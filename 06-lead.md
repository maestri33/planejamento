---
tags:
  - supletico
  - lead
aliases:
  - Lead
created: 2026-04-29
status: especificação
---

# 06 — Lead

Pré-matrícula. Estado entre "WhatsApp validado" e "aluno pagante". Status: 1 → 2 → 3 → 100.

> [!tip] Fluxo de status
> | Status | Significado | Ação esperada |
> | :---- | :---- | :---- |
> | 1 | Recém registrado | Lead preenche nome + email |
> | 2 | Dados básicos preenchidos | Lead preenche endereço (sem comprovante) |
> | 3 | Endereço preenchido — pronto para pagamento | Lead é redirecionado ao checkout do InfinitePay |
> | 100 | Pagou taxa de matrícula | Vira student → [[07-enrollment|enrollment.tools.new_student]] é chamado |

---

## Estrutura

```text
lead/
├── models/
│   ├── __init__.py
│   └── lead.py
├── api/
│   ├── __init__.py
│   ├── schemas.py
│   ├── public.py            # check, register, webhook InfinitePay
│   ├── authenticated.py     # status, update (role=lead)
│   └── by_role/
│       ├── promoter.py      # list dos leads do promoter
│       └── staff.py         # list todos
├── tools/
│   ├── __init__.py
│   ├── create_checkout.py
│   ├── get_url.py
│   ├── update_status.py
│   └── paid.py
├── services/
│   ├── __init__.py
│   └── lead_validators.py   # checa se status pode avançar
├── validators/
│   ├── __init__.py
│   └── promoter.py
├── normalizers/
│   ├── __init__.py
│   └── promoter.py          # se vazio, usa config.DEFAULT_PROMOTER
├── signals/
│   ├── __init__.py
│   └── post_save_lead.py
├── notify/
│   ├── new_lead.md
│   ├── complete_lead.md
│   ├── checkout.md
│   └── payment.md
└── apps.py
```

---

## 6.1 Models

### `lead/models/lead.py`

```python
from django.db import models
from core.models.base import BaseModel, ExternalIdMixin

class Lead(BaseModel, ExternalIdMixin):
    STATUS_BASIC = 1
    STATUS_ADDRESS = 2
    STATUS_AWAITING_PAYMENT = 3
    STATUS_PAID = 100

    STATUS_CHOICES = [
        (STATUS_BASIC, "Aguardando dados básicos"),
        (STATUS_ADDRESS, "Aguardando endereço"),
        (STATUS_AWAITING_PAYMENT, "Aguardando pagamento"),
        (STATUS_PAID, "Pago"),
    ]

    profile = models.OneToOneField("data.Profile", on_delete=models.CASCADE, related_name="lead")
    promoter = models.ForeignKey("data.Profile", on_delete=models.PROTECT, related_name="captured_leads")
    status = models.IntegerField(choices=STATUS_CHOICES, default=STATUS_BASIC)
    paid = models.BooleanField(default=False)
```

`promoter` é uma FK para [[05-data|Profile]] — o promoter captador. Quando vira student, o `enrollment.Student` herda esse vínculo para identificar o hub.

---

## 6.2 Schemas

```python
# lead/api/schemas.py
from ninja import Schema
from typing import Optional

class CheckIn(Schema):
    number: str

class CheckOut(Schema):
    external_id: Optional[str] = None
    role: Optional[str] = None
    first_name: Optional[str] = None
    whatsapp_valid: bool
    message: str
    redirect_to: Optional[str] = None  # se role bate, devolve frontend_url

class RegisterIn(Schema):
    number: str
    cpf: str
    promoter: Optional[str] = None  # external_id do promoter; se None, usa DEFAULT_PROMOTER

class StatusOut(Schema):
    status: int
    first_name: Optional[str] = None
    message: str
    instruction: Optional[str] = None
    checkout_url: Optional[str] = None
    redirect_url: Optional[str] = None

class UpdateOut(StatusOut):
    pass

class LeadListOut(Schema):
    external_id: str
    first_name: Optional[str]
    cpf: str
    status: int
    promoter_external_id: Optional[str] = None  # só staff vê
```

---

## 6.3 API

### `lead/api/public.py`

```python
from ninja import Router
from .schemas import CheckIn, CheckOut, RegisterIn, StatusOut
from auth.tools.check import check as auth_check
from auth.tools.otp import otp as auth_otp
from auth.tools.register import register as auth_register
from lead.tools.update_status import update_status
from lead.normalizers.promoter import resolve_promoter
from core.notify import service as notify_service
from data.models import Profile
from lead.models import Lead
import config

router = Router(tags=["lead-public"])

@router.get("/check/{number}", response=CheckOut)
def check(request, number: str):
    """
    Caso A: número não existe → whatsapp_valid=true → orienta a registrar
    Caso B: número não existe e wpp inválido → erro
    Caso C: existe e role=lead → dispara OTP automaticamente, devolve external_id
    Caso D: existe mas role!=lead → orienta redirecionar pra outro frontend
    """
    result = auth_check(number)
    if not result["external_id"]:
        if result["whatsapp_valid"]:
            return CheckOut(
                whatsapp_valid=True,
                message="Número válido. Pode prosseguir para registro.",
            )
        return CheckOut(
            whatsapp_valid=False,
            message="Número de WhatsApp inválido.",
        )

    # Existe — verificar role
    roles = result.get("roles", [])
    if "lead" in roles:
        # Caso C: dispara OTP e devolve external_id pra ele logar
        auth_otp(result["external_id"])
        return CheckOut(
            external_id=result["external_id"],
            first_name=result.get("first_name"),
            role="lead",
            whatsapp_valid=True,
            message="Te enviamos um código de acesso no WhatsApp.",
        )

    # Caso D: outra role — redireciona
    primary_role = roles[0]  # ou usar lógica de prioridade
    redirect_url = getattr(config, f"FRONTEND_URL_LOGIN_{primary_role.upper()}", None)
    return CheckOut(
        external_id=result["external_id"],
        role=primary_role,
        whatsapp_valid=True,
        message=f"Você já está cadastrado como {primary_role}.",
        redirect_to=redirect_url,
    )

@router.post("/register", response=StatusOut)
def register(request, payload: RegisterIn):
    promoter_external_id = resolve_promoter(payload.promoter)
    promoter_profile = Profile.objects.get(external_id=promoter_external_id)

    # Registra no auth (cria User, Profile, role=lead, recipient no notify)
    result = auth_register(numero=payload.number, cpf=payload.cpf, role="lead")
    profile = result["profile"]

    # Cria o Lead
    lead = Lead.objects.create(profile=profile, promoter=promoter_profile, status=Lead.STATUS_BASIC)

    # Notify para o lead
    notify_service.send(
        external_id=str(profile.external_id),
        template_path="lead/notify/new_lead.md",
        context={"first_name": "amigo(a)"},
    )

    # Notify para o promoter
    notify_service.send(
        external_id=str(promoter_profile.external_id),
        template_path="promoter/notify/new_lead_to_promotor.md",
        context={
            "lead_external_id": str(profile.external_id),
            "lead_cpf": profile.cpf,
        },
    )

    # Dispara OTP automaticamente
    auth_otp(str(profile.external_id))

    return StatusOut(
        status=Lead.STATUS_BASIC,
        message="Registro criado. Te enviamos um código de acesso no WhatsApp.",
        instruction="Faça login com external_id + OTP, depois preencha nome e e-mail.",
    )

@router.post("/webhook/infinitepay", response=dict)
def webhook_infinitepay(request, payload: dict):
    """
    Webhook chamado pelo InfinitePay quando pagamento é confirmado.
    Payload esperado (a confirmar): {"external_id": "...", "is_paid": true}
    """
    from lead.tools.paid import mark_lead_paid

    if payload.get("is_paid"):
        mark_lead_paid(payload["external_id"])
    return {"received": True}
```

### `lead/api/authenticated.py`

```python
from ninja import Router
from core.auth_helpers.role_required import require_role
from .schemas import StatusOut, UpdateOut
from lead.tools.update_status import update_status
from lead.tools.get_url import get_url
from lead.services.lead_validators import can_advance_from
import config

router = Router(tags=["lead"])

@router.get("/status", response=StatusOut, auth=JWTAuth())
def get_status(request, _: None = Depends(require_role("lead"))):
    profile = request.user.profile
    lead = profile.lead
    return _build_status_response(lead)

@router.post("/update", response=UpdateOut, auth=JWTAuth())
def post_update(request, _: None = Depends(require_role("lead"))):
    profile = request.user.profile
    lead = profile.lead

    # Verifica se pode avançar conforme status atual
    if can_advance_from(lead):
        update_status(lead)

    return _build_status_response(lead)

def _build_status_response(lead) -> StatusOut:
    profile = lead.profile

    if lead.status == 1:
        return StatusOut(
            status=1,
            message="Precisamos dos seus dados básicos: nome e e-mail.",
            instruction="PATCH /v1/data/profile (com 'name') e PATCH /v1/data/email",
        )

    if lead.status == 2:
        return StatusOut(
            status=2,
            first_name=profile.user.first_name,
            message=f"{profile.user.first_name}, agora preenche seu endereço.",
            instruction="PATCH /v1/data/address (somente dados, sem comprovante)",
        )

    if lead.status == 3:
        url_data = get_url(lead)
        return StatusOut(
            status=3,
            first_name=profile.user.first_name,
            message="Você será redirecionado para pagamento da taxa de matrícula.",
            instruction="Redirecione o usuário para a checkout_url",
            checkout_url=url_data["checkout_url"],
        )

    if lead.status == 100:
        return StatusOut(
            status=100,
            first_name=profile.user.first_name,
            message="Parabéns, a taxa de matrícula foi paga!",
            instruction="Redirecione para a área do aluno",
            redirect_url=config.FRONTEND_URL_LOGIN_STUDENT,
        )

    return StatusOut(status=lead.status, message="Status desconhecido.")
```

### `lead/api/by_role/promoter.py`

```python
from ninja import Router
from core.auth_helpers.role_required import require_role

router = Router(tags=["lead-promoter"])

@router.get("/", response=list[LeadListOut], auth=JWTAuth())
def list_my_leads(request, _: None = Depends(require_role("promoter"))):
    profile = request.user.profile
    leads = Lead.objects.filter(promoter=profile)
    if not leads.exists():
        return []
    return [_serialize(l) for l in leads]

@router.get("/{status}", response=list[LeadListOut], auth=JWTAuth())
def list_my_leads_by_status(request, status: int, _: None = Depends(require_role("promoter"))):
    profile = request.user.profile
    return [_serialize(l) for l in Lead.objects.filter(promoter=profile, status=status)]

@router.get("/{external_id}", response=LeadListOut, auth=JWTAuth())
def get_lead(request, external_id: str, _: None = Depends(require_role("promoter"))):
    profile = request.user.profile
    return _serialize(Lead.objects.get(promoter=profile, profile__external_id=external_id))
```

### `lead/api/by_role/staff.py`

Replica lógica do promoter, mas sem filtro de promoter e incluindo o campo `promoter_external_id`.

---

## 6.4 Tools

### `lead/tools/create_checkout.py`

```python
from core.infinitepay.client import InfinitePayClient

_client = InfinitePayClient()

def create_checkout(lead) -> dict:
    """Chamado quando lead avança para status 3."""
    profile = lead.profile
    address = profile.address
    return _client.create(
        external_id=str(profile.external_id),
        name=profile.name,
        number=profile.user.username,  # ou campo separado
        email=profile.user.email,
        address={
            "rua": address.rua,
            "numero": address.numero,
            "cep": address.cep,
            "cidade": address.cidade,
            "estado": address.estado,
            "bairro": address.bairro,
        },
    )
```

### `lead/tools/get_url.py`

```python
from core.infinitepay.client import InfinitePayClient

_client = InfinitePayClient()

def get_url(lead) -> dict:
    """Retorna {checkout_url, receipt_url} via core.infinitepay.client.check"""
    return _client.check(str(lead.profile.external_id))
```

### `lead/tools/update_status.py`

Implementa a transição entre estados.

```python
from django.db import transaction
from core.notify import service as notify_service
from lead.models import Lead

@transaction.atomic
def update_status(lead: Lead):
    if lead.status == Lead.STATUS_BASIC:
        # 1 → 2
        lead.status = Lead.STATUS_ADDRESS
        lead.save(update_fields=["status"])

    elif lead.status == Lead.STATUS_ADDRESS:
        # 2 → 3
        lead.status = Lead.STATUS_AWAITING_PAYMENT
        lead.save(update_fields=["status"])

        # Cria checkout no InfinitePay
        from lead.tools.create_checkout import create_checkout
        create_checkout(lead)

        # Notify
        notify_service.send(
            external_id=str(lead.profile.external_id),
            template_path="lead/notify/complete_lead.md",
            context={"first_name": lead.profile.user.first_name},
        )

    elif lead.status == Lead.STATUS_AWAITING_PAYMENT:
        # 3 → 100 (acionado pelo signal post_save quando paid=True)
        lead.status = Lead.STATUS_PAID
        lead.save(update_fields=["status"])

        # Cria student
        from enrollment.tools.new_student import new_student
        new_student(lead)

        # Comissão para promoter
        from finance.tools.new_commission_promotor import new_commission_promotor
        new_commission_promotor(promoter_profile=lead.promoter, lead=lead)

        # Notify do pagamento confirmado
        from lead.tools.get_url import get_url
        url_data = get_url(lead)
        notify_service.send(
            external_id=str(lead.profile.external_id),
            template_path="lead/notify/payment.md",
            context={
                "first_name": lead.profile.user.first_name,
                "receipt_url": url_data.get("receipt_url"),
            },
        )
```

### `lead/tools/paid.py`

```python
from lead.models import Lead

def mark_lead_paid(external_id: str):
    """
    Chamado pelo webhook do InfinitePay.
    Atualiza paid=True. O signal post_save dispara update_status.
    """
    lead = Lead.objects.get(profile__external_id=external_id)
    lead.paid = True
    lead.save(update_fields=["paid"])
```

---

## 6.5 Services

### `lead/services/lead_validators.py`

```python
def can_advance_from(lead) -> bool:
    """
    Verifica se os pré-requisitos do status atual estão atendidos
    para permitir avanço.
    """
    profile = lead.profile

    if lead.status == 1:
        return bool(profile.name and profile.user.email)

    if lead.status == 2:
        addr = profile.address
        if not addr:
            return False
        required = [addr.rua, addr.numero, addr.cep, addr.cidade, addr.estado, addr.bairro]
        return all(required)

    return False  # status 3 só avança via webhook (paid=True)
```

---

## 6.6 Normalizers

### `lead/normalizers/promoter.py`

```python
import config

def resolve_promoter(promoter_external_id: str | None) -> str:
    """Se vazio, retorna DEFAULT_PROMOTER."""
    return promoter_external_id or config.DEFAULT_PROMOTER
```

`DEFAULT_PROMOTER` é definido em [[01-config-e-env|config]].

---

## 6.7 Signals

### `lead/signals/post_save_lead.py`

```python
from django.db.models.signals import post_save
from django.dispatch import receiver
from lead.models import Lead
from lead.tools.update_status import update_status

@receiver(post_save, sender=Lead)
def on_lead_save(sender, instance, **kwargs):
    if instance.paid and instance.status == Lead.STATUS_AWAITING_PAYMENT:
        update_status(instance)
```

---

## 6.8 Templates de notify

### `lead/notify/new_lead.md`

> Olá! Parabéns por dar esse primeiro passo. Buscar concluir os estudos é uma decisão importante, e só o fato de você estar aqui já mostra que existe vontade de mudar, crescer e seguir em frente. Talvez esse sonho tenha ficado parado por falta de tempo, dificuldades da vida, trabalho, família ou falta de oportunidade. Mas agora você pode começar uma nova etapa. Continue seu cadastro e avance para o próximo passo. Cada informação preenchida aproxima você de encerrar esse ciclo e abrir novas portas para o seu futuro. Não pare agora. O primeiro passo já foi dado.

### `lead/notify/complete_lead.md`

```text
--tts
{{ first_name }}, parabéns! Você está muito perto de dar um dos passos mais importantes da sua vida. Talvez por dificuldades, falta de tempo, trabalho, família, problemas pessoais ou falta de oportunidade, você não tenha conseguido concluir essa etapa antes. Mas isso não define o seu futuro. O que importa é que agora você decidiu mudar essa história. {{ first_name }}, falta muito pouco para você encerrar esse ciclo e seguir em frente com mais confiança, mais oportunidades e mais orgulho da sua própria caminhada. Para continuar, clique no link que enviamos e faça o pagamento da taxa de matrícula. Esse é o próximo passo para manter seu processo ativo. Não pare agora, {{ first_name }}. Você já chegou até aqui. Agora avance.
```

### `lead/notify/checkout.md`

```text
{{ checkout_url }}
```

### `lead/notify/payment.md`

```text
Olá, {{ first_name }}! Parabéns, seu pagamento foi confirmado com sucesso. Esse é um passo muito importante para dar continuidade ao seu processo. Agora você está mais perto de concluir essa etapa e seguir em frente com mais segurança e tranquilidade. {{ first_name }}, se quiser conferir o comprovante do seu pagamento, é só acessar o link: {{ receipt_url }} Guarde esse comprovante para sua segurança. Seguimos com você nessa caminhada.
```

---

## 6.9 Variáveis de config usadas

| Variável | Uso |
| :---- | :---- |
| `DEFAULT_PROMOTER` | [[01-config-e-env|normalizers.promoter]] |
| `FRONTEND_URL_LOGIN_LEAD` | check (caso D — outras roles) |
| `FRONTEND_URL_LOGIN_STUDENT` | status response do status 100 |
| `FRONTEND_URL_LOGIN_<ROLE>` | check (cada role tem URL de redirect) |
| `LEAD_CHECKOUT_PRICE` | indireto (via `core.infinitepay.bootstrap`) |
| `LEAD_CHECKOUT_DESCRIPTION` | indireto |

---

## 6.10 Pontos abertos

> [!warning] Pontos abertos relacionados
> - ⚠️ Quando exatamente o checkout é criado? **Decisão tomada:** ao virar status 3 (em `update_status` na transição 2→3).
> - ⚠️ Validação de assinatura do webhook InfinitePay — adicionar quando confirmar nos [[02-integracoes-externas|docs do InfinitePay]].
> - ⚠️ Lead pode editar nome após avançar status 1? **Decisão padrão:** sim (até o status 100). Após virar student, regras do enrollment se aplicam.

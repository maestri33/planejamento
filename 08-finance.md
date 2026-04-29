---
tags:
  - supletico
  - finance
aliases:
  - Finance
created: 2026-04-29
status: especificação
---

# 08 — Finance

Pagamentos e comissões. Saída de dinheiro para promoters e coordenadores de hub. Toda integração de saída usa `core.asaas`.

## Conceitos

- **Commission** — direito a receber valor (gerado por evento de negócio: captação, conclusão, bonus)
- **Payment** — agrupamento de commissions de um mesmo `profile`, fechado semanalmente, enviado ao Asaas
- **Fluxo geral semanal:**
  1. Eventos do negócio criam Commissions (ao longo da semana)
  2. Sexta 18h: task de bonus checa metas e gera Commissions tipo `bonus`
  3. Em seguida: task de cash register closing soma commissions abertas por profile e cria 1 Payment por profile
  4. Signal post_save Payment → chama Asaas (`process_payment`)
  5. Webhook do Asaas confirma → atualiza `Payment.paid=True` → notifica e marca commissions como pagas

> [!warning] Payment vs OutgoingPayment
> `Payment` para QR Code de fee de matrícula externa: o profile dono é o coordenador do hub (que paga) ou nenhum (pagamento institucional)? A decisão atual é profile = coordenador do hub. Mas isso mistura "comissão a pagar" com "boleto a pagar". Talvez precise de model separado (`OutgoingPayment` vs `Payment`).

> [!warning] Webhook sem assinatura
> O webhook do Asaas não tem validação de assinatura — adicionar quando confirmar nos docs do Asaas.

---

## Estrutura

```text
finance/
├── models/
│   ├── __init__.py
│   ├── payment.py
│   └── commission.py
├── api/
│   ├── __init__.py
│   ├── schemas.py
│   ├── public.py            # webhook Asaas
│   ├── authenticated.py     # GET commissions/payments próprios (promoter, hub_coordinator)
│   └── by_role/
│       └── staff.py         # GET tudo + endpoints manuais (schedule_qrcode, etc)
├── tools/
│   ├── __init__.py
│   ├── new_commission_promotor.py
│   ├── new_commission_hub.py
│   ├── schedule_qrcode.py
│   ├── pay_qrcode.py
│   ├── pay.py
│   ├── update_payment.py
│   ├── process_bonus.py
│   ├── cash_register_closing.py
│   └── process_payment.py
├── services/
│   ├── __init__.py
│   └── asaas_pix_setup.py
├── tasks/
│   ├── __init__.py
│   └── friday_closing.py    # 2 tasks encadeadas: bonus → closing
├── signals/
│   ├── __init__.py
│   └── post_save_payment.py
├── notify/
│   ├── schedule.md
│   ├── promoter_commission_created.md
│   ├── hub_coordinator_commission_created.md
│   ├── payment_paid.md
│   └── congratulate_promoter_for_reaching_target.md
└── apps.py
```

---

## 8.1 Models

### `finance/models/payment.py`

```python
import uuid
from django.db import models
from core.models.base import BaseModel, ExternalIdMixin

class Payment(BaseModel, ExternalIdMixin):
    """
    Saída de dinheiro para um profile.
    Agrupa N commissions e é enviado ao Asaas.
    """
    payment_id = models.UUIDField(default=uuid.uuid4, unique=True, editable=False)
    profile = models.ForeignKey("data.Profile", on_delete=models.PROTECT, related_name="payments")
    status = models.CharField(max_length=30, default="pending")  # pending | sent | failed | scheduled
    processed = models.BooleanField(default=False)               # já enviou para Asaas
    paid = models.BooleanField(default=False)                    # confirmação de pagamento (via webhook)
    valor = models.DecimalField(max_digits=10, decimal_places=2)
    asaas_response = models.JSONField(null=True, blank=True)     # resposta crua do Asaas (auditoria)
```

### `finance/models/commission.py`

```python
class Commission(BaseModel, ExternalIdMixin):
    TIPO_CAPTACAO = "captacao"
    TIPO_CONCLUSION = "conclusion"
    TIPO_BONUS = "bonus"
    TIPO_CHOICES = [
        (TIPO_CAPTACAO, "Captação"),
        (TIPO_CONCLUSION, "Conclusão"),
        (TIPO_BONUS, "Bonus"),
    ]
    profile = models.ForeignKey("data.Profile", on_delete=models.PROTECT, related_name="commissions")
    tipo = models.CharField(max_length=15, choices=TIPO_CHOICES)
    processed = models.BooleanField(default=False)               # já foi incluída em algum Payment
    paid = models.BooleanField(default=False)
    valor = models.DecimalField(max_digits=10, decimal_places=2)
    payment = models.ForeignKey("finance.Payment", null=True, blank=True, on_delete=models.SET_NULL, related_name="commissions")
    # Referências para auditoria/exibição (qual lead/student gerou esta commission)
    lead = models.ForeignKey("lead.Lead", null=True, blank=True, on_delete=models.SET_NULL, related_name="commissions")
    student = models.ForeignKey("enrollment.Student", null=True, blank=True, on_delete=models.SET_NULL, related_name="commissions")
```

---

## 8.2 Tools

### `finance/tools/new_commission_promotor.py`

Chamado por `lead.tools.update_status` quando lead vira pago (3 → 100).

```python
import config
from decimal import Decimal
from core.notify import service as notify_service
from finance.models import Commission

def new_commission_promotor(promoter_profile, lead) -> Commission:
    """
    Cria Commission tipo 'captacao' para o promoter.
    Notifica com previsão de recebimento (próxima sexta 18h).
    """
    commission = Commission.objects.create(
        profile=promoter_profile,
        tipo=Commission.TIPO_CAPTACAO,
        valor=Decimal(str(config.PROMOTER_COMMISSION_VALUE)),
        lead=lead,
    )
    notify_service.send(
        external_id=str(promoter_profile.external_id),
        template_path="finance/notify/promoter_commission_created.md",
        context={
            "first_name": promoter_profile.user.first_name,
            "valor": str(commission.valor),
            "lead_name": lead.profile.name,
            "lead_cpf": lead.profile.cpf,
            "expected_date": _next_friday_18h(),
        },
    )
    return commission

def _next_friday_18h() -> str:
    from datetime import datetime, timedelta
    today = datetime.now()
    days_until_friday = (4 - today.weekday()) % 7
    if days_until_friday == 0 and today.hour >= 18:
        days_until_friday = 7
    friday = today + timedelta(days=days_until_friday)
    return friday.strftime("%d/%m/%Y às 18:00")
```

> [!info] Disparado por [[06-lead]]
> `new_commission_promotor` é chamado pelo `update_status` do lead quando o status atinge 100 (pago). O promoter recebe comissão de captação.

### `finance/tools/new_commission_hub.py`

Chamado por `enrollment/signals/post_save_student.py` quando student → 100.

```python
import config
from decimal import Decimal
from core.notify import service as notify_service
from finance.models import Commission

def new_commission_hub(coordinator_profile, student) -> Commission:
    commission = Commission.objects.create(
        profile=coordinator_profile,
        tipo=Commission.TIPO_CONCLUSION,
        valor=Decimal(str(config.HUB_COORDINATOR_COMMISSION_VALUE)),
        student=student,
    )
    notify_service.send(
        external_id=str(coordinator_profile.external_id),
        template_path="finance/notify/hub_coordinator_commission_created.md",
        context={
            "first_name": coordinator_profile.user.first_name,
            "valor": str(commission.valor),
            "student_name": student.profile.name,
            "expected_date": _next_friday_18h(),
        },
    )
    return commission
```

> [!info] Disparado por [[07-enrollment]]
> `new_commission_hub` é chamado pelo signal `post_save_student` quando o status do student atinge 100 (concluído). O hub coordinator recebe comissão de conclusão.

### `finance/tools/process_bonus.py`

Roda sexta 18h, antes do closing.

```python
import config
from decimal import Decimal
from django.db import transaction
from core.notify import service as notify_service
from finance.models import Commission
from data.models import Profile

@transaction.atomic
def process_bonus():
    """
    Para cada profile com >=WEEKLY_TARGET_QUANTITY commissions tipo captacao
    com processed=False, cria commission tipo bonus de WEEKLY_TARGET_BONUS_VALUE.
    """
    target_qty = int(config.WEEKLY_TARGET_QUANTITY)
    bonus_value = Decimal(str(config.WEEKLY_TARGET_BONUS_VALUE))

    profiles_with_open_captacao = Profile.objects.filter(
        commissions__tipo=Commission.TIPO_CAPTACAO,
        commissions__processed=False,
    ).distinct()

    for profile in profiles_with_open_captacao:
        qty = profile.commissions.filter(
            tipo=Commission.TIPO_CAPTACAO, processed=False
        ).count()
        if qty < target_qty:
            continue
        Commission.objects.create(
            profile=profile,
            tipo=Commission.TIPO_BONUS,
            valor=bonus_value,
        )
        notify_service.send(
            external_id=str(profile.external_id),
            template_path="finance/notify/congratulate_promoter_for_reaching_target.md",
            context={
                "first_name": profile.user.first_name,
                "qty": qty,
                "target": target_qty,
                "bonus_value": str(bonus_value),
            },
        )
```

> [!info] Metas de bonus via [[01-config-e-env]]
> `WEEKLY_TARGET_QUANTITY` e `WEEKLY_TARGET_BONUS_VALUE` são configuráveis em runtime via `config.py`.

### `finance/tools/cash_register_closing.py`

Roda imediatamente depois do bonus.

```python
from decimal import Decimal
from django.db import transaction
from finance.models import Commission, Payment
from data.models import Profile

@transaction.atomic
def cash_register_closing():
    """
    Para cada profile com commissions abertas (processed=False),
    soma valores e cria 1 Payment com o total.
    Marca todas as commissions desse profile como processed=True
    e relaciona-as ao payment.
    """
    profiles = Profile.objects.filter(commissions__processed=False).distinct()
    for profile in profiles:
        open_commissions = list(profile.commissions.filter(processed=False))
        if not open_commissions:
            continue
        total = sum((c.valor for c in open_commissions), Decimal("0"))
        payment = Payment.objects.create(
            profile=profile,
            valor=total,
            status="pending",
        )
        # Vincula commissions ao payment
        Commission.objects.filter(id__in=[c.id for c in open_commissions]).update(
            processed=True,
            payment=payment,
        )
        # post_save Payment dispara process_payment (ver signals)
```

### `finance/tools/process_payment.py`

Chamado pelo signal post_save Payment (criação).

```python
from core.asaas.client import AsaasClient
from finance.models import Payment

_client = AsaasClient()

def process_payment(payment_id: str) -> Payment:
    """
    Envia Payment ao Asaas. Atualiza processed=True e status='sent'.
    Não atualiza paid — isso vem via webhook.
    """
    payment = Payment.objects.get(payment_id=payment_id)
    profile = payment.profile
    # Descrição contendo as commissions
    description = _build_description(payment)

    response = _client.pay(
        amount=float(payment.valor),
        description=description,
        payment_id=str(payment.payment_id),
        pixkey_external_id=str(profile.external_id),
    )
    payment.processed = True
    payment.status = "sent"
    payment.asaas_response = response
    payment.save(update_fields=["processed", "status", "asaas_response"])
    return payment

def _build_description(payment: Payment) -> str:
    parts = []
    for c in payment.commissions.all():
        parts.append(f"{c.tipo} R${c.valor}")
    return " | ".join(parts)
```

> [!info] Integração com [[02-integracoes-externas|Asaas]]
> O client Asaas (`core.asaas.client`) encapsula as chamadas à API externa. Nenhum app fala direto com a URL do Asaas.

### `finance/tools/update_payment.py`

Chamado pelo webhook do Asaas.

```python
from django.db import transaction
from core.notify import service as notify_service
from finance.models import Payment, Commission

@transaction.atomic
def update_payment(payment_id: str, paid: bool):
    """
    Atualiza Payment.paid. Se True, propaga para commissions e dispara notify.
    """
    payment = Payment.objects.get(payment_id=payment_id)
    payment.paid = paid
    payment.save(update_fields=["paid"])
    if not paid:
        return
    # Propaga para commissions
    Commission.objects.filter(payment=payment).update(paid=True)
    # Notifica profile
    profile = payment.profile
    commissions_text = ", ".join([f"{c.tipo} R${c.valor}" for c in payment.commissions.all()])
    notify_service.send(
        external_id=str(profile.external_id),
        template_path="finance/notify/payment_paid.md",
        context={
            "first_name": profile.user.first_name,
            "valor": str(payment.valor),
            "commissions": commissions_text,
        },
    )
```

### `finance/tools/schedule_qrcode.py`

Usado pelo hub para agendar pagamento de fee de matrícula externa (segunda parcela).

```python
from core.asaas.client import AsaasClient
from core.notify import service as notify_service
from finance.models import Payment

_client = AsaasClient()

def schedule_qrcode(qrcode: str, context: dict | None = None) -> Payment:
    """
    1. Analisa qrcode (valida)
    2. Cria Payment local com valor + due_date extraídos
    3. Agenda no Asaas via pay_qrcode_scheduled
    4. Notifica
    """
    analysis = _client.analyze_qrcode(qrcode=qrcode)
    if not analysis.get("valid", True):
        raise ValueError("QR Code inválido.")

    # Cria Payment "manual" — não vinculado a profile do promoter, mas pago em nome do hub
    # ⚠️ ver pendência: "Payment para QR de hub não tem profile" — ajustar model?
    # Solução temporária: profile = coordenador do hub que disparou
    coordinator_profile = context.get("coordinator_profile") if context else None
    payment = Payment.objects.create(
        profile=coordinator_profile,
        valor=analysis["amount"],
        status="scheduled",
    )
    response = _client.pay_qrcode_scheduled(
        qrcode=qrcode,
        payment_id=str(payment.payment_id),
        scheduled_date=analysis["due_date"],
    )
    payment.asaas_response = response
    payment.save()

    notify_service.send(
        external_id=str(coordinator_profile.external_id) if coordinator_profile else None,
        template_path="finance/notify/schedule.md",
        context={
            "valor": str(analysis["amount"]),
            "due_date": analysis["due_date"],
        },
    )
    return payment
```

### `finance/tools/pay_qrcode.py`

Mesma ideia, sem agendamento (pagamento imediato).

```python
def pay_qrcode(qrcode: str, context: dict | None = None) -> Payment:
    """Análise + pagamento imediato. Mesma lógica do schedule, sem scheduled_date."""
    ...
```

### `finance/tools/pay.py`

Wrapper direto sobre `core.asaas.client.pay` quando há um Payment já criado.

```python
def pay(payment: Payment) -> Payment:
    """Atalho para enviar um Payment existente ao Asaas (mesmo que process_payment)."""
    return process_payment(str(payment.payment_id))
```

---

## 8.3 Services

### `finance/services/asaas_pix_setup.py`

```python
from core.asaas import service as asaas_service

def setup_pix_key_for_profile(profile) -> dict:
    """
    Garante que profile tem chave PIX (CPF) cadastrada no Asaas.
    Idempotente: se já existir, retorna o registro existente sem cadastrar de novo.
    Chamado quando candidate avança para status 20 (cadastro PIX automático),
    e também pode ser chamado manualmente por staff em caso de retentativa.
    """
    return asaas_service.ensure_pix_key(
        document=profile.cpf,
        external_id=str(profile.external_id),
        key=profile.cpf,
        key_type="CPF",
    )
```

> [!info] Usado por [[11-candidate]]
> `setup_pix_key_for_profile` é chamado quando o candidate atinge status 20 (cadastro PIX automático). Também pode ser chamado manualmente por staff para retentativa.

---

## 8.4 Tasks (Celery)

### `finance/tasks/friday_closing.py`

```python
from celery import shared_task
from celery.schedules import crontab
from finance.tools.process_bonus import process_bonus
from finance.tools.cash_register_closing import cash_register_closing

@shared_task
def run_friday_closing():
    """
    Sexta 18h: roda bonus primeiro, depois closing.
    Ordem importa: bonus cria commissions tipo bonus que precisam estar
    abertas quando o closing fizer a soma.
    """
    process_bonus()
    cash_register_closing()

# Configuração no celery.py do projeto:
# CELERY_BEAT_SCHEDULE = {
#     "friday-closing-18h": {
#         "task": "finance.tasks.friday_closing.run_friday_closing",
#         "schedule": crontab(day_of_week="friday", hour=18, minute=0),
#     },
# }
```

---

## 8.5 Signals

### `finance/signals/post_save_payment.py`

```python
from django.db.models.signals import post_save
from django.dispatch import receiver
from finance.models import Payment
from finance.tools.process_payment import process_payment

@receiver(post_save, sender=Payment)
def on_payment_save(sender, instance, created, **kwargs):
    if created and instance.status == "pending":
        # Dispara envio para Asaas
        process_payment(str(instance.payment_id))
```

O signal NÃO trata `paid` aqui — isso é feito explicitamente pelo `update_payment` chamado do webhook (mais rastreável).

---

## 8.6 API

### Schemas

```python
# finance/api/schemas.py
from ninja import Schema
from typing import Optional

class CommissionOut(Schema):
    external_id: str
    tipo: str
    valor: str
    processed: bool
    paid: bool
    created_at: str
    expected_date: Optional[str] = None  # próxima sexta se ainda não processada

class PaymentOut(Schema):
    payment_id: str
    valor: str
    status: str
    processed: bool
    paid: bool
    created_at: str
    commissions: list[CommissionOut]

class WebhookAsaasIn(Schema):
    payment_id: str
    paid: bool

class ScheduleQrcodeIn(Schema):
    qrcode: str
```

### `finance/api/public.py`

```python
from ninja import Router
from .schemas import WebhookAsaasIn

router = Router(tags=["finance-public"])

@router.post("/webhook/asaas", response=dict)
def webhook_asaas(request, payload: WebhookAsaasIn):
    """Webhook do Asaas confirmando pagamento."""
    from finance.tools.update_payment import update_payment
    update_payment(payload.payment_id, payload.paid)
    return {"received": True}
```

> [!warning] Endpoint público sem assinatura
> O webhook do Asaas é público (sem JWT) pois é chamado por um serviço externo. A validação de assinatura ainda precisa ser implementada.

### `finance/api/authenticated.py`

```python
from ninja import Router
from core.auth_helpers.role_required import require_role

router = Router(tags=["finance"])

@router.get("/commissions", response=list[CommissionOut], auth=JWTAuth())
def list_my_commissions(request, _: None = Depends(require_role("promoter", "hub_coordinator"))):
    profile = request.user.profile
    return [_serialize_commission(c) for c in profile.commissions.all()]

@router.get("/payments", response=list[PaymentOut], auth=JWTAuth())
def list_my_payments(request, _: None = Depends(require_role("promoter", "hub_coordinator"))):
    profile = request.user.profile
    return [_serialize_payment(p) for p in profile.payments.all()]
```

### `finance/api/by_role/staff.py`

```python
@router.get("/commissions", response=list[CommissionOut], auth=JWTAuth())
def list_all_commissions(request, profile_external_id: str = None, _: None = Depends(require_role("staff"))):
    qs = Commission.objects.all()
    if profile_external_id:
        qs = qs.filter(profile__external_id=profile_external_id)
    return [_serialize_commission(c) for c in qs]

@router.get("/payments", response=list[PaymentOut], auth=JWTAuth())
def list_all_payments(request, profile_external_id: str = None, _: None = Depends(require_role("staff"))):
    ...

@router.post("/schedule-qrcode", auth=JWTAuth())
def manual_schedule_qrcode(request, payload: ScheduleQrcodeIn, _: None = Depends(require_role("staff"))):
    """Staff agenda QR code manualmente (caso de uso fora dos fluxos automáticos)."""
    from finance.tools.schedule_qrcode import schedule_qrcode
    return schedule_qrcode(payload.qrcode, context={"coordinator_profile": request.user.profile})
```

---

## 8.7 Notifications

### `finance/notify/promoter_commission_created.md`

```
{{ first_name }}, você acaba de gerar uma comissão de R$ {{ valor }} pela captação de {{ lead_name }} (CPF: {{ lead_cpf }}). A previsão de pagamento é {{ expected_date }}.
```

### `finance/notify/hub_coordinator_commission_created.md`

```
{{ first_name }}, parabéns! Mais um aluno concluído: {{ student_name }}. Sua comissão de R$ {{ valor }} foi gerada e tem previsão de pagamento em {{ expected_date }}.
```

### `finance/notify/payment_paid.md`

```
Parabéns {{ first_name }}, seu pagamento no valor de R$ {{ valor }} referente a suas comissões ({{ commissions }}) está na conta. Obrigado por fazer parte da família Supletivo.net.
```

### `finance/notify/schedule.md`

```
Pagamento agendado com sucesso. Valor: R$ {{ valor }}. Vencimento: {{ due_date }}.
```

### `finance/notify/congratulate_promoter_for_reaching_target.md`

```
--tts
{{ first_name }}, parabéns! Você bateu a meta semanal: {{ qty }} captações (meta: {{ target }}). Um bonus de R$ {{ bonus_value }} foi adicionado às suas comissões dessa semana. Continue assim!
```

---

## 8.8 Variáveis de config usadas

| Variável | Uso |
| :--- | :--- |
| `PROMOTER_COMMISSION_VALUE` | new_commission_promotor |
| `HUB_COORDINATOR_COMMISSION_VALUE` | new_commission_hub |
| `WEEKLY_TARGET_QUANTITY` | process_bonus |
| `WEEKLY_TARGET_BONUS_VALUE` | process_bonus |
| `ASAAS_API_URL` | indireto via core.asaas.client |
| `ASAAS_API_TOKEN` ⚠️ | indireto |

> [!info] Configuração centralizada em [[01-config-e-env]]
> Todas essas variáveis são acessadas via `config.X`. `PROMOTER_COMMISSION_VALUE`, `HUB_COORDINATOR_COMMISSION_VALUE`, `WEEKLY_TARGET_QUANTITY` e `WEEKLY_TARGET_BONUS_VALUE` são editáveis em runtime via staff.

---

## 8.9 Pontos abertos

- ⚠️ **Payment vs OutgoingPayment:** `Payment` para QR Code de fee de matrícula externa: o profile dono é o coordenador do hub (que paga) ou nenhum (pagamento institucional)?
  - **Decisão padrão atual:** profile = coordenador do hub. Mas isso mistura "comissão a pagar" com "boleto a pagar". Talvez precise de model separado (`OutgoingPayment` vs `Payment`).
- ⚠️ **Ordem do friday closing:** bonus → closing está OK. Mas se algum erro acontece no bonus, o closing roda mesmo assim?
  - **Decisão padrão:** sim (closing é defensivo, só processa o que tem). Mas log e alerta.
- ⚠️ **Webhook do Asaas** não tem validação de assinatura — adicionar quando confirmar nos docs do Asaas.
- ⚠️ Quando uma `Commission.paid=True`, deve haver um histórico/log? Por enquanto o `created_at`/`updated_at` do BaseModel cobre.
- ⚠️ **Comissão de conclusion para promoter?** **Decisão atual:** apenas hub coordinator ganha na conclusão. Promoter ganha só na captação. Reverter se quiser.

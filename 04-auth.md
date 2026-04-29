---
tags:
  - supletico
  - auth
aliases:
  - Auth
created: 2026-04-29
---

# 04 — Auth

Usuário, login, OTP, JWT, roles. Lógica **polimórfica** — não tem API exposta direta; as funções são consumidas por outros apps (`lead`, `candidate`, etc.).

## Princípio do fluxo

> [!info] Fluxo de entrada
> *"usuário entra no frontend, põe número de celular. Se encontrar → segue pra login (retorna external_id e recebe OTP como notificação). Se não acha mas WhatsApp é válido → segue pra registro."*

A ideia é facilitar ao máximo o acesso. O telefone é a chave única.

---

## Estrutura

```
auth/
├── models/
│   ├── __init__.py
│   ├── role.py              # Role + ProfileRole (M2M)
│   └── otp_code.py          # OTPCode (geração e expiração)
├── api/
│   ├── __init__.py          # router principal
│   ├── schemas.py           # LoginIn, LoginOut, RefreshIn, etc.
│   ├── public.py            # POST /login (external_id + otp → JWT)
│   └── authenticated.py     # POST /refresh
├── tools/                   # PÚBLICO — outros apps importam daqui
│   ├── __init__.py
│   ├── check.py             # auth.tools.check(numero)
│   ├── otp.py               # auth.tools.otp(external_id) → gera + envia OTP
│   ├── register.py          # auth.tools.register(numero, cpf, role)
│   ├── add_role.py          # auth.tools.add_role(profile, role)
│   └── change_role.py       # auth.tools.change_role(profile, from_role, to_role)
├── services/                # PRIVADO — só auth mexe
│   ├── __init__.py
│   ├── jwt.py               # gera JWT com {external_id, roles}
│   └── otp_storage.py       # cria/valida/expira OTPCode
├── validators/
│   ├── __init__.py
│   └── cpf.py               # wraps core.validators.cpf
├── notify/
│   ├── otp.md
│   ├── magic_link_lead.md
│   ├── magic_link_student.md
│   ├── magic_link_promoter.md
│   ├── magic_link_candidate.md
│   ├── magic_link_hub.md
│   └── magic_link_staff.md
└── apps.py
```

> [!note] tools vs services
> Seguindo a convenção do [[00-arquitetura-geral#convencao-tools-vs-services|tools vs services]]: `tools/` é a interface pública, `services/` é privado e só o próprio app auth toca.

---

## 4.1 Models

### `auth/models/role.py`

```python
from django.db import models
from core.models.base import BaseModel

class Role(BaseModel):
    LEAD = "lead"
    STUDENT = "student"
    CANDIDATE = "candidate"
    PROMOTER = "promoter"
    HUB_COORDINATOR = "hub_coordinator"
    STAFF = "staff"

    CHOICES = [
        (LEAD, "Lead"),
        (STUDENT, "Student"),
        (CANDIDATE, "Candidate"),
        (PROMOTER, "Promoter"),
        (HUB_COORDINATOR, "Hub Coordinator"),
        (STAFF, "Staff"),
    ]

    name = models.CharField(max_length=20, choices=CHOICES, unique=True)


class ProfileRole(BaseModel):
    """
    M2M entre [[05-data|Profile]] e Role.
    Permite múltiplas roles simultâneas.
    """
    profile = models.ForeignKey("data.Profile", on_delete=models.CASCADE, related_name="profile_roles")
    role = models.ForeignKey(Role, on_delete=models.PROTECT)
    active = models.BooleanField(default=True)

    class Meta:
        unique_together = [("profile", "role")]
```

> [!warning] FK para data.Profile
> O `ProfileRole` tem FK para `data.Profile` como string. Em FastAPI, isso vira um relacionamento via `external_id` — ver [[05-data|05-data]].

### `auth/models/otp_code.py`

```python
import random
from datetime import timedelta

from django.db import models
from django.utils import timezone

from core.models.base import BaseModel


class OTPCode(BaseModel):
    profile = models.ForeignKey("data.Profile", on_delete=models.CASCADE, related_name="otp_codes")
    code = models.CharField(max_length=6)
    expires_at = models.DateTimeField()
    used = models.BooleanField(default=False)

    @classmethod
    def generate(cls, profile, ttl_minutes: int = 10):
        code = f"{random.randint(0, 999999):06d}"
        return cls.objects.create(
            profile=profile,
            code=code,
            expires_at=timezone.now() + timedelta(minutes=ttl_minutes),
        )

    def is_valid(self) -> bool:
        return (not self.used) and timezone.now() < self.expires_at
```

> [!note] Herda de BaseModel
> OTPCode usa `BaseModel` de [[03-core]], herdando UUID, timestamps e outras convenções.

---

## 4.2 Tools (interface pública)

### `auth/tools/check.py`

Recebe número, devolve estado do usuário.

```python
# auth/tools/check.py
from core.notify import service as notify_service
from core.validators.phone import normalize_phone
from data.models import Profile

def check(numero: str) -> dict:
    """
    Retorna dict com:
      - external_id (se já existe)
      - first_name (se já existe e tem profile preenchido)
      - roles: list[str] (se já existe)
      - whatsapp_valid: bool (se não existe)
    """
    numero = normalize_phone(numero)
    notify_result = notify_service.check(numero)

    if notify_result.get("external_id"):
        # Existe no notify → procurar no nosso DB
        try:
            profile = Profile.objects.get(external_id=notify_result["external_id"])
            return {
                "external_id": str(profile.external_id),
                "first_name": profile.user.first_name or None,
                "roles": [pr.role.name for pr in profile.profile_roles.filter(active=True)],
                "whatsapp_valid": True,
            }
        except Profile.DoesNotExist:
            # Existe no notify mas não no nosso DB — caso raro
            return {"external_id": None, "whatsapp_valid": True}

    return {"external_id": None, "whatsapp_valid": notify_result.get("whatsapp_valid", False)}
```

> [!info] Notify como fonte de verdade para WhatsApp
> O `check()` primeiro consulta o [[02-integracoes-externas#notify|Notify]] para validar se o número existe no WhatsApp. O notify é a fonte de verdade para `whatsapp_valid`.

### `auth/tools/otp.py`

Gera código de 6 dígitos, salva, envia notify.

```python
# auth/tools/otp.py
import config

from core.notify import service as notify_service
from data.models import Profile
from auth.models.otp_code import OTPCode


def otp(external_id: str) -> dict:
    """
    Gera OTP e envia 2 notificações:
      1. auth/notify/otp.md → texto com o código
      2. auth/notify/magic_link_<role>.md → link com OTP embutido
    """
    profile = Profile.objects.get(external_id=external_id)
    otp_code = OTPCode.generate(profile)

    # Determinar role principal para o magic link
    active_roles = [pr.role.name for pr in profile.profile_roles.filter(active=True)]
    role = _resolve_primary_role(active_roles)

    frontend_url = getattr(config, f"FRONTEND_URL_LOGIN_{role.upper()}")

    # 1. Notify do código
    notify_service.send(
        external_id=str(profile.external_id),
        template_path="auth/notify/otp.md",
        context={"otp": otp_code.code, "first_name": profile.user.first_name or "amigo(a)"},
    )

    # 2. Notify do magic link
    notify_service.send(
        external_id=str(profile.external_id),
        template_path=f"auth/notify/magic_link_{role}.md",
        context={
            "otp": otp_code.code,
            "external_id": str(profile.external_id),
            "frontend_url": frontend_url,
            "first_name": profile.user.first_name or "amigo(a)",
        },
    )

    return {"sent": True, "expires_at": otp_code.expires_at.isoformat()}


def _resolve_primary_role(roles: list[str]) -> str:
    """
    Quando user tem múltiplas roles, escolhe a "mais alta" para o magic link.
    Ordem: staff > hub_coordinator > promoter > candidate > student > lead
    """
    priority = ["staff", "hub_coordinator", "promoter", "candidate", "student", "lead"]
    for r in priority:
        if r in roles:
            return r
    return "lead"  # fallback
```

> [!tip] Duas notificações por OTP
> O `otp()` envia **duas** mensagens via [[02-integracoes-externas#notify|Notify]]: uma com o código puro e outra com magic link. A URL do frontend vem de [[01-config-e-env]] (`FRONTEND_URL_LOGIN_{ROLE}`).

### `auth/tools/register.py`

Registro polimórfico — chamado por `lead.register` e `candidate.register`.

```python
# auth/tools/register.py
from django.contrib.auth import get_user_model
from django.db import transaction

from core.notify import service as notify_service
from core.validators.cpf import validate_cpf
from core.validators.phone import normalize_phone

from auth.tools.check import check
from auth.models.role import Role, ProfileRole
from data.tools.new_profile import new_profile

User = get_user_model()


@transaction.atomic
def register(numero: str, cpf: str, role: str) -> dict:
    """
    Registra novo usuário com role específica.

    Pré-condições:
      - número válido (whatsapp_valid=True)
      - CPF não existe ainda
      - CPF é válido

    Retorna {external_id, profile, user}.
    """
    numero = normalize_phone(numero)
    cpf_clean = validate_cpf(cpf)  # tira pontos/traços + valida algoritmo

    # 1. Confere se número existe no notify e é válido
    check_result = check(numero)
    if check_result["external_id"]:
        raise ValueError("Já existe usuário com esse número.")
    if not check_result["whatsapp_valid"]:
        raise ValueError("WhatsApp inválido.")

    # 2. Confere CPF não duplicado
    from data.models import Profile
    if Profile.objects.filter(cpf=cpf_clean).exists():
        raise ValueError("CPF já cadastrado.")

    # 3. Cria User (sem senha — só OTP)
    user = User.objects.create(username=cpf_clean, is_active=True)

    # 4. Cria Profile (data.tools.new_profile faz toda criação cascata)
    profile = new_profile(user_id=user.id, cpf=cpf_clean)

    # 5. Adiciona role
    role_obj = Role.objects.get(name=role)
    ProfileRole.objects.create(profile=profile, role=role_obj)

    # 6. Cadastra recipient no notify
    notify_service.new_recipient(numero=numero, external_id=str(profile.external_id))

    return {"external_id": str(profile.external_id), "profile": profile, "user": user}
```

> [!warning] CPF e WhatsApp são pré-condições
> O registro exige: (1) WhatsApp válido via [[02-integracoes-externas#notify|Notify]], (2) CPF validado e único. Sem senha — autenticação é só por OTP.

### `auth/tools/add_role.py` e `change_role.py`

```python
# auth/tools/add_role.py
def add_role(profile, role_name: str):
    """Adiciona role mantendo as existentes (ex: lead vira lead+student)."""
    ...

# auth/tools/change_role.py
def change_role(profile, from_role: str, to_role: str):
    """
    Substitui uma role por outra.
    Ex: lead → student (após pagamento da matrícula).
    Mantém histórico (ProfileRole.active=False).
    """
    ...
```

> [!note] Histórico preservado
> `change_role` mantém `active=False` na role antiga, preservando o histórico de transições — consistente com os [[00-arquitetura-geral#roles-disponiveis|princípios de roles]].

---

## 4.3 API

### Schemas (`auth/api/schemas.py`)

```python
from ninja import Schema

class LoginIn(Schema):
    external_id: str
    otp: str

class LoginOut(Schema):
    access: str
    refresh: str

class RefreshIn(Schema):
    refresh: str

class RefreshOut(Schema):
    access: str
```

### `auth/api/public.py`

```python
from ninja import Router
from .schemas import LoginIn, LoginOut
from auth.services.jwt import generate_tokens
from auth.services.otp_storage import validate_otp
from data.models import Profile

router = Router(tags=["auth"])

@router.post("/login", response=LoginOut)
def login(request, payload: LoginIn):
    profile = Profile.objects.get(external_id=payload.external_id)

    if not validate_otp(profile, payload.otp):
        return router.create_response(
            request, {"detail": "OTP inválido ou expirado"}, status=401
        )

    return generate_tokens(profile)
```

> [!info] Endpoint público
> `POST /login` não exige autenticação prévia. O OTP é a única credencial. A assinatura JWT usa `JWT_SIGNING_KEY` de [[01-config-e-env]].

### `auth/api/authenticated.py`

```python
from ninja import Router
from .schemas import RefreshIn, RefreshOut

router = Router(tags=["auth"])

@router.post("/refresh", response=RefreshOut)
def refresh(request, payload: RefreshIn):
    ...  # usa django-ninja-jwt
```

### JWT payload

```json
{
  "external_id": "uuid",
  "roles": ["lead", "candidate"],
  "exp": 1234567890
}
```

> [!note] Payload mínimo e portátil
> O JWT carrega apenas `external_id` e `roles`. Nada de dados pessoais — o frontend busca o resto via `/v1/data/profile/me/` no [[05-data]].

---

## 4.4 Templates de notify

### `auth/notify/otp.md`

```
Olá {{ first_name }}! Seu código de acesso é {{ otp }}. Ele expira em 10 minutos.
```

### `auth/notify/magic_link_lead.md`

```
Olá {{ first_name }}! Para entrar, clique no link: {{ frontend_url }}/{{ external_id }}?otp={{ otp }}
```

> [!note] Demais magic links
> Os outros `magic_link_*.md` seguem o mesmo padrão, mudando apenas o `frontend_url` injetado pela função `otp()`. A URL vem de [[01-config-e-env]] (`FRONTEND_URL_LOGIN_{ROLE}`).

---

## 4.5 Polimorfismo — como `lead.register` usa isto

```python
# lead/tools/register.py (próximo doc)
from auth.tools.register import register as auth_register
from auth.tools.otp import otp as auth_otp

def register_lead(numero, cpf, promoter_external_id):
    result = auth_register(numero=numero, cpf=cpf, role="lead")
    # ... cria Lead model, vincula promoter, etc
    auth_otp(result["external_id"])  # já dispara OTP automaticamente
    return result
```

> [!tip] Polimorfismo no registro
> `auth.register` é genérico — recebe `role` como parâmetro. Cada app (`lead`, `candidate`, etc.) chama com sua role específica e depois faz o setup adicional. Ver [[06-lead]] para o fluxo completo do lead.

---

## 4.6 Variáveis de config usadas

| Variável | Uso |
| :--- | :--- |
| `FRONTEND_URL_LOGIN_LEAD` | magic_link_lead.md |
| `FRONTEND_URL_LOGIN_STUDENT` | magic_link_student.md |
| `FRONTEND_URL_LOGIN_PROMOTER` | magic_link_promoter.md |
| `FRONTEND_URL_LOGIN_CANDIDATE` | magic_link_candidate.md |
| `FRONTEND_URL_LOGIN_HUB` | magic_link_hub.md |
| `FRONTEND_URL_LOGIN_STAFF` | magic_link_staff.md |
| `JWT_SIGNING_KEY` | services.jwt |
| `JWT_ACCESS_TTL` | services.jwt |
| `JWT_REFRESH_TTL` | services.jwt |

> [!info] Configuração centralizada
> Todas essas variáveis vêm de [[01-config-e-env]] via `config.X`. As URLs de frontend são de **negócio** (editáveis via [[12-staff|staff]] em runtime). As keys/TTLs do JWT são **técnicas** (imutáveis, só `.env`).

---
tags: [supletico, core]
aliases: [Core]
created: 2026-04-29
---

# 03 — Core

Base técnica compartilhada. Não tem API exposta. Não tem regra de negócio. Outros apps importam de `core.X` para reaproveitar.

## Estrutura

```
core/
├── models/
│   ├── __init__.py
│   └── base.py              # BaseModel abstrato (created_at, updated_at, created_by, updated_by)
├── notify/                  # cliente da integração de notificações
│   ├── __init__.py
│   ├── client.py            # HTTP cru
│   ├── render.py            # parser de .md (diretivas + Jinja)
│   └── service.py           # API pública: send/check/new_recipient
├── infinitepay/             # cliente da integração de checkout
│   ├── __init__.py
│   ├── client.py
│   ├── bootstrap.py         # PATCH /config/ no boot
│   └── service.py
├── asaas/                   # cliente da integração financeira
│   ├── __init__.py
│   ├── client.py
│   └── service.py
├── files/
│   ├── __init__.py
│   ├── upload.py            # validação png/jpg/pdf, tamanho máximo
│   └── url.py               # build_external_url(file_field)
├── auth_helpers/            # decorators / dependências de role para django-ninja
│   ├── __init__.py
│   ├── role_required.py
│   └── jwt_authenticator.py
├── validators/              # validators reutilizáveis (não específicos de um app)
│   ├── __init__.py
│   ├── cpf.py
│   ├── phone.py
│   └── email.py
├── exceptions/
│   ├── __init__.py
│   └── api.py               # exceções padrão (NotFound, Forbidden, etc.)
└── apps.py                  # AppConfig — chama bootstrap do infinitepay no ready()
```

---

## 3.1 core/models/base.py

Todos os models de todos os apps herdam do [[00-arquitetura-geral|BaseModel]].

```python
import uuid
from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()

class BaseModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    created_by = models.ForeignKey(
        User, null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name="+",
    )
    updated_by = models.ForeignKey(
        User, null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name="+",
    )

    class Meta:
        abstract = True

class ExternalIdMixin(models.Model):
    """
    Mixin para models que precisam de UUID público.
    Usar em todos os models expostos via API.
    """
    external_id = models.UUIDField(default=uuid.uuid4, editable=False, unique=True, db_index=True)

    class Meta:
        abstract = True
```

Quem preenche `created_by`/`updated_by`? O middleware de auditoria (em `staff/middleware/audit.py`) injeta `request.user` no save via thread-local. Detalhar em 12-staff.md.

---

## 3.2 core/notify/

### client.py

Cliente HTTP cru. Não tem regra de negócio, só faz request.

```python
# core/notify/client.py
import httpx
from config import NOTIFY_API_URL, NOTIFY_API_TOKEN  # token a confirmar nos docs

class NotifyClient:
    def __init__(self):
        self.base_url = NOTIFY_API_URL
        self.headers = {"Authorization": f"Bearer {NOTIFY_API_TOKEN}"}

    def send_notification(self, external_id: str, content: str, is_tts: bool = False, media_url: str | None = None) -> dict:
        payload = {"external_id": external_id, "content": content, "is_tts": is_tts}
        if media_url:
            payload["media_url"] = media_url
        r = httpx.post(f"{self.base_url}/api/v1/notifications", json=payload, headers=self.headers, timeout=10)
        r.raise_for_status()
        return r.json()

    def check_number(self, numero: str) -> dict:
        r = httpx.get(f"{self.base_url}/v1/recipients/check", params={"q": numero}, headers=self.headers, timeout=10)
        r.raise_for_status()
        return r.json()  # {"external_id": "..."} OR {"whatsapp_valid": true|false}

    def new_recipient(self, numero: str, external_id: str) -> dict:
        r = httpx.post(f"{self.base_url}/api/v1/recipients", json={"numero": numero, "external_id": external_id}, headers=self.headers, timeout=10)
        r.raise_for_status()
        return r.json()
```

> [!warning] Endpoints a validar
> Token `NOTIFY_API_TOKEN` e paths exatos (ex: `/api/v1/notifications`, `/v1/recipients/check`) precisam ser confirmados nos docs da integração [[02-integracoes-externas|Notify]].

### render.py

Parser dos templates `.md`.

```python
# core/notify/render.py
from pathlib import Path
from jinja2 import Template

DIRECTIVE_TTS = "--tts"
DIRECTIVE_MEDIA = "--media:"

def render_template(template_path: str, context: dict) -> dict:
    """
    Lê arquivo .md, extrai diretivas da 1ª linha (se houver),
    renderiza Jinja com context, retorna {text, is_tts, media_url}.
    """
    raw = Path(template_path).read_text(encoding="utf-8")
    lines = raw.splitlines()
    is_tts = False
    media_url = None
    body_start = 0

    if lines and lines[0].strip() == DIRECTIVE_TTS:
        is_tts = True
        body_start = 1
    elif lines and lines[0].strip().startswith(DIRECTIVE_MEDIA):
        media_template = lines[0].strip()[len(DIRECTIVE_MEDIA):].strip()
        media_url = Template(media_template).render(**context)
        body_start = 1

    body = "\n".join(lines[body_start:]).strip()
    text = Template(body).render(**context)
    return {"text": text, "is_tts": is_tts, "media_url": media_url}
```

### service.py

API pública que outros apps chamam.

```python
# core/notify/service.py
from .client import NotifyClient
from .render import render_template

_client = NotifyClient()

def send(
    external_id: str | None = None,
    template_path: str | None = None,
    text: str | None = None,
    context: dict | None = None,
    is_tts: bool = False,
    media_url: str | None = None,
    user=None,
) -> dict:
    """
    Envia notificação.
    - Se external_id None, pega do `user.profile.external_id` (caller responsabiliza-se de passar user)
    - Template_path tem precedência sobre text
    """
    if external_id is None:
        if user is None:
            raise ValueError("external_id ou user obrigatório")
        external_id = str(user.profile.external_id)

    if template_path:
        rendered = render_template(template_path, context or {})
        text = rendered["text"]
        is_tts = rendered["is_tts"] or is_tts
        media_url = rendered["media_url"] or media_url

    if text is None:
        raise ValueError("text ou template_path obrigatório")

    return _client.send_notification(external_id, text, is_tts, media_url)

def check(numero: str) -> dict:
    """
    Retorna {external_id, whatsapp_valid} normalizado.
    """
    raw = _client.check_number(numero)
    return {
        "external_id": raw.get("external_id"),
        "whatsapp_valid": raw.get("whatsapp_valid", False) if "external_id" not in raw else True,
    }

def new_recipient(numero: str, external_id: str) -> dict:
    return _client.new_recipient(numero, external_id)
```

---

## 3.3 core/infinitepay/

### bootstrap.py

Roda no `apps.py ready()` do core — envia `PATCH /config/` para o serviço InfinitePay com as configs do nosso `.env`.

```python
# core/infinitepay/bootstrap.py
import httpx
import config

def patch_infinitepay_config():
    """
    Chamado no boot do app. Envia somente os campos definidos em config.
    """
    payload = {}
    if hasattr(config, "INFINITEPAY_HANDLE"):
        payload["handle"] = config.INFINITEPAY_HANDLE
    if hasattr(config, "LEAD_CHECKOUT_PRICE"):
        payload["price"] = float(config.LEAD_CHECKOUT_PRICE)
    if hasattr(config, "LEAD_CHECKOUT_DESCRIPTION"):
        payload["description"] = config.LEAD_CHECKOUT_DESCRIPTION

    payload["redirect_url"] = f"{config.EXTERNAL_URL}/api/v1/lead/webhook/infinitepay/"

    if hasattr(config, "INFINITEPAY_BACKEND_WEBHOOK"):
        payload["backend_webhook"] = config.INFINITEPAY_BACKEND_WEBHOOK

    if not payload:
        return  # nada para configurar

    try:
        r = httpx.patch(f"{config.INFINITEPAY_API_URL}/config/", json=payload, timeout=10)
        r.raise_for_status()
    except Exception as e:
        # logar mas não quebrar boot
        import logging
        logging.getLogger(__name__).error("InfinitePay bootstrap falhou: %s", e)
```

### client.py

```python
# core/infinitepay/client.py
import httpx
import config

class InfinitePayClient:
    def __init__(self):
        self.base_url = config.INFINITEPAY_API_URL

    def create(self, *, external_id: str, name: str, number: str, email: str, address: dict) -> dict:
        """
        POST /checkout/  → cria checkout para um lead.
        Retorna {checkout_url}.
        """
        payload = {
            "external_id": external_id,
            "name": name,
            "number": number,
            "email": email,
            "address": address,
        }
        r = httpx.post(f"{self.base_url}/checkout/", json=payload, timeout=15)
        r.raise_for_status()
        return r.json()

    def check(self, external_id: str) -> dict:
        """
        GET /checkout/<external_id>  → retorna {checkout_url, receipt_url, is_paid}.
        """
        r = httpx.get(f"{self.base_url}/checkout/{external_id}", timeout=10)
        r.raise_for_status()
        return r.json()
```

> [!warning] Endpoints a validar nos docs do InfinitePay
> Consulte [[02-integracoes-externas|InfinitePay]] — docs em `http://10.10.10.120:8000/docs`:
> - Path exato (`/checkout/` vs `/v1/checkout/?`)
> - Autenticação (token bearer? api key?)
> - Formato exato do retorno

---

## 3.4 core/asaas/

### client.py

```python
# core/asaas/client.py
import httpx
import config

class AsaasClient:
    def __init__(self):
        self.base_url = config.ASAAS_API_URL

    def check_pix_key(self, *, external_id: str) -> dict | None:
        """
        Verifica se já existe chave PIX cadastrada para esse external_id.
        Retorna dict com dados da chave se existir, None caso contrário.
        ⚠️ Endpoint exato a confirmar nos docs do Asaas (http://10.10.10.121/docs).
        Possibilidades comuns:
          - GET /pixkey/{external_id}
          - GET /pixkey?external_id={external_id}
        """
        try:
            r = httpx.get(f"{self.base_url}/pixkey/{external_id}", timeout=10)
            if r.status_code == 404:
                return None
            r.raise_for_status()
            return r.json()
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 404:
                return None
            raise

    def cadastra_pix_key(self, *, document: str, external_id: str, key: str, key_type: str) -> dict:
        """
        Cadastra chave PIX. Idempotente: se já existir, retorna o registro existente
        (verificação feita no `service.py`, não aqui — o client é cru).
        """
        payload = {
            "document": document,
            "external_id": external_id,
            "key": key,
            "key_type": key_type,
        }
        r = httpx.post(f"{self.base_url}/pixkey", json=payload, timeout=10)
        r.raise_for_status()
        return r.json()

    def pay(self, *, amount: float, description: str, payment_id: str, pixkey_external_id: str) -> dict:
        payload = {
            "amount": amount,
            "description": description,
            "payment_id": payment_id,
            "pixkey_external_id": pixkey_external_id,
        }
        r = httpx.post(f"{self.base_url}/payment", json=payload, timeout=15)
        r.raise_for_status()
        return r.json()

    def pay_scheduled(self, *, amount: float, description: str, payment_id: str, pixkey_external_id: str, scheduled_date: str) -> dict:
        payload = {
            "amount": amount,
            "description": description,
            "payment_id": payment_id,
            "pixkey_external_id": pixkey_external_id,
            "scheduled_date": scheduled_date,  # ISO YYYY-MM-DD
        }
        r = httpx.post(f"{self.base_url}/payment/scheduled", json=payload, timeout=15)
        r.raise_for_status()
        return r.json()

    def pay_qrcode(self, *, qrcode: str, payment_id: str) -> dict:
        r = httpx.post(f"{self.base_url}/payment/qrcode", json={"qrcode": qrcode, "payment_id": payment_id}, timeout=15)
        r.raise_for_status()
        return r.json()

    def pay_qrcode_scheduled(self, *, qrcode: str, payment_id: str, scheduled_date: str) -> dict:
        r = httpx.post(f"{self.base_url}/payment/qrcode/scheduled", json={"qrcode": qrcode, "payment_id": payment_id, "scheduled_date": scheduled_date}, timeout=15)
        r.raise_for_status()
        return r.json()

    def analyze_qrcode(self, *, qrcode: str) -> dict:
        """
        Valida QR Code e retorna {amount, due_date, payee, ...}.
        """
        r = httpx.post(f"{self.base_url}/payment/qrcode/analyze", json={"qrcode": qrcode}, timeout=10)
        r.raise_for_status()
        return r.json()
```

> [!warning] Endpoints a validar nos docs do Asaas
> Consulte [[02-integracoes-externas|Asaas]] — docs em `http://10.10.10.121/`:
> - Path exato e versionamento
> - Autenticação
> - Campos obrigatórios em cada payload
> - Endpoint correto para verificar existência de pix_key (GET por external_id)

### service.py

API pública. Outros módulos chamam **daqui**, não do `client.py` direto. É aqui que mora a lógica de idempotência.

```python
# core/asaas/service.py
from .client import AsaasClient

_client = AsaasClient()

def ensure_pix_key(*, document: str, external_id: str, key: str, key_type: str = "CPF") -> dict:
    """
    Garante que existe uma chave PIX cadastrada para este external_id.
    - Se já existir, retorna o registro existente sem fazer POST
    - Se não existir, cadastra e retorna
    Idempotente — pode ser chamado múltiplas vezes sem efeito colateral.
    Usado por candidate.services.asaas_setup quando candidate avança para status 20.
    """
    existing = _client.check_pix_key(external_id=external_id)
    if existing:
        return existing
    return _client.cadastra_pix_key(
        document=document,
        external_id=external_id,
        key=key,
        key_type=key_type,
    )

# Wrappers diretos para os outros métodos (sem lógica adicional por enquanto)
def pay(**kwargs) -> dict:
    return _client.pay(**kwargs)

def pay_scheduled(**kwargs) -> dict:
    return _client.pay_scheduled(**kwargs)

def pay_qrcode(**kwargs) -> dict:
    return _client.pay_qrcode(**kwargs)

def pay_qrcode_scheduled(**kwargs) -> dict:
    return _client.pay_qrcode_scheduled(**kwargs)

def analyze_qrcode(**kwargs) -> dict:
    return _client.analyze_qrcode(**kwargs)
```

---

## 3.5 core/files/

### upload.py

```python
# core/files/upload.py
ALLOWED_IMAGE_EXTENSIONS = {".jpg", ".jpeg", ".png"}
ALLOWED_DOCUMENT_EXTENSIONS = {".jpg", ".jpeg", ".png", ".pdf"}
MAX_FILE_SIZE_MB = 10

def validate_image(file) -> None:
    """Aceita só jpg/png. Levanta ValueError."""
    ...

def validate_document(file) -> None:
    """Aceita jpg/png/pdf. Levanta ValueError."""
    ...
```

### url.py

```python
# core/files/url.py
import config

def build_external_url(file_field) -> str | None:
    """
    Recebe um FileField e retorna URL absoluta:
    "{EXTERNAL_URL}/media/{caminho}"
    """
    if not file_field or not file_field.name:
        return None
    return f"{config.EXTERNAL_URL}/media/{file_field.name}"
```

Schemas Ninja que retornam arquivos usam essa função no `from_orm`/serializer.

---

## 3.6 core/auth_helpers/

### jwt_authenticator.py

Wrapper sobre django-ninja-jwt que carrega roles da claim do JWT e injeta em `request.roles`.

### role_required.py

Decorator/dependência para proteger rotas por role:

```python
# core/auth_helpers/role_required.py
from ninja.errors import HttpError

def require_role(*roles: str):
    """
    Dependência ninja que valida se o user logado tem alguma das roles.
    Uso:
        @router.get("/", auth=JWTAuth())
        def list_leads(request, _: None = Depends(require_role("promoter", "staff"))):
            ...
    """
    def _check(request):
        user_roles = set(getattr(request, "roles", []))
        if not user_roles.intersection(roles):
            raise HttpError(403, f"Requer role(s): {', '.join(roles)}")
        return None
    return _check
```

---

## 3.7 core/validators/

Validators **realmente** genéricos. Os específicos de cada app ficam em `<app>/validators/`.

- `cpf.py` — algoritmo de validação de CPF (dígitos verificadores)
- `phone.py` — normalização para E.164 (`+559****9999`)
- `email.py` — wrapper sobre `email_validator` lib

---

## 3.8 core/apps.py

```python
# core/apps.py
from django.apps import AppConfig

class CoreConfig(AppConfig):
    name = "core"

    def ready(self):
        from .infinitepay.bootstrap import patch_infinitepay_config
        patch_infinitepay_config()
```

---

## 3.9 Convenções importantes

> [!tip] Convenções
> - **Nenhum app importa diretamente de `core/*/client.py`** — sempre via `core/*/service.py` ou tools dos próprios apps
> - **core não tem migrations** (só models abstratos) — exceto se aparecer necessidade futura
> - **Toda exceção HTTP usa `core.exceptions.api`** — padronização do formato de erro
> - **Logging:** todos clientes usam `logging.getLogger(__name__)` — outputs para stdout em DEBUG, para arquivo/loki em prod (a definir)

---

## 3.10 Variáveis de config usadas neste módulo

| Variável | Uso |
| :--- | :--- |
| `EXTERNAL_URL` | `core.files.url.build_external_url` |
| `NOTIFY_API_URL` | `core.notify.client` |
| `NOTIFY_API_TOKEN` ⚠️ | `core.notify.client` (a confirmar) |
| `INFINITEPAY_API_URL` | `core.infinitepay.client/bootstrap` |
| `INFINITEPAY_HANDLE` ⚠️ | `core.infinitepay.bootstrap` |
| `INFINITEPAY_BACKEND_WEBHOOK` ⚠️ | `core.infinitepay.bootstrap` |
| `LEAD_CHECKOUT_PRICE` | `core.infinitepay.bootstrap` |
| `LEAD_CHECKOUT_DESCRIPTION` | `core.infinitepay.bootstrap` |
| `ASAAS_API_URL` | `core.asaas.client` |
| `ASAAS_API_TOKEN` ⚠️ | `core.asaas.client` (a confirmar) |

⚠️ = adicionar ao `.env.example` quando confirmar nos docs das integrações. Ver [[01-config-e-env|variáveis de ambiente]] para detalhes.

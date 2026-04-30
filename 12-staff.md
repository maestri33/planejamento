---
tags:
  - supletico
  - staff
aliases:
  - Staff
created: 2026-04-29
status: especificação
---

# 12 — Staff

Administração centralizada. O app `staff` é o **ponto único de gestão** do sistema: endpoints que expõem dados de **todos os apps** com visão irrestrita, gerenciamento de configuração em runtime e auditoria completa.

> [!info] Conexões
> - [[00-arquitetura-geral]] — padrão de módulo replicado, RBAC, logs centralizados
> - [[01-config-e-env]] — SystemConfig com fallback `.env`, cache Redis, audit log
> - [[03-core]] — BaseModel, `created_by`/`updated_by` injetado via middleware
> - [[04-auth]] — role `staff`, magic_link_staff, `FRONTEND_URL_LOGIN_STAFF`
> - [[05-data]] — list_all_profiles, get_profile, patch_profile
> - [[06-lead]] — list_all, get_by_id (sem filtro de promoter)
> - [[07-enrollment]] — pendencies, document_analysis, upload_final_documents
> - [[08-finance]] — list_all_commissions, list_all_payments, schedule_qrcode manual
> - [[11-candidate]] — list_all, approve, reject
> - [[02-integracoes-externas]] — staff pode disparar ações manuais nas integrações

> [!warning] Regras fundamentais
> 1. **Staff NÃO mexe no DB de nenhum outro app.** Só chama `tools.*` dos outros apps.
> 2. **Staff é o ÚNICO app com acesso irrestrito** a listagens de todos os domínios (sem filtro de dono/hub/promoter).
> 3. **Staff gerencia config runtime** (`SystemConfig`) com cache Redis e invalidação no PATCH.
> 4. **Toda chamada (API + CLI) é auditada** nos models `ApiLog` e `CommandLog`.

---

## Estrutura

staff/

├── models/
│   ├── __init__.py
│   ├── system_config.py
│   ├── system_config_log.py
│   ├── api_log.py
│   └── command_log.py
├── api/
│   ├── __init__.py              # router principal, prefixo /v1/staff
│   ├── schemas.py               # In/Out compartilhados
│   ├── leads.py                 # GET /leads, GET /leads/{status}, GET /leads/{external_id}
│   ├── students.py              # POST/DELETE pendency, PATCH document, document_analysis
│   ├── finance.py               # GET commissions/payments, POST schedule_qrcode
│   ├── candidates.py            # GET candidates, POST approval
│   ├── profiles.py              # GET/PATCH profiles
│   └── config.py                # GET/PATCH config, GET /config/{key}/history
├── services/
│   ├── __init__.py
│   └── system_config.py         # get_config, set_config, get_all, invalidate_cache
├── middleware/
│   ├── __init__.py
│   └── audit.py                 # ApiLog + CommandLog + thread-local request.user
├── notify/
│   ├── asaas_pix_setup_failed.md
│   ├── payment_processing_failed.md
│   ├── friday_closing_error.md
│   ├── webhook_validation_failed.md
│   └── weekly_report.md
└── apps.py

---

## 12.1 Models

### `staff/models/system_config.py`

```python
from django.db import models

from core.models.base import BaseModel


class SystemConfig(BaseModel):
    """Configuração de negócio editável em runtime.
    Todas as vars da categoria NEGÓCIO em [[01-config-e-env]] podem ser
    persistidas aqui, com fallback pro .env quando não existirem.
    """

    key = models.CharField(max_length=255, unique=True)
    value = models.TextField()

    # Auditoria
    updated_by = models.ForeignKey(
        "data.Profile",
        null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name="config_changes",
    )

    class Meta:
        db_table = "staff_system_config"
        verbose_name = "System Config"
        verbose_name_plural = "System Configs"

    def __str__(self):
        return f"{self.key} = {self.value}"
```

As chaves suportadas são as variáveis de negócio de [[01-config-e-env#Negócio (editáveis via staff em runtime)]]:

- `PROMOTER_COMMISSION_VALUE`, `HUB_COORDINATOR_COMMISSION_VALUE`
- `WEEKLY_TARGET_QUANTITY`, `WEEKLY_TARGET_BONUS_VALUE`
- `ENROLLMENT_FIRST_PART_FEE`, `ENROLLMENT_SECOND_PART_FEE`
- `LEAD_CHECKOUT_PRICE`, `LEAD_CHECKOUT_DESCRIPTION`
- `DEFAULT_PROMOTER`, `DEFAULT_HUB`
- `FRONTEND_URL_LOGIN_LEAD`, `FRONTEND_URL_LOGIN_STUDENT`, `FRONTEND_URL_LOGIN_PROMOTER`
- `FRONTEND_URL_LOGIN_CANDIDATE`, `FRONTEND_URL_LOGIN_HUB`, `FRONTEND_URL_LOGIN_STAFF`
- `FRONTEND_CANDIDATE_URL`

### `staff/models/system_config_log.py`

```python
from django.db import models

from core.models.base import BaseModel


class SystemConfigLog(BaseModel):
    """Audit log de toda alteração em SystemConfig."""

    config = models.ForeignKey(
        "staff.SystemConfig",
        on_delete=models.CASCADE,
        related_name="logs",
    )
    previous_value = models.TextField(null=True, blank=True)
    new_value = models.TextField()
    changed_by = models.ForeignKey(
        "data.Profile",
        null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name="config_logs",
    )

    class Meta:
        db_table = "staff_system_config_log"
        verbose_name = "System Config Log"
        verbose_name_plural = "System Config Logs"
        ordering = ["-created_at"]

    def __str__(self):
        return f"{self.config.key}: {self.previous_value} → {self.new_value}"
```

> [!info] Tabela de auditoria cresce
> `SystemConfigLog` acumula entradas a cada PATCH. A API staff expõe GET `/config/{key}/history` para consulta com paginação. Ver [[#12.2.6 Config]].

### `staff/models/api_log.py`

```python
from django.db import models

from core.models.base import BaseModel


class ApiLog(BaseModel):
    """Toda chamada HTTP é logada aqui pelo middleware de auditoria."""

    user = models.ForeignKey(
        "data.Profile",
        null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name="api_logs",
    )
    method = models.CharField(max_length=10)
    path = models.CharField(max_length=512)
    status_code = models.IntegerField()
    ip = models.GenericIPAddressField(null=True, blank=True)
    duration_ms = models.IntegerField(null=True, blank=True)

    class Meta:
        db_table = "staff_api_log"
        verbose_name = "API Log"
        verbose_name_plural = "API Logs"
        ordering = ["-created_at"]

    def __str__(self):
        return f"{self.method} {self.path} → {self.status_code}"
```

### `staff/models/command_log.py`

```python
from django.db import models

from core.models.base import BaseModel


class CommandLog(BaseModel):
    """Toda execução de CLI (manage.py) é logada aqui."""

    user = models.ForeignKey(
        "data.Profile",
        null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name="command_logs",
    )
    command = models.CharField(max_length=255)
    args = models.TextField(blank=True)

    class Meta:
        db_table = "staff_command_log"
        verbose_name = "Command Log"
        verbose_name_plural = "Command Logs"
        ordering = ["-created_at"]

    def __str__(self):
        return f"{self.command} {self.args}"
```

---

## 12.2 API

Regra de ouro: **nenhum endpoint de staff acessa models de outros apps diretamente**. Toda listagem, criação ou modificação é feita via `tools.*` dos apps donos dos dados.

> [!info] Prefixo
> Todos os endpoints abaixo são montados em `/v1/staff`. O router principal (`staff/api/__init__.py`) agrega os sub-routers com `require_role("staff")`.

### 12.2.1 Leads (`staff/api/leads.py`)

```python
from ninja import Router
from ninja.security import HttpBearer

from auth.dependencies import require_role
from lead.tools.list_all import list_all as lead_list_all
from lead.tools.get_by_id import get_by_id as lead_get_by_id

router = Router(tags=["staff-leads"])


@router.get("/leads", auth=HttpBearer())
def list_all_leads(request, _: None = require_role("staff")):
    """Lista todos os leads do sistema, sem filtro de promoter."""
    return lead_list_all()


@router.get("/leads/{status}", auth=HttpBearer())
def list_leads_by_status(request, status: int, _: None = require_role("staff")):
    """Filtra leads por status (1-100, 999)."""
    return lead_list_all(status=status)


@router.get("/leads/{external_id}", auth=HttpBearer())
def get_lead(request, external_id: str, _: None = require_role("staff")):
    """Busca lead específico por external_id."""
    return lead_get_by_id(external_id)
```

> [!info] Sem filtro de dono
> Ao contrário do endpoint de promoter (`GET /leads` em [[06-lead]] que filtra por `promoter=request.user`), o endpoint staff retorna **todos** os registros. Esta é a única visão irrestrita de leads no sistema.

### 12.2.2 Students (`staff/api/students.py`)

```python
from ninja import Router

from auth.dependencies import require_role
from enrollment.tools.create_pendency import create_pendency
from enrollment.tools.list_pendencies import list_pendencies
from enrollment.tools.resolve_pendency import resolve_pendency
from enrollment.tools.approve_document_analysis import approve_document_analysis
from enrollment.tools.upload_final_documents import upload_final_documents

router = Router(tags=["staff-students"])


@router.post("/{external_id}/pendency", auth=HttpBearer())
def post_pendency(request, external_id: str, payload: PendencyIn, file: UploadedFile = File(None), _: None = require_role("staff")):
    """Cria pendência documental para um aluno. Opcionalmente anexa arquivo."""
    return create_pendency(external_id=external_id, content=payload.content, file=file)


@router.get("/{external_id}/pendencies", auth=HttpBearer())
def get_pendencies(request, external_id: str, _: None = require_role("staff", "hub_coordinator")):
    """Lista pendências de um aluno."""
    return list_pendencies(external_id=external_id)


@router.patch("/{external_id}/pendency/{pendency_id}", auth=HttpBearer())
def patch_resolve_pendency(request, external_id: str, pendency_id: int, _: None = require_role("staff", "hub_coordinator")):
    """Resolve (marca como concluída) uma pendência."""
    return resolve_pendency(external_id=external_id, pendency_id=pendency_id)


@router.post("/{external_id}/document-analysis-approved", auth=HttpBearer())
def post_document_analysis_approved(request, external_id: str, _: None = require_role("staff")):
    """Aprova análise documental do aluno (status 30 → 32).
    Se houver pendências não resolvidas, o status vai para 31 (PENDENCY)."""
    return approve_document_analysis(external_id=external_id)


@router.patch("/{external_id}/document", auth=HttpBearer())
def patch_final_documents(request, external_id: str, historic: UploadedFile = File(...), certificate: UploadedFile = File(...), _: None = require_role("staff")):
    """Upload de documentos finais (histórico + certificado) após aprovação da secretaria.
    Dispara status 32 → 33 (disponível no polo)."""
    return upload_final_documents(external_id=external_id, historic=historic, certificate=certificate)
```

> [!info] Fluxo documental
> O fluxo de análise documental é detalhado em [[07-enrollment#7.7 Fluxo staff: análise documental]]. Os status 30 → 31 → 32 → 33 são gerenciados exclusivamente pelo staff.

### 12.2.3 Finance (`staff/api/finance.py`)

```python
from ninja import Router

from auth.dependencies import require_role
from finance.tools.list_all_commissions import list_all_commissions
from finance.tools.list_all_payments import list_all_payments
from finance.tools.schedule_qrcode import schedule_qrcode

router = Router(tags=["staff-finance"])


@router.get("/commissions", auth=HttpBearer())
def get_commissions(request, profile_external_id: str = None, _: None = require_role("staff")):
    """Lista todas as comissões do sistema.
    Opcionalmente filtra por profile_external_id."""
    return list_all_commissions(profile_external_id=profile_external_id)


@router.get("/payments", auth=HttpBearer())
def get_payments(request, profile_external_id: str = None, _: None = require_role("staff")):
    """Lista todos os pagamentos do sistema."""
    return list_all_payments(profile_external_id=profile_external_id)


@router.post("/schedule-qrcode", auth=HttpBearer())
def post_schedule_qrcode(request, payload: ScheduleQrcodeIn, _: None = require_role("staff")):
    """Agenda QR code manualmente (casos fora dos fluxos automáticos)."""
    return schedule_qrcode(payload.qrcode, context={"coordinator_profile": request.user.profile})
```

> [!info] Comissões e pagamentos
> Os endpoints de promoter e hub_coordinator em [[08-finance]] filtram por dono. O staff vê **tudo**. O `schedule_qrcode` manual é para cenários de retentativa ou correção.

### 12.2.4 Candidates (`staff/api/candidates.py`)

```python
from ninja import Router

from auth.dependencies import require_role
from candidate.tools.list_all import list_all as candidate_list_all
from candidate.tools.approve import approve as candidate_approve
from candidate.tools.reject import reject as candidate_reject

router = Router(tags=["staff-candidates"])


@router.get("/candidates", auth=HttpBearer())
def list_candidates(request, status: int = None, _: None = require_role("staff")):
    """Lista todos os candidates do sistema. Opcionalmente filtra por status."""
    return candidate_list_all(status=status)


@router.post("/candidates/{external_id}/approval", auth=HttpBearer())
def post_approval(request, external_id: str, payload: ApprovalIn, _: None = require_role("staff")):
    """Aprova ou rejeita um candidate.
    payload.approved=True → approve → candidate vira promoter (status 100)
    payload.approved=False → reject → status 999 terminal"""
    if payload.approved:
        return candidate_approve(external_id=external_id, approved_by=request.user.profile)
    return candidate_reject(external_id=external_id, reason=payload.reason, rejected_by=request.user.profile)
```

> [!warning] Candidate rejeitado é terminal
> Status 999 não tem caminho de volta. Para nova tentativa, staff precisa intervir manualmente (resetar status ou criar novo Candidate). Ver [[11-candidate#Status]] — decisão consciente para evitar loops de aprovação.

### 12.2.5 Profiles (`staff/api/profiles.py`)

```python
from ninja import Router

from auth.dependencies import require_role
from data.tools.list_all_profiles import list_all_profiles
from data.tools.get_profile import get_profile
from data.tools.patch_profile import patch_profile

router = Router(tags=["staff-profiles"])


@router.get("/profiles", auth=HttpBearer())
def list_profiles(request, _: None = require_role("staff")):
    """Lista todos os profiles do sistema (visão irrestrita)."""
    return list_all_profiles()


@router.get("/profiles/{external_id}", auth=HttpBearer())
def get_profile_by_id(request, external_id: str, _: None = require_role("staff")):
    """Busca profile por external_id."""
    return get_profile(external_id=external_id)


@router.patch("/profiles/{external_id}", auth=HttpBearer())
def patch_profile_by_id(request, external_id: str, payload: ProfilePatchIn, _: None = require_role("staff", "hub_coordinator")):
    """Atualiza campos do profile (nome, email, etc)."""
    return patch_profile(external_id=external_id, data=payload.dict(exclude_unset=True))
```

### 12.2.6 Config (`staff/api/config.py`)

```python
from ninja import Router

from auth.dependencies import require_role
from staff.services.system_config import get_all, set_config, get_history

router = Router(tags=["staff-config"])


@router.get("/config", auth=HttpBearer())
def get_all_config(request, _: None = require_role("staff")):
    """Retorna todas as SystemConfig (key/value) + fallback do .env."""
    return get_all()


@router.patch("/config", auth=HttpBearer())
def patch_config(request, payload: dict, _: None = require_role("staff")):
    """Atualiza uma ou mais chaves de configuração.
    Exemplo: {"PROMOTER_COMMISSION_VALUE": "60.00"}

    Fluxo:
    1. Salva em SystemConfig (key/value)
    2. Gera SystemConfigLog (auditoria)
    3. Invalida cache Redis da chave
    4. Próxima leitura config.X retorna novo valor
    """
    results = {}
    for key, value in payload.items():
        results[key] = set_config(key=key, value=str(value), changed_by=request.user.profile)
    return results


@router.get("/config/{key}/history", auth=HttpBearer())
def get_config_history(request, key: str, _: None = require_role("staff")):
    """Histórico de alterações de uma chave de configuração."""
    return get_history(key=key)
```

> [!info] Fluxo completo de config
> O comportamento do PATCH é detalhado em [[01-config-e-env#Comportamento do PATCH /v1/staff/config/]]. Em resumo: salva no banco → gera audit log → invalida Redis → `config.X` lê do banco no próximo acesso.

---

## 12.3 Services

### `staff/services/system_config.py`

O service é a camada de acesso a `SystemConfig`. É chamado tanto pela API staff quanto pelo `config.py` (raiz do projeto) via `__getattr__`.

```python
import redis

from django.core.cache import cache

from staff.models import SystemConfig, SystemConfigLog

REDIS_CONFIG_PREFIX = "staff:config:"
TTL = 600  # 10 minutos


def get_config(key: str) -> str | None:
    """Busca valor de config. Cache Redis primeiro, depois banco, depois None.
    Chamado por config.py __getattr__ como fallback."""
    cached = cache.get(f"{REDIS_CONFIG_PREFIX}{key}")
    if cached is not None:
        return cached

    try:
        obj = SystemConfig.objects.get(key=key)
        cache.set(f"{REDIS_CONFIG_PREFIX}{key}", obj.value, timeout=TTL)
        return obj.value
    except SystemConfig.DoesNotExist:
        return None


def set_config(key: str, value: str, changed_by=None) -> dict:
    """Persiste ou atualiza uma config, gera audit log, invalida cache."""
    obj, created = SystemConfig.objects.update_or_create(
        key=key,
        defaults={"value": value, "updated_by": changed_by},
    )

    # Audit log
    previous = obj.value if not created else None
    SystemConfigLog.objects.create(
        config=obj,
        previous_value=previous,
        new_value=value,
        changed_by=changed_by,
    )

    # Invalida cache
    invalidate_cache(key)

    return {"key": key, "value": value, "created": created}


def get_all() -> dict:
    """Retorna todas as configs do banco como dict {key: value}."""
    return {c.key: c.value for c in SystemConfig.objects.all()}


def get_history(key: str) -> list[dict]:
    """Histórico de alterações de uma chave."""
    try:
        config = SystemConfig.objects.get(key=key)
    except SystemConfig.DoesNotExist:
        return []

    return [
        {
            "previous_value": log.previous_value,
            "new_value": log.new_value,
            "changed_by": str(log.changed_by) if log.changed_by else None,
            "changed_at": log.created_at.isoformat(),
        }
        for log in config.logs.all()
    ]


def invalidate_cache(key: str):
    """Invalida cache Redis de uma chave específica."""
    cache.delete(f"{REDIS_CONFIG_PREFIX}{key}")
```

> [!info] Cache Redis com TTL
> Cache de 10 minutos. Na leitura: Redis → banco → None. Na escrita: banco → audit log → invalida Redis. A próxima leitura faz cold read no banco e reaquece o cache. Isso garante consistência sem stale reads.

---

## 12.4 Middleware

### `staff/middleware/audit.py`

Middleware de auditoria que intercepta **toda chamada HTTP** e **todo comando CLI**.

```python
import time
import threading

from staff.models import ApiLog, CommandLog

_thread_local = threading.local()


class AuditMiddleware:
    """Middleware Django que loga toda requisição HTTP em ApiLog.
    Também injeta request.user no thread-local para BaseModel.created_by/updated_by."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Injeta user no thread-local
        _thread_local.user = getattr(request, "user", None)

        start = time.monotonic()

        response = self.get_response(request)

        duration_ms = int((time.monotonic() - start) * 1000)

        # Loga a chamada
        ApiLog.objects.create(
            user=request.user.profile if hasattr(request.user, "profile") else None,
            method=request.method,
            path=request.path,
            status_code=response.status_code,
            ip=request.META.get("REMOTE_ADDR"),
            duration_ms=duration_ms,
        )

        return response


def get_current_user():
    """Retorna o usuário da request atual (thread-local).
    Usado por BaseModel.save() para preencher created_by/updated_by."""
    return getattr(_thread_local, "user", None)


# ---- CLI audit ----

def log_command(user, command: str, args: str = ""):
    """Chamado por management commands para logar execução.
    Exemplo de uso no handle() do comando:
        from staff.middleware.audit import log_command
        log_command(user=profile, command="fix_lead_status", args="--external-id abc123")
    """
    CommandLog.objects.create(
        user=user,
        command=command,
        args=args,
    )
```

> [!info] Thread-local para auditoria
> `_thread_local.user` é populado no início de cada request e consumido por `BaseModel.save()` (em [[03-core]]) para preencher automaticamente `created_by`/`updated_by`. Comandos CLI não passam pelo middleware HTTP — nesse caso o `created_by` é passado explicitamente ou fica `None`.

---

## 12.5 Variáveis de config usadas

| Variável | Uso |
|:---------|:----|
| `FRONTEND_URL_LOGIN_STAFF` | Redirect do magic link ([[04-auth]]) |
| `REDIS_URL` | Cache do SystemConfig ([[01-config-e-env#Técnicas (somente .env, imutáveis em runtime)]]) |
| `DATABASE_URL` | Persistência dos models staff |

Além disso, o app staff **expõe edição** de todas as variáveis de negócio listadas em [[01-config-e-env#Negócio (editáveis via staff em runtime)]].

---

## 12.6 Notify — alertas operacionais

> [!info] Notificações do staff
> Diferente dos outros apps (que notificam o usuário sobre seu progresso), as notificações do staff
> são **alertas operacionais**: falhas de sistema, relatórios periódicos e eventos que exigem atenção
> da administração. Quem dispara essas notificações são os `services/` e `signals/` dos apps de origem
> — o staff só define os templates.

### `staff/notify/asaas_pix_setup_failed.md`

Disparado por `candidate/services/asaas_setup.py` quando o cadastro PIX falha no status 20.
O candidate fica travado nesse status até retentativa manual.

```
--tts
⚠️ FALHA: Cadastro PIX — candidate {{ candidate_external_id }}

Não foi possível cadastrar a chave PIX de {{ candidate_name }} (CPF: {{ candidate_cpf }}) no Asaas.
Erro: {{ error_message }}

O candidate permanece no status 20 aguardando retentativa.
Acesse: {{ admin_url }}
```

### `staff/notify/payment_processing_failed.md`

Disparado por `finance/signals/post_save_payment.py` quando `process_payment` falha.

```
⚠️ FALHA: Pagamento — Payment {{ payment_id }}

Não foi possível processar o pagamento de R$ {{ valor }} para {{ profile_name }} ({{ profile_external_id }}).
Commissions: {{ commissions }}

Erro: {{ error_message }}
O payment permanece com status "failed" até retentativa.
```

### `staff/notify/friday_closing_error.md`

Disparado por `finance/tasks/friday_closing.py` quando bonus ou closing encontram erro.

```
⚠️ FALHA: Friday Closing — {{ date }}

O fechamento semanal encontrou erros:

{% if bonus_error %}
Bonus: {{ bonus_error }}
{% endif %}

{% if closing_error %}
Closing: {{ closing_error }}
{% endif %}

O fechamento parcial foi executado para os profiles sem erro.
Os profiles com erro precisam de intervenção manual.
```

### `staff/notify/webhook_validation_failed.md`

Disparado pelos endpoints de webhook (`lead/webhook/infinitepay`, `finance/webhook/asaas`) quando a validação de assinatura/origem falha.

```
⚠️ Webhook inválido recebido — {{ source }}

Um webhook do {{ source }} foi recebido mas a validação falhou.
IP de origem: {{ remote_ip }}
Timestamp: {{ timestamp }}
Motivo: {{ reason }}

Payload (truncado): {{ payload_preview }}

Possível tentativa de fraude ou configuração incorreta do webhook no {{ source }}.
Verifique a assinatura/configuração no painel do {{ source }}.
```

### `staff/notify/weekly_report.md`

Relatório semanal automático — disparado pelo `friday_closing` ao final do processamento.

```
📊 Relatório Semanal — {{ week_start }} a {{ week_end }}

**Leads:** {{ new_leads }} novos | {{ leads_paid }} pagaram
**Matrículas:** {{ new_students }} novos alunos
**Conclusões:** {{ graduations }} concluíram
**Comissões:** R$ {{ total_commissions }} em {{ total_payments }} pagamentos
**Promoters ativos:** {{ active_promoters }}
**Meta batida:** {{ bonus_recipients }} promoters ({{ total_bonus }} em bonus)

Resumo completo: {{ admin_url }}
```

---

> [!warning] Pontos abertos
> - ⚠️ **GET `/config` deve expor quais vars?** Apenas as de negócio (categoria editável) ou todas? Por segurança, expor apenas as de negócio. As técnicas (`JWT_SIGNING_KEY`, etc.) são imutáveis e não devem ser acessíveis via API.
> - ⚠️ **Paginação nos endpoints de listagem.** Leads, candidates, profiles podem ter milhares de registros. Todos os GETs de lista precisam de paginação (a definir: cursor-based ou offset/limit).
> - ⚠️ **CLI audit — como autenticar?** Comandos Django não têm request HTTP. O `log_command` recebe `user` como parâmetro explícito. Commands que exigem auditoria devem exigir `--staff-uuid` e resolver o Profile.
> - ⚠️ **Staff pode ver dados sensíveis.** O modelo de segurança atual confia que staff tem acesso total. Precisa de log específico para acessos a dados sensíveis (CPF, documentos)? Ver [[05-data]].
> - ⚠️ **Rotação de cache.** TTL de 10 min no Redis é suficiente para config? Se uma config for alterada e o cache não for invalidado corretamente, o sistema opera com valor stale por até 10 min. O PATCH invalida explicitamente — mas e se o Redis falhar? Fallback: cold read no banco.

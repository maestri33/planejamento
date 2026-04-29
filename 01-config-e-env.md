---
tags:
  - supletico
  - config
aliases:
  - Config e .env
created: 2026-04-29
---

# 01 — Config e .env

## Filosofia

- O `.env` é a **fonte** das configurações no boot
- O módulo `config.py` (na raiz do projeto) é a **interface única** de acesso
- Nenhum código fora de `config.py` chama `os.getenv()` diretamente
- Variáveis de **negócio** podem ser editadas em runtime via `staff` (GET/PATCH `/v1/staff/config/`) — nesse caso ficam num model `staff.models.SystemConfig` e o `config.py` consulta esse model com fallback pro `.env`
- Variáveis **técnicas/infra** são imutáveis em runtime (só `.env`)

> [!info] Interface única de config
> `config.X` é a única forma de acessar qualquer configuração no sistema. O módulo [[00-arquitetura-geral]] define que `config.py` fica na raiz do projeto e resolve variáveis técnicas (direto do `.env`) e de negócio (via [[12-staff|SystemConfig]] com fallback).

## Categorização

### Técnicas (somente `.env`, imutáveis em runtime)

| Variável | Uso |
| :---- | :---- |
| `DJANGO_SECRET_KEY` | Chave do Django |
| `DJANGO_DEBUG` | Modo debug |
| `DATABASE_URL` | URL do PostgreSQL |
| `REDIS_URL` | URL do Redis (Celery + cache) |
| `ALLOWED_HOSTS` | Hosts permitidos |
| `EXTERNAL_URL` | URL pública do backend (prefixo de `/media/...`) |
| `NOTIFY_API_URL` | URL do serviço de notificações (`10.10.10.119:8000`) |
| `ASAAS_API_URL` | URL do Asaas (`10.10.10.121`) |
| `INFINITEPAY_API_URL` | URL do InfinitePay (`10.10.10.120:8000`) |
| `JWT_SIGNING_KEY` | Chave de assinatura JWT |
| `JWT_ACCESS_TTL` | TTL do access token |
| `JWT_REFRESH_TTL` | TTL do refresh token |

> [!warning] Segredos e tokens
> Todas as variáveis acima que contêm `_SECRET_KEY`, `_SIGNING_KEY` ou credenciais são **sensíveis**. Jamais devem ser commitadas em repositório, logadas ou expostas em respostas de API. O `.env.example` (abaixo) usa `change-me` como placeholder.

### Negócio (editáveis via `staff` em runtime)

| Variável | Uso | Origem |
| :---- | :---- | :---- |
| `FRONTEND_URL_LOGIN_LEAD` | Redirect quando role bate | lead |
| `FRONTEND_URL_LOGIN_STUDENT` | Redirect pós-matrícula | lead |
| `FRONTEND_URL_LOGIN_PROMOTER` | Redirect quando role bate | auth |
| `FRONTEND_URL_LOGIN_CANDIDATE` | Redirect quando role bate | auth |
| `FRONTEND_URL_LOGIN_HUB` | Redirect coordenador hub | auth |
| `FRONTEND_URL_LOGIN_STAFF` | Redirect staff | auth |
| `FRONTEND_CANDIDATE_URL` | Base do frontend de candidate (link `/veteran/<id>`) | enrollment |
| `DEFAULT_PROMOTER` | external_id do promoter padrão (quando lead chega sem ref) | lead |
| `DEFAULT_HUB` | external_id do hub padrão (quando candidate chega sem ref) | candidate |
| `PROMOTER_COMMISSION_VALUE` | Valor fixo da comissão de captação (R$) | finance |
| `HUB_COORDINATOR_COMMISSION_VALUE` | Valor fixo da comissão de conclusão (R$) | finance |
| `WEEKLY_TARGET_QUANTITY` | Qtd de captações para meta semanal | finance |
| `WEEKLY_TARGET_BONUS_VALUE` | Valor do bonus quando bate meta | finance |
| `ENROLLMENT_FIRST_PART_FEE` | Valor da taxa "first part" (matrícula externa) | enrollment |
| `ENROLLMENT_SECOND_PART_FEE` | Valor da taxa "second part" | enrollment |
| `LEAD_CHECKOUT_PRICE` | Valor cobrado no checkout do Lead (taxa de matrícula) | lead |
| `LEAD_CHECKOUT_DESCRIPTION` | Descrição do produto no InfinitePay | lead |

> [!note] Runtime editável via Staff
> Essas variáveis podem ser alteradas em produção via `PATCH /v1/staff/config/` sem redeploy. Os novos valores são persistidos em [[12-staff|SystemConfig]] e cacheados em Redis. Veja seção [[#Comportamento do PATCH /v1/staff/config/]] abaixo.

## Módulo `config.py` (proposta)

```python
# config.py (raiz do projeto)

from functools import lru_cache

from environ import Env

env = Env()

Env.read_env()

# Categoria: TÉCNICAS (somente env)

SECRET_KEY = env("DJANGO_SECRET_KEY")

DEBUG = env.bool("DJANGO_DEBUG", default=False)

DATABASE_URL = env("DATABASE_URL")

REDIS_URL = env("REDIS_URL")

EXTERNAL_URL = env("EXTERNAL_URL")

NOTIFY_API_URL = env("NOTIFY_API_URL")

ASAAS_API_URL = env("ASAAS_API_URL")

INFINITEPAY_API_URL = env("INFINITEPAY_API_URL")

# ...

# Categoria: NEGÓCIO (sobreescritas por staff.SystemConfig)

def __getattr__(name: str):
    """
    Fallback dinâmico: para variáveis de negócio,
    consulta staff.SystemConfig primeiro, depois .env.
    Cacheado em Redis com invalidação no PATCH do staff.
    """
    from staff.services.system_config import get_config
    return get_config(name) or env(name)
```

> [!warning] Módulo `config.py` é sensível
> Esse módulo carrega `DJANGO_SECRET_KEY`, `JWT_SIGNING_KEY` e outras credenciais em memória. Nunca exponha o objeto `config` em logs, responses de API ou tracing. Use logging sanitizado se precisar debugar valores.

## `.env.example` (template para gerar)

```bash
# ===== Técnicas =====

DJANGO_SECRET_KEY=change-me
DJANGO_DEBUG=False
DATABASE_URL=postgres://user:***@localhost:5432/supletivo
REDIS_URL=redis://localhost:6379/0
ALLOWED_HOSTS=localhost,127.0.0.1
EXTERNAL_URL=http://localhost:8000
NOTIFY_API_URL=http://10.10.10.119:8000
ASAAS_API_URL=http://10.10.10.121
INFINITEPAY_API_URL=http://10.10.10.120:8000
JWT_SIGNING_KEY=change-me
JWT_ACCESS_TTL=900
JWT_REFRESH_TTL=2592000

# ===== Negócio (também editáveis via /v1/staff/config/) =====

FRONTEND_URL_LOGIN_LEAD=https://app.exemplo.com/lead/login
FRONTEND_URL_LOGIN_STUDENT=https://app.exemplo.com/student/login
FRONTEND_URL_LOGIN_PROMOTER=https://app.exemplo.com/promoter/login
FRONTEND_URL_LOGIN_CANDIDATE=https://app.exemplo.com/candidate/login
FRONTEND_URL_LOGIN_HUB=https://app.exemplo.com/hub/login
FRONTEND_URL_LOGIN_STAFF=https://app.exemplo.com/staff/login
FRONTEND_CANDIDATE_URL=https://app.exemplo.com/candidate
DEFAULT_PROMOTER=00000000-0000-0000-0000-000000000000
DEFAULT_HUB=00000000-0000-0000-0000-000000000000
PROMOTER_COMMISSION_VALUE=50.00
HUB_COORDINATOR_COMMISSION_VALUE=100.00
WEEKLY_TARGET_QUANTITY=10
WEEKLY_TARGET_BONUS_VALUE=200.00
ENROLLMENT_FIRST_PART_FEE=150.00
ENROLLMENT_SECOND_PART_FEE=150.00
LEAD_CHECKOUT_PRICE=49.90
LEAD_CHECKOUT_DESCRIPTION="Taxa de matrícula - Supletivo"
```

> [!warning] Placeholders de segurança
> Os valores `change-me` e `00000000-0000-...` são placeholders explícitos. Em produção, **todos** os segredos (`DJANGO_SECRET_KEY`, `JWT_SIGNING_KEY`) devem ser substituídos por valores fortes gerados aleatoriamente.

## Comportamento do PATCH /v1/staff/config/

1. Staff envia `{"PROMOTER_COMMISSION_VALUE": "60.00"}`
2. Salva em `staff.models.SystemConfig` (key/value + audit log)
3. Invalida cache Redis da chave
4. Próxima leitura `config.PROMOTER_COMMISSION_VALUE` → busca em [[12-staff|SystemConfig]] → retorna novo valor

> [!info] Fluxo de cache
> O cache Redis é invalidado no momento do PATCH. A próxima chamada a `config.X` (via `__getattr__`) faz cold read no banco, depois reaquece o cache. Isso garante consistência e evita stale reads.

## Auditoria

Toda alteração de config gera `staff.models.SystemConfigLog`:

- quem alterou, quando, valor anterior, valor novo

Essa tabela cresce — precisa expor visualização via API staff (GET histórico).

---
tags:
  - supletico
  - arquitetura
aliases:
  - Arquitetura Geral
created: 2026-04-29
status: especificação
---

# 00 — Arquitetura Geral

## Stack

- **Django** + **django-ninja** ([https://django-ninja.dev/](https://django-ninja.dev/))
- **django-ninja-jwt** ([https://github.com/eadwinCode/django-ninja-jwt](https://github.com/eadwinCode/django-ninja-jwt)) — JWT
- **Celery** + **Redis** (Redis instalado no host do servidor)
- **SQLite** (em desenvolvimento)
- **PostgreSQL** (em produção)

> [!note] Stack alvo de implementação
> A especificação original foi escrita em sintaxe Django. A implementação real será em **FastAPI** com serviços independentes, seguindo o mesmo padrão dos serviços já existentes ([[02-integracoes-externas|Notify, Asaas, InfinitePay]]).

## Princípios fundamentais

1. **Toda API tem validação e normalização** — diretório `validators/` e `normalizers/` por app
2. **GET de arquivo retorna URL absoluta**: `{config.EXTERNAL_URL}/media/{caminho}`
3. **Acesso a config sempre via `config.X`** — nunca `os.getenv()` espalhado. O `.env` é a fonte, mas o módulo `config.py` é a interface
4. **Notificações declarativas em `.md`** com variáveis Jinja-like e diretivas (`--tts`, `--media`)
5. **Padrão de módulo replicado** em todos os apps (ver seção abaixo)
6. **Máquina de estados** — `status` numérico em dezenas, cada dezena = uma etapa
7. **RBAC por endpoint** — mesmo path com comportamento por role
8. **Logs centralizados em `staff`** — toda chamada (API ou CLI) é auditada
9. **Polimorfismo no `auth`** — `register`, `check`, `otp` são funções reutilizadas por outros apps com parâmetro `role`
10. **Apps não mexem no DB de outros apps** — usam `tools/` (público) e `services/` (privado)

## Apps

| App | Responsabilidade | Documento |
|:----|:-----------------|:----------|
| `core` | Base técnica compartilhada (clients de integrações, mixins, BaseModel) | [[03-core]] |
| `auth` | Usuário, login, OTP, JWT, roles (lógica polimórfica) | [[04-auth]] |
| `data` | Perfil, CPF, endereço, documentos, dados educacionais | [[05-data]] |
| `lead` | Pré-matrícula | [[06-lead]] |
| `enrollment` | Aluno/matrícula | [[07-enrollment]] |
| `finance` | Pagamentos e comissões | [[08-finance]] |
| `hub` | Polo físico | *pendente* |
| `promoter` | Vendedor | *pendente* |
| `candidate` | Futuro vendedor | [[11-candidate]] |
| `staff` | Administração + GET/PATCH em config + logs centralizados | *pendente* |

## Estrutura padrão de cada app

> [!warning] Apenas o que faz sentido
> Nem todo app terá todos os diretórios — só o que faz sentido para o domínio.

```text
<app>/
├── models/                 # 1 arquivo por model
│   └── __init__.py         # exporta todos
├── api/                    # endpoints (django-ninja)
│   ├── __init__.py         # router principal do app
│   ├── schemas.py          # Pydantic In/Out/Patch (ou pasta schemas/ se grande)
│   ├── public.py           # endpoints públicos
│   ├── authenticated.py    # endpoints autenticados (qualquer role)
│   └── by_role/            # endpoints protegidos por role
│       ├── lead.py
│       ├── promoter.py
│       └── staff.py
├── validators/             # 1 arquivo por campo (cpf.py, name.py, email.py)
├── normalizers/            # idem
├── signals/                # post_save etc. — 1 arquivo por signal
├── tools/                  # FUNÇÕES PÚBLICAS — chamáveis por outros apps
├── services/               # FUNÇÕES INTERNAS — só este app mexe no próprio DB
├── tasks/                  # Celery (agendamentos)
├── notify/                 # templates .md
└── management/commands/    # CLI (PATCH de logado exige só uuid)
```

### Apps implementados seguindo este padrão

- [[03-core]] — BaseModel, clients, validators, helpers
- [[04-auth]] — register, check, otp polimórficos
- [[05-data]] — Profile, Address, documentos
- [[06-lead]] — pré-matrícula com máquina de estados
- [[07-enrollment]] — matrícula com 15 status
- [[08-finance]] — comissões, pagamentos, fechamento semanal
- [[11-candidate]] — pipeline promoter com 10 status

### Convenção `tools/` vs `services/`

> [!tip] Regra de ouro
> - **`tools/`** — interface pública do app. Exemplo: `enrollment.tools.new_student` é chamado pelo `lead`.
> - **`services/`** — função interna que mexe no DB do próprio app. Outros apps **não importam** de `services/`.
>
> Outros apps que precisam alterar dados deste app chamam `tools/`, que internamente chama `services/`.

## Estrutura padrão de `api/` (django-ninja)

Cada arquivo dentro de `api/` define um `Router`. O `__init__.py` do `api/` agrega tudo:

```python
# <app>/api/__init__.py
from ninja import Router

from .public import router as public_router
from .authenticated import router as authenticated_router
from .by_role.staff import router as staff_router

router = Router()
router.add_router("", public_router)
router.add_router("", authenticated_router)
router.add_router("", staff_router)
```

E no `urls.py` do projeto:

```python
# project/api.py
from ninja import NinjaAPI

api = NinjaAPI()
api.add_router("/v1/lead/", "lead.api.router")
api.add_router("/v1/enrollment/", "enrollment.api.router")
# ...
```

## Versionamento

Todos endpoints sob `/api/v1/<app>/...`. Versão futura `v2` sem quebrar `v1`.

## Auditoria e BaseModel

Todo model herda de `core.models.base.BaseModel`:

```python
class BaseModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    created_by = models.ForeignKey(User, null=True, ...)
    updated_by = models.ForeignKey(User, null=True, ...)

    class Meta:
        abstract = True
```

Toda chamada de API e CLI é logada em `staff.models.api_log` / `staff.models.command_log` via middleware (ver *pendente*).

## Convenção de nomes de arquivos `.md` de notify

- Sempre `snake_case.md`
- Primeira linha pode conter diretivas: `--tts` (envia como áudio), `--media: {{ url }}` (envia mídia anexa)
- Variáveis com Jinja: `{{ first_name }}`, `{{ checkout_url }}`
- O parser fica em `core/notify/render.py`

> [!tip] Exemplo de template de notificação
> ```text
> --tts
> Olá {{ first_name }}, seu pagamento de R$ {{ amount }} foi confirmado!
> ```

## Convenção de `external_id`

- Todo `data.Profile` tem um `external_id` UUID v4 imutável
- Esse UUID é a chave usada em todas as integrações externas e em URLs públicas
- **Nunca expor `id` (PK numérica) em APIs**

## Roles disponíveis

| Role | Descrição | Transição |
|:-----|:----------|:----------|
| `lead` | Pré-aluno | → vira `student` ao concluir matrícula |
| `student` | Aluno matriculado | — |
| `candidate` | Pré-promoter | → vira `promoter` ao ser aprovado |
| `promoter` | Vendedor ativo | — |
| `hub_coordinator` | Coordenador de polo | Sempre é também `promoter` |
| `staff` | Administração | — |

> [!note] Múltiplas roles
> Um usuário pode ter **múltiplas roles simultaneamente**. JWT carrega `roles: list[str]`.

## Checagem rápida do que vem nos próximos docs

| Doc | Conteúdo |
|:----|:---------|
| [[01-config-e-env]] | Consolidação de todas variáveis `config.*` |
| [[02-integracoes-externas]] | Notify, Asaas, InfinitePay |
| [[03-core]] | Detalhamento dos clients e helpers |
| [[04-auth]] | Auth, OTP, JWT, roles |
| [[05-data]] | Profile, Address, documentos |
| [[06-lead]] | Pré-matrícula |
| [[07-enrollment]] | Aluno/matrícula + máquina de estados |
| [[08-finance]] | Pagamentos e comissões |
| [[11-candidate]] | Pipeline promoter |
| *pendente* | `09-hub.md` — Polo físico |
| *pendente* | `10-promoter.md` — Vendedor ativo |
| *pendente* | `12-staff.md` — Admin, config runtime, logs |
| *pendente* | `13-pendencias-e-decisoes.md` — Questões abertas |
| *pendente* | `enrollment-state-machine.md` — Diagrama da máquina de estados (entregue separado) |

> [!note] Referência
> Consulte o [[Supletico MOC]] para visão geral do projeto, diagramas e status dos subsistemas.

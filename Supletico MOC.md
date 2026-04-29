---
tags:
  - supletico
  - moc
  - arquitetura
  - fastapi
created: 2026-04-29
status: especificação
---

# Supletivo.net — MOC

> [!abstract] Stack alvo
> **FastAPI** microserviços + **PostgreSQL** + **Redis** + **Celery**
> 
> A especificação original foi escrita em sintaxe Django. A implementação real será em **FastAPI** com serviços independentes, seguindo o mesmo padrão dos serviços já existentes ([[02-integracoes-externas|Notify, Asaas, InfinitePay]]).

---

## Estrutura do sistema

```mermaid
graph TD
    A[Frontend] --> B[Auth Service]
    B --> C[Data Service]
    C --> D[Lead Service]
    D --> E[Enrollment Service]
    E --> F[Finance Service]
    B --> G[Hub Service]
    B --> H[Promoter Service]
    B --> I[Candidate Service]
    B --> J[Staff Service]
    
    F --> K[Asaas API]
    D --> L[InfinitePay API]
    B --> M[Notify API]
```

---

## Apps / Módulos

| # | Módulo | Documento | Responsabilidade |
|:--|:-------|:----------|:-----------------|
| 00 | Arquitetura | [[00-arquitetura-geral]] | Visão geral, stack, princípios |
| 01 | Config | [[01-config-e-env]] | `.env`, `config.py`, SystemConfig |
| 02 | Integrações | [[02-integracoes-externas]] | Notify, Asaas, InfinitePay |
| 03 | Core | [[03-core]] | BaseModel, clients, validators, helpers |
| 04 | Auth | [[04-auth]] | OTP, JWT, roles, register/check polimórfico |
| 05 | Data | [[05-data]] | Profile, Address, Educational, documentos |
| 06 | Lead | [[06-lead]] | Pré-matrícula (status 1→2→3→100) |
| 07 | Enrollment | [[07-enrollment]] | Aluno/matrícula (15 status) |
| 08 | Finance | [[08-finance]] | Comissões, pagamentos, friday closing |
| 09 | Hub | *pendente* | Polo físico |
| 10 | Promoter | *pendente* | Vendedor ativo |
| 11 | Candidate | [[11-candidate]] | Pipeline promoter (10 status) |
| 12 | Staff | *pendente* | Admin, config runtime, logs |
| 13 | Pendências | *pendente* | Decisões em aberto |

---

## Fluxos principais

### Jornada do aluno
```mermaid
sequenceDiagram
    participant L as Lead
    participant P as Promoter
    participant IP as InfinitePay
    participant S as Student
    participant H as Hub
    participant E as Exam
    
    L->>L: 1. Cadastro ([[06-lead]])
    L->>L: 2. Dados + endereço
    L->>IP: 3. Pagamento matrícula
    IP-->>L: Webhook confirmado
    L->>S: 100. Vira Student ([[07-enrollment]])
    S->>S: 1-3. Dados pessoais + educacionais
    S->>H: 4. Matrícula externa (hub paga fees)
    S->>S: 10-15. Documentos
    S->>E: 20-21. Prova
    S->>S: 30-33. Análise + secretaria
    S->>S: 100. Concluído 🎓
```

### Jornada do promoter
```mermaid
sequenceDiagram
    participant C as Candidate
    participant A as Asaas
    participant P as Promoter
    participant F as Finance
    
    C->>C: 1-3. Dados pessoais ([[11-candidate]])
    C->>C: 10-12. Documentos + selfie
    C->>A: 20. Cadastro PIX
    C->>C: 30. Aguardando aprovação
    C->>P: 100. Aprovado → Promoter
    P->>P: Captura leads
    F->>F: Comissão por captação ([[08-finance]])
    F->>F: Bonus semanal (meta batida)
```

### Fluxo financeiro semanal
```mermaid
graph LR
    A[Eventos da semana] --> B[Commissions criadas]
    B --> C{Sexta 18h}
    C --> D[Process Bonus]
    D --> E[Cash Register Closing]
    E --> F[1 Payment por Profile]
    F --> G[Asaas - PIX]
    G --> H[Webhook confirma]
    H --> I[Notifica pago]
```

---

## Conceitos transversais

- **[[00-arquitetura-geral#external-id|external_id]]** — UUID v4 imutável, chave pública para tudo
- **[[00-arquitetura-geral#roles-disponiveis|Roles]]** — lead, student, candidate, promoter, hub_coordinator, staff
- **[[01-config-e-env#filosofia|Config]]** — `config.X` como interface única, `.env` como fonte
- **[[00-arquitetura-geral#convencao-tools-vs-services|tools vs services]]** — `tools/` público, `services/` privado
- **[[00-arquitetura-geral#estrutura-padrao-de-api-django-ninja|API por role]]** — `public.py`, `authenticated.py`, `by_role/`
- **[[00-arquitetura-geral#convencao-de-nomes-de-arquivos-md-de-notify|Notificações declarativas]]** — `.md` com Jinja + diretivas `--tts`, `--media`

---

## Status dos subsistemas

| Serviço | IP | Status | Doc |
|:--------|:---|:------|:----|
| Notify | `10.10.10.119:8000` | ✅ Em produção | [[02-integracoes-externas#notify]] |
| InfinitePay | `10.10.10.120:8000` | ✅ Em produção | [[02-integracoes-externas#infinitepay]] |
| Asaas | `10.10.10.121` | ✅ Em produção | [[02-integracoes-externas#asaas]] |
| Auth | `10.10.10.122` | 🔨 Especificado | [[04-auth]] |
| Data | — | 🔨 Especificado | [[05-data]] |
| Lead | — | 🔨 Especificado | [[06-lead]] |
| Enrollment | — | 🔨 Especificado | [[07-enrollment]] |
| Finance | — | 🔨 Especificado | [[08-finance]] |
| Candidate | — | 🔨 Especificado | [[11-candidate]] |
| Hub | — | ❌ Pendente | — |
| Promoter | — | ❌ Pendente | — |
| Staff | — | ❌ Pendente | — |

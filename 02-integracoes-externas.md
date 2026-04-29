---
tags:
  - supletico
  - integracoes
aliases:
  - Integrações Externas
created: 2026-04-29
status: especificação
---

# 02 — Integrações Externas

Tudo que sai do nosso backend para serviços de terceiros. Encapsulado em [[03-core|core/<serviço>/client.py]] — nenhum app fala direto com essas URLs.

---

## 2.1 Notify — `http://10.10.10.119:8000`

**Documentação:** [http://10.10.10.119:8000/docs](http://10.10.10.119:8000/docs)

> [!warning] Validar quando construir
> A documentação do Notify está acessível, mas a estrutura exata do retorno de `/recipients/check` precisa ser confirmada contra a implementação real.

### Encapsulamento: `core/notify/`

```
core/notify/
├── client.py          # cliente HTTP cru
├── render.py          # parser dos .md (extrai diretivas + renderiza Jinja)
└── service.py         # API pública: send(), check(), new_recipient()
```

### Endpoints consumidos

| Método/Path | Função no client | Quando usar |
|:------------|:-----------------|:------------|
| `POST /api/v1/notifications` | `client.send_notification(...)` | Enviar texto/áudio/mídia para external_id |
| `GET /v1/recipients/check?q=<numero>` | `client.check_number(numero)` | Verificar se número existe / é WhatsApp válido |
| `POST /api/v1/recipients` | `client.new_recipient(...)` | Criar recipient (número + external_id) |

### Comportamento de `service.send()`

```python
core.notify.service.send(
    external_id: str | None = None,   # se None, pega do user logado (request.user.profile.external_id)
    template_path: str | None = None, # ex: "lead/notify/payment.md"
    text: str | None = None,          # alternativa: texto cru
    context: dict | None = None,      # variáveis Jinja
    is_tts: bool = False,             # se template tem `--tts` na 1ª linha, vira True automático
    media_url: str | None = None      # se template tem `--media: {{ url }}`, extrai daqui
) -> dict
```

### Parser de `.md` (`core/notify/render.py`)

Lê a primeira linha do arquivo. Diretivas suportadas:

| Diretiva | Efeito |
|:---------|:-------|
| `--tts` | `is_tts = True` (envia como áudio) |
| `--media: {{ url }}` | extrai `media_url` da context |

Variáveis no corpo: `{{ first_name }}`, `{{ checkout_url }}`, etc.

### Comportamento de `service.check(numero)`

```json
{
  "external_id": "uuid-or-null",
  "whatsapp_valid": true|false
}
```

Lógica:

- Se backend Notify retorna `external_id` → existe usuário com esse número
- Se não retorna → testa `whatsapp_valid` (true = pode registrar; false = número inválido)

---

## 2.2 InfinitePay — `http://10.10.10.120:8000`

**Documentação:** [http://10.10.10.120:8000/docs](http://10.10.10.120:8000/docs)

> [!warning] Validar quando construir
> Payload exato do webhook e método de autenticação precisam ser confirmados na documentação do InfinitePay.

### Encapsulamento: `core/infinitepay/`

```
core/infinitepay/
├── client.py
├── bootstrap.py       # PATCH /config/ no boot do app
└── service.py
```

### Bootstrap no boot do app

Ao iniciar (signal de ready), executa `PATCH /config/` enviando:

```json
{
  "handle": "<config.INFINITEPAY_HANDLE>",
  "price": "<config.LEAD_CHECKOUT_PRICE>",
  "description": "<config.LEAD_CHECKOUT_DESCRIPTION>",
  "redirect_url": "<config.EXTERNAL_URL>/api/v1/lead/webhook/infinitepay/",
  "backend_webhook": "<config.INFINITEPAY_BACKEND_WEBHOOK>"
}
```

Só envia os campos que existem em `config`. Os outros ficam intactos no InfinitePay.

> [!warning] Adicionar ao `.env.example`
> Chaves pendentes: `INFINITEPAY_HANDLE`, `INFINITEPAY_BACKEND_WEBHOOK`

### Funções públicas

| Função | Endpoint InfinitePay | Uso |
|:-------|:---------------------|:----|
| `client.create(profile_data)` | `POST /checkout/` | Cria checkout para um lead |
| `client.check(external_id)` | `GET /checkout/<external_id>` | Retorna `{checkout_url, receipt_url}` |

### Webhook recebido

`POST /api/v1/lead/webhook/infinitepay/` — ver [[06-lead]]

Payload esperado (a confirmar com docs):

```json
{ "external_id": "...", "is_paid": true }
```

Handler chama `lead.tools.paid.py` → atualiza `Lead.paid=True` → signal post_save dispara `update_status` (3→100).

---

## 2.3 Asaas — `http://10.10.10.121`

**Documentação:** [http://10.10.10.121/](http://10.10.10.121/)

> [!warning] Validar quando construir
> Payload exato do webhook, formato de `qrcode` (PIX copia-cola) e método de autenticação precisam ser confirmados na documentação do Asaas.

### Encapsulamento: `core/asaas/`

```
core/asaas/
├── client.py
└── service.py
```

### Funções públicas

| Função | Endpoint Asaas | Uso |
|:-------|:---------------|:----|
| `client.cadastra_pix_key(profile)` | `POST /pixkey` | Cadastra chave PIX (CPF) do promoter/coord |
| `client.pay(amount, description, payment_id, pixkey_external_id)` | `POST /payment` | Paga commission via PIX |
| `client.pay_scheduled(...)` | `POST /payment/scheduled` | Agenda pagamento |
| `client.pay_qrcode(qrcode, ...)` | `POST /payment/qrcode` | Paga via "PIX cola" (QR copia-cola) |
| `client.pay_qrcode_scheduled(...)` | `POST /payment/qrcode/scheduled` | Agenda pagamento por QR Code |
| `client.analyze_qrcode(qrcode)` | `POST /payment/qrcode/analyze` | Valida QR Code antes de pagar/agendar |

### Quando `cadastra_pix_key` é chamado?

- Promoter aprovado (candidate → promoter): cadastra com CPF do profile
- Coordenador de hub criado: idem
- Sem isso, não recebe comissão

### Webhook recebido

`POST /api/v1/finance/webhook/asaas/` — ver [[08-finance]]

Confirma pagamento de commission:

```json
{ "payment_id": "uuid", "paid": true }
```

Handler chama `finance.tools.update_payment.py` → atualiza `Payment.paid=True` → signal dispara `payment_paid.md` e atualiza commissions relacionadas para `paid=True`.

---

## 2.4 Resumo de webhooks expostos pelo nosso backend

| Webhook | Origem | Disparador |
|:--------|:-------|:-----------|
| `POST /api/v1/lead/webhook/infinitepay/` | InfinitePay | Lead pagou taxa de matrícula |
| `POST /api/v1/finance/webhook/asaas/` | Asaas | Comissão paga ao promoter/coordenador. E quando qrcode é pago |

> [!warning] Ambos públicos (sem JWT) mas com validação de assinatura/origem — a definir

---

## 2.5 Itens a validar

> [!warning] Validar quando chegar a hora

| Item | Onde validar |
|:-----|:-------------|
| Estrutura exata do retorno do `/recipients/check` | [http://10.10.10.119:8000/docs](http://10.10.10.119:8000/docs) |
| Payload exato do webhook InfinitePay | [http://10.10.10.120:8000/docs](http://10.10.10.120:8000/docs) |
| Payload exato do webhook Asaas | [http://10.10.10.121/](http://10.10.10.121/) |
| Formato de `qrcode` (PIX copia-cola string) | [http://10.10.10.121/](http://10.10.10.121/) |
| Como autenticar requests no Asaas (token?) | [http://10.10.10.121/](http://10.10.10.121/) |
| Como autenticar requests no InfinitePay | [http://10.10.10.120:8000/docs](http://10.10.10.120:8000/docs) |
| Como autenticar requests no Notify | [http://10.10.10.119:8000/docs](http://10.10.10.119:8000/docs) |
| Validação de assinatura nos webhooks | docs de cada serviço |

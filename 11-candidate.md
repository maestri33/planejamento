---
tags:
  - supletico
  - candidate
aliases:
  - Candidate
created: 2026-04-29
status: especificação
---

# 11 — Candidate

Futuro promoter. Mesma filosofia do Lead, com mais etapas de coleta (porque promoter precisa ter dados completos para receber comissão via PIX).

> [!info] Conexões
> - [[06-lead]] — mesma filosofia de API (check, register, status, update)
> - [[08-finance]] — cadastro PIX no Asaas (status 20) + comissões futuras
> - [[10-promoter]] — vira promoter ao ser aprovado (status 100)
> - [[07-enrollment]] — veteran: aluno concluído recebe convite
> - [[04-auth]] — add_role, change_role, OTP
> - [[05-data]] — Profile, documentos (RG/CNH)
> - [[01-config-e-env]] — DEFAULT_HUB, FRONTEND_URL_LOGIN_PROMOTER

Status: 1 → 2 → 3 → 10 → 11 → 12 → 20 → 30 → 100

## Status

| Status | Significado | Ação esperada |
| :---- | :---- | :---- |
| 1 | Recém registrado | Candidate preenche nome \+ email |
| 2 | Dados básicos preenchidos | Candidate preenche profile completo |
| 3 | Profile completo | Candidate preenche endereço (sem comprovante) |
| 10 | Endereço preenchido | Candidate envia comprovante de endereço |
| 11 | Comprovante recebido | Candidate envia RG ou CNH (frente \+ verso) |
| 12 | Documento ID recebido | Candidate envia selfie (assinatura do contrato) |
| 20 | Selfie recebida — cadastrando PIX no Asaas | Sistema chama `[[03-core|core.asaas.service.ensure_pix_key]]` |
| 30 | Aguardando aprovação do hub | Hub coordinator aprova ou rejeita |
| 100 | Aprovado — virou promoter | Redirect para FRONTEND\_URL\_LOGIN\_PROMOTER |
| 999 | Rejeitado — terminal | Sem path de volta (apenas registro) |

Dezenas seguem mesma filosofia do enrollment para consistência mental: 1-3 dados pessoais, 10-12 documentos, 20 integração externa, 30 aprovação, 100 done.

**Importante:** candidate rejeitado **não vira promoter**, **não recebe comissão**, e a role `candidate` permanece (status 999 é terminal). Caso a pessoa queira tentar de novo, staff precisa intervir.

---

## Origem do candidate — duas formas

1. **Cadastro direto** — pessoa entra no frontend e se cadastra. Hub vai pelo `[[01-config-e-env|DEFAULT_HUB]]`.  
2. **Convite veteran** — aluno concluiu o supletivo (status 100\) e recebeu o link `[[01-config-e-env|FRONTEND_CANDIDATE_URL]]/veteran/<external_id_do_aluno>`. Ao clicar e iniciar o fluxo:  
   - A role `candidate` é **adicionada** ao perfil dele (mantém role `student`)  
   - O hub do candidate é **herdado automaticamente** do hub onde ele estudou  
   - Não é uma role nova "veteran" — veteran é só o **status** semântico de um student que concluiu  
   - O `Candidate.veteran=True` marca que esse cadastro veio pelo convite (apenas rastreabilidade)

---

## Estrutura

candidate/

├── models/

│   ├── \_\_init\_\_.py

│   └── candidate.py

├── api/

│   ├── \_\_init\_\_.py

│   ├── schemas.py

│   ├── public.py            \# check, register, register\_veteran

│   ├── authenticated.py     \# status, update (role=candidate)

│   └── by\_role/

│       ├── hub.py           \# list candidates do hub, aprovar/rejeitar

│       └── staff.py         \# list todos, aprovar/rejeitar

├── tools/

│   ├── \_\_init\_\_.py

│   ├── update\_status.py

│   ├── approve.py           \# candidate → promoter

│   ├── reject.py

│   └── start\_from\_veteran.py \# cria Candidate a partir de aluno concluído

├── services/

│   ├── \_\_init\_\_.py

│   ├── candidate\_validators.py

│   └── asaas\_setup.py       \# wraps finance.services.asaas\_pix\_setup (já idempotente)

├── validators/

│   ├── \_\_init\_\_.py

│   └── hub.py

├── normalizers/

│   ├── \_\_init\_\_.py

│   └── hub.py               \# se vazio, usa config.DEFAULT\_HUB

├── signals/

│   ├── \_\_init\_\_.py

│   └── post\_save\_candidate.py

├── notify/

│   ├── new\_candidate.md

│   ├── basic\_data\_done.md

│   ├── profile\_complete.md

│   ├── address\_done.md

│   ├── address\_proof\_done.md

│   ├── id\_document\_done.md

│   ├── selfie\_done.md

│   ├── awaiting\_approval.md

│   ├── approved.md

│   └── rejected.md

└── apps.py

---

## 11.1 Models

### `candidate/models/candidate.py`

from django.db import models

from core.models.base import BaseModel, ExternalIdMixin

class Candidate(BaseModel, ExternalIdMixin):

    STATUS\_BASIC \= 1

    STATUS\_PROFILE\_FULL \= 2

    STATUS\_ADDRESS \= 3

    STATUS\_ADDRESS\_PROOF \= 10

    STATUS\_ID\_DOCUMENT \= 11

    STATUS\_SELFIE \= 12

    STATUS\_ASAAS\_REGISTRATION \= 20

    STATUS\_AWAITING\_APPROVAL \= 30

    STATUS\_APPROVED \= 100

    STATUS\_REJECTED \= 999

    STATUS\_CHOICES \= \[

        (STATUS\_BASIC, "Aguardando dados básicos"),

        (STATUS\_PROFILE\_FULL, "Aguardando profile completo"),

        (STATUS\_ADDRESS, "Aguardando endereço"),

        (STATUS\_ADDRESS\_PROOF, "Aguardando comprovante de endereço"),

        (STATUS\_ID\_DOCUMENT, "Aguardando RG ou CNH"),

        (STATUS\_SELFIE, "Aguardando selfie"),

        (STATUS\_ASAAS\_REGISTRATION, "Cadastrando chave PIX no Asaas"),

        (STATUS\_AWAITING\_APPROVAL, "Aguardando aprovação do hub"),

        (STATUS\_APPROVED, "Aprovado"),

        (STATUS\_REJECTED, "Rejeitado (terminal)"),

    \]

    DOCUMENT\_RG \= "rg"

    DOCUMENT\_CNH \= "cnh"

    DOCUMENT\_CHOICES \= \[(DOCUMENT\_RG, "RG"), (DOCUMENT\_CNH, "CNH")\]

    profile \= models.OneToOneField("data.Profile", on\_delete=models.CASCADE, related\_name="candidate")

    hub \= models.ForeignKey("hub.Hub", on\_delete=models.PROTECT, related\_name="candidates")

    status \= models.IntegerField(choices=STATUS\_CHOICES, default=STATUS\_BASIC)

    document\_choice \= models.CharField(max\_length=4, choices=DOCUMENT\_CHOICES, blank=True)

    \# Marca se veio por convite veteran (aluno concluído).

    \# Quando True, o hub foi herdado do student.

    \# NÃO é uma role separada — veteran é só status semântico do Student.

    veteran \= models.BooleanField(default=False)

    \# Auditoria de aprovação/rejeição

    approved\_by \= models.ForeignKey(

        "data.Profile",

        null=True, blank=True,

        on\_delete=models.SET\_NULL,

        related\_name="approved\_candidates",

    )

    approved\_at \= models.DateTimeField(null=True, blank=True)

    rejection\_reason \= models.TextField(blank=True)

O `Profile.address` (já existente em `data`) é reusado — endereço não fica em Candidate. Documentos (RG/CNH) também ficam em `data` — Candidate só registra qual o candidate escolheu enviar.

---

## 11.2 Schemas

\# candidate/api/schemas.py

from ninja import Schema

from typing import Optional

class CheckIn(Schema):

    number: str

class CheckOut(Schema):

    external\_id: Optional\[str\] \= None

    role: Optional\[str\] \= None

    first\_name: Optional\[str\] \= None

    whatsapp\_valid: bool

    message: str

    redirect\_to: Optional\[str\] \= None

class RegisterIn(Schema):

    """Cadastro direto (sem ser veteran)."""

    number: str

    cpf: str

    hub: Optional\[str\] \= None       \# external\_id do hub; se None, usa DEFAULT\_HUB

class StatusOut(Schema):

    status: int

    first\_name: Optional\[str\] \= None

    message: str

    instruction: Optional\[str\] \= None

    extra: Optional\[dict\] \= None

    redirect\_url: Optional\[str\] \= None

class UpdateOut(StatusOut):

    pass

class DocumentChoiceIn(Schema):

    document: str   \# "rg" ou "cnh"

class ApprovalIn(Schema):

    approve: bool

    rejection\_reason: Optional\[str\] \= None

class CandidateListOut(Schema):

    external\_id: str

    first\_name: Optional\[str\]

    cpf: str

    status: int

    hub\_external\_id: str

    veteran: bool

---

## 11.3 API

### `candidate/api/public.py`

Espelho do `[[06-lead|lead/api/public.py]]`, com `role="candidate"`. Inclui o endpoint `register_veteran`.

from ninja import Router

from .schemas import CheckIn, CheckOut, RegisterIn, StatusOut

from auth.tools.check import check as auth\_check

from [[04-auth|auth.tools.otp]] import otp as auth\_otp

from auth.tools.register import register as auth\_register

from candidate.normalizers.hub import resolve\_hub

from candidate.tools.start\_from\_veteran import start\_from\_veteran

from core.notify import service as notify\_service

from data.models import Profile

from candidate.models import Candidate

import config

router \= Router(tags=\["candidate-public"\])

@router.get("/check/{number}", response=CheckOut)

def check(request, number: str):

    """

    Mesma lógica do Lead/check, mas para candidate.

    Importante: alguém que já é student pode também ser candidate (multi-role).

    Nesse caso, NÃO redirecionamos — deixamos seguir como candidate.

    """

    result \= auth\_check(number)

    if not result\["external\_id"\]:

        if result\["whatsapp\_valid"\]:

            return CheckOut(

                whatsapp\_valid=True,

                message="Número válido. Pode prosseguir para registro como candidato a promoter.",

            )

        return CheckOut(whatsapp\_valid=False, message="Número de WhatsApp inválido.")

    roles \= result.get("roles", \[\])

    if "candidate" in roles:

        auth\_otp(result\["external\_id"\])

        return CheckOut(

            external\_id=result\["external\_id"\],

            first\_name=result.get("first\_name"),

            role="candidate",

            whatsapp\_valid=True,

            message="Te enviamos um código de acesso no WhatsApp.",

        )

    \# Já tem outra role MAS NÃO candidate — orienta o caminho certo:

    \# \- se for 'student' (potencial veteran), oferece o fluxo veteran

    \# \- se for outra role, redireciona pra o frontend dela

    if "student" in roles:

        return CheckOut(

            external\_id=result\["external\_id"\],

            role="student",

            whatsapp\_valid=True,

            message="Você já é aluno. Para virar promoter, use o link de convite veteran.",

            redirect\_to=f"{config.FRONTEND\_CANDIDATE\_URL}/veteran/{result\['external\_id'\]}",

        )

    primary\_role \= roles\[0\]

    redirect\_url \= getattr(config, f"FRONTEND\_URL\_LOGIN\_{primary\_role.upper()}", None)

    return CheckOut(

        external\_id=result\["external\_id"\],

        role=primary\_role,

        whatsapp\_valid=True,

        message=f"Você já está cadastrado como {primary\_role}.",

        redirect\_to=redirect\_url,

    )

@router.post("/register", response=StatusOut)

def register(request, payload: RegisterIn):

    """

    Cadastro DIRETO de candidate (pessoa que não é nosso aluno).

    Hub: se não vier no payload, usa DEFAULT\_HUB.

    """

    from hub.models import Hub

    hub\_external\_id \= resolve\_hub(payload.hub)

    hub \= Hub.objects.get(external\_id=hub\_external\_id)

    \# auth\_register cria User, Profile, role=candidate, recipient no notify

    result \= auth\_register(numero=payload.number, cpf=payload.cpf, role="candidate")

    profile \= result\["profile"\]

    Candidate.objects.create(

        profile=profile,

        hub=hub,

        veteran=False,

        status=Candidate.STATUS\_BASIC,

    )

    \_notify\_new\_candidate(profile, hub)

    auth\_otp(str(profile.external\_id))

    return StatusOut(

        status=Candidate.STATUS\_BASIC,

        message="Registro criado. Te enviamos um código de acesso no WhatsApp.",

        instruction="Faça login com external\_id \+ OTP, depois preencha nome e e-mail.",

    )

@router.post("/register-veteran/{student\_external\_id}", response=StatusOut)

def register\_veteran(request, student\_external\_id: str):

    """

    Endpoint chamado quando o aluno concluído clica no link de convite.

    \- Adiciona role 'candidate' ao Profile existente (mantém role 'student')

    \- Cria Candidate com hub herdado do Student

    \- Marca veteran=True (rastreabilidade)

    \- Dispara OTP (se a sessão já não estiver ativa)

    NÃO chama auth\_register — o usuário JÁ existe (é student).

    """

    candidate \= start\_from\_veteran(student\_external\_id)

    return StatusOut(

        status=candidate.status,

        message=f"Bem-vindo de volta, {candidate.profile.user.first\_name}\! Vamos completar seu cadastro como promoter.",

        instruction="Faça login com seu external\_id \+ OTP e siga o fluxo.",

    )

def \_notify\_new\_candidate(profile, hub):

    """Notify para o candidate \+ para o coordenador do hub."""

    notify\_service.send(

        external\_id=str(profile.external\_id),

        template\_path="candidate/notify/new\_candidate.md",

        context={"first\_name": profile.user.first\_name or "amigo(a)"},

    )

    notify\_service.send(

        external\_id=str(hub.coordinator.external\_id),

        template\_path="hub/notify/new\_candidate\_to\_hub.md",

        context={

            "candidate\_external\_id": str(profile.external\_id),

            "candidate\_cpf": profile.cpf,

            "hub\_name": hub.name,

        },

    )

### `candidate/api/authenticated.py`

from ninja import Router

from core.auth\_helpers.role\_required import require\_role

from .schemas import StatusOut, UpdateOut, DocumentChoiceIn

from candidate.tools.update\_status import update\_status

from candidate.services.candidate\_validators import can\_advance\_from

from candidate.models import Candidate

import config

router \= Router(tags=\["candidate"\])

@router.get("/status", response=StatusOut, auth=JWTAuth())

def get\_status(request, \_: None \= Depends(require\_role("candidate"))):

    candidate \= request.user.profile.candidate

    return \_build\_status\_response(candidate)

@router.post("/update", response=UpdateOut, auth=JWTAuth())

def post\_update(request, \_: None \= Depends(require\_role("candidate"))):

    candidate \= request.user.profile.candidate

    if can\_advance\_from(candidate):

        update\_status(candidate)

    return \_build\_status\_response(candidate)

@router.post("/document-choice", auth=JWTAuth())

def choose\_document(request, payload: DocumentChoiceIn, \_: None \= Depends(require\_role("candidate"))):

    """Candidate escolhe enviar RG ou CNH. Salva escolha em Candidate.document\_choice."""

    candidate \= request.user.profile.candidate

    candidate.document\_choice \= payload.document

    candidate.save(update\_fields=\["document\_choice"\])

    return {"ok": True, "document": payload.document}

def \_build\_status\_response(candidate) \-\> StatusOut:

    profile \= candidate.profile

    first\_name \= profile.user.first\_name or None

    if candidate.status \== Candidate.STATUS\_BASIC:

        return StatusOut(

            status=1,

            message="Precisamos dos seus dados básicos: nome e e-mail.",

            instruction="PATCH /v1/data/profile (com 'name') e PATCH /v1/data/email",

        )

    if candidate.status \== Candidate.STATUS\_PROFILE\_FULL:

        return StatusOut(

            status=2, first\_name=first\_name,

            message=f"{first\_name}, agora preenche seus dados completos.",

            instruction="PATCH /v1/data/profile (mãe, pai, sangue, civil, sexo)",

        )

    if candidate.status \== Candidate.STATUS\_ADDRESS:

        return StatusOut(

            status=3, first\_name=first\_name,

            message="Preenche seu endereço.",

            instruction="PATCH /v1/data/address (somente dados, sem comprovante)",

        )

    if candidate.status \== Candidate.STATUS\_ADDRESS\_PROOF:

        return StatusOut(

            status=10, first\_name=first\_name,

            message="Anexa o comprovante de endereço.",

            instruction="PATCH /v1/data/address (campo proof)",

        )

    if candidate.status \== Candidate.STATUS\_ID\_DOCUMENT:

        doc \= candidate.document\_choice

        if not doc:

            return StatusOut(

                status=11, first\_name=first\_name,

                message="Envia frente e verso do seu RG ou CNH.",

                instruction="POST /v1/candidate/document-choice (escolhe rg ou cnh) depois PATCH /v1/data/\<doc\>",

            )

        return StatusOut(

            status=11, first\_name=first\_name,

            message=f"Envia frente e verso do seu {doc.upper()}.",

            instruction=f"PATCH /v1/data/{doc}",

        )

    if candidate.status \== Candidate.STATUS\_SELFIE:

        return StatusOut(

            status=12, first\_name=first\_name,

            message="Envia uma selfie. Ela serve como sua assinatura no contrato de promoter.",

            instruction="PATCH /v1/data/profile (campo self\_image)",

        )

    if candidate.status \== Candidate.STATUS\_ASAAS\_REGISTRATION:

        return StatusOut(

            status=20, first\_name=first\_name,

            message="Estamos cadastrando sua chave PIX. Aguarde um momento.",

        )

    if candidate.status \== Candidate.STATUS\_AWAITING\_APPROVAL:

        return StatusOut(

            status=30, first\_name=first\_name,

            message=f"Tudo certo, {first\_name}\! Seu cadastro está em análise pelo polo.",

        )

    if candidate.status \== Candidate.STATUS\_APPROVED:

        return StatusOut(

            status=100, first\_name=first\_name,

            message=f"Parabéns {first\_name}\! Você é oficialmente um promoter agora.",

            instruction="Redirecione para a área do promoter",

            redirect\_url=config.FRONTEND\_URL\_LOGIN\_PROMOTER,

        )

    if candidate.status \== Candidate.STATUS\_REJECTED:

        return StatusOut(

            status=999, first\_name=first\_name,

            message="Seu cadastro não foi aprovado neste momento. Entre em contato com o polo se quiser conversar a respeito.",

            extra={"rejection\_reason": candidate.rejection\_reason},

        )

    return StatusOut(status=candidate.status, message="Status desconhecido.")

### `candidate/api/by_role/hub.py`

Coordenador aprova ou rejeita candidates do **próprio hub**.

from ninja import Router

from core.auth\_helpers.role\_required import require\_role

from candidate.models import Candidate

from candidate.tools.approve import approve\_candidate

from candidate.tools.reject import reject\_candidate

from .schemas import ApprovalIn, CandidateListOut

router \= Router(tags=\["candidate-hub"\])

@router.get("/", response=list\[CandidateListOut\], auth=JWTAuth())

def list\_my\_hub\_candidates(request, \_: None \= Depends(require\_role("hub\_coordinator"))):

    coordinator\_profile \= request.user.profile

    hub \= coordinator\_profile.coordinated\_hub

    return \[\_serialize(c) for c in Candidate.objects.filter(hub=hub)\]

@router.get("/{status}", response=list\[CandidateListOut\], auth=JWTAuth())

def list\_my\_hub\_candidates\_by\_status(request, status: int, \_: None \= Depends(require\_role("hub\_coordinator"))):

    coordinator\_profile \= request.user.profile

    hub \= coordinator\_profile.coordinated\_hub

    return \[\_serialize(c) for c in Candidate.objects.filter(hub=hub, status=status)\]

@router.get("/{external\_id}", response=CandidateListOut, auth=JWTAuth())

def get\_candidate\_detail(request, external\_id: str, \_: None \= Depends(require\_role("hub\_coordinator"))):

    coordinator\_profile \= request.user.profile

    hub \= coordinator\_profile.coordinated\_hub

    return \_serialize(Candidate.objects.get(hub=hub, profile\_\_external\_id=external\_id))

@router.post("/{external\_id}/approval", auth=JWTAuth())

def approve\_or\_reject(request, external\_id: str, payload: ApprovalIn, \_: None \= Depends(require\_role("hub\_coordinator"))):

    """

    Aprova ou rejeita candidate.

    Status 30 → 100 (aprovado) ou 30 → 999 (rejeitado, terminal).

    Candidate rejeitado NÃO vira promoter, NÃO recebe comissão e NÃO pode

    voltar a tentar. Status 999 é definitivo.

    """

    coordinator\_profile \= request.user.profile

    candidate \= Candidate.objects.get(

        hub=coordinator\_profile.coordinated\_hub,

        profile\_\_external\_id=external\_id,

    )

    if candidate.status \!= Candidate.STATUS\_AWAITING\_APPROVAL:

        raise HttpError(400, "Candidate não está aguardando aprovação.")

    if payload.approve:

        approve\_candidate(candidate, approver\_profile=coordinator\_profile)

    else:

        reject\_candidate(candidate, approver\_profile=coordinator\_profile, reason=payload.rejection\_reason or "")

    return {"ok": True, "new\_status": candidate.status}

### `candidate/api/by_role/staff.py`

Mesma lógica, sem filtro de hub (vê todos), pode aprovar/rejeitar qualquer um.

---

## 11.4 Tools

### `candidate/tools/start_from_veteran.py`

Aluno concluído clicou no link. **Adiciona role candidate** ao perfil existente, **herda hub** do Student.

from django.db import transaction

from auth.tools.add\_role import add\_role

from [[04-auth|auth.tools.otp]] import otp as auth\_otp

from candidate.models import Candidate

from core.notify import service as notify\_service

from data.models import Profile

@transaction.atomic

def start\_from\_veteran(student\_external\_id: str) \-\> Candidate:

    """

    Inicia fluxo de candidate para um aluno concluído (veteran).

    \- Adiciona role 'candidate' ao Profile (mantém role 'student')

    \- Cria Candidate com hub herdado do Student

    \- Marca veteran=True

    \- Notifica candidate e coordenador do hub

    \- Dispara OTP

    Veteran NÃO é uma role — é apenas o Candidate.veteran=True dizendo

    "esse cadastro veio de aluno concluído". A role da pessoa é multi:

    fica student E candidate ao mesmo tempo.

    """

    profile \= Profile.objects.get(external\_id=student\_external\_id)

    \# Sanity check: profile precisa ter Student concluído

    student \= getattr(profile, "student", None)

    if student is None or student.status \!= student.STATUS\_DONE:

        raise ValueError("Apenas alunos com status concluído podem virar candidate via veteran.")

    \# Idempotência: se já existe candidate, retorna sem recriar

    existing \= getattr(profile, "candidate", None)

    if existing:

        return existing

    \# Adiciona role candidate (mantém student)

    add\_role(profile, "candidate")

    \# Cria Candidate com hub herdado do Student

    candidate \= Candidate.objects.create(

        profile=profile,

        hub=student.hub,

        veteran=True,

        status=Candidate.STATUS\_BASIC,

    )

    \# Notify candidate (mensagem específica de veteran)

    notify\_service.send(

        external\_id=str(profile.external\_id),

        template\_path="candidate/notify/new\_candidate\_veteran.md",

        context={"first\_name": profile.user.first\_name},

    )

    \# Notify coordenador do hub

    notify\_service.send(

        external\_id=str(student.hub.coordinator.external\_id),

        template\_path="hub/notify/new\_candidate\_to\_hub.md",

        context={

            "candidate\_external\_id": str(profile.external\_id),

            "candidate\_cpf": profile.cpf,

            "hub\_name": student.hub.name,

            "is\_veteran": True,

        },

    )

    \# Dispara OTP pra ele logar

    auth\_otp(str(profile.external\_id))

    return candidate

### `candidate/tools/update_status.py`

Implementa as transições. Cascata 12 → 20 → 30 (cadastra Asaas e segue).

from django.db import transaction

from core.notify import service as notify\_service

from candidate.models import Candidate

from candidate.services.candidate\_validators import can\_advance\_from

@transaction.atomic

def update\_status(candidate: Candidate) \-\> int:

    if not can\_advance\_from(candidate):

        return candidate.status

    transitions \= {

        Candidate.STATUS\_BASIC:           Candidate.STATUS\_PROFILE\_FULL,        \# 1 → 2

        Candidate.STATUS\_PROFILE\_FULL:    Candidate.STATUS\_ADDRESS,              \# 2 → 3

        Candidate.STATUS\_ADDRESS:         Candidate.STATUS\_ADDRESS\_PROOF,        \# 3 → 10

        Candidate.STATUS\_ADDRESS\_PROOF:   Candidate.STATUS\_ID\_DOCUMENT,          \# 10 → 11

        Candidate.STATUS\_ID\_DOCUMENT:     Candidate.STATUS\_SELFIE,               \# 11 → 12

        Candidate.STATUS\_SELFIE:          Candidate.STATUS\_ASAAS\_REGISTRATION,   \# 12 → 20

    }

    next\_status \= transitions.get(candidate.status)

    if next\_status is None:

        return candidate.status

    candidate.status \= next\_status

    candidate.save(update\_fields=\["status"\])

    \_notify\_step\_done(candidate)

    \# Cascata: status 20 dispara cadastro Asaas e tenta seguir pra 30

    if next\_status \== Candidate.STATUS\_ASAAS\_REGISTRATION:

        from candidate.services.asaas\_setup import register\_pix\_key\_for\_candidate

        try:

            register\_pix\_key\_for\_candidate(candidate)  \# idempotente — vide service

            candidate.status \= Candidate.STATUS\_AWAITING\_APPROVAL

            candidate.save(update\_fields=\["status"\])

            \_notify\_awaiting\_approval(candidate)

        except Exception as e:

            import logging

            logging.getLogger(\_\_name\_\_).error(

                "Falha ao cadastrar PIX no Asaas para candidate %s: %s",

                candidate.profile.external\_id, e,

            )

            \# mantém em 20 — staff pode retentar manualmente

    return candidate.status

def \_notify\_step\_done(candidate):

    profile \= candidate.profile

    first\_name \= profile.user.first\_name or "amigo(a)"

    template\_map \= {

        Candidate.STATUS\_PROFILE\_FULL:       "candidate/notify/basic\_data\_done.md",

        Candidate.STATUS\_ADDRESS:            "candidate/notify/profile\_complete.md",

        Candidate.STATUS\_ADDRESS\_PROOF:      "candidate/notify/address\_done.md",

        Candidate.STATUS\_ID\_DOCUMENT:        "candidate/notify/address\_proof\_done.md",

        Candidate.STATUS\_SELFIE:             "candidate/notify/id\_document\_done.md",

        Candidate.STATUS\_ASAAS\_REGISTRATION: "candidate/notify/selfie\_done.md",

    }

    template \= template\_map.get(candidate.status)

    if template:

        notify\_service.send(

            external\_id=str(profile.external\_id),

            template\_path=template,

            context={"first\_name": first\_name},

        )

def \_notify\_awaiting\_approval(candidate):

    profile \= candidate.profile

    notify\_service.send(

        external\_id=str(profile.external\_id),

        template\_path="candidate/notify/awaiting\_approval.md",

        context={"first\_name": profile.user.first\_name, "hub\_name": candidate.hub.name},

    )

    notify\_service.send(

        external\_id=str(candidate.hub.coordinator.external\_id),

        template\_path="hub/notify/candidate\_ready\_for\_approval.md",

        context={

            "candidate\_external\_id": str(profile.external\_id),

            "candidate\_name": profile.name,

            "candidate\_cpf": profile.cpf,

        },

    )

### `candidate/tools/approve.py`

Aprova candidate → vira promoter.

from django.db import transaction

from django.utils import timezone

from core.notify import service as notify\_service

from auth.tools.change\_role import change\_role

from candidate.models import Candidate

import config

@transaction.atomic

def approve\_candidate(candidate: Candidate, approver\_profile) \-\> Candidate:

    """

    Aprova candidate:

    \- status 30 → 100

    \- troca role candidate → promoter (se candidate é veteran, mantém student também)

    \- cria promoter.Promoter vinculado ao mesmo hub

    """

    candidate.status \= Candidate.STATUS\_APPROVED

    candidate.approved\_by \= approver\_profile

    candidate.approved\_at \= timezone.now()

    candidate.save(update\_fields=\["status", "approved\_by", "approved\_at"\])

    \# Troca candidate por promoter.

    \# Se for veteran, change\_role mantém as outras roles (student) intactas.

    change\_role(candidate.profile, from\_role="candidate", to\_role="promoter")

    \# Cria registro de promoter

    from promoter.tools.new\_promoter import new\_promoter

    new\_promoter(profile=candidate.profile, hub=candidate.hub)

    notify\_service.send(

        external\_id=str(candidate.profile.external\_id),

        template\_path="candidate/notify/approved.md",

        context={

            "first\_name": candidate.profile.user.first\_name,

            "frontend\_url": config.FRONTEND\_URL\_LOGIN\_PROMOTER,

        },

    )

    return candidate

### `candidate/tools/reject.py`

Rejeita candidate. **Status terminal**.

from django.utils import timezone

from core.notify import service as notify\_service

from candidate.models import Candidate

def reject\_candidate(candidate: Candidate, approver\_profile, reason: str) \-\> Candidate:

    """

    Rejeita candidate. Status 999 é TERMINAL:

    \- candidate NÃO vira promoter

    \- role 'candidate' permanece (não some)

    \- candidate NÃO pode voltar a tentar (precisa staff intervir)

    """

    candidate.status \= Candidate.STATUS\_REJECTED

    candidate.approved\_by \= approver\_profile

    candidate.approved\_at \= timezone.now()

    candidate.rejection\_reason \= reason

    candidate.save()

    notify\_service.send(

        external\_id=str(candidate.profile.external\_id),

        template\_path="candidate/notify/rejected.md",

        context={

            "first\_name": candidate.profile.user.first\_name,

            "reason": reason,

        },

    )

    return candidate

---

## 11.5 Services

### `candidate/services/candidate_validators.py`

from candidate.models import Candidate

def can\_advance\_from(candidate) \-\> bool:

    profile \= candidate.profile

    if candidate.status \== Candidate.STATUS\_BASIC:

        return bool(profile.name and profile.user.email)

    if candidate.status \== Candidate.STATUS\_PROFILE\_FULL:

        return bool(

            profile.mother\_name

            and profile.father\_name

            and profile.blood\_type

            and profile.civil\_status

            and profile.sexo

        )

    if candidate.status \== Candidate.STATUS\_ADDRESS:

        addr \= profile.address

        if not addr:

            return False

        return all(\[addr.rua, addr.numero, addr.cep, addr.cidade, addr.estado, addr.bairro\])

    if candidate.status \== Candidate.STATUS\_ADDRESS\_PROOF:

        return bool(profile.address and profile.address.proof)

    if candidate.status \== Candidate.STATUS\_ID\_DOCUMENT:

        if candidate.document\_choice \== Candidate.DOCUMENT\_RG:

            rg \= getattr(profile, "rg", None)

            return bool(rg and rg.front and rg.back)

        if candidate.document\_choice \== Candidate.DOCUMENT\_CNH:

            cnh \= getattr(profile, "cnh", None)

            return bool(cnh and cnh.front and cnh.back)

        return False  \# ainda não escolheu

    if candidate.status \== Candidate.STATUS\_SELFIE:

        return bool(profile.self\_image)

    \# 20, 30, 100, 999 não avançam por update\_status (automáticos ou manuais)

    return False

### `candidate/services/asaas_setup.py`

Wrapper sobre `[[08-finance|finance.services.asaas_pix_setup]]` (que já chama `[[03-core|core.asaas.service.ensure_pix_key]]`, que é idempotente — verifica antes se já tem cadastro).

from finance.services.asaas\_pix\_setup import setup\_pix\_key\_for\_profile

def register\_pix\_key\_for\_candidate(candidate) \-\> dict:

    """

    Garante que candidate.profile tem chave PIX cadastrada no Asaas.

    Idempotente — \`core.asaas.service.ensure\_pix\_key\` verifica se já existe

    antes de tentar criar, então pode ser chamado múltiplas vezes sem efeito colateral.

    """

    return setup\_pix\_key\_for\_profile(candidate.profile)

---

## 11.6 Normalizers

### `candidate/normalizers/hub.py`

import config

def resolve\_hub(hub\_external\_id: str | None) \-\> str:

    """Se vazio, retorna DEFAULT\_HUB."""

    return hub\_external\_id or config.DEFAULT\_HUB

Note: o resolve\_hub só é usado no cadastro **direto**. No fluxo veteran, o hub é herdado direto do Student (em `tools/start_from_veteran.py`).

---

## 11.7 Signals

### `candidate/signals/post_save_candidate.py`

\# Placeholder. Toda lógica de transição está em candidate.tools.update\_status,

\# chamado explicitamente pelo POST /update.

---

## 11.8 Templates de notify

### `candidate/notify/new_candidate.md`

Olá\! Você acaba de iniciar seu cadastro como candidato a promoter da Supletivo.net. Esse é um caminho que pode mudar a sua história financeira: você ajuda outras pessoas a concluir os estudos e ganha por isso. Continue seu cadastro e avance para o próximo passo.

### `candidate/notify/new_candidate_veteran.md`

{{ first\_name }}, que alegria te ver dando esse próximo passo\! Você concluiu seu supletivo e agora vai poder ajudar outras pessoas a fazerem o mesmo, ganhando comissão por cada uma. Vamos completar alguns dados que ainda não temos no seu cadastro pra você começar como promoter o quanto antes.

### `candidate/notify/basic_data_done.md`

Boa, {{ first\_name }}\! Recebemos seus dados básicos. Próximo passo: complete os dados do seu perfil (nome da mãe, pai, tipo sanguíneo, estado civil).

### `candidate/notify/profile_complete.md`

{{ first\_name }}, perfil completo\! Agora preencha o seu endereço para seguirmos.

### `candidate/notify/address_done.md`

{{ first\_name }}, endereço registrado. Agora envie o comprovante de endereço (conta de luz, água ou similar).

### `candidate/notify/address_proof_done.md`

Comprovante recebido, {{ first\_name }}\! Próximo passo: envie frente e verso do seu RG ou CNH (você escolhe).

### `candidate/notify/id_document_done.md`

Documento recebido, {{ first\_name }}\! Última etapa: envie uma selfie. Ela serve como sua assinatura no contrato de promoter.

### `candidate/notify/selfie_done.md`

{{ first\_name }}, selfie recebida. Estamos cadastrando sua chave PIX automaticamente. Em instantes seu cadastro vai pra análise do polo.

### `candidate/notify/awaiting_approval.md`

Pronto, {{ first\_name }}\! Seu cadastro completo foi enviado para o polo {{ hub\_name }}. O coordenador vai revisar e te avisamos assim que for aprovado.

### `candidate/notify/approved.md`

\--tts

Parabéns, {{ first\_name }}\! Você é oficialmente um promoter da Supletivo.net agora. A partir de agora, cada pessoa que você levar até nós e que pagar a matrícula gera comissão para você. E quando essa pessoa concluir os estudos, você também recebe um bonus de conclusão. Acesse a área do promoter no link que enviamos e comece sua jornada.

### `candidate/notify/rejected.md`

{{ first\_name }}, infelizmente seu cadastro como promoter não foi aprovado neste momento. Motivo: {{ reason }}. Se quiser conversar sobre isso, entre em contato com o polo.

---

## 11.9 Variáveis de config usadas

| Variável | Uso |
| :---- | :---- |
| `[[01-config-e-env|DEFAULT_HUB]]` | normalizers.hub (cadastro direto) |
| `[[01-config-e-env|FRONTEND_URL_LOGIN_CANDIDATE]]` | check (caso D — outras roles) |
| `[[01-config-e-env|FRONTEND_URL_LOGIN_PROMOTER]]` | status response do status 100, approved.md |
| `[[01-config-e-env|FRONTEND_CANDIDATE_URL]]` | check (caso student — sugere link veteran) |
| `[[01-config-e-env|ASAAS_API_URL]]` | indireto via core.asaas.service |

⚠️ Adicionar `[[01-config-e-env|DEFAULT_HUB]]` ao `.env.example` (UUID do hub padrão para candidates sem indicação direta).

---

## 11.10 Pontos abertos

- ⚠️ **Cadastro Asaas idempotente** — `[[03-core|core.asaas.service.ensure_pix_key]]` verifica se já existe antes de criar. Endpoint exato do GET de pix\_key precisa ser confirmado em `[[02-integracoes-externas#asaas|http://10.10.10.121/docs]]`.  
- ⚠️ **Hub padrão por região?** Por enquanto único (`[[01-config-e-env|DEFAULT_HUB]]`). Pode evoluir pra "hub mais próximo do CEP".  
- ⚠️ **Candidate rejeitado pode tentar de novo?** **Não**. Status 999 é terminal. Para nova tentativa, staff precisa criar novo Candidate manualmente (resetar status). Decisão consciente — evita reaprovações em loop.  
- ⚠️ **Veteran é STATUS do Student, não role.** Quando aluno conclui (Student.status=100), o `Candidate.veteran=True` marca a origem do cadastro. A pessoa fica com roles `student` \+ `candidate` (e depois `student` \+ `promoter` se aprovado).


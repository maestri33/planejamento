---
tags:
  - supletico
  - data
aliases:
  - Data
created: 2026-04-29
status: especificação
---

# 05 — Data

Onde fica o `external_id` (UUID principal do usuário) e todos os dados genéricos que qualquer pessoa do sistema (lead, student, promoter, etc) tem.

## Estrutura

```text
data/
├── models/
│   ├── __init__.py
│   ├── profile.py
│   ├── address.py
│   ├── educational.py
│   ├── birth_info.py
│   └── documents/
│       ├── __init__.py
│       ├── rg.py
│       ├── military_document.py
│       ├── cnh.py
│       └── certidao.py
├── api/
│   ├── __init__.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── profile.py
│   │   ├── address.py
│   │   ├── educational.py
│   │   └── documents.py
│   ├── public.py            # GET /cpf/<numero>
│   ├── authenticated.py     # PATCH/GET próprio perfil
├── tools/
│   ├── __init__.py
│   ├── get_cpf.py           # busca CPF → external_id
│   ├── new_profile.py       # cria Profile + cascata
│   ├── update_name.py       # split first/last name
│   ├── list_profiles_by_promoter.py  # Profile.objects.filter(lead__promoter=promoter_profile)
│   ├── list_profiles_by_hub.py       # Profile.objects.filter(student__hub=hub)
│   └── list_all_profiles.py          # Profile.objects.all()
├── services/
│   ├── __init__.py
│   └── document_factory.py  # cria documents conforme regras (sexo masculino → military_document)
├── validators/
│   ├── __init__.py
│   ├── cpf.py
│   ├── name.py
│   ├── email.py
│   ├── phone.py
│   ├── address.py
│   └── cep.py
├── normalizers/
│   ├── __init__.py
│   ├── name.py
│   └── address.py
├── signals/
│   ├── __init__.py
│   ├── post_save_profile.py
│   └── post_save_address.py
├── notify/                  # intencionalmente vazio — data não dispara notificações (só mudanças de estado)
└── apps.py
```

---

## 5.1 Models

### `data/models/profile.py`

```python
import uuid
from django.db import models
from django.contrib.auth import get_user_model
from core.models.base import BaseModel, ExternalIdMixin

User = get_user_model()

def profile_self_path(instance, filename):
    return f"data/{instance.external_id}/self.jpg"

class Profile(BaseModel, ExternalIdMixin):
    SEXO_M = "M"
    SEXO_F = "F"
    SEXO_CHOICES = [(SEXO_M, "Masculino"), (SEXO_F, "Feminino")]

    BLOOD_CHOICES = [
        ("A+", "A+"), ("A-", "A-"),
        ("B+", "B+"), ("B-", "B-"),
        ("O+", "O+"), ("O-", "O-"),
        ("AB+", "AB+"), ("AB-", "AB-"),
    ]

    CIVIL_SOLTEIRO = "solteiro"
    CIVIL_CASADO = "casado"
    CIVIL_VIUVO = "viuvo"
    CIVIL_DIVORCIADO = "divorciado"
    CIVIL_CHOICES = [
        (CIVIL_SOLTEIRO, "Solteiro"),
        (CIVIL_CASADO, "Casado"),
        (CIVIL_VIUVO, "Viúvo"),
        (CIVIL_DIVORCIADO, "Divorciado"),
    ]

    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name="profile")
    address = models.OneToOneField("data.Address", null=True, blank=True, on_delete=models.SET_NULL, related_name="profile")
    self_image = models.ImageField(upload_to=profile_self_path, null=True, blank=True)
    name = models.CharField(max_length=200, blank=True)
    cpf = models.CharField(max_length=11, unique=True, db_index=True)  # apenas dígitos
    sexo = models.CharField(max_length=1, choices=SEXO_CHOICES, blank=True)
    mother_name = models.CharField(max_length=200, blank=True)
    father_name = models.CharField(max_length=200, blank=True)
    blood_type = models.CharField(max_length=3, choices=BLOOD_CHOICES, blank=True)
    civil_status = models.CharField(max_length=15, choices=CIVIL_CHOICES, blank=True)
```

CPF é único e não pode ser vazio. Validação no `validators/cpf.py`.

> [!warning] `external_password` texto cru
> O campo `password` do [[04-auth|User]] é armazenado como texto cru (`external_password`) no modelo `User` do auth — **não** usar `make_password` do Django. A autenticação é delegada ao backend externo (Asaas/InfinitePay).

### `data/models/address.py`

```python
def address_proof_path(instance, filename):
    return f"data/{instance.profile.external_id}/address_proof.{filename.split('.')[-1]}"

class Address(BaseModel):
    ESTADOS_CHOICES = [...]  # 27 estados

    rua = models.CharField(max_length=200)
    numero = models.CharField(max_length=10)  # só dígitos, mas string para "S/N" futuramente
    cep = models.CharField(max_length=8)
    cidade = models.CharField(max_length=100)
    estado = models.CharField(max_length=2, choices=ESTADOS_CHOICES)
    bairro = models.CharField(max_length=100)
    referencia = models.CharField(max_length=200, blank=True)
    proof = models.FileField(upload_to=address_proof_path, null=True, blank=True)
```

Referência é opcional. Comprovante (`proof`) também — o lead/enrollment exige em momentos diferentes.

### `data/models/educational.py`

```python
class Educational(BaseModel):
    NIVEL_FUND_INC = "fund_inc"
    NIVEL_FUND_COMP = "fund_comp"
    NIVEL_MED_INC = "med_inc"
    NIVEL_MED_COMP = "med_comp"
    NIVEL_SUP_INC = "sup_inc"
    NIVEL_SUP_COMP = "sup_comp"
    NIVEL_CHOICES = [
        (NIVEL_FUND_INC, "Fundamental incompleto"),
        (NIVEL_FUND_COMP, "Fundamental completo"),
        (NIVEL_MED_INC, "Médio incompleto"),
        (NIVEL_MED_COMP, "Médio completo"),
        (NIVEL_SUP_INC, "Superior incompleto"),
        (NIVEL_SUP_COMP, "Superior completo"),
    ]

    # Mapeamento "série antiga" → "ano novo" (fundamental)
    ANO_FUND_PRE = "pre"        # pré
    ANO_FUND_1 = "1ano"         # 1 ano
    ANO_FUND_2 = "2ano"         # 1ª série / 2 ano
    ANO_FUND_3 = "3ano"
    ANO_FUND_4 = "4ano"
    ANO_FUND_5 = "5ano"
    ANO_FUND_6 = "6ano"
    ANO_FUND_7 = "7ano"
    ANO_FUND_8 = "8ano"
    ANO_FUND_9 = "9ano"         # 8ª série / 9 ano
    ANO_FUND_CHOICES = [...]

    ANO_MED_1 = "1ano_med"
    ANO_MED_2 = "2ano_med"
    ANO_MED_3 = "3ano_med"
    ANO_MED_CHOICES = [...]

    profile = models.OneToOneField("data.Profile", on_delete=models.CASCADE, related_name="educational")
    nivel = models.CharField(max_length=10, choices=NIVEL_CHOICES, blank=True)
    ultimo_ano_fund = models.CharField(max_length=10, choices=ANO_FUND_CHOICES, blank=True)
    fund_concluido = models.BooleanField(null=True)
    ano_fund = models.IntegerField(null=True, blank=True)  # ano civil de conclusão
    historico_fund = models.FileField(upload_to=lambda i,f: f"data/{i.profile.external_id}/history_of_elementary_education.{f.split('.')[-1]}", null=True, blank=True)
    certificado_fund = models.FileField(upload_to=lambda i,f: f"data/{i.profile.external_id}/certificate_fund.{f.split('.')[-1]}", null=True, blank=True)
    ultimo_ano_med = models.CharField(max_length=15, choices=ANO_MED_CHOICES, blank=True)
    med_concluido = models.BooleanField(null=True)
    historico_med = models.FileField(upload_to=lambda i,f: f"data/{i.profile.external_id}/history_of_high_school.{f.split('.')[-1]}", null=True, blank=True)
```

### `data/models/birth_info.py`

```python
class BirthInfo(BaseModel):
    profile = models.OneToOneField("data.Profile", on_delete=models.CASCADE, related_name="birth_info")
    estado = models.CharField(max_length=2)  # choices estados BR
    cidade = models.CharField(max_length=100)
    data = models.DateField()
```

### `data/models/documents/rg.py`

```python
class RG(BaseModel):
    profile = models.OneToOneField("data.Profile", on_delete=models.CASCADE, related_name="rg")
    numero = models.CharField(max_length=20)
    estado_emissao = models.CharField(max_length=2)  # choices
    orgao_emissao = models.CharField(max_length=20)
    data_emissao = models.DateField(null=True)
    front = models.FileField(upload_to=lambda i,f: f"data/{i.profile.external_id}/rg_front.{f.split('.')[-1]}", null=True, blank=True)
    back = models.FileField(upload_to=lambda i,f: f"data/{i.profile.external_id}/rg_back.{f.split('.')[-1]}", null=True, blank=True)
```

### `data/models/documents/military_document.py`

```python
class MilitaryDocument(BaseModel):
    profile = models.OneToOneField("data.Profile", on_delete=models.CASCADE, related_name="military_document")
    front = models.FileField(upload_to=lambda i,f: f"data/{i.profile.external_id}/military_document_front.{f.split('.')[-1]}", null=True, blank=True)
    back = models.FileField(upload_to=lambda i,f: f"data/{i.profile.external_id}/military_document_back.{f.split('.')[-1]}", null=True, blank=True)
```

### `data/models/documents/cnh.py`

```python
class CNH(BaseModel):
    profile = models.OneToOneField("data.Profile", on_delete=models.CASCADE, related_name="cnh")
    numero = models.CharField(max_length=20)
    data_emissao = models.DateField(null=True)
    front = models.FileField(...)
    back = models.FileField(...)
```

### `data/models/documents/certidao.py`

```python
class Certidao(BaseModel):
    TIPO_NASCIMENTO = "nascimento"
    TIPO_CASAMENTO = "casamento"
    TIPO_DIVORCIO = "divorcio"
    TIPO_OBITO = "obito"
    TIPO_CHOICES = [...]

    profile = models.ForeignKey("data.Profile", on_delete=models.CASCADE, related_name="certidoes")  # FK pq pode ter casamento+divorcio
    tipo = models.CharField(max_length=15, choices=TIPO_CHOICES)
    arquivo = models.FileField(upload_to=lambda i,f: f"data/{i.profile.external_id}/certificate_{i.tipo}.{f.split('.')[-1]}", null=True, blank=True)

    class Meta:
        unique_together = [("profile", "tipo")]
```

**Regra de criação de Certidão por estado civil:**

- solteiro → 1 certidão tipo `nascimento`
- casado → 1 certidão tipo `casamento`
- divorciado → 2 certidões: `casamento` + `divorcio`
- viúvo → 2 certidões: `casamento` + `obito`

---

## 5.2 Tools (interface pública)

### `data/tools/get_cpf.py`

```python
def get_cpf(cpf: str) -> str | None:
    """Busca CPF (já normalizado) → retorna external_id ou None."""
    from data.models import Profile
    try:
        return str(Profile.objects.get(cpf=cpf).external_id)
    except Profile.DoesNotExist:
        return None
```

### `data/tools/new_profile.py`

Recebe `user_id` + CPF, cria Profile e tudo que cascata.

```python
from django.db import transaction
from data.models import Profile, Educational, BirthInfo
from data.services.document_factory import create_documents_for_profile

@transaction.atomic
def new_profile(user_id: int, cpf: str) -> Profile:
    profile = Profile.objects.create(user_id=user_id, cpf=cpf)
    Educational.objects.create(profile=profile)
    BirthInfo.objects.create(profile=profile, estado="", cidade="", data=None)  # vazios — preencher depois

    # Documents (RG, CNH, Certidão) são criados por demanda no enrollment.
    # Military Document é criado pelo signal post_save_profile quando sexo=M

    return profile
```

### `data/tools/update_name.py`

Signal `post_save` chama isto quando `name` é atualizado.

```python
def update_name(profile):
    """
    Pega profile.name, separa primeiro e último,
    salva em user.first_name e user.last_name.
    """
    parts = (profile.name or "").strip().split()
    if not parts:
        return
    profile.user.first_name = parts[0]
    profile.user.last_name = parts[-1] if len(parts) > 1 else ""
    profile.user.save(update_fields=["first_name", "last_name"])
```

---

## 5.3 Services (privado)

### `data/services/document_factory.py`

```python
from data.models import Certidao
from data.models.documents.military_document import MilitaryDocument

def create_military_document(profile):
    """Chamado pelo signal quando sexo=M."""
    if profile.sexo == "M":
        MilitaryDocument.objects.get_or_create(profile=profile)

def create_certidao_for_civil_status(profile):
    """Chamado pelo signal quando civil_status muda."""
    mapping = {
        "solteiro": ["nascimento"],
        "casado": ["casamento"],
        "divorciado": ["casamento", "divorcio"],
        "viuvo": ["casamento", "obito"],
    }
    tipos = mapping.get(profile.civil_status, [])
    for tipo in tipos:
        Certidao.objects.get_or_create(profile=profile, tipo=tipo)
```

---

## 5.4 Signals

### `data/signals/post_save_profile.py`

```python
from django.db.models.signals import post_save
from django.dispatch import receiver
from data.models import Profile
from data.services.document_factory import create_military_document, create_certidao_for_civil_status
from data.tools.update_name import update_name

@receiver(post_save, sender=Profile)
def on_profile_save(sender, instance, created, **kwargs):
    # Documento militar se masculino
    if instance.sexo == "M":
        create_military_document(instance)

    # Certidões conforme estado civil
    if instance.civil_status:
        create_certidao_for_civil_status(instance)

    # Atualiza first_name/last_name se name preenchido
    if instance.name:
        update_name(instance)
```

---

## 5.5 Validators

### `data/validators/cpf.py`

```python
from core.validators.cpf import validate_cpf as _validate_algorithm
from data.models import Profile

def validate_cpf(cpf: str) -> str:
    """
    1. Tira pontos e traços
    2. Valida algoritmo
    3. Retorna apenas dígitos
    """
    cpf_clean = "".join(c for c in cpf if c.isdigit())
    if not _validate_algorithm(cpf_clean):
        raise ValueError("CPF inválido.")
    return cpf_clean

def cpf_already_exists(cpf_clean: str) -> bool:
    return Profile.objects.filter(cpf=cpf_clean).exists()
```

### `data/validators/name.py`

```python
EXCEPTIONS = {"de", "da", "do", "das", "dos", "e"}

def validate_and_normalize_name(name: str) -> str:
    """
    1. Exige ao menos 2 palavras
    2. Capitaliza primeira letra de cada palavra (exceto exceções)
    Ex: "joao DA silva" → "Joao da Silva"
    """
    parts = name.strip().split()
    if len(parts) < 2:
        raise ValueError("Nome deve conter ao menos 2 palavras.")
    return " ".join(
        p.lower() if p.lower() in EXCEPTIONS else p.capitalize()
        for p in parts
    )
```

### `data/validators/email.py`, `phone.py`, `address.py`, `cep.py`

Cada um com validação específica. CEP pode opcionalmente consultar ViaCEP para preencher campos faltantes (a decidir em `13-pendencias-e-decisoes.md`).

---

## 5.6 API

### Schemas

```python
# data/api/schemas/profile.py
from ninja import Schema, ModelSchema
from data.models import Profile

class ProfileOut(ModelSchema):
    self_image_url: str | None = None  # via build_external_url

    class Config:
        model = Profile
        model_fields = ["external_id", "name", "cpf", "sexo", "mother_name", "father_name", "blood_type", "civil_status"]

class ProfilePatchIn(Schema):
    name: str | None = None
    sexo: str | None = None
    mother_name: str | None = None
    father_name: str | None = None
    blood_type: str | None = None
    civil_status: str | None = None

class SelfUploadIn(Schema):
    """Upload de selfie via multipart."""
    pass  # arquivo via UploadedFile no endpoint
```

### `data/api/public.py`

```python
from ninja import Router
from data.tools.get_cpf import get_cpf

router = Router(tags=["data-public"])

@router.get("/cpf/{numero}", response={200: dict, 404: dict})
def check_cpf(request, numero: str):
    """Público — verifica se CPF já existe."""
    from data.validators.cpf import validate_cpf
    try:
        cpf_clean = validate_cpf(numero)
    except ValueError as e:
        return 400, {"detail": str(e)}
    external_id = get_cpf(cpf_clean)
    if external_id:
        return {"exists": True, "external_id": external_id}
    return 404, {"exists": False}
```

### `data/api/authenticated.py`

```python
@router.get("/profile", response=ProfileOut, auth=JWTAuth())
def get_my_profile(request):
    return request.user.profile

@router.patch("/profile", response=ProfileOut, auth=JWTAuth())
def patch_my_profile(request, payload: ProfilePatchIn):
    profile = request.user.profile
    for k, v in payload.dict(exclude_unset=True).items():
        setattr(profile, k, v)
    profile.save()
    return profile

@router.get("/email", response={"email": str}, auth=JWTAuth())
def get_my_email(request):
    return {"email": request.user.email}

@router.patch("/email", response={"email": str}, auth=JWTAuth())
def patch_my_email(request, payload: dict):
    request.user.email = payload["email"]
    request.user.save()
    return {"email": request.user.email}

# ... GET/PATCH para Address, Educational, BirthInfo, RG, etc.
```

### `data/tools/list_profiles_by_promoter.py`

```python
from data.models import Profile

def list_profiles_by_promoter(promoter_profile) -> list[Profile]:
    """Retorna profiles dos leads de um promoter."""
    return Profile.objects.filter(lead__promoter=promoter_profile)
```

### `data/tools/list_profiles_by_hub.py`

```python
from data.models import Profile

def list_profiles_by_hub(hub) -> list[Profile]:
    """Retorna profiles dos alunos de um hub."""
    return Profile.objects.filter(student__hub=hub)
```

### `data/tools/list_all_profiles.py`

```python
from data.models import Profile

def list_all_profiles() -> list[Profile]:
    """Retorna todos os profiles (staff)."""
    return Profile.objects.all()
```

> [!info] Consumido por
> [[10-promoter]], [[09-hub]], [[12-staff]]

---

## 5.7 Convenções de upload de arquivo

- Sempre normalizar para nome canônico (`rg_front.pdf`, não `meu rg.pdf`)
- Nomenclatura de path: `media/data/<external_id>/<arquivo>.<ext>`
- Validação: `core.files.upload.validate_image` ou `validate_document`
- GET retorna URL absoluta via `core.files.url.build_external_url`

---

## 5.8 Variáveis de config usadas

| Variável | Uso |
|:---------|:----|
| `EXTERNAL_URL` | URL absoluta dos arquivos |

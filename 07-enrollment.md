---
tags:
  - supletico
  - enrollment
aliases:
  - Enrollment
created: 2026-04-29
status: especificação
---

# 07 — Enrollment

Aluno/matrícula. Domínio mais complexo do sistema. Ver `enrollment-state-machine.md` para visualização das transições.

> [!info] Conexões
> - [[06-lead]] — `new_student` é chamado quando lead vira pago (status 100)
> - [[08-finance]] — comissão para hub coordinator na conclusão
> - [[11-candidate]] — convite veteran ao concluir
> - [[02-integracoes-externas]] — pagamento de fees via QR code (Asaas)
> - [[05-data]] — Profile, documentos, educational

## Status (resumo)

| Dezena | Etapa | Status detalhado |
| :---- | :---- | :---- |
| 1-3 | Dados pessoais | 1 selfie / 2 profile completo / 3 educacional |
| 4 | Matrícula externa | aguardando hub pagar fees \+ matricular |
| 10-15 | Documentos | 10 RG / 11 endereço / 12 certidão / 13 histórico / 14 militar (M) / 15 pulado (F) |
| 20-21 | Prova | 20 agendar / 21 aguardando |
| 30-33 | Análise e secretaria | 30 análise / 31 pendências / 32 secretaria / 33 disponível no polo |
| 100 | Concluído | self-certified \+ foto com coordenador |

---

## Estrutura

enrollment/

├── models/

│   ├── \_\_init\_\_.py

│   ├── student.py

│   ├── exam.py

│   ├── pendency.py

│   ├── document.py

│   ├── conclusion.py

│   └── enrollment\_fee.py        \# registra first\_part\_fee \+ second\_part\_fee \+ payment relacionados

├── api/

│   ├── \_\_init\_\_.py

│   ├── schemas/

│   │   ├── \_\_init\_\_.py

│   │   ├── student.py

│   │   ├── exam.py

│   │   ├── pendency.py

│   │   └── conclusion.py

│   ├── authenticated.py         \# status, update (role=student)

│   └── by\_role/

│       ├── hub.py               \# coordenador: paga fees, matricula externamente, aplica prova, lista exames

│       └── staff.py             \# análise documental, registra pendências, libera docs

├── tools/

│   ├── \_\_init\_\_.py

│   ├── new\_student.py

│   ├── student\_trajectory.py

│   ├── check\_exam\_schedules\_for\_date\_and\_time.py

│   └── update\_status.py         \# core da máquina de estados

├── services/

│   ├── \_\_init\_\_.py

│   ├── student\_validators.py    \# checa pré-requisitos para avançar de cada status

│   └── exam\_scheduler.py

├── tasks/

│   ├── \_\_init\_\_.py

│   ├── test\_reminder\_today.py        \# celery: lembrete no dia da prova

│   └── test\_reminder\_one\_hour.py     \# celery: lembrete 1h antes

├── validators/

│   └── ...

├── normalizers/

│   └── ...

├── signals/

│   ├── \_\_init\_\_.py

│   └── post\_save\_student.py

├── notify/

│   ├── new\_student.md

│   ├── scheduled\_test.md

│   ├── test\_reminder\_today.md

│   ├── test\_reminder\_one\_hour\_from\_now.md

│   ├── missed\_the\_test.md

│   ├── failed\_the\_test.md

│   ├── passed\_the\_test.md

│   ├── pending.md

│   ├── document\_analysis\_approved.md

│   ├── documentation\_available.md

│   ├── historic.md

│   ├── certificate.md

│   ├── external\_enrollment.md

│   └── invitation\_to\_be\_a\_promoter.md

└── apps.py

> [!info] Consumido por
> - **[[09-hub]]** — pay fees, external enrollment, exam result, conclusion, list exams
> - **[[12-staff]]** — pendencies, document analysis, final documents
>
> O app enrollment EXPÕE funções em `tools/` para que hub e staff consumam via seus próprios endpoints.
> Nenhum app externo acessa diretamente os models Student, Exam, Pendency, EnrollmentFee ou Conclusion.


---

## 7.1 Models

### `enrollment/models/student.py`

from django.db import models

from core.models.base import BaseModel, ExternalIdMixin

def student\_self\_path(instance, filename):

    return f"enrollment/{instance.profile.external\_id}/self\_certified.{filename.split('.')\[-1\]}"

def student\_external\_pwd\_path(instance, filename):

    return f"enrollment/{instance.profile.external\_id}/external\_credentials.txt"

class Student(BaseModel, ExternalIdMixin):

    STATUS\_SELFIE \= 1

    STATUS\_PROFILE\_FULL \= 2

    STATUS\_EDUCATIONAL \= 3

    STATUS\_AWAITING\_HUB\_EXTERNAL \= 4

    STATUS\_RG \= 10

    STATUS\_ADDRESS\_PROOF \= 11

    STATUS\_CERTIDAO \= 12

    STATUS\_HISTORIC \= 13

    STATUS\_MILITARY \= 14

    STATUS\_FEMININE\_SKIP \= 15  \# placeholder — feminino pula 13 → 20

    STATUS\_SCHEDULE\_EXAM \= 20

    STATUS\_AWAITING\_EXAM \= 21

    STATUS\_ANALYSIS \= 30

    STATUS\_PENDENCY \= 31

    STATUS\_AWAITING\_SECRETARIA \= 32

    STATUS\_AVAILABLE\_AT\_HUB \= 33

    STATUS\_DONE \= 100

    STATUS\_CHOICES \= \[

        (STATUS\_SELFIE, "Aguardando selfie"),

        (STATUS\_PROFILE\_FULL, "Aguardando profile completo"),

        (STATUS\_EDUCATIONAL, "Aguardando dados educacionais"),

        (STATUS\_AWAITING\_HUB\_EXTERNAL, "Aguardando matrícula externa do hub"),

        (STATUS\_RG, "Aguardando RG"),

        (STATUS\_ADDRESS\_PROOF, "Aguardando comprovante de endereço"),

        (STATUS\_CERTIDAO, "Aguardando certidão"),

        (STATUS\_HISTORIC, "Aguardando histórico escolar"),

        (STATUS\_MILITARY, "Aguardando documento militar"),

        (STATUS\_SCHEDULE\_EXAM, "Aguardando agendamento de prova"),

        (STATUS\_AWAITING\_EXAM, "Aguardando realização da prova"),

        (STATUS\_ANALYSIS, "Em análise documental"),

        (STATUS\_PENDENCY, "Com pendências"),

        (STATUS\_AWAITING\_SECRETARIA, "Aguardando secretaria de educação"),

        (STATUS\_AVAILABLE\_AT\_HUB, "Documentação disponível no polo"),

        (STATUS\_DONE, "Concluído"),

    \]

    profile \= models.OneToOneField("data.Profile", on\_delete=models.CASCADE, related\_name="student")

    hub \= models.ForeignKey("hub.Hub", on\_delete=models.PROTECT, related\_name="students")

    status \= models.IntegerField(choices=STATUS\_CHOICES, default=STATUS\_SELFIE)

    \# Credenciais da plataforma externa (preenchidos pelo hub no status 4\)

    external\_username \= models.CharField(max\_length=100, blank=True)

    external\_password \= models.CharField(max\_length=100, blank=True)  \# ⚠️ texto cru — ver pendências (criptografar?)

### `enrollment/models/enrollment_fee.py`

class EnrollmentFee(BaseModel):

    """

    Registra as duas taxas externas que o coordenador do hub paga

    em nome do aluno (boletos da secretaria de educação).

    """

    KIND\_FIRST \= "first\_part"

    KIND\_SECOND \= "second\_part"

    KIND\_CHOICES \= \[(KIND\_FIRST, "First part"), (KIND\_SECOND, "Second part")\]

    student \= models.ForeignKey("enrollment.Student", on\_delete=models.CASCADE, related\_name="fees")

    kind \= models.CharField(max\_length=15, choices=KIND\_CHOICES)

    qrcode \= models.TextField()                                    \# PIX copia-cola enviado pelo hub

    payment \= models.ForeignKey("finance.Payment", null=True, on\_delete=models.SET\_NULL, related\_name="enrollment\_fees")

    paid \= models.BooleanField(default=False)

    class Meta:

        unique\_together \= \[("student", "kind")\]

### `enrollment/models/exam.py`

def exam\_pdf\_path(instance, filename):

    return f"enrollment/{instance.student.profile.external\_id}/exam\_{instance.id}.pdf"

class Exam(BaseModel, ExternalIdMixin):

    student \= models.ForeignKey("enrollment.Student", on\_delete=models.CASCADE, related\_name="exams")

    scheduled\_date \= models.DateField()

    scheduled\_time \= models.TimeField()

    attended \= models.BooleanField(null=True)         \# null \= ainda não aconteceu

    approved \= models.BooleanField(null=True)

    pdf \= models.FileField(upload\_to=exam\_pdf\_path, null=True, blank=True)

### `enrollment/models/pendency.py`

def pendency\_media\_path(instance, filename):

    return f"enrollment/{instance.student.profile.external\_id}/pendency\_{instance.id}.{filename.split('.')\[-1\]}"

class Pendency(BaseModel, ExternalIdMixin):

    student \= models.ForeignKey("enrollment.Student", on\_delete=models.CASCADE, related\_name="pendencies")

    content \= models.TextField()

    media \= models.FileField(upload\_to=pendency\_media\_path, null=True, blank=True)

    resolved \= models.BooleanField(default=False)

### `enrollment/models/document.py`

def historic\_path(instance, filename):

    return f"enrollment/{instance.student.profile.external\_id}/historic.pdf"

def certificate\_path(instance, filename):

    return f"enrollment/{instance.student.profile.external\_id}/certificate.pdf"

class StudentDocument(BaseModel):

    """

    Documentos finais entregues pela secretaria de educação.

    Inseridos pelo staff no PATCH /document.

    """

    student \= models.OneToOneField("enrollment.Student", on\_delete=models.CASCADE, related\_name="document")

    historic \= models.FileField(upload\_to=historic\_path, null=True, blank=True)

    certificate \= models.FileField(upload\_to=certificate\_path, null=True, blank=True)

### `enrollment/models/conclusion.py`

def conclusion\_self\_path(instance, filename):

    return f"enrollment/{instance.student.profile.external\_id}/conclusion\_self.{filename.split('.')\[-1\]}"

def conclusion\_photo\_path(instance, filename):

    return f"enrollment/{instance.student.profile.external\_id}/conclusion\_photo.{filename.split('.')\[-1\]}"

class Conclusion(BaseModel):

    student \= models.OneToOneField("enrollment.Student", on\_delete=models.CASCADE, related\_name="conclusion")

    self\_certified \= models.ImageField(upload\_to=conclusion\_self\_path, null=True, blank=True)

    photo\_with\_hub\_coordinator \= models.ImageField(upload\_to=conclusion\_photo\_path, null=True, blank=True)

    completed \= models.BooleanField(default=False)

---

## 7.2 Tools (interface pública)

### `enrollment/tools/new_student.py`

Chamado por `[[06-lead|lead.tools.update_status]]` quando lead vira pago (status 100).

from django.db import transaction

from [[03-core|core.notify]] as notify\_service

from auth.tools.change\_role import change\_role

from enrollment.models import Student

from enrollment.models.enrollment\_fee import EnrollmentFee

@transaction.atomic

def new\_student(lead) \-\> "Student":

    """

    Cria Student a partir de um Lead pago.

    \- Atualiza role lead → student

    \- Hub é o do promoter que captou (promoter.hub)

    \- Status inicial \= 1 (aguardando selfie)

    \- Notify para aluno (parabéns) \+ notify para hub coordinator (novo aluno)

    """

    profile \= lead.profile

    \# 1\. Troca role

    change\_role(profile, from\_role="lead", to\_role="student")

    \# 2\. Identifica hub via promoter

    promoter\_profile \= lead.promoter

    hub \= promoter\_profile.promoter\_record.hub  \# [[10-promoter|promoter.Promoter.hub]]

    \# 3\. Cria Student

    student \= Student.objects.create(profile=profile, hub=hub, status=Student.STATUS\_SELFIE)

    \# 4\. Cria as duas EnrollmentFee vazias (qrcode é preenchido pelo hub)

    EnrollmentFee.objects.create(student=student, kind=EnrollmentFee.KIND\_FIRST, qrcode="")

    EnrollmentFee.objects.create(student=student, kind=EnrollmentFee.KIND\_SECOND, qrcode="")

    \# 5\. Notify aluno

    notify\_service.send(

        external\_id=str(profile.external\_id),

        template\_path="enrollment/notify/new\_student.md",

        context={"first\_name": profile.user.first\_name},

    )

    \# 6\. Notify coordenador do hub

    coordinator \= hub.coordinator  \# data.Profile do coordenador

    notify\_service.send(

        external\_id=str(coordinator.external\_id),

        template\_path="hub/notify/new\_student.md",

        context={

            "student\_external\_id": str(profile.external\_id),

            "student\_name": profile.name,

            "student\_cpf": profile.cpf,

            "student\_phone": profile.user.username,

        },

    )

    return student

### `enrollment/tools/student_trajectory.py`

Junta dados desde lead até conclusão.

def student\_trajectory(student) \-\> dict:

    """

    Retorna visão completa do aluno: lead, profile, address, educational,

    documents, exams, pendencies, payments, conclusion.

    Usado tanto por aluno (status 100\) quanto por staff.

    """

    profile \= student.profile

    return {

        "external\_id": str(profile.external\_id),

        "name": profile.name,

        "cpf": profile.cpf,

        "promoter": str(profile.lead.promoter.external\_id) if hasattr(profile, "lead") else None,

        "hub": {

            "external\_id": str(student.hub.external\_id),

            "name": student.hub.name,

            "address": student.hub.address.rua \+ ", " \+ student.hub.address.numero,

        },

        "current\_status": student.status,

        "exams": \[

            {"date": str(e.scheduled\_date), "approved": e.approved, "pdf": \_file\_url(e.pdf)}

            for e in student.exams.all()

        \],

        "pendencies": \[

            {"content": p.content, "resolved": p.resolved} for p in student.pendencies.all()

        \],

        "documents": {

            "historic": \_file\_url(student.document.historic) if hasattr(student, "document") else None,

            "certificate": \_file\_url(student.document.certificate) if hasattr(student, "document") else None,

        },

        "conclusion": {

            "completed": student.conclusion.completed if hasattr(student, "conclusion") else False,

        },

    }

### `enrollment/tools/check_exam_schedules_for_date_and_time.py`

from enrollment.models import Exam

def check\_exam\_schedules\_for\_date\_and\_time(hub, date, time) \-\> int:

    """Retorna quantidade de exames agendados em um hub para um dia/hora específicos."""

    return Exam.objects.filter(

        student\_\_hub=hub,

        scheduled\_date=date,

        scheduled\_time=time,

    ).count()

### `enrollment/tools/update_status.py`

Implementa toda a máquina de estados.

from django.db import transaction

from [[03-core|core.notify]] as notify\_service

from enrollment.services.student\_validators import can\_advance\_from

from enrollment.models import Student

@transaction.atomic

def update\_status(student: Student) \-\> int:

    """

    Avança o status do aluno se os pré-requisitos estão atendidos.

    Retorna o novo status. Não avança \= retorna mesmo status.

    """

    if not can\_advance\_from(student):

        return student.status

    transitions \= {

        Student.STATUS\_SELFIE: Student.STATUS\_PROFILE\_FULL,

        Student.STATUS\_PROFILE\_FULL: Student.STATUS\_EDUCATIONAL,

        Student.STATUS\_EDUCATIONAL: Student.STATUS\_AWAITING\_HUB\_EXTERNAL,

        \# status 4 → 10 acontece quando hub registra matrícula externa (api by\_role/hub)

        Student.STATUS\_RG: Student.STATUS\_ADDRESS\_PROOF,

        Student.STATUS\_ADDRESS\_PROOF: Student.STATUS\_CERTIDAO,

        Student.STATUS\_CERTIDAO: Student.STATUS\_HISTORIC,

        \# Status 13 → depende de sexo (M=14, F=20)

        Student.STATUS\_MILITARY: Student.STATUS\_SCHEDULE\_EXAM,

        \# status 20 → 21 acontece quando exame é agendado (api authenticated)

        \# status 21 → 30 acontece quando aluno é aprovado na prova

        \# status 30 → 31 ou 32 depende de pendências (staff manualmente)

        \# status 32 → 33 acontece quando staff sobe documentos

        \# status 33 → 100 acontece quando aluno+coordenador completam conclusion

    }

    if student.status \== Student.STATUS\_HISTORIC:

        \# transição condicional: feminino pula 13 → 20

        next\_status \= (

            Student.STATUS\_MILITARY

            if student.profile.sexo \== "M"

            else Student.STATUS\_SCHEDULE\_EXAM

        )

    else:

        next\_status \= transitions.get(student.status)

    if next\_status is None:

        return student.status

    student.status \= next\_status

    student.save(update\_fields=\["status"\])

    \# Triggers automáticos por status

    if next\_status \== Student.STATUS\_AWAITING\_HUB\_EXTERNAL:

        \# 3 → 4: notifica hub para iniciar matrícula externa

        notify\_service.send(

            external\_id=str(student.hub.coordinator.external\_id),

            template\_path="hub/notify/to\_enroll\_externally.md",

            context={

                "student\_external\_id": str(student.profile.external\_id),

                "student\_name": student.profile.name,

            },

        )

    return next\_status

---



### `enrollment/tools/pay_first_fee.py`

Chamado por `[[09-hub|hub]]` — coordenador paga a primeira taxa externa.

```python
from django.db import transaction
from finance.tools.pay_qrcode import pay_qrcode
from enrollment.models import Student, EnrollmentFee

@transaction.atomic
def pay_first_fee(student_external_id: str, qrcode: str):
    """
    Registra QR code e efetua pagamento da first_part_fee.
    Verifica status=4 (AWAITING_HUB_EXTERNAL).
    """
    student = Student.objects.get(profile__external_id=student_external_id)
    if student.status != Student.STATUS_AWAITING_HUB_EXTERNAL:
        raise ValueError("Status do aluno não permite pagar fee.")
    
    fee = student.fees.get(kind="first_part")
    fee.qrcode = qrcode
    payment = pay_qrcode(qrcode=qrcode, context={"enrollment_fee_id": fee.id})
    fee.payment = payment
    fee.save()
    return {"ok": True}
```

### `enrollment/tools/schedule_second_fee.py`

Chamado por `[[09-hub|hub]]` — agenda pagamento da segunda taxa.

```python
from finance.tools.schedule_qrcode import schedule_qrcode

def schedule_second_fee(student_external_id: str, qrcode: str):
    student = Student.objects.get(profile__external_id=student_external_id)
    fee = student.fees.get(kind="second_part")
    fee.qrcode = qrcode
    payment = schedule_qrcode(qrcode=qrcode, context={"enrollment_fee_id": fee.id})
    fee.payment = payment
    fee.save()
    return {"ok": True}
```

### `enrollment/tools/register_external_enrollment.py`

Chamado por `[[09-hub|hub]]` — registra matrícula externa concluída.

```python
@transaction.atomic
def register_external_enrollment(
    student_external_id: str,
    username: str,
    password: str,
) -> Student:
    """
    Verifica ambas fees pagas, salva credenciais, status 4 → 10.
    Notifica aluno com credenciais da plataforma externa.
    """
    student = Student.objects.get(profile__external_id=student_external_id)
    
    if not all(f.paid for f in student.fees.all()):
        raise ValueError("Ambas as taxas precisam estar pagas.")
    
    student.external_username = username
    student.external_password = password  # ⚠️ texto cru
    student.status = Student.STATUS_RG
    student.save()
    
    notify_service.send(
        external_id=str(student.profile.external_id),
        template_path="enrollment/notify/external_enrollment.md",
        context={
            "first_name": student.profile.user.first_name,
            "username": username,
            "password": password,
            "external_platform_url": config.EXTERNAL_PLATFORM_URL,
        },
    )
    return student
```

### `enrollment/tools/post_exam_result.py`

Chamado por `[[09-hub|hub]]` — registra resultado da prova.

```python
def post_exam_result(
    student_external_id: str,
    attended: bool,
    approved: bool | None = None,
    pdf_file=None,
) -> dict:
    """
    attended=False → status volta pra 20, notify missed
    attended=True, approved=False → status volta pra 20, notify failed (com pdf)
    attended=True, approved=True → status 21 → 30, notify passed
    """
    student = Student.objects.get(profile__external_id=student_external_id)
    exam = student.exams.latest("created_at")
    exam.attended = attended
    exam.approved = approved
    if pdf_file:
        exam.pdf = pdf_file
    exam.save()
    
    if not attended:
        student.status = Student.STATUS_SCHEDULE_EXAM
        student.save()
        notify_service.send(..., template_path="enrollment/notify/missed_the_test.md")
        return {"ok": True, "new_status": student.status}
    
    if approved is False:
        student.status = Student.STATUS_SCHEDULE_EXAM
        student.save()
        notify_service.send(..., template_path="enrollment/notify/failed_the_test.md")
        return {"ok": True, "new_status": student.status}
    
    student.status = Student.STATUS_ANALYSIS
    student.save()
    notify_service.send(..., template_path="enrollment/notify/passed_the_test.md")
    return {"ok": True, "new_status": student.status}
```

### `enrollment/tools/upload_conclusion.py`

Chamado por `[[09-hub|hub]]` — recebe selfie + foto com coordenador.

```python
def upload_conclusion(
    student_external_id: str,
    self_certified,
    photo_with_hub_coordinator,
) -> Conclusion:
    """
    Salva os 2 arquivos, marca completed=True, status 33 → 100.
    O signal post_save Student cuida de comissão + convite veteran.
    """
    student = Student.objects.get(profile__external_id=student_external_id)
    conclusion, _ = Conclusion.objects.get_or_create(student=student)
    conclusion.self_certified = self_certified
    conclusion.photo_with_hub_coordinator = photo_with_hub_coordinator
    conclusion.completed = True
    conclusion.save()
    
    student.status = Student.STATUS_DONE
    student.save()
    return conclusion
```

### `enrollment/tools/list_exams.py`

Chamado por `[[09-hub|hub]]` e `[[12-staff|staff]]`.

```python
def list_exams(hub, date=None, time=None, scheduled_only=False):
    """Lista exames de um hub com filtros opcionais."""
    qs = Exam.objects.filter(student__hub=hub)
    if scheduled_only:
        qs = qs.filter(attended__isnull=True)
    if date:
        qs = qs.filter(scheduled_date=date)
    if time:
        qs = qs.filter(scheduled_time=time)
    return qs
```

### `enrollment/tools/create_pendency.py`

Chamado por `[[12-staff|staff]]`.

```python
def create_pendency(student_external_id: str, content: str, media=None) -> Pendency:
    """Cria pendência. Se status=30, avança pra 31."""
    student = Student.objects.get(profile__external_id=student_external_id)
    pendency = Pendency.objects.create(student=student, content=content, media=media)
    if student.status == Student.STATUS_ANALYSIS:
        student.status = Student.STATUS_PENDENCY
        student.save()
    notify_service.send(
        external_id=str(student.profile.external_id),
        template_path="enrollment/notify/pending.md",
        context={"first_name": student.profile.user.first_name, "content": content},
    )
    return pendency
```

### `enrollment/tools/resolve_pendency.py`

Chamado por `[[09-hub|hub]]` e `[[12-staff|staff]]`.

```python
def resolve_pendency(pendency_id: int) -> dict:
    """Marca pendência como resolvida. Se foi a última, volta status pra 30."""
    p = Pendency.objects.get(id=pendency_id)
    p.resolved = True
    p.save()
    student = p.student
    if student.status == Student.STATUS_PENDENCY and not student.pendencies.filter(resolved=False).exists():
        student.status = Student.STATUS_ANALYSIS
        student.save()
    return {"ok": True}
```

### `enrollment/tools/approve_document_analysis.py`

Chamado por `[[12-staff|staff]]`.

```python
def approve_document_analysis(student_external_id: str) -> dict:
    """Status 30 → 32. Aguardando secretaria."""
    student = Student.objects.get(profile__external_id=student_external_id)
    student.status = Student.STATUS_AWAITING_SECRETARIA
    student.save()
    notify_service.send(
        external_id=str(student.profile.external_id),
        template_path="enrollment/notify/document_analysis_approved.md",
        context={"first_name": student.profile.user.first_name},
    )
    return {"ok": True}
```

### `enrollment/tools/upload_final_documents.py`

Chamado por `[[12-staff|staff]]`.

```python
def upload_final_documents(
    student_external_id: str,
    historic,
    certificate,
) -> StudentDocument:
    """Staff sobe histórico + certificado. Status 32 → 33. Notifica aluno."""
    student = Student.objects.get(profile__external_id=student_external_id)
    doc, _ = StudentDocument.objects.get_or_create(student=student)
    doc.historic = historic
    doc.certificate = certificate
    doc.save()
    student.status = Student.STATUS_AVAILABLE_AT_HUB
    student.save()
    # 3 notifies: documentation_available, historic, certificate
    ...
    return doc
```

## 7.3 Services (privado)

### `enrollment/services/student_validators.py`

Validação dos pré-requisitos antes de cada transição.

from enrollment.models import Student

def can\_advance\_from(student) \-\> bool:

    profile \= student.profile

    if student.status \== Student.STATUS\_SELFIE:

        return bool(profile.self\_image)

    if student.status \== Student.STATUS\_PROFILE\_FULL:

        \# nome, mãe, sangue, civil \+ RG dados (sem arquivo)

        rg \= getattr(profile, "rg", None)

        return bool(

            profile.name

            and profile.mother\_name

            and profile.blood\_type

            and profile.civil\_status

            and rg

            and rg.numero

            and rg.estado\_emissao

        )

    if student.status \== Student.STATUS\_EDUCATIONAL:

        edu \= profile.educational

        return bool(edu.nivel and (edu.fund\_concluido is not None or edu.med\_concluido is not None))

    \# 4 não avança automaticamente — depende do hub

    if student.status \== Student.STATUS\_AWAITING\_HUB\_EXTERNAL:

        return False

    if student.status \== Student.STATUS\_RG:

        rg \= profile.rg

        return bool(rg.front and rg.back)

    if student.status \== Student.STATUS\_ADDRESS\_PROOF:

        return bool(profile.address and profile.address.proof)

    if student.status \== Student.STATUS\_CERTIDAO:

        \# todas as certidões esperadas têm arquivo

        return all(c.arquivo for c in profile.certidoes.all())

    if student.status \== Student.STATUS\_HISTORIC:

        edu \= profile.educational

        \# histórico obrigatório; certificado fundamental obrigatório se concluiu fundamental

        if edu.fund\_concluido and not edu.certificado\_fund:

            return False

        return bool(edu.historico\_fund or edu.historico\_med)

    if student.status \== Student.STATUS\_MILITARY:

        if profile.sexo \!= "M":

            return True  \# feminino sempre avança

        mil \= getattr(profile, "military\_document", None)

        return bool(mil and mil.front)

    return False  \# outros status não avançam por update\_status

### `enrollment/services/exam_scheduler.py`

from enrollment.tools.check\_exam\_schedules\_for\_date\_and\_time import check\_exam\_schedules\_for\_date\_and\_time

def check\_availability\_to\_schedule(hub, date, time) \-\> dict:

    """

    Combina hub.tools.check\_availability \+ enrollment.check\_exam\_schedules.

    Retorna {available: bool, reason: str|None, slots\_used, capacity}.

    """

    from hub.tools.check\_availability\_to\_schedule\_a\_test import check\_availability\_to\_schedule\_a\_test

    hub\_check \= check\_availability\_to\_schedule\_a\_test(hub, date, time)

    if not hub\_check\["available"\]:

        return hub\_check

    used \= check\_exam\_schedules\_for\_date\_and\_time(hub, date, time)

    if used \>= hub.capacity:

        return {"available": False, "reason": f"Horário lotado ({used}/{hub.capacity}). Escolha outro."}

    return {"available": True, "reason": None, "slots\_used": used, "capacity": hub.capacity}

---

## 7.4 API

### Schemas (resumo)

\# enrollment/api/schemas/student.py

from ninja import Schema

from typing import Optional

class StudentStatusOut(Schema):

    status: int

    first\_name: Optional\[str\] \= None

    message: str

    instruction: Optional\[str\] \= None

    extra: Optional\[dict\] \= None  \# campos contextuais (data, hora, endereço, urls)

class ScheduleExamIn(Schema):

    date: str  \# YYYY-MM-DD

    time: str  \# HH:MM

class ExamPostIn(Schema):

    """Coordenador registra resultado da prova."""

    attended: bool

    approved: Optional\[bool\] \= None  \# None se attended=False

class PendencyIn(Schema):

    content: str

class ConclusionIn(Schema):

    """Multipart: 2 arquivos."""

    pass

class ExternalEnrollmentIn(Schema):

    username: str

    password: str

### `enrollment/api/authenticated.py` (role=student)

Mesma lógica do Lead — `GET /status` e `POST /update`. A resposta varia por status.

from ninja import Router

from core.auth\_helpers.role\_required import require\_role

router \= Router(tags=\["enrollment"\])

@router.get("/status", response=StudentStatusOut, auth=JWTAuth())

def get\_status(request, \_: None \= Depends(require\_role("student"))):

    student \= request.user.profile.student

    return \_build\_status\_response(student)

@router.post("/update", response=StudentStatusOut, auth=JWTAuth())

def post\_update(request, \_: None \= Depends(require\_role("student"))):

    student \= request.user.profile.student

    update\_status(student)

    return \_build\_status\_response(student)

@router.post("/schedule-exam", response=StudentStatusOut, auth=JWTAuth())

def schedule\_exam(request, payload: ScheduleExamIn, \_: None \= Depends(require\_role("student"))):

    student \= request.user.profile.student

    if student.status \!= Student.STATUS\_SCHEDULE\_EXAM:

        raise HttpError(400, "Status atual não permite agendamento.")

    from enrollment.services.exam\_scheduler import check\_availability\_to\_schedule

    avail \= check\_availability\_to\_schedule(student.hub, payload.date, payload.time)

    if not avail\["available"\]:

        return StudentStatusOut(status=student.status, message=avail\["reason"\])

    exam \= Exam.objects.create(

        student=student,

        scheduled\_date=payload.date,

        scheduled\_time=payload.time,

    )

    student.status \= Student.STATUS\_AWAITING\_EXAM

    student.save(update\_fields=\["status"\])

    \# Notifica aluno \+ coordenador

    notify\_service.send(

        external\_id=str(student.profile.external\_id),

        template\_path="enrollment/notify/scheduled\_test.md",

        context={

            "first\_name": student.profile.user.first\_name,

            "date": payload.date,

            "time": payload.time,

            "hub\_address": student.hub.full\_address,

        },

    )

    notify\_service.send(

        external\_id=str(student.hub.coordinator.external\_id),

        template\_path="hub/notify/scheduled\_test.md",

        context={"student\_name": student.profile.name, "date": payload.date, "time": payload.time},

    )

    \# Agenda lembretes via Celery

    from enrollment.tasks.test\_reminder\_today import schedule\_today\_reminder

    from enrollment.tasks.test\_reminder\_one\_hour import schedule\_one\_hour\_reminder

    schedule\_today\_reminder(exam.id)

    schedule\_one\_hour\_reminder(exam.id)

    return \_build\_status\_response(student)

#### `_build_status_response(student)`

Retorna a mensagem/instrução conforme status. Resumo do que cada status devolve:

| Status | message | instruction / extra |
| :---- | :---- | :---- |
| 1 | "Envie sua selfie. Ela serve como assinatura do contrato" | PATCH /v1/data/profile (self\_image) |
| 2 | "Preencha seus dados completos" | PATCH /v1/data/profile (mãe, pai, sangue, civil) \+ dados de RG |
| 3 | "Envie seus dados educacionais" | PATCH /v1/data/educational |
| 4 | "Aguardando matrícula externa pelo polo" | (sem ação do aluno) |
| 10 | "Anexe seu RG (frente e verso)" | PATCH /v1/data/rg (front/back) |
| 11 | "Anexe comprovante de endereço" | PATCH /v1/data/address (proof) |
| 12 | "Anexe sua(s) certidão(ões)" | PATCH /v1/data/certidao/ |
| 13 | "Anexe histórico escolar (e certificado se concluiu)" | PATCH /v1/data/educational (arquivos) |
| 14 | "Anexe documento militar" | PATCH /v1/data/military\_document |
| 20 | "Agende sua prova" | POST /v1/enrollment/schedule-exam |
| 21 | "Aguardando realização da prova" | extra: {date, time, hub\_address} |
| 30 | "Estamos analisando sua documentação" | (sem ação) |
| 31 | "Resolva pendências no polo" | extra: lista de pendências não resolvidas |
| 32 | "Aguardando documentação da secretaria" | (sem ação) |
| 33 | "Documentação disponível no polo" | extra: {hub\_address, historic\_url, certificate\_url} |
| 100 | "Concluído" | extra: student\_trajectory() |

## 7.5 Tasks (Celery)

### `enrollment/tasks/test_reminder_today.py`

from celery import shared\_task

from datetime import datetime

from [[03-core|core.notify]] as notify\_service

from enrollment.models import Exam

def schedule\_today\_reminder(exam\_id: int):

    """Agenda task para rodar no dia da prova às 8:00."""

    exam \= Exam.objects.get(id=exam\_id)

    eta \= datetime.combine(exam.scheduled\_date, datetime.strptime("08:00", "%H:%M").time())

    send\_today\_reminder.apply\_async(args=\[exam\_id\], eta=eta)

@shared\_task

def send\_today\_reminder(exam\_id: int):

    exam \= Exam.objects.get(id=exam\_id)

    if exam.attended is not None:

        return  \# já foi feita ou faltou

    notify\_service.send(

        external\_id=str(exam.student.profile.external\_id),

        template\_path="enrollment/notify/test\_reminder\_today.md",

        context={

            "first\_name": exam.student.profile.user.first\_name,

            "time": exam.scheduled\_time.strftime("%H:%M"),

            "hub\_address": exam.student.hub.full\_address,

        },

    )

### `enrollment/tasks/test_reminder_one_hour.py`

Mesma lógica, agendado 1h antes do horário marcado.

---

## 7.6 Signals

### `enrollment/signals/post_save_student.py`

from django.db.models.signals import post\_save

from django.dispatch import receiver

from enrollment.models import Student

import config

@receiver(post\_save, sender=Student)

def on\_student\_save(sender, instance, \*\*kwargs):

    if instance.status \== Student.STATUS\_DONE:

        \# Comissão para coordenador do hub

        from finance.tools.new\_commission\_hub import new\_commission\_hub

        new\_commission\_hub(coordinator\_profile=instance.hub.coordinator, student=instance)

        \# Convite para virar promoter

        from [[03-core|core.notify]] as notify\_service

        notify\_service.send(

            external\_id=str(instance.profile.external\_id),

            template\_path="enrollment/notify/invitation\_to\_be\_a\_promoter.md",

            context={

                "first\_name": instance.profile.user.first\_name,

                "url": f"{config.FRONTEND\_CANDIDATE\_URL}/veteran/{instance.profile.external\_id}",

            },

        )

        \# Notifica coordenador (parabéns \+ comissão criada)

        notify\_service.send(

            external\_id=str(instance.hub.coordinator.external\_id),

            template\_path="hub/notify/congratulations\_on\_graduating\_another\_student.md",

            context={

                "student\_name": instance.profile.name,

                "commission\_value": config.HUB\_COORDINATOR\_COMMISSION\_VALUE,

            },

        )

---

## 7.7 Variáveis de config usadas

| Variável | Uso |
| :---- | :---- |
| `[[01-config-e-env|FRONTEND_CANDIDATE_URL]]` | invitation\_to\_be\_a\_promoter.md |
| `[[01-config-e-env|EXTERNAL_PLATFORM_URL]]` | external\_enrollment.md (URL da plataforma externa onde aluno acessa com user/senha) |
| `[[01-config-e-env|HUB_COORDINATOR_COMMISSION_VALUE]]` | usado indiretamente via finance.tools.new\_commission\_hub |
| `ENROLLMENT_FIRST_PART_FEE` | (referência — valor é definido pelo qrcode) |
| `ENROLLMENT_SECOND_PART_FEE` | (referência — valor é definido pelo qrcode) |

⚠️ Adicionar `[[01-config-e-env|EXTERNAL_PLATFORM_URL]]` ao `.env.example`.

---

## 7.8 Pontos abertos

- ⚠️ `external_password` salvo em texto cru — criptografar? (ver `13-pendencias-e-decisoes.md`)  
- ⚠️ Quando aluno volta de 21 → 20 (faltou/reprovou), Exam antigo é mantido como histórico ou deletado?  
  - **Decisão padrão:** mantido (histórico). Novo agendamento cria novo Exam.  
- ⚠️ `feminine skip` (status 15\) é dezena reservada ou pulado realmente? **Decisão padrão:** dezena 15 nunca é atribuída; status pula direto 13 → 20 para feminino.  
- ⚠️ `convite para virar promoter` deve criar Candidate automaticamente ou só notificar?  
  - **Decisão padrão:** só notifica. Aluno opta clicando no link e iniciando fluxo de candidate.


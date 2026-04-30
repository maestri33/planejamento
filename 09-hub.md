---
tags:
  - supletico
  - hub
aliases:
  - Hub
created: 2026-04-29
status: especificação
---

# 09 — Hub

Polo físico de atendimento. Onde alunos fazem prova, retiram documentos e são atendidos presencialmente. O coordenador do hub (`hub_coordinator`) gerencia os alunos do polo: paga taxas de matrícula externa, agenda provas, aprova/rejeita candidates.

> [!info] Conexões
> - [[07-enrollment]] — hub paga fees (`pay_first_fee`, `schedule_second_fee`), matricula externamente, aplica prova, faz upload de conclusão
> - [[11-candidate]] — hub aprova/rejeita candidates do seu polo
> - [[05-data]] — `address` (FK) e `coordinator` (FK pra Profile)
> - [[08-finance]] — comissão por aluno concluído (`HUB_COORDINATOR_COMMISSION_VALUE`)
> - [[06-lead]] — `new_student` notifica coordenador quando lead vira pago
> - [[10-promoter]] — `coordinator` é um Profile com role `hub_coordinator` (que também pode ser `promoter`)
> - [[01-config-e-env]] — `DEFAULT_HUB`

> [!warning] Hub NÃO mexe no DB de outros apps
> O hub **nunca** acessa models de `enrollment`, `candidate`, `data` ou qualquer outro app diretamente. Toda comunicação é via `tools/` (interface pública) dos respectivos apps. O hub só tem permissão de escrita no próprio model `Hub`.

---

## Estrutura

```text
hub/
├── models/
│   ├── __init__.py
│   └── hub.py
├── api/
│   ├── __init__.py
│   ├── students.py           # pay-first-fee, schedule-second-fee, external-enrollment, exam-result, conclusion
│   ├── exams.py              # list exams do hub
│   ├── candidates.py         # list candidates, approve/reject
│   ├── profiles.py           # list profiles dos alunos do hub
│   └── authenticated.py      # GET /hub — dados do próprio hub
├── tools/
│   ├── __init__.py
│   └── check_availability.py # exposto para enrollment/services/exam_scheduler.py
├── validators/
│   ├── __init__.py
│   └── hub.py
├── normalizers/
│   ├── __init__.py
│   └── hub.py                # resolve DEFAULT_HUB quando necessário
├── notify/
│   ├── new_student.md
│   ├── new_candidate_to_hub.md
│   ├── to_enroll_externally.md
│   ├── scheduled_test.md
│   ├── congratulations_on_graduating_another_student.md
│   └── candidate_ready_for_approval.md
└── apps.py
```

---

## 9.1 Models

### `hub/models/hub.py`

```python
from django.db import models
from core.models.base import BaseModel, ExternalIdMixin

class Hub(BaseModel, ExternalIdMixin):
    name = models.CharField(max_length=200)
    address = models.ForeignKey(
        "data.Address",
        on_delete=models.PROTECT,
        related_name="hubs",
    )
    coordinator = models.ForeignKey(
        "data.Profile",
        on_delete=models.PROTECT,
        related_name="coordinated_hubs",
        help_text="Profile com role hub_coordinator (que também pode ser promoter)",
    )
    capacity = models.IntegerField(
        default=50,
        help_text="Capacidade máxima de alunos ativos simultâneos no polo",
    )

    @property
    def full_address(self) -> str:
        """Endereço completo formatado."""
        addr = self.address
        parts = [
            f"{addr.rua}, {addr.numero}",
            addr.bairro,
            f"{addr.cidade} - {addr.estado}",
            f"CEP {addr.cep}",
        ]
        return " — ".join(p for p in parts if p)

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields=["coordinator"],
                name="unique_coordinator_per_hub",
            )
        ]

    def __str__(self):
        return f"{self.name} ({self.external_id})"
```

`address` é FK para `data.Address` — o endereço fica em `data` e é reusado. O hub só guarda a referência.

`coordinator` é FK para `data.Profile` — a role `hub_coordinator` existe em `auth` e é atribuída ao Profile. Um mesmo Profile pode ter role `promoter` + `hub_coordinator` (multi-role).

`capacity` é a capacidade máxima de alunos ativos simultâneos. Não é um hard limit (soft-cap com alerta).

---

## 9.2 API (endpoints do coordenador)

Todos os endpoints exigem autenticação JWT + `require_role("hub_coordinator")`. O hub do coordenador é obtido via `request.user.profile.coordinated_hubs.first()` (o constraint garante 1:1).

> [!info] Regra de ouro
> Nenhum endpoint do hub acessa `enrollment.models.*`, `candidate.models.*` ou qualquer model externo diretamente. Tudo é delegado para os `tools/` dos apps donos.

### `hub/api/students.py`

```python
from ninja import Router
from ninja_jwt.authentication import JWTAuth
from auth.decorators import require_role

router = Router(tags=["hub-students"], auth=JWTAuth())

def _get_hub(request):
    """Helper: retorna o Hub do coordenador logado."""
    return request.user.profile.coordinated_hubs.first()


@router.post("/{external_id}/pay-first-fee")
@require_role("hub_coordinator")
def pay_first_fee(request, external_id: str):
    """
    Coordenador pagou a primeira taxa de matrícula externa.
    Envia o QR code PIX (copia-cola) para o enrollment.
    Delega para enrollment.tools.pay_first_fee().
    """
    from enrollment.tools.pay_first_fee import pay_first_fee as _pay
    hub = _get_hub(request)
    return _pay(hub=hub, student_external_id=external_id)


@router.post("/{external_id}/schedule-second-fee")
@require_role("hub_coordinator")
def schedule_second_fee(request, external_id: str):
    """
    Agenda/registra o boleto da segunda taxa de matrícula externa.
    Delega para enrollment.tools.schedule_second_fee().
    """
    from enrollment.tools.schedule_second_fee import schedule_second_fee as _sched
    hub = _get_hub(request)
    return _sched(hub=hub, student_external_id=external_id)


@router.post("/{external_id}/external-enrollment")
@require_role("hub_coordinator")
def external_enrollment(request, external_id: str):
    """
    Matricula o aluno na plataforma externa e registra credenciais.
    Delega para enrollment.tools.register_external_enrollment().
    """
    from enrollment.tools.register_external_enrollment import register_external_enrollment as _reg
    hub = _get_hub(request)
    return _reg(hub=hub, student_external_id=external_id)


@router.post("/{external_id}/exam-result")
@require_role("hub_coordinator")
def exam_result(request, external_id: str):
    """
    Coordenador posta o resultado da prova (PDF da prova + aprovação).
    Delega para enrollment.tools.post_exam_result().
    """
    from enrollment.tools.post_exam_result import post_exam_result as _post
    hub = _get_hub(request)
    return _post(hub=hub, student_external_id=external_id)


@router.patch("/{external_id}/conclusion")
@require_role("hub_coordinator")
def upload_conclusion(request, external_id: str):
    """
    Upload da selfie + foto com coordenador para conclusão do aluno.
    Delega para enrollment.tools.upload_conclusion().
    """
    from enrollment.tools.upload_conclusion import upload_conclusion as _upload
    hub = _get_hub(request)
    return _upload(hub=hub, student_external_id=external_id)
```

### `hub/api/exams.py`

```python
from ninja import Router
from ninja_jwt.authentication import JWTAuth
from auth.decorators import require_role

router = Router(tags=["hub-exams"], auth=JWTAuth())


@router.get("/exams")
@require_role("hub_coordinator")
def list_exams(request):
    """
    Lista todas as provas agendadas para o hub do coordenador.
    Delega para enrollment.tools.list_exams(hub=...).
    """
    from enrollment.tools.list_exams import list_exams as _list
    hub = request.user.profile.coordinated_hubs.first()
    return _list(hub=hub)
```

### `hub/api/candidates.py`

```python
from ninja import Router
from ninja_jwt.authentication import JWTAuth
from auth.decorators import require_role

router = Router(tags=["hub-candidates"], auth=JWTAuth())


@router.get("/candidates")
@require_role("hub_coordinator")
def list_candidates(request):
    """
    Lista todos os candidates do hub do coordenador.
    Delega para candidate.tools.list_by_hub(hub=...).
    """
    from candidate.tools.list_by_hub import list_by_hub
    hub = request.user.profile.coordinated_hubs.first()
    return list_by_hub(hub=hub)


@router.get("/candidates/{status}")
@require_role("hub_coordinator")
def list_candidates_by_status(request, status: int):
    """
    Lista candidates do hub filtrados por status.
    Delega para candidate.tools.list_by_hub(hub=..., status=...).
    """
    from candidate.tools.list_by_hub import list_by_hub
    hub = request.user.profile.coordinated_hubs.first()
    return list_by_hub(hub=hub, status=status)


@router.post("/candidates/{external_id}/approval")
@require_role("hub_coordinator")
def approve_or_reject_candidate(request, external_id: str):
    """
    Aprova ou rejeita um candidate do hub.
    Delega para candidate.tools.approve() ou candidate.tools.reject().
    """
    from candidate.tools.approve import approve
    from candidate.tools.reject import reject
    hub = request.user.profile.coordinated_hubs.first()

    # Payload determina se é approve ou reject
    approve_flag = request.body.get("approve", False)
    rejection_reason = request.body.get("rejection_reason", "")

    if approve_flag:
        return approve(candidate_external_id=external_id, hub=hub)
    else:
        return reject(candidate_external_id=external_id, hub=hub, reason=rejection_reason)
```

### `hub/api/profiles.py`

```python
from ninja import Router
from ninja_jwt.authentication import JWTAuth
from auth.decorators import require_role

router = Router(tags=["hub-profiles"], auth=JWTAuth())


@router.get("/profiles")
@require_role("hub_coordinator")
def list_profiles(request):
    """
    Lista profiles dos alunos e candidates do hub do coordenador.
    Delega para data.tools.list_profiles_by_hub(hub=...).
    """
    from data.tools.list_profiles_by_hub import list_profiles_by_hub
    hub = request.user.profile.coordinated_hubs.first()
    return list_profiles_by_hub(hub=hub)
```

### `hub/api/authenticated.py`

```python
from ninja import Router
from ninja_jwt.authentication import JWTAuth
from auth.decorators import require_role

router = Router(tags=["hub-authenticated"], auth=JWTAuth())


@router.get("/hub")
@require_role("hub_coordinator")
def get_my_hub(request):
    """
    Retorna os dados do hub do coordenador logado.
    Inclui name, address completo, capacity, contagem de alunos ativos.
    """
    hub = request.user.profile.coordinated_hubs.first()
    return {
        "external_id": str(hub.external_id),
        "name": hub.name,
        "full_address": hub.full_address,
        "capacity": hub.capacity,
        "active_students_count": hub.students.filter(status__lt=100).count(),
        "coordinator_name": hub.coordinator.name,
    }
```

---

## 9.3 Tools (interface pública)

O hub expõe **uma** tool para outros apps — a verificação de disponibilidade para agendar prova.

### `hub/tools/check_availability.py`

```python
def check_availability_to_schedule_a_test(hub, date, time) -> bool:
    """
    Verifica se o hub tem vaga para agendar uma prova na data/hora informada.

    Chamado por enrollment/services/exam_scheduler.py antes de criar um Exam.

    Regras:
    - Máximo de alunos por slot de prova (definido pela capacidade do hub)
    - Não pode agendar em horário de almoço (12:00-13:00)
    - Sábado apenas manhã (até 12:00)
    - Domingo e feriados não agendam

    Retorna True se disponível, False se lotado ou inválido.
    """
    from datetime import time as dt_time
    from enrollment.models import Exam

    # Validação de horário
    if dt_time(12, 0) <= time < dt_time(13, 0):
        return False  # horário de almoço

    if date.weekday() == 6:  # domingo
        return False

    if date.weekday() == 5 and time >= dt_time(12, 0):  # sábado tarde
        return False

    # Conta quantos exams já estão agendados nesse slot
    slot_count = Exam.objects.filter(
        student__hub=hub,
        scheduled_date=date,
        scheduled_time=time,
    ).count()

    # Capacidade por slot = capacidade total / 10 (heurística)
    max_per_slot = max(1, hub.capacity // 10)

    return slot_count < max_per_slot
```

> [!info] Única tool pública do hub
> Diferente de `enrollment` e `candidate` que expõem várias tools, o hub só expõe esta. O hub é primariamente um **consumidor** de tools de outros apps, não um provedor.

---

## 9.4 Validators

### `hub/validators/hub.py`

```python
def validate_hub_name(name: str) -> str:
    """Nome do hub: mínimo 3 caracteres, máximo 200."""
    name = name.strip()
    if len(name) < 3:
        raise ValueError("Nome do hub deve ter ao menos 3 caracteres.")
    return name


def validate_capacity(capacity: int) -> int:
    """Capacidade: entre 1 e 500."""
    if not (1 <= capacity <= 500):
        raise ValueError("Capacidade deve ser entre 1 e 500.")
    return capacity


def validate_hub_exists(external_id: str) -> bool:
    """Verifica se um hub com o external_id informado existe."""
    from hub.models import Hub
    return Hub.objects.filter(external_id=external_id).exists()
```

---

## 9.5 Notify templates

Templates de notificação disparados pelo hub ou recebidos pelo coordenador do hub:

### `hub/notify/new_student.md`

Disparado por `enrollment.tools.new_student` quando um lead vira student e é alocado neste hub. Recebido pelo coordenador.

```
--tts
**Novo aluno no hub {{hub_name}}!**

{{student_name}} (CPF {{student_cpf}}) acabou de ser matriculado.
WhatsApp: {{student_phone}}

📋 Status inicial: aguardando selfie.
```

### `hub/notify/new_candidate_to_hub.md`

Disparado por `candidate` quando um novo candidate é registrado no hub (direto ou veteran). Recebido pelo coordenador.

```
**Novo candidato a promoter no hub {{hub_name}}!**

{{candidate_name}} (CPF {{candidate_cpf}})
Hub: {{hub_name}}
Origem: {{origin}}  # "cadastro direto" ou "veteran (aluno concluído)"

⚠️ Este candidato precisará da sua aprovação ao chegar no status 30.
```

### `hub/notify/to_enroll_externally.md`

Disparado quando um aluno atinge status 4 (aguardando matrícula externa). Alerta o coordenador para agir.

```
--tts
**Aluno aguardando matrícula externa!**

{{student_name}} ({{student_external_id}}) chegou no status 4.
📌 Ação necessária: pagar as taxas e matricular externamente.

1. POST /hub/{{hub_external_id}}/students/{{student_external_id}}/pay-first-fee
2. POST /hub/{{hub_external_id}}/students/{{student_external_id}}/external-enrollment
```

### `hub/notify/scheduled_test.md`

Disparado quando uma prova é agendada para um aluno do hub. Recebido pelo coordenador.

```
**Prova agendada no hub {{hub_name}}!**

Aluno: {{student_name}} ({{student_external_id}})
📅 Data: {{exam_date}}
🕐 Horário: {{exam_time}}

Total de provas neste slot: {{slot_count}}
```

### `hub/notify/congratulations_on_graduating_another_student.md`

Disparado quando um aluno do hub conclui (status 100). Recebido pelo coordenador.

```
--tts
🎉 **Mais um aluno concluiu no hub {{hub_name}}!**

{{student_name}} concluiu o supletivo!
Comissão: R$ {{commission_value}}
💸 Será paga no próximo fechamento de caixa (sexta-feira).
```

### `hub/notify/candidate_ready_for_approval.md`

Disparado quando um candidate do hub atinge status 30 (aguardando aprovação). Recebido pelo coordenador.

```
**Candidate pronto para aprovação!**

{{candidate_name}} (CPF {{candidate_cpf}}) finalizou o cadastro e aguarda sua aprovação.

👉 Aprovar: POST /hub/candidates/{{candidate_external_id}}/approval {"approve": true}
👎 Rejeitar: POST /hub/candidates/{{candidate_external_id}}/approval {"approve": false, "rejection_reason": "..."}
```

---

## 9.6 Config vars

| Variável | Uso | Origem |
| :---- | :---- | :---- |
| `DEFAULT_HUB` | external_id do hub padrão (quando candidate chega sem ref) | candidate |

Ver [[01-config-e-env]] para detalhes sobre como configurar.

> [!warning] Pontos abertos
> - **Criação de Hub**: quem cria? Staff via CLI ou endpoint? Ainda não especificado — provavelmente `[[12-staff]]`.
> - **Troca de coordenador**: como transferir um hub para outro coordenador? Implica em reatribuir `Hub.coordinator` e atualizar roles.
> - **Capacity enforcement**: hoje `capacity` é soft-cap. Deveria bloquear novas matrículas ao atingir o limite? Impacta `enrollment.tools.new_student`.
> - **Múltiplos hubs por coordenador**: o `UniqueConstraint` atual força 1:1. Deveria permitir 1 coordenador → N hubs?
> - **Endereço do hub**: criado junto com o Hub ou reusa endereço existente? Se criado junto, vai em `data.tools.new_address`.

## Архитектура приложения и распределение ответственности

```mermaid
classDiagram
    direction TB

    class UserRole {
        <<enumeration>>
        APPLICANT
        OFFICER
        ADMIN
    }

    class ApplicationStatus {
        <<enumeration>>
        SUBMITTED
        IN_REVIEW
        NEEDS_SUPPLEMENT
        COMPLETED
        REJECTED
    }

    class EntryGoal {
        <<enumeration>>
        WORK
        NON_WORK
    }

    class FindingAssessment {
        <<enumeration>>
        COMPLIANT
        NEEDS_CORRECTION
        WARNING
    }

    class SessionService {
        -int codeTtlSeconds
        +requestSmsCode(phone: String) bool
        +verifySmsCode(phone: String, code: String) String
        +loginStaff(login: String, password: String) String
        +requireApplicant(sessionToken: String) UUID
        +requireOfficer(sessionToken: String) UUID
        +requireAdmin(sessionToken: String) UUID
    }

    class ApplicationController {
        +submitApplication(sessionToken: String, command: SubmitApplicationCommand) SubmitResult
        +supplementApplication(sessionToken: String, appId: UUID, command: SupplementCommand) SubmitResult
        +getMyApplications(sessionToken: String) List~Application~
        +startReview(sessionToken: String, appId: UUID) Review
        +saveReviewFindings(sessionToken: String, reviewId: UUID, findings: List~FindingDto~) Review
        +completeProcedure(sessionToken: String, appId: UUID, recordNumber: String, date: Date) Result
    }

    class AdminController {
        +upsertService(sessionToken: String, dto: ServiceDto) Service
        +upsertDivision(sessionToken: String, dto: DivisionDto) Division
        +createSlots(sessionToken: String, divisionId: UUID, slots: List~SlotDto~) List~Slot~
        +getStatistics(sessionToken: String) StatisticsReport
    }

    class ApplicationValidator {
        -List~String~ allowedMimeTypes
        -long maxFileSizeBytes
        +validateSubmission(command: SubmitApplicationCommand) ValidationResult
        +validateAttachments(files: List~AttachmentDto~) ValidationResult
        +validateDates(entryDate: Date, patentDate: Date, stayDays: int) ValidationResult
    }

    class FingerprintingPolicy {
        -int workDeadlineDays
        -int nonWorkStayThresholdDays
        -int nonWorkDeadlineDays
        +evaluateApplicability(goal: EntryGoal, stayDays: int, hasPrevPrints: bool) ApplicabilityResult
        +calculateDeadline(goal: EntryGoal, entryDate: Date, hasPatent: bool, patentDate: Date) Date
        +checkOverdue(calculatedDeadline: Date, currentDate: Date) bool
    }

    class ApplicationService {
        +submitApplication(applicantId: UUID, command: SubmitApplicationCommand) SubmitResult
        +supplementApplication(applicantId: UUID, appId: UUID, command: SupplementCommand) SubmitResult
        +startReview(officerId: UUID, appId: UUID) Review
        +evaluateAndDecideReview(officerId: UUID, reviewId: UUID, findings: List~FindingDto~) Review
        +completeApplication(officerId: UUID, appId: UUID, recordNumber: String, procedureDate: Date) Result
        +getStatistics() StatisticsReport
    }

    class CatalogService {
        +getAvailableDivisionsAndSlots(serviceId: UUID) List~Slot~
        +saveService(dto: ServiceDto) Service
        +saveDivision(dto: DivisionDto) Division
        +generateSlots(divisionId: UUID, slots: List~SlotDto~) List~Slot~
    }

    class ApplicationRepository {
        +beginAndLockSlot(slotId: UUID, serviceId: UUID) LockedContext
        +save(changeSet: ApplicationChangeSet) void
        +commit() void
        +rollback() void
        +rollbackIfActive() void
        +findById(appId: UUID) Application
        +findByApplicant(applicantId: UUID) List~Application~
        +countByStatus() Map~ApplicationStatus, int~
    }

    class CatalogRepository {
        +findActiveServices() List~Service~
        +findDivisionsByService(serviceId: UUID) List~Division~
        +findFreeSlots(divisionId: UUID) List~Slot~
        +saveService(service: Service) Service
        +saveDivision(division: Division) Division
        +saveSlots(slots: List~Slot~) List~Slot~
    }

    class User {
        +UUID id
        +String fullName
        +String phone
        +UserRole role
        +DateTime createdAt
        +isStaff() bool
    }

    class Service {
        +UUID id
        +String code
        +String name
        +String description
        +bool isActive
        +activate() void
        +deactivate() void
    }

    class Division {
        +UUID id
        +String name
        +String address
        +String phone
        +String workSchedule
        +updateSchedule(schedule: String) void
    }

    class Slot {
        +UUID id
        +UUID divisionId
        +Date slotDate
        +Time startTime
        +Time endTime
        +bool isAvailable
        +reserve() void
        +release() void
    }

    class Application {
        +UUID id
        +UUID applicantId
        +UUID serviceId
        +UUID slotId
        +EntryGoal entryGoal
        +Date entryDate
        +int plannedStayDays
        +bool hasPreviousFingerprinting
        +bool appliesForPatent
        +Date patentApplyDate
        +Date calculatedDeadline
        +ApplicationStatus currentStatus
        +DateTime createdAt
        +createNew(applicantId: UUID, cmd: SubmitApplicationCommand, deadline: Date)$ ApplicationChangeSet
        +transitionTo(newStatus: ApplicationStatus, actorId: UUID, comment: String) StatusEvent
        +canBeCompleted() bool
    }

    class Document {
        +UUID id
        +UUID applicationId
        +String documentKind
        +String fileName
        +String filePath
        +DateTime uploadedAt
        +isMandatoryKind() bool
    }

    class Review {
        +UUID id
        +UUID applicationId
        +UUID officerId
        +DateTime startedAt
        +DateTime finishedAt
        +String decision
        +complete(decision: String) void
    }

    class Finding {
        +UUID id
        +UUID reviewId
        +String requirementCode
        +FindingAssessment assessment
        +String comment
        +requiresApplicantAction() bool
    }

    class Result {
        +UUID id
        +UUID applicationId
        +UUID officerId
        +String recordNumber
        +Date procedureDate
        +DateTime issuedAt
        +formatConfirmation() String
    }

    class StatusEvent {
        +UUID id
        +UUID applicationId
        +UUID actorId
        +ApplicationStatus fromStatus
        +ApplicationStatus toStatus
        +String comment
        +DateTime occurredAt
    }

    ApplicationController --> SessionService : проверяет сеанс
    ApplicationController --> ApplicationService : делегирует операции
    AdminController --> SessionService : проверяет права админа
    AdminController --> CatalogService : управляет справочниками
    AdminController --> ApplicationService : запрашивает статистику

    ApplicationService --> ApplicationValidator : валидирует входные данные
    ApplicationService --> FingerprintingPolicy : применяет правила п. 2.1–2.2
    ApplicationService --> ApplicationRepository : управляет транзакциями
    CatalogService --> CatalogRepository : читает и сохраняет справочники

    ApplicationRepository ..> Application : сохраняет / загружает
    CatalogRepository ..> Service : управляет
    CatalogRepository ..> Division : управляет
    CatalogRepository ..> Slot : управляет

    User "1" --> "0..*" Application : подаёт
    User "1" --> "0..*" Review : проводит
    User "1" --> "0..*" Result : оформляет
    User "1" --> "0..*" StatusEvent : инициирует

    Service "1" --> "0..*" Application : определяет услугу
    Division "1" --> "0..*" Slot : предоставляет расписание
    Slot "1" --> "0..1" Application : бронируется под

    Application "1" *-- "1..*" Document : содержит вложения
    Application "1" *-- "0..*" Review : проверяется
    Application "1" *-- "0..1" Result : завершается результатом
    Application "1" *-- "1..*" StatusEvent : фиксирует историю
    Review "1" *-- "1..*" Finding : включает оценки

    User --> UserRole
    Application --> ApplicationStatus
    Application --> EntryGoal
    Finding --> FindingAssessment
```

Диаграмма классов (`classDiagram`) проектирует объектно-ориентированную структуру приложения для оказания государственной услуги **«Дактилоскопия и фотографирование»** (раздел 2, пункты 2.1–2.2 дорожной карты). Модель объединяет сущности предметной области из [ER-модели](er.md), роли и прецеденты из [диаграммы прецедентов](use_case.md), а также сервисные объекты, задействованные в [диаграмме последовательностей классов](class_sequence.md).

Система разделена на три архитектурных слоя для соблюдения принципа единственной ответственности

1. **Слой управления и аутентификации (Controllers & Auth)**
   * `SessionService` — обеспечивает регистрацию и вход заявителей по номеру телефона и одноразовому SMS-коду (`requestSmsCode`, `verifySmsCode`), служебную авторизацию сотрудников и администраторов (`loginStaff`), а также проверку токенов и ролей (`requireApplicant`, `requireOfficer`, `requireAdmin`).
   * `ApplicationController` — точка входа для пользовательских сценариев заявителя и сотрудника МВД/ПВС: приём первичного заявления (`submitApplication`), отправка доработок (`supplementApplication`), просмотр истории (`getMyApplications`), проведение проверки (`startReview`, `saveReviewFindings`) и фиксация очного прохождения процедуры (`completeProcedure`).
   * `AdminController` — точка входа для административных сценариев (`UC17`–`UC20`): управление услугами (`upsertService`), подразделениями МВД/ПВС (`upsertDivision`), расписанием слотов (`createSlots`) и просмотр статистики (`getStatistics`).

2. **Слой прикладной логики и предметных правил (Services & Domain Policies)**
   * `ApplicationValidator` — выполняет формальную проверку входных команд, обязательных реквизитов анкеты, логической согласованности дат и допустимых форматов/размеров приложенных сканов документов.
   * `FingerprintingPolicy` — инкапсулирует бизнес-правила пунктов 2.1 и 2.2 дорожной карты: определяет применимость процедуры (`evaluateApplicability`), рассчитывает нормативный крайний срок (`calculateDeadline`) с учётом 30-дневного срока, даты обращения за патентом или 90-дневного срока пребывания, а также проверяет факт просрочки при проверке (`checkOverdue`).
   * `ApplicationService` — координирует жизненный цикл обращения: управляет транзакциями при подаче заявки, запускает цикл проверки и возврата на доработку, контролирует наличие подтверждённого результата перед переводом заявки в статус «Завершена».
   * `CatalogService` — отвечает за выборку свободных слотов по подразделениям и ведение справочников услуг и расписания.

3. **Слой доступа к данным и доменная модель (Repositories & Domain Entities)**
   * `ApplicationRepository` и `CatalogRepository` — изолируют логику работы с БД, включая транзакционную блокировку выбранного слота (`beginAndLockSlot`) и атомарное сохранение набора изменений (`save`, `commit`, `rollback`).
   * Доменные классы (`User`, `Service`, `Division`, `Slot`, `Application`, `Document`, `Review`, `Finding`, `Result`, `StatusEvent`) и перечисления (`UserRole`, `ApplicationStatus`, `EntryGoal`, `FindingAssessment`) отражают структуру сущностей из [ER-модели](er.md) и дополняют её поведением (методы смены статуса `transitionTo`, резервирования слота `reserve`, проверки готовности к завершению `canBeCompleted`).

## Реализация ключевых бизнес-правил в методах классов

* **Применимость и расчёт срока (`FingerprintingPolicy`)**
  * метод `evaluateApplicability` блокирует создание заявки при цели `WORK` (п. 2.1), если `hasPrevPrints == true`, и при цели `NON_WORK` (п. 2.2), если `stayDays <= 90`;
  * метод `calculateDeadline` реализует формулу `min(entryDate + 30 дн., patentDate)` для цели `WORK` и `entryDate + 90 дн.` для цели `NON_WORK` (> 90 дней);
  * метод `checkOverdue` сравнивает текущую дату с `calculatedDeadline` при открытии заявки сотрудником (`startReview`) и при превышении срока автоматически добавляет замечание с оценкой `WARNING` без блокировки проверки.
* **Транзакционная целостность бронирования (`Slot` и `ApplicationRepository`)**
  * при подаче заявки `ApplicationRepository.beginAndLockSlot` блокирует строку `Slot`, метод `Slot.reserve()` переводит `isAvailable` в `false`, а статический фабричный метод `Application.createNew(...)` формирует согласованный набор (`Application`, `Document`, стартовый `StatusEvent`).
* **Контроль завершённости (`Application` и `Result`):**
  * метод `Application.canBeCompleted()` проверяет наличие связанного объекта `Result` (с номером записи `recordNumber` и датой `procedureDate`) перед вызовом `transitionTo(COMPLETED, ...)`, гарантируя, что заявка не может быть завершена без отметки о фактическом прохождении дактилоскопии и фотографирования в отделении МВД/ПВС.

## Типы связей и согласованность с ER-моделью

* **Композиция (`*--`):** дочерние сущности жизненного цикла обращения (`Document`, `Review`, `Result`, `StatusEvent`) не существуют отдельно от `Application`, а оценки (`Finding`) принадлежат конкретной проверке `Review`. При удалении родительской заявки связанные записи удаляются каскадно.
* **Ассоциация (`-->`):** отражает ссылки между независимыми сущностями справочников и участниками (`User` подаёт `Application`, `Division` предоставляет `Slot`, `Slot` связывается с одной `Application` с кратностью `1` к `0..1`).
* **Зависимость (`..>`):** репозитории создают и материализуют доменные объекты при чтении и записи в хранилище данных.

## Основания

Формат документа (`classes.md`, ровно один блок `classDiagram` с указанием атрибутов, методов, типов связей и текстовым обоснованием) соответствует требованиям лабораторной работы № 5 курса ПИС. Состав классов, ролей, статусов и предметных правил полностью согласован с пунктами 2.1–2.2 дорожной карты, [диаграммой прецедентов](use_case.md), [ER-моделью](er.md), [системной диаграммой последовательностей](system_sequence.md) и [диаграммой последовательностей классов](class_sequence.md).

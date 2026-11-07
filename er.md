# Лабораторная работа № 2. ER-диаграмма предметной области

```mermaid
erDiagram
    USER ||..o{ APPLICATION : submits
    USER ||..o{ REVIEW : performs
    USER ||..o{ STATUS_EVENT : initiates
    APPLICATION ||..|{ DOCUMENT : includes
    APPLICATION ||..|{ STATUS_EVENT : has_history
    APPLICATION ||..o{ REVIEW : undergoes
    APPLICATION ||..o| RESULT : produces
    DIVISION ||..o{ SLOT : offers
    SLOT ||..o| APPLICATION : selected_in
    REVIEW ||..o{ FINDING : records

    USER {
        uuid user_id PK
        string phone UK "Уникальный телефон или логин"
        string role "Заявитель, Сотрудник или Админ"
        string password "Хеш пароля для служебных ролей"
        datetime registered_at
    }
    APPLICATION {
        uuid application_id PK
        uuid applicant_id FK
        uuid slot_id FK
        string entry_goal "Работа или не работа"
        date entry_date
        boolean has_previous_fingerprinting
        string current_status "Подана, На проверке, Требует дополнения, Завершена"
        date calculated_deadline "Дедлайн (30 или 90 дней)"
        datetime submitted_at
    }
    DOCUMENT {
        uuid document_id PK
        uuid application_id FK
        string document_kind "Паспорт, Миграционная карта, Виза"
        string file_name
        string media_type
        bytes file_content
        datetime attached_at
    }
    REVIEW {
        uuid review_id PK
        uuid application_id FK
        uuid specialist_id FK
        string state "Открыта, Оценка сохранена, Завершена"
        datetime started_at
        datetime assessed_at "NULL до сохранения оценки"
    }
    FINDING {
        uuid finding_id PK
        uuid review_id FK
        string assessment "Соответствует или Требует исправления"
        string comment
    }
    RESULT {
        uuid result_id PK
        uuid application_id FK, UK
        string record_number "Номер записи о прохождении"
        date completed_at "Дата прохождения процедуры"
        datetime issued_at
    }
    DIVISION {
        uuid division_id PK
        string name "Название подразделения МВД/ПВС"
        string address
        string phone
        string work_hours
    }
    SLOT {
        uuid slot_id PK
        uuid division_id FK
        datetime slot_time
        boolean is_available
    }
    STATUS_EVENT {
        uuid event_id PK
        uuid application_id FK
        uuid initiator_id FK
        int event_number "Уникален внутри заявки"
        string previous_status "NULL у первого события"
        string new_status
        datetime occurred_at
        string comment
    }

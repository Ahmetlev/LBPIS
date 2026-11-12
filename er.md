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
        uuid user_id PK "ИД пользователя"
        string phone UK "Уникальный телефон или логин"
        string role "Заявитель, Сотрудник или Админ"
        string password "Хеш пароля для служебных ролей"
        datetime registered_at "Дата регистрации"
    }
    APPLICATION {
        uuid application_id PK "ИД заявки"
        uuid applicant_id FK  "ИД заявителя"
        uuid slot_id FK      "ИД слота"
        string entry_goal "Цель въезда (Работа / Не работа)"
        int planned_stay_days "Планируемое количество дней пребывания"
        boolean applies_for_patent "Подает ли на патент (только для работы)"
        date entry_date "Дата въезда"
        boolean has_previous_fingerprinting "Была ли дактилоскопия"
        string current_status "Текущий статус"
        date calculated_deadline "Дедлайн (30 или 90 дней)"
        datetime submitted_at "Дата подачи"
    }
    DOCUMENT {
        uuid document_id PK "ИД документа"
        uuid application_id FK "ИД заявки"
        string document_kind "Тип документа"
        string file_name  "Имя файла"
        string media_type  "Формат файла"
        bytes file_content "Содержимое"
        datetime attached_at "Дата загрузки"
    }
    REVIEW {
        uuid review_id PK "ИД проверки"
        uuid application_id FK "ИД заявки"
        uuid specialist_id FK  "ИД сотрудника"
        string state "Состояние"
        datetime started_at "Дата начала"
        datetime assessed_at "Дата окончания"
    }
    FINDING {
        uuid finding_id PK "ИД замечания"
        uuid review_id FK "ИД проверки"
        string assessment "Оценка"
        string comment "Комментарий"
    }
    RESULT {
        uuid result_id PK "ИД результата"
        uuid application_id FK, UK "ИД заявки"
        string record_number "Номер записи о прохождении"
        date completed_at "Дата прохождения процедуры"
        datetime issued_at "Дата выдачи"
    }
    DIVISION {
        uuid division_id PK "ИД подразделения"
        string name "Название подразделения МВД/ПВС"
        string address "Адрес"
        string phone "Телефон"
        string work_hours "Часы работы"
    }
    SLOT {
        uuid slot_id PK  "ИД слота"
        uuid division_id FK "ИД подразделения"
        datetime slot_time "Время слота"
        boolean is_available "Доступен?"
    }
    STATUS_EVENT {
        uuid event_id PK "ИД события"
        uuid application_id FK "ИД заявки"
        uuid initiator_id FK "ИД инициатора"
        int event_number "Номер события"
        string previous_status "Старый статус"
        string new_status "Новый статус"
        datetime occurred_at "Дата события"
        string comment "Комментарий"
    }

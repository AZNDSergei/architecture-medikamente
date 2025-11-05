@startuml
!define C4P https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master
!includeurl C4P/C4_Container.puml
LAYOUT_WITH_LEGEND()

' ==== Персоны ====
Person(patient, "Пациент", "Запись, ЛК, согласия")
Person(staff,   "Сотрудник (врач/ресепшн/кассир/бухгалтер)", "Рабочее место в портале")
Person(analyst, "Аналитик/DS", "Строит отчёты и модели (через Privacy Gate)")
Person(engineer,"Data/Platform инженер", "Разворачивает пайплайны в Airflow")
Person(admin,   "Администратор", "Управляет инфраструктурой/безопасностью")

System_Boundary(sys, "Medikamente Cloud (Target + Analytical Layer, Privacy by Design)") {

  ' ==== Прикладной слой (порталы/интеграции) ====
  Container(cp,  "Client Portal", "Web/Mobile", "Privacy-first формы, сбор/отзыв согласий")
  Container(sp,  "Staff Portal",  "Web",        "Ролевой доступ, минимизация полей")
  Container(api, "API Layer",     "Web/API",    "Валидация, нормализация, маршрутизация")
  Container(auth,"IAM / OAuth2 (Yandex)", "Auth", "SSO, роли, политики")
  Container(labgw,"Lab Integration", "Gateway",  "VPN/TLS, псевдонимизация направлений")
  Container(gost,"GOST Gateway (preview)", "Net","ГОСТ Р 57580.1-2017 канал к партнёрам")
  Container(mail,"Mail (Cloud/Relay)", "Email",  "Правила и шаблоны без PII (по умолчанию)")

  ' ==== Хранение / СУБД (общие для платформы и аналитики) ====
  Container(s3, "Object Storage (S3)", "Storage", "Файлы/сканы; SSE-KMS, ACL, versioning, lifecycle (Bronze/Silver files)")
  Container(pg, "Managed PostgreSQL",  "DB",      "Операционные данные, очищенные витрины (Silver)")
  Container(ch, "Managed ClickHouse",  "DB",      "Аналитические витрины/март (Gold)")

  ' ==== Безопасность и аудит ====
  Container(kms, "KMS",      "Keys",    "Ключи шифрования (S3, DB, приложения)")
  Container(lock,"Lockbox",  "Secrets", "Секреты/токены (токенизация, соли)")
  Container(log, "Cloud Logging", "Logs","Централизованные журналы (аудит/мониторинг)")

  ' ==== Аналитический слой (Privacy by Design) ====
  Container(ing,   "Ingestion",               "S3 landing / API", "Приём файлов из 1С/Excel/сканов; триггеры; SSE-KMS")
  Container(af,    "Managed Apache Airflow",  "Yandex Cloud",     "Оркестрация DAG: ingest-DeID-Quality-Publish")
  Container(deid,  "De-ID & Privacy Pipeline","Python + NER",     "PII/MED детекция (Natasha/DeepPavlov), MIN, MASK, TOK (Lockbox/KMS)")
  Container(quality,"Quality/Normalize",      "Python/SQL",       "Проверки схем/качества, очистка/нормализация")
  Container(pgate, "Privacy Gate",            "Service (Python)", "k-anon, small-groups, маски, правила доступа перед Gold")
  Container(fs,    "Feature Store",           "ClickHouse/PostgreSQL","Псевдоним. признаки для ML")
}

' ==== Пользовательские потоки (порталы) ====
Rel(patient, cp, "Регистрация/согласие", "HTTPS/OAuth2")
Rel(staff,   sp, "Рабочие операции",     "HTTPS/SSO")
Rel(cp,  api, "CRUD/заявки/записи", "HTTPS")
Rel(sp,  api, "CRUD/операции/отчёты", "HTTPS")
Rel(api, auth, "Аутентификация/авторизация", "OIDC/OAuth2")
Rel(api, s3,   "Загрузка/скачивание файлов (сканы)", "TLS + SSE-KMS")
Rel(api, pg,   "Транзакционный доступ (операционные данные / Silver)", "TLS")
Rel(api, ch,   "Запросы отчётов/витрин (Gold)", "TLS")
Rel(staff, mail, "Оповещения", "шаблоны без PII")

' ==== Интеграции с лабораториями ====
Rel(labgw, gost, "Трафик к партнёрам", "ГОСТ-канал")
Rel(gost,  api,  "Направления/результаты", "VPN/TLS")

' ==== Аналитический конвейер (общие хранилища) ====
Rel(engineer, af, "Пишет и запускает DAG", "IAM/SSO")
Rel(ing,     af, "Триггеры загрузок", "events")
Rel(af,    deid, "Запуск De-ID задач", "DAG")
Rel(deid,   s3,  "Raw→обезличенные артефакты (Bronze в S3)", "S3")
Rel(af,  quality,"Запуск проверок качества/схем", "DAG")
Rel(quality, pg, "Publish очищенных данных (Silver)", "SQL")
Rel(af,      ch, "Публикация витрин (Gold)", "SQL")
Rel(pgate,   ch, "Доступ только после проверок", "SQL")
Rel(analyst, pgate, "Запросы к данным (BI/SQL)", "IAM/SSO")
Rel(ch,      fs, "Фичи для ML", "SQL")

' ==== Безопасность / управление ====
Rel(deid, lock, "Секреты/соли/токены")
Rel(deid, kms,  "Ключи шифрования (Transit)")
Rel(s3,   log,  "Аудит операций с объектами")
Rel(pg,   log,  "Аудит запросов/изменений")
Rel(ch,   log,  "Аудит запросов/публикаций")
Rel(af,   log,  "Аудит DAG/пайплайнов")
Rel(pgate,log,  "Аудит проверок (k-anon/DP/small-groups)")
Rel(api,  log,  "Аудит API-вызовов")
Rel(admin,log,  "Просмотр логов/алертов")
Rel(admin,kms,  "Управление ключами")
Rel(admin,lock, "Управление секретами")
@enduml

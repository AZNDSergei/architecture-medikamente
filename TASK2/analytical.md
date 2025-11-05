@startuml
!define C4P https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master
!includeurl C4P/C4_Container.puml
LAYOUT_WITH_LEGEND()

Person(analyst, "Аналитик/DS", "Строит отчёты и модели (через Privacy Gate)")
Person(engineer, "Data/Platform инженер", "Разворачивает пайплайны в Airflow")

System_Boundary(system, "Analytical Layer (Privacy by Design)") {

  Container(ing, "Ingestion", "Yandex Object Storage (S3)", "Приём файлов из 1С/Excel/сканов; SSE-KMS, versioning")
  Container(af,  "Managed Apache Airflow", "Yandex Cloud", "Оркестрация DAG: ingest-DeID-Quality-Publish")
  Container(deid,"De-ID & Privacy Pipeline", "Python + NER (Natasha/DeepPavlov)", "PII/MED детекция, MIN, MASK, TOK (Lockbox/KMS)")
  Container(bronze, "Bronze (raw)", "S3 (Parquet/Raw)", "Сырые данные (закрыто), шифр. SSE-KMS, lifecycle")
  Container(quality,"Quality/Normalize", "Python/SQL", "Проверки схем/качества, очищение")
  Container(silver, "Silver (обработанные)", "Managed PostgreSQL / S3", "Очищенные и тегированные данные (операционные витрины)")
  Container(pgate, "Privacy Gate", "Service (Python/Flask)", "k-anon, small groups, маски, правила доступа")
  Container(gold,  "Gold", "Managed ClickHouse", "Аналитические витрины/март (только через Gate)")
  Container(fs,    "Feature Store", "ClickHouse/PostgreSQL", "Псевдоним. признаки для ML")

  ContainerDb(kms, "KMS", "Yandex KMS", "Ключи шифрования (S3, DB, приложения)")
  Container(lock, "Lockbox", "Yandex Lockbox", "Секреты/токены (токенизация)")
  Container(log, "Cloud Logging", "Yandex", "Централизованные журналы (аудит)")
  Container(iam, "IAM/OAuth2", "Yandex", "Роли/политики, SSO")

}

Rel(engineer, af, "Пишет и запускает DAG", "IAM")
Rel(ing, af, "Триггеры загрузок", "events")
Rel(af, deid, "Запуск De-ID задач", "DAG")
Rel(deid, bronze, "Raw→обезличенные артефакты", "S3")
Rel(af, quality, "Запуск проверок", "DAG")
Rel(quality, silver, "Publish очищенных данных", "SQL/S3")
Rel(pgate, gold, "Доступ только после проверок", "SQL")
Rel(af, gold, "Публикация витрин", "SQL")
Rel(gold, fs, "Фичи для ML", "SQL")
Rel(analyst, pgate, "Запросы к данным", "IAM/SSO")

Rel(deid, lock, "Секреты/соли/токены")
Rel(deid, kms, "Ключи шифрования")
Rel(bronze, log, "Аудит операций")
Rel(silver, log, "Аудит доступа/изменений")
Rel(pgate, log, "Аудит правил/отказов")
Rel(af, log, "Аудит DAG")

Rel(analyst, iam, "SSO/OAuth2")
@enduml

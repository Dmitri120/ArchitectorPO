\# **14. Основные представления**





\## **14.1 Функциональное представление**



Модули монолита и их функции:

|**Модуль**|**Функции**|**Внешние зависимости**|
|-|-|-|
|Training|CRUD тренировок, графики динамики, сравнение с собой/регионом|PostgreSQL, ClickHouse, Kafka|
|Social|Группы, поиск людей рядом, лента, события|Redis (гео), PostgreSQL, Kafka|
|Gamification|Уровни, значки, челленджи, рекорды|PostgreSQL, Kafka (подписка на тренировки)|
|Inventory|Инвентарь (обувь, часы), подсказки об износе|PostgreSQL|
|Promotion|Региональные акции, новости спорта, персонализация|PostgreSQL, Kafka (подписка)|
|Device Integration|Адаптеры для Garmin, Polar, Wahoo, BLE|MQTT, HealthKit/Google Fit|
|Shop Adapter|REST-клиент к существующему магазину|HTTP|
|User \& Profile|Регистрация, профиль, настройки приватности|PostgreSQL|
|Notification|Push (FCM/APNS), email, in-app|Kafka (подписка)|









\## **14.2 Информационное представление**



**Основные сущности:**

* User (id, name, email, phone, encrypted\_pii)
* Training (id, user\_id, type, distance, duration, avg\_hr, track\_s3\_key, datetime)
* InventoryItem (id, user\_id, type, brand, distance\_covered\_km, expiry\_notification)
* SocialGroup (id, name, sport\_type, location, owner\_id)
* GroupMember (user\_id, group\_id, role, joined\_at)
* Achievement (id, user\_id, type, earned\_at)
* Notification (id, user\_id, type, status, sent\_at)



**ClickHouse (аналитика):**

* training\_aggregates (user\_id, date, sport\_type, total\_distance, avg\_hr, count\_trainings)
* region\_aggregates (region, date, sport\_type, avg\_distance, percentile\_50, percentile\_90)



**Redis (гео + кеш):**

* geo:active:{user\_id} → координаты + TTL 10 мин
* idempotency:{key} → training\_id + TTL 24 ч
* cache:comparison:{user\_id} → JSON сравнения + TTL 1 ч







\## **14.3 Многозадачность (Concurrency)**



**Проблема:** одновременная запись тренировок, обновление рекордов, отправка уведомлений.

**Решения:**

|**Сценарий**|**Решение**|
|-|-|
|Запись тренировки|Транзакция PostgreSQL (READ COMMITTED), уникальный idempotency\_key|
|Обновление рекорда|SELECT FOR UPDATE на строке user\_achievement, Kafka asynch|
|Геопоиск|Redis — атомарные операции, нет блокировок|
|Челленджи с тысячами участников|Асинхронный расчёт раз в час (Kafka Streams), не в реальном времени|
|Рейтинг в группе|Материализованное представление в ClickHouse, обновление по событию|





**Обработка гонок:**

* Оптимистичные блокировки (version field в PostgreSQL)
* Идемпотентность через Redis
* Kafka с partition по user\_id (сохраняем порядок событий)







\## **14.4 Инфраструктурное представление**



**Регионы**: EU (Frankfurt), US (Ohio), Asia (Singapore) — на старте только EU.



**Компоненты в одном регионе**:

Internet -> CloudFlare (DNS, DDoS) -> Kong Gateway (3 pods) -> Модульный монолит (10-30 pods) -> PostgreSQL (primary + replica) / Redis Cluster / ClickHouse (3 nodes)-> Kafka (3 brokers) → Consumers (Gamification, Notification, Analytics)



**K8s (EKS / GKE / AKS):**

* Namespaces: gateway, app, kafka, observability, analytics
* HPA для монолита (CPU > 70%, масштабирование)
* PDB (PodDisruptionBudget): minAvailable 80%



**Сеть**:

* Внутри кластера: сервис-меш (Linkerd) для mTLS и наблюдаемости
* Доступ к БД только из приложения (NetworkPolicy)
* Бэкапы PostgreSQL в S3 (каждые 6 часов) + WAL архивация







\## **14.5 Безопасность**



**Угрозы и защита**:

|**Угроза**|**Защита**|
|-|-|
|A1: Injection|Параметризованные запросы, валидация GPX-треков|
|A2: Broken Auth|JWT с коротким TTL (15 мин), refresh token, MFA для sensitive ops|
|A3: PII утечка|Шифрование email/телефона в БД (AES-256), маскирование в логах|
|A4: XXE|Отключить external entities в XML (если используем GPX)|
|A5: Broken Access Control|Проверка owner\_id на каждом запросе, row-level security в PostgreSQL|
|A6: Misconfig|Регулярные аудиты, kube-bench, запрет privileged контейнеров|
|A7: XSS|Экранирование вывода в веб-версии|
|A8: Insecure Deserialization|Использовать JSON Schema validation|
|A9: Known Vulnerabilities|Сканирование образов (Trivy), регулярные обновления|
|A10: Logging Fail|Логи всех access + audit в отдельный Kafka topic|





**Дополнительно:**

* Доступ к админке только через VPN
* Rate limiting: 1000 запросов/час на пользователя, 100/час на геопоиск
* Безопасность устройств: API ключи на устройство, возможность отозвать












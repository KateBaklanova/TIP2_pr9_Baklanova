<img width="900" height="481" alt="image" src="https://github.com/user-attachments/assets/d0185702-dec2-4ee4-810e-4a483a13f622" /># Практическое занятие №9
## Реализация распределённого кэша (Redis cluster)

**ФИО:** Бакланова Е.С.
**Группа:** ЭФМО-01-25

## Цели работы

- Освоить внедрение распределённого кэша в backend-приложение на Go и реализовать стратегию cache-aside с использованием Redis, корректного TTL, jitter и устойчивого поведения сервиса при недоступности кэша.

## Задачи

1.	Изучить назначение кэширования в серверных приложениях.
2.	Понять, когда кэширование действительно ускоряет систему.
3.	Освоить роль Redis как внешней инфраструктурной зависимости.
4.	Реализовать стратегию cache-aside для чтения сущности по идентификатору.
5.	Научиться формировать ключи кэша по понятной и стабильной схеме.
6.	Освоить использование TTL и случайного разброса времени жизни ключа.
7.	Реализовать инвалидацию кэша при изменении и удалении данных.
8.	Обеспечить деградацию сервиса при недоступности Redis без отказа основного API.
9.	Научиться проверять hit, miss и fallback-сценарии на учебном стенде.

## Теория

Когда серверное приложение получает запрос на чтение данных, оно чаще всего обращается к базе данных. Если запросов много, а часть данных запрашивается повторно, БД начинает выполнять лишнюю работу. В такой ситуации кэш позволяет временно хранить уже полученный результат и быстрее отдавать его при повторном обращении.
Кэширование особенно полезно, когда:

- данные читаются чаще, чем изменяются;
- чтение из БД сравнительно дороже, чем чтение из памяти;
- одни и те же сущности часто запрашиваются повторно;
- нужно снизить нагрузку на основное хранилище.

В этой работе Redis рассматривается именно как ускоритель чтения.

Redis — это очень быстрый in-memory key-value store. Его удобно использовать как внешний кэш, потому что:

-	доступ к данным быстрый;
-	можно задавать TTL;
-	можно удалять и обновлять ключи выборочно;
-	Redis легко поднять локально в Docker;
-	он хорошо подходит для cache-aside сценариев.


### Содержание проекта

**Auth service** (порт 8081 +  gRPC порт 50051)
- Аутентификация пользователей
- Выдача токенов
- Проверка токенов через gRPC

**Tasks service** (порт 8082)
- CRUD для задач (TODO-список)
- Перед каждой операцией проверяет токен, отправляя gRPC-запрос в Auth service

### Инструкция по запуску

```bash
cd deploy

# Сборка и запуск всех сервисов
docker-compose up -d --build

# Просмотр статуса
docker-compose ps

# Просмотр логов
docker-compose logs -f

# Остановка
docker-compose down
```

<img width="900" height="548" alt="image" src="https://github.com/user-attachments/assets/ea9a582d-3808-4acd-ae64-4108a8525a75" />



### Структура

<img width="210" height="838" alt="image" src="https://github.com/user-attachments/assets/51fea5b3-e1a0-4ee9-bcfd-7111606b2159" />


### Тестирование

1. Получить токен для дальнейшей работы

```bash
curl -i -X POST http://localhost:8081/v1/auth/login \
  -H "Content-Type: application/json" \
  -H "X-Request-ID: test_kottia" \
  -d "{\"username\":\"kate\",\"password\":\"secret\"}"
```

2. Создаем таск, копируем id

```bash
  curl -i -X POST http://localhost:8082/v1/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer demo-token-test_kottia:kate" \
  -d '{"title":"","description":"","due_date":"2026-03-21"}'
```

<img width="900" height="477" alt="image" src="https://github.com/user-attachments/assets/b7cea2e3-38e8-4aef-84b3-c0279cea8d1f" />


3. Получить задачу по id

```bash
  curl -X GET http://localhost:8082/v1/tasks/{id} \
  -H "Authorization: Bearer demo-token-test_kottia:kate"
```

<img width="900" height="475" alt="image" src="https://github.com/user-attachments/assets/b03822e2-e980-4d1d-9b08-5a8b665c7508" />

Задачи пока нет в кэше [CACHE MISS], записываем в кэш [CACHE SET]

<img width="900" height="93" alt="image" src="https://github.com/user-attachments/assets/bab75c5d-b737-4448-b183-6340c55b7152" />

После повторного запроса, данные в кэше [CACHE HIT]

<img width="900" height="91" alt="image" src="https://github.com/user-attachments/assets/b0ba0993-eaaa-404b-87cb-3bd16223be31" />


4. Проверяем инвалидацию 

Выполняем PATCH, кэш инвалидируется

<img width="900" height="478" alt="image" src="https://github.com/user-attachments/assets/ca88df30-0601-424a-b3fc-5ea9fb8c9536" />

<img width="900" height="41" alt="image" src="https://github.com/user-attachments/assets/0962f168-cf83-4d09-a1d8-c6054a9ae849" />

После получаем [CACHE MISS]

<img width="900" height="500" alt="image" src="https://github.com/user-attachments/assets/1cc81d64-6c64-4342-ba98-c26455dbb52b" />

<img width="900" height="128" alt="image" src="https://github.com/user-attachments/assets/a0d4ca87-1dc9-42b0-9407-7effa743bad5" />


Выполняем DELETE, кэш инвалидируется

<img width="900" height="348" alt="image" src="https://github.com/user-attachments/assets/265e833d-eac5-42a2-a592-ee953f7e743c" />

<img width="900" height="173" alt="image" src="https://github.com/user-attachments/assets/381c2568-a571-4ded-86ce-8bf39e0ab733" />



5. Проверка остановки Redis

останавливает контейнр с Redis

<img width="824" height="73" alt="image" src="https://github.com/user-attachments/assets/d186b59d-9ec9-49d0-b678-0a7a82d5168e" />

Создаем новую задачу, получаем [CHACHE ERROR]

<img width="900" height="476" alt="image" src="https://github.com/user-attachments/assets/da05a6a9-10dd-4de4-8852-0ab280e69225" />

<img width="900" height="108" alt="image" src="https://github.com/user-attachments/assets/3bfaff25-3332-4cd3-8bb9-24d932fdb17b" />


### Контрольные вопросы

1. Что такое cache-aside?

Это стратегия, при которой приложение сначала пытается прочитать данные из кэша, при промахе (cache miss) идёт в основное хранилище (БД), получает данные и сохраняет их в кэш для последующих запросов.

2. Почему Redis не должен быть источником истины?

Потому что Redis - это временное in-memory хранилище

3. Зачем нужен TTL?

TTL (Time To Live) нужен для автоматического удаления устаревших данных из кэша через заданный промежуток времени

4. Что такое jitter?

Jitter - это случайный разброс времени жизни ключа (TTL)

5. Почему одинаковый TTL для всех ключей может быть проблемой?

Потому что при одновременном истечении большого количества ключей возникает всплеск нагрузки на БД

6. Как должен вести себя сервис при недоступности Redis?

Сервис должен логировать ошибку, но продолжать работать, обращаясь напрямую к БД

7. Почему кэш нужно инвалидировать после изменения данных?

Чтобы клиент не получал устаревшие данные из кэша после того, как данные в основном хранилище были изменены или удалены

8. Чем кэширование одной сущности проще, чем кэширование списка?

Кэширование одной сущности проще, потому что ключ однозначен (tasks:task:id), инвалидация тривиальна (удалить один ключ), и нет проблем с пагинацией и фильтрацией.

9. В чём смысл ключа вида tasks:task:<id>?

Смысл в том, что ключ уникален, предсказуем, легко формируется по ID задачи, позволяет быстро найти нужные данные в Redis и инвалидировать задачу

10. Почему Redis рассматривается как внешняя инфраструктурная зависимость?

Потому что Redis - это отдельный сервис, работающий по сети, который может быть недоступен (независимо от основного приложения)


# Retrospective

Backend для проведения анонимных командных ретроспектив.

## Идея

Сервис позволяет команде создать комнату для проведения ретроспективы, пригласить участников по ссылке и анонимно собирать их записи.

Участники комнаты видят только количество/плашки созданных записей, но не могут узнать, кому принадлежит конкретная запись или прочитать её содержимое до момента раскрытия.

---

## Основные возможности

### Пользователи

Пользователь может:

* зарегистрироваться;
* авторизоваться;
* создать комнату;
* присоединиться к комнате по invite-ссылке;
* просматривать комнаты, в которых он участвует.

---

### Комнаты

Комната представляет собой отдельную сессию ретроспективы.

В комнате есть:

* название;
* создатель;
* список участников;
* набор полей для записей;
* записи участников;
* статус ретроспективы.

Пример полей:

```text
What went well?
What went wrong?
What should we improve?
Ideas
```

---

## Приглашение в комнату

В комнату нельзя попасть через публичный поиск.

Вход осуществляется только по специальной ссылке.

Предусмотрено два типа invite-ссылок.

### Одноразовая ссылка

Ссылка создаётся для конкретного участника и может быть использована только один раз.

После использования ссылка становится недействительной.

Пример:

```text
https://example.com/invite/8f3a2c...
```

### Временная ссылка

Ссылка не привязана к конкретному пользователю и может использоваться несколькими участниками.

У ссылки есть срок действия.

Например:

```text
https://example.com/invite/8f3a2c...
```

Ссылка действительна в течение 30 минут.

После истечения срока новые участники не смогут присоединиться по ней.

---

## Участники

После входа в комнату пользователь появляется в списке участников.

Пример:

```text
Participants

● Alexander
● John
● Maria
● Anonymous
```

При этом автор записи не должен быть виден другим участникам.

---

## Анонимные записи

После начала ретроспективы каждый участник выбирает поле, в котором хочет оставить запись.

Например:

```text
┌──────────────────────────────┐
│ What went well?              │
│                              │
│ + + + + +                    │
└──────────────────────────────┘

┌──────────────────────────────┐
│ What went wrong?             │
│                              │
│ + + +                        │
└──────────────────────────────┘
```

Каждый `+` представляет отдельную запись.

Содержимое записей скрыто.

Например, участники видят:

```text
What went well?

[ + ] [ + ] [ + ] [ + ]
```

но не видят:

```text
Alexander:
"Communication between teams improved."
```

или:

```text
Maria:
"Deployments became much faster."
```

Связь между пользователем и записью должна оставаться скрытой.

---

## Раскрытие записей

После завершения этапа добавления записей комната переходит в состояние просмотра.

В этот момент записи становятся доступны всем участникам:

```text
What went well?

┌────────────────────────────────────┐
│ Communication between teams        │
│ improved.                          │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│ Deployments became much faster.    │
└────────────────────────────────────┘
```

При этом автор записи по-прежнему не раскрывается.

---

## Жизненный цикл комнаты

Комната проходит несколько состояний:

```text
CREATED
   ↓
WAITING
   ↓
ACTIVE
   ↓
REVIEW
   ↓
FINISHED
```

### CREATED

Комната создана, но ретроспектива ещё не началась.

Создатель может:

* настроить поля;
* создать invite-ссылку;
* пригласить участников.

### WAITING

Участники подключаются к комнате.

### ACTIVE

Участники оставляют анонимные записи.

Содержимое записей скрыто.

### REVIEW

Добавление новых записей завершено.

Участники могут просматривать и обсуждать записи.

### FINISHED

Ретроспектива завершена.

---

## Основные сущности

Предполагаемые сущности:

```text
User
Room
RoomParticipant
Invite
RetroColumn
Note
```

Связи:

```text
User
 │
 ├──────────────┐
 │              │
 ▼              ▼
Room       RoomParticipant
 │
 ├── Invite
 │
 ├── RetroColumn
 │       │
 │       └── Note
 │
 └── Participants
```

### User

```text
id
email
password_hash
created_at
```

### Room

```text
id
name
owner_id
status
created_at
started_at
finished_at
```

### RoomParticipant

Связывает пользователя с комнатой.

```text
id
room_id
user_id
joined_at
```

### Invite

```text
id
room_id
token
type
expires_at
used_at
created_at
```

Для одноразовой ссылки дополнительно необходимо хранить информацию о том, кому она предназначена.

### RetroColumn

Поле/категория ретроспективы.

```text
id
room_id
title
position
```

Например:

```text
What went well?
What went wrong?
Ideas
```

### Note

Анонимная запись пользователя.

```text
id
room_id
column_id
author_id
content
created_at
```

`author_id` используется только сервером для контроля доступа и предотвращения повторного использования/изменения записи.

Клиент никогда не должен получать информацию, позволяющую определить автора записи.

---

## API

Предварительный набор endpoints:

### Authentication

```http
POST /auth/register
POST /auth/login
POST /auth/logout
GET  /auth/me
```

### Rooms

```http
POST   /rooms
GET    /rooms
GET    /rooms/{room_id}
PATCH  /rooms/{room_id}
DELETE /rooms/{room_id}
```

### Participants

```http
GET    /rooms/{room_id}/participants
POST   /rooms/{room_id}/join
DELETE /rooms/{room_id}/participants/{user_id}
```

### Invites

```http
POST /rooms/{room_id}/invites
GET  /rooms/{room_id}/invites
POST /invites/{token}/join
```

### Columns

```http
POST   /rooms/{room_id}/columns
GET    /rooms/{room_id}/columns
PATCH  /rooms/{room_id}/columns/{column_id}
DELETE /rooms/{room_id}/columns/{column_id}
```

### Notes

```http
POST /rooms/{room_id}/columns/{column_id}/notes
GET  /rooms/{room_id}/columns/{column_id}/notes
```

До перехода комнаты в `REVIEW` endpoint получения записей не должен возвращать их содержимое.

---

## Real-time

Для совместной работы участников желательно использовать WebSocket.

WebSocket может использоваться для:

* появления нового участника;
* выхода участника;
* изменения статуса комнаты;
* появления новой анонимной записи;
* начала/завершения этапа;
* обновления количества записей.

Например:

```text
Client
   │
   │ WebSocket
   ▼
Go Server
   │
   ├── Participant joined
   ├── Note created
   ├── Room started
   └── Room finished
```

---

## Технологии

Основной стек:

* Go
* PostgreSQL
* Redis
* WebSocket
* Docker
* Docker Compose

Возможный стек Go:

* `net/http` или `chi` для HTTP;
* `pgx` для PostgreSQL;
* `golang-migrate` для миграций;
* `go-redis` для Redis;
* стандартный `log/slog` для логирования.

---

## Предполагаемая архитектура

```text
                    ┌──────────────┐
                    │    Client    │
                    └──────┬───────┘
                           │
                    HTTP / WebSocket
                           │
                           ▼
                  ┌─────────────────┐
                  │   HTTP Handler  │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │     Service     │
                  └────────┬────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
              ┌──────────┐   ┌──────────┐
              │PostgreSQL│   │  Redis   │
              └──────────┘   └──────────┘
```

Основной принцип:

```text
Handler
   ↓
Service
   ↓
Repository
   ↓
Database
```

Бизнес-логика не должна находиться непосредственно в HTTP handlers.

---

## Анонимность

Анонимность является одной из основных особенностей проекта.

Клиент не должен получать:

* `author_id`;
* email автора;
* username автора;
* другие данные, позволяющие определить автора записи.

Например, сервер хранит:

```json
{
  "id": 123,
  "column_id": 5,
  "author_id": 42,
  "content": "Deployments became faster"
}
```

Но клиент получает:

```json
{
  "id": 123,
  "column_id": 5,
  "content": "Deployments became faster"
}
```

До завершения этапа записи клиент может получать только:

```json
{
  "id": 123
}
```

Таким образом, содержимое записи и её автор не раскрываются до перехода комнаты в соответствующее состояние.

---

## Цель проекта

Проект создаётся как практический backend-проект для изучения Go.

Основные цели:

* изучить Go;
* реализовать REST API;
* разобраться с PostgreSQL;
* реализовать WebSocket;
* изучить concurrency в Go;
* реализовать authentication/authorization;
* поработать с Redis;
* реализовать работу с invite-ссылками;
* разобраться с транзакциями;
* написать тесты;
* контейнеризировать приложение через Docker.

---

## Roadmap

### Phase 1 — Go basics

* [ ] Go modules
* [ ] Structs
* [ ] Interfaces
* [ ] Errors
* [ ] Pointers
* [ ] Goroutines
* [ ] Channels
* [ ] Context
* [ ] Testing

### Phase 2 — Backend foundation

* [ ] HTTP server
* [ ] Routing
* [ ] Middleware
* [ ] JSON
* [ ] Error handling
* [ ] Configuration
* [ ] Logging

### Phase 3 — Database

* [ ] PostgreSQL
* [ ] Database schema
* [ ] Migrations
* [ ] Repository layer
* [ ] Transactions
* [ ] Indexes

### Phase 4 — Authentication

* [ ] Registration
* [ ] Login
* [ ] Password hashing
* [ ] JWT/session authentication
* [ ] Authorization

### Phase 5 — Retrospective

* [ ] Rooms
* [ ] Participants
* [ ] Invite links
* [ ] Columns
* [ ] Anonymous notes
* [ ] Room states
* [ ] Reveal stage

### Phase 6 — Real-time

* [ ] WebSocket
* [ ] Room events
* [ ] Participant events
* [ ] Note events
* [ ] Concurrent connections

### Phase 7 — Infrastructure

* [ ] Redis
* [ ] Docker
* [ ] Docker Compose
* [ ] Environment configuration
* [ ] Health checks
* [ ] Graceful shutdown

### Phase 8 — Testing

* [ ] Unit tests
* [ ] Repository tests
* [ ] HTTP handler tests
* [ ] Integration tests
* [ ] WebSocket tests
* [ ] Race detector

---

## Future ideas

В дальнейшем можно добавить:

* голосование за записи;
* объединение похожих записей;
* комментарии;
* таймер этапа;
* несколько шаблонов ретроспектив;
* историю ретроспектив команды;
* роли `owner / moderator / participant`;
* автоматическое удаление старых комнат;
* экспорт результатов;
* несколько команд в рамках одного аккаунта.

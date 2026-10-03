# Проектирование GraphQL API

## 1. Анализ REST-контракта

| REST endpoint | Ресурс | Операция |
|---|---|---|
| `GET /v1/clients/{id}` | Client | Получить клиента |
| `GET /v1/clients/{id}/documents` | Document[] | Получить документы |
| `GET /v1/clients/{id}/relatives` | Relative[] | Получить родственников |

Ключевые ресурсы: **Client**, **Document**, **Relative**.
Операции: 3 × GET (read-only).

## 2. Сопоставление REST → GraphQL

| REST | GraphQL |
|---|---|
| `GET /v1/clients/{id}` | `query { client(id) { ... } }` |
| `GET /v1/clients/{id}/documents` | `query { documents(clientId) { ... } }` |
| `GET /v1/clients/{id}/relatives` | `query { relatives(clientId) { ... } }` |
| — | `query { clients(first, after) { ... } }` (расширение) |
| — | `mutation { updateClient / addDocument / addRelative / ... }` (расширение) |

## 3. Что даёт GraphQL

### Устранение over-fetching
Клиент запрашивает ровно те поля, которые нужны экрану. При 500 атрибутах это критично — не тянем всё.

### Устранение under-fetching
Один запрос заменяет N REST-вызовов для связанных ресурсов (client + documents + relatives).

### Снижение RPS
3 REST-вызова → 1 GraphQL-запрос. При пике это ощутимо снижает нагрузку на client-info.

### Гибкость без роста API
Не нужно добавлять новый endpoint под каждый сценарий продажи. Клиент сам комбинирует поля.

### Строгая типизация
SDL-схема = контракт. Интроспекция даёт автодополнение в IDE и генерацию клиентов.

### Пагинация и фильтры
Relay-style connections позволяют листать большие списки документов/родственников без перегрузки ответа.

## 4. Решения по схеме

- **Client.documents** и **Client.relatives** — вложенные поля. Это ключевое преимущество: получить всё одним запросом.
- Дополнительно оставлены top-level `documents(clientId)` и `relatives(clientId)` — прямой эквивалент REST для совместимости.
- **Enum** для `DocumentType` и `RelationType` — строгая типизация вместо строк.
- **Relay connections** для пагинации — стандарт GraphQL-сообщества.
- **Payload-типы с UserError** для мутаций — стандарт для предсказуемой обработки бизнес-ошибок (вместо исключений).
- **Расширение Mutation** — в REST контракте операций записи нет, но GraphQL-схема сразу готова к их добавлению без breaking changes.

## 5. Оптимизация на стороне сервера

- **DataLoader** для батчинга N+1: если клиент запрашивает 100 клиентов и для каждого documents — не будет 100 отдельных SQL-запросов.
- **Query depth / complexity limit** — защита от тяжёлых запросов.
- **Persisted queries** — кэширование часто используемых запросов на CDN/edge.
- **@defer / @stream** (если поддерживается) — отдавать базовые поля сразу, тяжёлые — по мере готовности.

## 6. Что НЕ меняется

- Декомпозиция сервисов и БД — прежняя.
- HTTP/REST до внешнего API для B2B остаётся (GraphQL используется только между внутренними потребителями и client-info).
- Контракт БД — прежний.
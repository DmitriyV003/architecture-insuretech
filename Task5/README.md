# Task 5. GraphQL API для client-info

Файл схемы: `schema.graphql` (валидна по graphql-core).

## Анализ REST-контракта (Swagger)
3 операции чтения и 3 сущности:

| REST-операция | Возвращает |
|---|---|
| `GET /clients/{id}` | `Client { id, name, age }` |
| `GET /clients/{id}/documents` | `[Document { id, type, number, issueDate, expiryDate }]` |
| `GET /clients/{id}/relatives` | `[Relative { id, relationType, name, age }]` |

Проблема: под один сценарий нужно несколько объектов → несколько отдельных REST-запросов → рост RPS
client-info; при этом один «толстый» ресурс на все ~500 атрибутов отдавать нельзя (много лишних данных).

## Маппинг REST → GraphQL

| REST | GraphQL |
|---|---|
| `GET /clients/{id}` | `Query.client(id)` + поля типа `Client` |
| `GET /clients/{id}/documents` | `Client.documents` (вложенно) или `Query.clientDocuments(clientId)` |
| `GET /clients/{id}/relatives` | `Client.relatives` (вложенно) или `Query.clientRelatives(clientId)` |

## Как GraphQL снимает проблему
- **Один запрос вместо N.** Документы и родственники — вложенные поля `Client`, поэтому весь нужный
  набор берётся за один round-trip:
  ```graphql
  query {
    client(id: "1") {
      name
      documents { type number expiryDate }
      relatives { name relationType }
    }
  }
  ```
- **Ровно нужные поля** (нет over-fetching): потребитель выбирает поля под свой сценарий — веб-приложение
  и core-app запрашивают разные подмножества из одной схемы, без дублирования ресурсов.
- **Только документы** (без остальной карточки) — прямой запрос:
  ```graphql
  query { clientDocuments(clientId: "1") { type number issueDate } }
  ```
- **Расширяемость** до ~500 атрибутов: новые под-ресурсы (contacts и т.д.) добавляются полями/типами
  тем же приёмом, без роста числа эндпоинтов.

## Заметки по проектированию
- `id` — non-null; прочие поля nullable (в Swagger 2.0 required не задан).
- Списки `[T!]!` — всегда массив (пустой вместо null).
- `issueDate`/`expiryDate` — `String` (как в Swagger); при желании заменить на кастомный scalar `Date`.
- Контракт read-only → только `Query`. `Mutation` добавится при появлении операций изменения
  (редактирование в личном кабинете).

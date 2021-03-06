# HTTP API Design Guide

## Вступление

Это руководство описывает методы проектирования HTTP+JSON API, полученные
в процессе работы над [Heroku Platform API](https://devcenter.heroku.com/articles/platform-api-reference).
This guide describes a set of HTTP+JSON API design practices, originally
extracted from work on the [Heroku Platform API](https://devcenter.heroku.com/articles/platform-api-reference).

Это руководство информирует о дополнениях в этом API и
рассказывает о новых внутренних API Heroku. Мы надеемся, что
что это будет интересно разработчикам, не являющимся частью команды Heroku.
This guide informs additions to that API and also guides new internal
APIs at Heroku. We hope it’s also of interest to API designers
outside of Heroku.


Наша цель здесь - последовательность, сосредоточение на бизнес-логике
и избежание бесполезной траты времени на обсуждение незначительных технических деталей
во время проектирования архитектуры.
<!-- NOTE см. Parkinson's law of triviality -->
Our goals here are consistency and focusing on business logic while
avoiding design bikeshedding.
Мы ищем _хороший, последовательный,
хорошо документированный способ_ проектирования API,
но не обязательно _единственный/идеальный вариант_
We’re looking for _a good, consistent,
well-documented way_ to design APIs, not necessarily _the only/ideal
way_.

Мы предполагаем, что вы знакомы с основами HTTP+JSON API и не будем
рассказывать о них в этом руководстве.
We assume you’re familiar with the basics of HTTP+JSON APIs and won’t
cover all of the fundamentals of those in this guide.

Мы приветствуем [дополнения](CONTRIBUTING.md) к этому руководству.
We welcome [contributions](CONTRIBUTING.md) to this guide.

## Содержание

*  [Возвращайте соответствующие коды состояния](#return-appropriate-status-codes)
*  [Предоставляйте ресурс полностью, когда это возможно](#provide-full-resources-where-available)
*  [Принимайте сериализованный JSON в теле запроса](#accept-serialized-json-in-request-bodies)
*  [Предоставляйте (UU)ID ресурса](#provide-resource-uuids)
*  [Предоставляйте стандартные timestamp'ы](#provide-standard-timestamps)
*  [Используйте время в стандарте UTC отформатированное согласно ISO8601](#use-utc-times-formatted-in-iso8601)
*  [Используйте однообразное форматирование путей](#use-consistent-path-formats)
*  [Указывайте пути и атрибуты в нижнем регистре](#downcase-paths-and-attributes)
*  [Делайте атрибуты связанных ассоциаций вложенными](#nest-foreign-key-relations)
*  [Оставляйте возможность получение ресурса не только по идентификатору](#support-non-id-dereferencing-for-convenience)
*  [Генерируйте структурированные ошибки](#generate-structured-errors)
*  [Поддерживайте кэширование с помощью Etags](#support-caching-with-etags)
*  [Отслеживайте запросы с помощью Request-Id](#trace-requests-with-request-ids)
*  [Организуйте постраницную разбивку с помощью диапазонов](#paginate-with-ranges)
*  [Показывайте состояние ограничения частоты запросов](#show-rate-limit-status)
*  [Указывайте версию с помощью заголовков Accept](#version-with-accepts-header)
*  [Минимизируйте вложенность путей](#minimize-path-nesting)
*  [Предоставляйте машиночитаемую JSON схему](#provide-machine-readable-json-schema)
*  [Предоставляйте человекочитаемую документацию](#provide-human-readable-docs)
*  [Предоставляйте примеры использования](#provide-executable-examples)
*  [Предоставляйте информацию о стабильности](#describe-stability)
*  [Требуйте использование TLS](#require-tls)
*  [Форматируйте JSON по умолчанию](#pretty-print-json-by-default)

### Возвращайте соответствующие коды состояния
### Return appropriate status codes

Возвращайте соответствующие коды состояния HTTP с каждым
ответом. Успешные запросы должны возвращать коды в соответствии со
следующими соглашениями:

Return appropriate HTTP status codes with each response. Successful
responses should be coded according to this guide:

* `200`: Для успешно выполненного `GET` запроса и `DELETE` или
  `PATCH` запроса, выполненного синхронно
* `200`: Request succeeded for a `GET` calls, and for `DELETE` or
  `PATCH` calls that complete synchronously
* `201`: Успешно выполненный синхронный `POST` запрос
* `201`: Request succeeded for a `POST` call that completes
  synchronously
* `202`: Успешно завершенный `POST`, `DELETE`, или `PATCH` запрос,
  обработанный асинхронно
* `202`: Request accepted for a `POST`, `DELETE`, or `PATCH` call that
  will be processed asynchronously
* `206`: Успешно выполненный `GET` запрос, для которого был возвращен частичный ответ: see [above on ranges](#paginate-with-ranges)
* `206`: Request succeeded on `GET`, but only a partial response
  returned: см. [далее про диапазоны](#paginate-with-ranges)

Обращайтесь к [спецификации кодов возврата HTTP](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)
за рекомендациями по кодам возврата для пользовательских ошибок и ошибок сервера.

Refer to the [HTTP response code spec](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)
for guidance on status codes for user error and server error cases.

### Предоставляйте ресурс полностью, когда это возможно
### Provide full resources where available

Предоставляйте в ответе полное представление ресурса (т.е. объект со всеми
атрибутами) когда это возможно. Всегда возвращайте полный ресурс в ответах
с кодами 200 и 201, включая `PUT`/`PATCH` и `DELETE` запросы, например:

Provide the full resource representation (i.e. the object with all
attributes) whenever possible in the response. Always provide the full
resource on 200 and 201 responses, including `PUT`/`PATCH` and `DELETE`
requests, e.g.:

```
$ curl -X DELETE \
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

Ответы с кодом 202 не должны включать полное представление ресурса, например:

202 responses will not include the full resource representation,
e.g.:

```
$ curl -X DELETE \
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

### Принимайте сериализованный JSON в теле запроса
### Accept serialized JSON in request bodies

Принимайте сериалиpованный JSON в теле `PUT`/`PATCH`/`POST` запросов
либо вместо, либо в дополнение к данным из формы. Это создает симметрию
с сериализованным в JSON телом запроcа, например:

Accept serialized JSON on `PUT`/`PATCH`/`POST` request bodies, either
instead of or in addition to form-encoded data. This creates symmetry
with JSON-serialized response bodies, e.g.:

```
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

### Предоставляйте (UU)ID ресурса
### Provide resource (UU)IDs


Дайте каждому ресурсу атрибут `id` по умолчанию. Используйте UUID всегда,
если только у вас нет веской причины не делать этого. Не используйте
идентификаторы, которые не являются уникальными между всеми экземплярами
сервиса или могут совпадать с идентификаторами другого ресурса в сервисе,
особенно автоматически увеличиваемые идентификаторы.

Give each resource an `id` attribute by default. Use UUIDs unless you
have a very good reason not to. Don’t use IDs that won’t be globally
unique across instances of the service or other resources in the
service, especially auto-incrementing IDs.

Отображайте UUID в нижнем регистре, в формате `8-4-4-4-12`, например:

Render UUIDs in downcased `8-4-4-4-12` format, e.g.:

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

### Предоставляйте стандартные timestamp'ы
### Provide standard timestamps

По умолчанию предоставляйте временные метки (timestamp) `created_at` и `updated_at`
ресурса, например:

Provide `created_at` and `updated_at` timestamps for resources by default,
e.g:

```json
{
  ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  ...
}
```

Эти временные метки могут не иметь смысла для некоторых ресурсов и, в
таком случае, могут быть опущены.

These timestamps may not make sense for some resources, in which case
they can be omitted.

### Используйте время в стандарте UTC отформатированное согласно ISO8601
### Use UTC times formatted in ISO8601

Принимайте и возвращайте время только в стандарте UTC. Отображайте
время, отформатированное согласно спецификации ISO8601, например:

Accept and return times in UTC only. Render times in ISO8601 format,
e.g.:

```
"finished_at": "2012-01-01T12:00:00Z"
```

### Используйте однообразное форматирование путей
### Use consistent path formats

#### Именование ресурсов
#### Resource names

Используйте множественное число в имени ресурса, за исключением тех случаев, когда ресурс представлен в системе в единственном числе (например, в большинстве систем у пользователя только одна учетная запись). Это позволит сделать доступ к конкретным ресурсам однообразным, например:

```
/books
/posts/{id}/comments
```

Use the plural version of a resource name unless the resource in question is a singleton within the system (for example, in most systems a given user would only ever have one account). This keeps it consistent in the way you refer to particular resources.

#### Действия
#### Actions

Отдавайте предпочтение таким схемам бэкенда, которые не требуют специальных
действий для отдельных ресурсов. В случаях, когда специальные действия
необходимы, располагайте их после стандартного префикса `actions`, чтобы
четко отделить их:

Prefer endpoint layouts that don’t need any special actions for
individual resources. In cases where special actions are needed, place
them under a standard `actions` prefix, to clearly delineate them:

```
/resources/:resource/actions/:action
```

например
e.g.

```
/runs/{run_id}/actions/stop
```


### Указывайте пути и атрибуты в нижнем регистре
### Downcase paths and attributes

Указывайте пути в нижнем регистре, разделяя слова знаком [дефис](https://ru.wikipedia.org/wiki/%D0%94%D0%B5%D1%84%D0%B8%D1%81),
таким же образом, как пишутся адреса хостов, например:
Use downcased and dash-separated path names, for alignment with
hostnames, e.g:

```
service-api.com/users
service-api.com/app-setups
```

Имена атрибутов также рекомендуется писать в нижнем регистре, только
использовать в качестве разделителя подчеркивание, чтобы они
могли быть написаны в JavaScript без кавычек, например:
Downcase attributes as well, but use underscore separators so that
attribute names can be typed without quotes in JavaScript, e.g.:

```
service_class: "first"
```

### Делайте атрибуты связанных ассоциаций вложенными
### Nest foreign key relations

Сериализуйте внешние ключи во вложенные объекты, например:
Serialize foreign key references with a nested object, e.g.:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  ...
}
```

Вместо:
Instead of e.g:

```json
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  ...
}
```

Этот подход позволит добавить дополнительную информацию о
связанном ресурсе без изменения структуры ответа, например:
This approach makes it possible to inline more information about the
related resource without having to change the structure of the response
or introduce more top-level response fields, e.g.:

```json
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  ...
}
```

### Оставляйте возможность получение ресурса не только по идентификатору
### Support non-id dereferencing for convenience

В некоторых случаях конечным пользователям может быть неудобно использовать
идентификаторы для получения ресурса. Например, пользователь может думать
о приложениях Heroku используя их имена, но приложения могут идентифицироваться
по UUID. В этом случае вы можете реализовать получение ресурса и
по идентификатору и по имени, например:

In some cases it may be inconvenient for end-users to provide IDs to
identify a resource. For example, a user may think in terms of a Heroku
app name, but that app may be identified by a UUID. In these cases you
may want to accept both an id or name, e.g.:

```
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

Но не нужно использовать только имена, исключая доступ по идентификатору.

Do not accept only names to the exclusion of IDs.

### Генерируйте структурированные ошибки
### Generate structured errors

Генерируйте единообразные структурированные ответы в случае ошибки.
Включайте идентификатор ошибки в поле `id`, описание в поле `message` и,
опционально, поле `url` указывающее клиенту на
дополнительную информацию об ошибке и способы ее устранения, например:

Generate consistent, structured response bodies on errors. Include a
machine-readable error `id`, a human-readable error `message`, and
optionally a `url` pointing the client to further information about the
error and how to resolve it, e.g.:

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

Задокументируйте формат тех ошибок и возможных `id` с которыми может
встретиться пользователь.

Document your error format and the possible error `id`s that clients may
encounter.

### Поддерживайте кэширование с помощью Etags
### Support caching with Etags

Включайте заголовок [ETag](http://ru.wikipedia.org/wiki/HTTP_ETag) во все
ответы для идентификации версии возвращенного ресурса. Пользователь
должен иметь возможность проверить актуальность данных в последующих
запросах с помощью заголовка
[`If-None-Match`](http://ru.wikipedia.org/wiki/HTTP_ETag#.D0.A2.D0.B8.D0.BF.D0.B8.D1.87.D0.BD.D0.BE.D0.B5_.D0.B8.D1.81.D0.BF.D0.BE.D0.BB.D1.8C.D0.B7.D0.BE.D0.B2.D0.B0.D0.BD.D0.B8.D0.B5).

Include an `ETag` header in all responses, identifying the specific
version of the returned resource. The user should be able to check for
staleness in their subsequent requests by supplying the value in the
`If-None-Match` header.

### Отслеживайте запросы с помощью Request-Id
### Trace requests with Request-Ids

Включайте заголовок `Request-Id` с UUID в каждый ответ. Если и сервер
и клиент логируют значение этого заголовка в дальнейшем это может помочь
в отслеживании и отладке запросов.

Include a `Request-Id` header in each API response, populated with a
UUID value. If both the server and client log these values, it will be
helpful for tracing and debugging requests.

### Организуйте постраничную разбивку с помощью диапазонов
### Paginate with Ranges

Разбивайте на страницы ответы, которые могут вернуть большое количество
данных. Используйте заголовок `Content-Range` для обработки постраничных
запросов. В разделе [Heroku Platform API on Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges)
вы можете найти информацию про заголовки запросов и ответов, коды состояния,
ограничения, сортировку и переходы по страницам.

Paginate any responses that are liable to produce large amounts of data.
Use `Content-Range` headers to convey pagination requests. Follow the
example of the [Heroku Platform API on Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges)
for the details of request and response headers, status codes, limits,
ordering, and page-walking.

### Показывайте состояние ограничения частоты запросов
### Show rate limit status

Ограничивайте количество запросов от клиентов, чтобы поддерживать качество
сервиса. Вы можете использовать [token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket),
чтобы ограничивать колчество запросов.

Rate limit requests from clients to protect the health of the service
and maintain high service quality for other clients. You can use a
[token bucket algorithm](http://en.wikipedia.org/wiki/Token_bucket) to
quantify request limits.

Возвращайте оставшееся количество запросов с каждым ответом в заголовке
`RateLimit-Remaining`.

Return the remaining number of request tokens with each request in the
`RateLimit-Remaining` response header.

### Указывайте версию с помощью заголовков Accept
### Version with Accepts header

Версионируйте API с самого начала. Используйте заголовок `Accept` чтобы
сообщить версию вместе с нестандартным типом содержимого, например:

Version the API from the start. Use the `Accepts` header to communicate
the version, along with a custom content type, e.g.:

```
Accept: application/vnd.heroku+json; version=3
```

Предпочтительнее не иметь значения по умолчанию и, вместо этого,
требовать у клиентов явного указания версии в каждом запросе.

Prefer not to have a default version, instead requiring clients to
explicitly peg their usage to a specific version.

### Уменьшайте вложенность путей
### Minimize path nesting

Для моделей данных с вложенными родительскими/дочерними ресурсами пути
могут иметь большую вложенность, например:

In data models with nested parent/child resource relationships, paths
may become deeply nested, e.g.:

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

Ограничивайте вложенность располагая ресурсы в корневом пути.
Используйте вложенность, чтобы обозначить область видимости коллекции.
Например, когда dyno принадлежит приложению, а приложение организации:

Limit nesting depth by preferring to locate resources at the root
path. Use nesting to indicate scoped collections. For example, for the
case above where a dyno belongs to an app belongs to an org:

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### Предоставляйте машиночитаемую JSON схему
### Provide machine-readable JSON schema

Предоставляйте машиночитаемую схему, чтобы описать API. Используйте
[prmd](https://github.com/interagent/prmd) для управления схемой и
проверки ее правильности с помощью `prmd verify`.

Provide a machine-readable schema to exactly specify your API. Use
[prmd](https://github.com/interagent/prmd) to manage your schema, and ensure
it validates with `prmd verify`.

### Предоставляйте документацию
### Provide human-readable docs

Предоставляйте документацию, которую сторонние разработчики смогут использовать,
чтобы лучше понять ваш API
Provide human-readable documentation that client developers can use to
understand your API.

Если вы описываете схему используя prmd, как говорилось выше, вы можете легко
генерировать документацию в формате Markdown для всех бэкендов с помощью `prmd doc`.
If you create a schema with prmd as described above, you can easily
generate Markdown docs for all endpoints with with `prmd doc`.

Кроме деталей бэкенда, делайте обзор API, содержащий
информацию о таких вещах, как:
In addition to endpoint details, provide an API overview with
information about:

* Аутентификация, в том числе acquiring и использование токенов аутентификации
* Authentication, including acquiring and using authentication tokens.

* Стабильность и версионирование API, в том числе то, как выбрать желаемую версию API
* API stability and versioning, including how to select the desired API
  version.

* Распространенные заголовки запросов и ответов
* Common request and response headers.

* Формат сериализации ошибок
* Error serialization format.

* Примеры использования API с клиентами на разных языках
* Examples of using the API with clients in different languages.

### Предоставляйте исполняемые примеры
### Provide executable examples

Предоставляйте примеры, показывающие вызовы API в действии,
которые можно вставить и выполнить в терминале.
В идеале, примеры должны быть такими, чтобы их можно было использовать дословно.
Таким образом, вы упростите использование API для пользователя.
Provide executable examples that users can type directly into their
terminals to see working API calls. To the greatest extent possible,
these examples should be usable verbatim, to minimize the amount of
work a user needs to do to try the API, e.g.:

```
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

Если вы используете [prmd](https://github.com/interagent/prmd),
то примеры будут сгененированы вместе с документацией в формате Markdown.
If you use [prmd](https://github.com/interagent/prmd) to generate Markdown
docs, you will get examples for each endpoint for free.

### Предоставляйте информацию о стабильности
### Describe stability

Предоставляйте информацию о стабильности API в соответствии со степенью его
готовности и стабильности, например с помощью флагов протопип/в разработке/продакшн.
Describe the stability of your API or its various endpoints according to
its maturity and stability, e.g. with prototype/development/production
flags.

Просмотрите [политику совместимости API Heroku](https://devcenter.heroku.com/articles/api-compatibility-policy),
чтобы узнать о возможных подходах в управлении изменениями и стабильностью.
See the [Heroku API compatibility policy](https://devcenter.heroku.com/articles/api-compatibility-policy)
for a possible stability and change management approach.

Если ваш API заявлен как стабильный и готовый к использованию в продакшене,
не вносите изменений, нарушающих обратную совместимость внутри одной версии.
Если вам нужно сделать такие изменения, создайте новую версию API.
Once your API is declared production-ready and stable, do not make
backwards incompatible changes within that API version. If you need to
make backwards-incompatible changes, create a new API with an
incremented version number.

### Требуйте использование TLS
### Require TLS

Требуйте использование TLS для доступа к API без исключений. Не стоит тратить время на то,
чтобы попытаться понять или объяснить, когда использовать TLS, а когда нет.
Просто требуйте использование TLS для всего.
Require TLS to access the API, without exception. It’s not worth trying
to figure out or explain when it is OK to use TLS and when it’s not.
Just require TLS for everything.

### Форматируйте JSON по умолчанию
### Pretty-print JSON by default

Вероятно, в первый раз пользователь воспрользуется вашим API через командную строку
с помощью curl. Намного легче понять ответ API в командной строке, если он
будет отформатирован, например:
The first time a user sees your API is likely to be at the command line,
using curl. It’s much easier to understand API responses at the
command-line if they are pretty-printed. For the convenience of these
developers, pretty-print JSON responses, e.g.:

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

Вместо:
Instead of e.g.:

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

Убедитесь, что вставляете дополнительную пустую строку в конце ответа, чтобы
не портить внешний вид командной строки пользователя.
Be sure to include a trailing newline so that the user’s terminal prompt
isn’t obstructed.

Для большинства API форматирование ответов не отражается на производительности.
Вы можете решить не форматировать ответы некоторых бэкендов, чувствительных
к производительности (например, это касается систем с очень большим трафиком),
или не делать этого для некоторых клиентов (например, если известно,
что они будут использоваться программами без GUI).
For most APIs it will be fine performance-wise to pretty-print responses
all the time. You may consider for performance-sensitive APIs not
pretty-printing certain endpoints (e.g. very high traffic ones) or not
doing it for certain clients (e.g. ones known to be used by headless
programs).

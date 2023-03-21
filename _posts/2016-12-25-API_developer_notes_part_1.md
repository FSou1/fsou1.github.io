---
title: Заметки ASP.NET Web API/Backend разработчика (часть 1)
tags: ASP.NET OWIN IIS MSSQL NHibernate
redirect_from: "/API_developer_notes_part_1/"
---

В сегодняшней небольшой заметке хотел бы поделиться некоторыми вещами, с которыми столкнулся за последнее время и о которых бы не хотелось забыть.

## Бесконечный refresh token в oauth 2 (Microsoft.Owin)

При реализации авторизации с использованием oauth 2 [Refresh token](https://tools.ietf.org/html/rfc6749#section-1.5) используется для обновления выданного ранее [access token](https://tools.ietf.org/html/rfc6749#section-1.4). Срок жизни (expiration time) refresh token'a обычно больше срока жизни access token'a, однако, мне захотелось сделать его бесконечным (по аналогии с [OAuth 2.0 to access google APIs](https://developers.google.com/identity/protocols/OAuth2)) (его срок протухания наступает в тот момент, когда его отзовут вручную).

Однако в реализации Microsoft этого сделать нельзя ввиду [следующего условия при валидации токена](https://github.com/jchannon/katanaproject/blob/24aa43641adac57cb5bd523b77bdc0ced9f0455d/src/Microsoft.Owin.Security.OAuth/OAuthAuthorizationServerHandler.cs#L627-L633).

По умолчанию срок его жизни устанавливается таким же, как и срок жизни access token'a, однако, есть возможность его изменить (сделать больше или меньше). В случае же присваивания ему значения null, сам токен станет невалидным (попытка продлить им access token будет безуспешной).

В итоге я остановился на варианте 1 месяц для access и 1 год для refresh.

## Заглушка для IIS приложений

Время от времени бывает необходимо все запросы к ASP.NET приложениям, которые хостятся с использованием IIS, перенаправлять на некую статическую страницу, например:

1. На сайте ведутся работы;
2. В данный момент сайт недоступен, попробуйте позже;
3. Страница недоступности сайта (для симуляции экстренного выключения).

Самым простым и автоматизируемым решением, на мой взгляд, является размещение файла с именем app_offline.htm в корне приложения. В этом случае произойдёт следующее:

1. Приложение будет завершено;
2. Все занимаемые приложением ресурсы будут освобождены;
3. Все новые входящие запросы не будут обработаны;

Главное не забудьте удалить этот файл!

## NHibernate и nvarchar(max)

К сожалению (или к счастью) NHibernate по умолчанию не умеет работать с некоторыми типами MS SQL (будь-то datetime2 или nvarchar(max)). В этом случае придётся на соответствующем поле вызвать `CustomType` и `CustomSqlType`, где явно указать, какому типу в базе данных соответствует данное поле:

```csharp
Map(x => x.Description).CustomType("StringClob").CustomSqlType("nvarchar(max)");
```

## SCOPE_IDENTITY() for Guids (uniqueidentifier)

В том случае, когда вы используете Guid'ы (mssql- uniqueidentifier) и вам требуется соблюдение их уникальности (например, в случае с primary key), вам потребуется функция, которая будет формировать уникальные ключи (SCOPE_IDENTITY()). Есть 2 варианта это сделать:

### Генерация на стороне mssql

Для этого используем встроенную функцию `newsequentialid()`:

```sql
CREATE TABLE dbo.GuidTest (
    GuidColumn uniqueidentifier NOT NULL DEFAULT NewSequentialID(),
    IntColumn int NOT NULL
)
```

### Генерация на стороне ORM

В случае с NHibernate эту функцию можно задать с использованием маппинга `Id`:

```csharp
Id(p => p.Id).GeneratedBy.GuidComb();
```

Вариант с использованием `GuidComb()` является более приоритетным ввиду повышения производительности базы данных при вставках новых записей вследствие уменьшения фрагментации индексов, т.к. элементы последовательности получаются близкими по значению.

В случае с Entity Framework:

```csharp
[Key]
[DatabaseGenerated(DatabaseGeneratedOption.Identity)]
public Guid Id { get; set; }
```

Используемые материалы:

1. [The length of the string value exceeds the length configured in the mapping/parameter](http://stackoverflow.com/questions/12708171/the-length-of-the-string-value-exceeds-the-length-configured-in-the-mapping-para);
2. [Expiration of refresh tokens Microsoft.Owin](http://stackoverflow.com/questions/41066253/expiration-of-refresh-tokens-microsoft-owin/41144221#41144221);
3. [Katanaproject](https://github.com/jchannon/katanaproject);
4. [ASP.NET 2.0 - How to use app_offline.htm](http://stackoverflow.com/questions/1153449/asp-net-2-0-how-to-use-app-offline-htm);
5. [SCOPE_IDENTITY() for GUIDs?](http://stackoverflow.com/questions/1509947/scope-identity-for-guids).

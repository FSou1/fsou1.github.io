---
title: NHibernate & JSON mapping- реализуем собственный CustomType (IUserType)
tags: .NET NHibernate JSON
redirect_from: "/NHibernate_custom_type_implementation_for_json/"
---

В последнее время у меня часто возникают ситуации, когда необходимо сохранить произвольные наборы данных, которые, с большей долей вероятности, не понадобятся при выборках (если мы говорим про отчёты или срезы) и не будут участвовать в фильтрациях и объединениях таблиц (говоря про join'ы, фильтры, группировки).

К таким данным, например, относятся:

- контекст для конечного автомата ([state machine](https://en.wikipedia.org/wiki/Finite-state_machine) / [state design pattern](https://en.wikipedia.org/wiki/State_pattern)) - при попытке перехода к следующему состоянию первым делом восстанавливается текущее состояние из контекста;
- история изменений / snapshot сущностей - актуальная и исторические версии данных редко хранятся в одинаковом формате и одном и том же месте- для исторических версий зачастую выбирают отдельный формат или хранилище;
- входные параметры для возобновляемых сценариев - для возможности возобновлять выполнение сценариев с момента их остановки (п.1) или их повторного запуска (по прошествию некоторого времеми или в случае возникновения ошибок исполнения);

В случае с реляционными базами данных для решения подобных задач я предпочитаю использовать хранение данных в ненормализованном виде, например, в формате JSON (в рамках данной заметки формат хранения не имеет принципиального значения и может быть с лёгкостью заменён путём использования собственных функций сериализации и десериализации). Не буду спорить, что для JSON (или XML) существуют более удобные и для этого приспособленные решения, в которых этот формат является нативным (те же NoSQL решения, в частности MongoDB или Cassandra), однако, и в реляционных базах данных (особенно с учётом введения поддержки JSON в MSSQL 2016 и PostgreSQL) этот подход имеет право на жизнь.

## Учим NHibernate понимать JSON

Одним из способов научить NHibernate взаимодействовать с данными в требуемом нам формате является реализация собственного типа, в котором и будут реализованы операции распознования (десериализации) и подготовки к записи (сериализации). Для этого потребуется унаследовать наш тип от интерфейса [IUserType](https://github.com/nhibernate/nhibernate-core/blob/b08e2b9a94b75fed21a455daaef320cb0df105a3/src/NHibernate/UserTypes/IUserType.cs) и реализовать необходимые методы:

```csharp
public class JsonMappableType<T> : IUserType where T : class
{
    public new bool Equals(object x, object y)
    {
        if (x == null && y == null)
            return true;

        if (x == null || y == null)
            return false;

        return JsonFormatter.Serialize(x) == JsonFormatter.Serialize(y);
    }

    public int GetHashCode(object x)
    {
        if (x == null)
            return 0;

        return x.GetHashCode();
    }

    public object NullSafeGet(IDataReader rs, string[] names, object owner)
    {
        if(names.Length != 1)
            throw new InvalidOperationException("Expect only one column");

        var val = rs[names[0]] as string;

        if (!string.IsNullOrWhiteSpace(val))
        {
            return JsonFormatter.Deserialize<T>(val);
        }

        return null;
    }

    public void NullSafeSet(IDbCommand cmd, object value, int index)
    {
        var parameter = (DbParameter) cmd.Parameters[index];

        parameter.Value = value == null
            ? (object) DBNull.Value
            : JsonFormatter.Serialize(value);
    }

    public object DeepCopy(object value)
    {
        if (value == null)
            return null;

        var serialized = JsonFormatter.Serialize(value);

        return JsonFormatter.Deserialize<T>(serialized);
    }

    public object Replace(object original, object target, object owner)
    {
        return original;
    }

    public object Assemble(object cached, object owner)
    {
        var str = cached as string;

        if (string.IsNullOrWhiteSpace(str))
            return null;

        return JsonFormatter.Deserialize<T>(str);
    }

    public object Disassemble(object value)
    {
        if (value == null)
            return null;

        return JsonFormatter.Serialize(value);
    }

    public SqlType[] SqlTypes
        => new SqlType[] {new StringClobSqlType() };

    public Type ReturnedType
        => typeof(T);

    public bool IsMutable
        => true;
}
```

Пример использования:

```csharp
public class Scenario {
    public virtual ISet<Call> Calls { get; set; }
}

public class ScenarioMap : ClassMap<Scenario> {
    public ScenarioMap() {
        Map(x => x.Calls).CustomType<JsonMappableType<ISet<Call>>>();
    }
}
```

Для полноты картины, на всякий случай, приведу содержимое класса `Call`:

```csharp
public class Call
{
    public virtual long Id { get; set; }
    public virtual decimal Cost { get; set; }
    public virtual int Duration { get; set; }
    public virtual DateTime StartTimeUtc { get; set; }
}
```

### Будьте готовы к изменениям

Хочу отдельно отметить и, возможно, предостеречь вас от вероятных неприятностей, связанных с изменением структуры данных.

Предположим, что сейчас всё что вам нужно- это сохранять коллекцию звонков, и вы решили последовать примеру выше. Всё работает замечательно и вы радуетесь жизни ровно до тех пор, пока не поступит новое требование - кроме информации о звонках наш сценарий должен так же хранить коллекцию сообщений, sms уведомлений или чего угодно ещё (не стоит забывать о том, какие данные и для каких нужд мы храним). В этот момент лично я могу себе представить только 2 пути:

- добавить ещё одно поле для хранения новой информации;
- начать использовать агрегаты;

В связи с этим я настоятельно рекомендую вам использовать агрегаты для всех данных, которые вы захотите хранить в ненормализованном виде. В случае со звонками это может выглядеть следующим образом:

```csharp
public class Scenario
{
    public virtual ScenarioData Data { get; set; }
}

public class ScenarioData
{
    public ISet<Call> Calls { get; set; }
    public ISet<Message> Messages { get; set; }

    /* агрегат позволит расширять (вносить любые изменения в) нашу структуру данных */
}

public class ScenarioMap : ClassMap<Scenario>
{
    public ScenarioMap()
    {
        Map(x => x.Data)
            .Column("ScenarioData")
            .CustomType<JsonMappableType<ScenarioData>>();
    }
}
```

**Важно:**

- Использование в качестве `SqlType` типа `StringClobSqlType` позволяет использовать sql'ный тип данных `nvarchar(max)` (в случае использования `StringSqlType` максимальная длина строки будет ограничена 4000 символами);

Код доступен на [github gist](https://gist.github.com/FSou1/9e5b646ab4ccc47bf0fccda6c3a77d66).

Ссылки по теме:

- [JSON serialized object in NHibernate](http://blog.denouter.net/2015/03/json-serialized-object-in-nhibernate.html);
- [NHibernate Custom User Type for serializing a property to JSON](https://gist.github.com/phillip-haydon/1936188).

---
title: Десериализация нетипизированного JSON поля на строготипизированный объект .NET с использованием Newtonsoft.Json
tags: .NET C# JSON
redirect_from: "/Deserialize_json_multiple_types_property/"
---

Встречали ли вы JSON объекты, поля которых не имеют строгой типизации, а точнее, тип которых может варьироваться от `string` и `numeric` до `Object`? Если ответ положительный, то вы могли заметить, что красотой решения десериализации подобных объектов не блещут, однако, об одном из них я бы и хотел сегодня написать.

## Задача

Внешний (неподконтрольный нам API) предоставляет следующий формат ответа, который нам требуется десериализовать на строготипизированный объект:

Ответ:

```javascript
{
   "contact":[
      {
         "type":"cell",
         "value":{
            "country":"7",
            "city":"123",
            "number":"4567890",
            "formatted":"+71234567890"
         }
      },
      {
         "type":"email",
         "value":"applicant@example.com"
      },
      {
          "type":"vk_id",
          "value": 12342341
      }
   ]
}
```

Как вы могли заметить, значение поля `value` объекта `contact` не имеет строго определённого типа:

1. Если тип контакта `cell` то тип значения `Object`;
2. Если тип контакта `email` то тип значения `string`;
3. Возможны так же значения типа `int` и `Array`;

В .NET нет возможности объявить тип, который будет единообразно хорошо и удобно поддерживать весь предполагаемый зоопарк значений, в связи с этим для каждого возможного варианта мы будем использовать собственный тип:

1. Для телефона- `class`;
2. Для email- `string`;
3. Для идентификатора- `long`;

# Решение с использованием CustomCreationConverter

Одним из возможных способов решения может быть написание собственного `CustomCreationConverter` (наследник `JsonConverter`), который отвечает непосредственно за создание объекта на этапе десериализации, generic типом у которого будет указан интерфейс `IContact`. В этом случае на этапе десериализации мы получим контроль над выполнением и сможем реализовать создание экземпляра требуемого нам типа контакта основываясь на значении `type`.

Делается это в 3 шага:

## Модель

Для строгой типизации была объявлена следующая модель, состаящая из:

1. root объекта - `Resume`;
2. интерфейса `IContact` на место которого будут подставляться конкретные реализации;
3. реализации контактов: `EmailContact`, `PhoneContact`, `VkContact`;

```csharp
public class Resume {
    public IList<IContact> Contacts { get; set; }
}

public interface IContact {
    string Type { get; set; }
}

public class EmailContact : IContact {
    public string Type { get; set; }
    public string Value { get; set; }
}

public class PhoneContact : IContact {
    public string Type { get; set; }
    public PhoneContactValue Value { get; set; }
}

public class VkContact : IContact {
    public string Type { get; set; }
    public int Value { get; set; }
}

public class PhoneContactValue {
    public string Code { get; set; }
    public string Number { get; set; }
    public int CountryId { get; set; }
}
```

## Реализация CustomCreationConverter

Пример реализации указан далее:

```csharp
public class ContactConverter : CustomCreationConverter<IContact> {
    public override IContact Create(Type objectType) {
        throw new NotImplementedException();
    }

    public override object ReadJson(JsonReader reader, Type objectType, object existingValue, JsonSerializer serializer) {
        var jObject = JObject.Load(reader);

        var target = CreateContact(jObject);

        return target;
    }

    /// <summary>
    /// Create concrete contact base on contactType value
    /// </summary>
    /// <param name="jObject"></param>
    /// <returns></returns>
    private IContact CreateContact(JObject jObject) {
        var contactType = (string)jObject.Property("type");
        var json = jObject.ToString();

        switch (contactType) {
            case "email": {
                return DeserializeObject<EmailContact>(json);
            }
            case "cell": {
                return DeserializeObject<PhoneContact>(json);
            }
            case "vk_id": {
                return DeserializeObject<VkContact>(json);
            }
            default:
                throw new NotSupportedException("Unexpected contactType: " + contactType);
        }
    }

    private T DeserializeObject<T>(string json) => JsonConvert.DeserializeObject<T>(json);
}
```

## Подключение

Теперь всё что осталось сделать, так это подключить реализованный ранее конвертер в настройках `JsonSerializer`:

```csharp
var resume = JsonConvert.DeserializeObject<Resume>(response, new ContactConverter());
```

## Результат

В результате получаем строготипизиронное представление конкретных контактов с удобной возможностью дальнейшего расширение (на случай появления новых типов контактов потребуется лишь объявить модель и добавить новое значение `type` в оператор `switch`).

![mt event](/assets/images/posts/deserialize_json_multiple.png)

Исходники доступны на [GitHubGist src](https://gist.github.com/FSou1/daeff50471419de025f7dab9c744df1c).

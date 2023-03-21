---
title: Конструкция «controller as» в AngularJS
tags: AngularJS JavaScript
redirect_from: "/AngularJS_controller_as/"
---

Сегодня хочу поделиться информацией о конструкции «controller as» в AngularJS, для чего его понадобилось добавлять и разобрать внутренее устройство его работы.

## Что это за конструкция «controller as»

Как уже было выше сказано,- это специализированный синтаксис описания контроллера (например, в директиве `ng-controller`), который позволяет результат вызова конструктора неявно присвоить в поле сопутствующего (локального) `$scope`.

Напомню, что при создании каждого контроллера, в функцию `$controller` передаётся объект `locals`, который содержит в себе поля `$scope`, `$element`, `$attrs` и `$transclude`, присущие экземпляру создаваемого контроллера.

Если ранее, чтобы отобразить данные, предоставляемые контроллером, нам необходимо было явно инжектить `$scope` и инициализировать его поле:

```javascript
app.controller("ProductCtrl", [
  "$scope",
  function ($scope) {
    $scope.products = ["Apple", "Banana", "Milk", "Coffee"];
  },
]);
```

{% raw %}

```html
<div ng-controller="ProductCtrl">
  <ul>
    <li ng-repeat="product in products">{{ product }}</li>
  </ul>
</div>
```

{% endraw %}

```
Apple
Banana
Milk
Coffee
```

То теперь, благодаря новому синтаксису, мы можем внутри контроллера оперировать указателем `this`, а результат будет автоматически присвоен полю локального `$scope`:

```javascript
app.controller("ProductCtrlAs", [
  function () {
    this.products = ["Apple", "Banana", "Milk", "Coffee"];
  },
]);
```

{% raw %}

```html
<div ng-controller="ProductCtrlAs as ctrl">
  <ul>
    <li ng-repeat="product in ctrl.products">{{ product }}</li>
  </ul>
</div>
```

{% endraw %}

```
Apple
Banana
Milk
Coffee
```

> Важно: В случае ручного вызова `$controller`, вторым параметром необходимо явно передавать объект, содержащий поле `$scope` (см. тест `'should throw an error if $scope is not provided'`).

## Для чего потребовался новый синтаксис

Необходимость данной возможности обусловлена несколькими причинами:

Общая область видимости `$scope`’ов вложенных контроллеров при пересечении наименований:

{% raw %}

```html
<div ng-controller="TopCtrl">
  {{ title || 'undefined;' }}

  <div ng-controller="MiddleCtrl">
    {{ title || 'undefined;' }}

    <div ng-controller="BottomCtrl">{{ title || 'undefined;' }}</div>
  </div>
</div>
```

{% endraw %}

В том случае, когда одна из переменных не инициализирована, то на её замену ищется одноимённая из родительского `$scope` (`$parent`), что может приводить к следующему результату:

```
I am top ctrl!
I am top ctrl! ($scope MiddleCtrl не содержит значение title, поэтому оно берётся из родительского $scope)
I am bottom ctrl!
```

Таким образом, в коде появляется неоднозначность и сложность отслеживания источника данных. Для этого, мы можем переписать код представления с использованием синтаксиса «controller as», явно указывая какая переменная из какого `$scope` должна браться:

{% raw %}

```html
<div ng-controller="TopCtrlAs as top">
  {{ top.title || 'undefined;' }}

  <div ng-controller="MiddleCtrlAs as middle">
    {{ middle.title || 'undefined;' }}

    <div ng-controller="BottomCtrlAs as bottom">
      {{ bottom.title || 'undefined;' }}
    </div>
  </div>
</div>
```

{% endraw %}

Как итог, переменная будет искаться исключительно в `$scope` связанного контроллера:

```
I am top ctrl!
undefined
I am bottom ctrl!
```

## Внутреннее устройство и реализация

На этапе создания экземпляра контроллера с помощью `$controllerProvider`, в том случае, когда первый параметр является строкой (строковое выражение контроллера), проводится поиск вхождения в ней частей регулярного выражения:

```javascript
var CNTRL_REG = /^(\S+)(\s+as\s+([\w$]+))?$/;

[0] TopCtrlAs as top
[1] TopCtrlAs
[2] as top
[3] top
```

С помощью первого (`[1] TopCtrlAs`) элемента (наименование контроллера) ищется зарегистрированный ранее в объекте `controllers` конструктор контроллера и создаётся его экземпляр.

При наличии третьего (`[3] top`) элемента (наименование поля в `$scope`, которому присвоить результат вызова конструктора), вызывается функция `addIdentifier`, которая делает простое присвоение:

```javascript
locals.$scope[identifier] = instance;
```

## Тестовое покрытие

Описанное выше поведение успешно покрыто следующими тестами:

Поиск зарегистрированного ранее контроллера, результат вызова конструктора присваивается в поле `foo` объекта `$scope`.

```javascript
it("should publish controller instance into scope", function () {
  var scope = {};

  $controllerProvider.register("FooCtrl", function () {
    this.mark = "foo";
  });

  var foo = $controller("FooCtrl as foo", { $scope: scope });
  expect(scope.foo).toBe(foo);
  expect(scope.foo.mark).toBe("foo");
});
```

Обязательность явной передачи локального объекта `$scope` в случае использования синтаксиса «controller as»:

```javascript
it("should throw an error if $scope is not provided", function () {
  $controllerProvider.register("a.b.FooCtrl", function () {
    this.mark = "foo";
  });

  expect(function () {
    $controller("a.b.FooCtrl as foo");
  }).toThrowMinErr(
    "$controller",
    "noscp",
    "Cannot export controller 'a.b.FooCtrl' as 'foo'! No $scope object provided via `locals`."
  );
});
```

## Бонусы

Как можно заметить из вышесказанного, если не возвращать из конструктора контроллера явно результат, то в локальную переменную `$scope` будет присвоен `this`. Однако, ничто не мешает нам сделать следующим образом и, всё же, вернуть результат:

```javascript
app.controller("MainCtrl", function () {
  this.message = "I should be in scope";

  return { message: "Wrong!" };
});
```

В таком случае, на экран будет выведено именно 'Wrong!'.

Так же, в процессе разбора исходников, наткнулся на довольно интересный тест, о котором не стоит забывать имея дело с объектами javascript:

```javascript
it('should throw an exception if a controller is called "hasOwnProperty"', function () {
  expect(function () {
    $controllerProvider.register("hasOwnProperty", function ($scope) {});
  }).toThrowMinErr(
    "ng",
    "badname",
    "hasOwnProperty is not a valid controller name"
  );
});
```

Т.к. регистрация и выявление зарегистрированных ранее контроллеров происходит с помощью объекта `controllers`, то запрещено иметь наименование контроллера `hasOwnProperty`, дабы в последствии при выявлении `controllers[constructor]` не вернуть встроенную функцию языка (или не перегрузить её конструктором контроллера),- весьма предусмотрительно.

См. также:

1. [Digging into Angular’s “Controller as” syntax](https://toddmotto.com/digging-into-angulars-controller-as-syntax/)

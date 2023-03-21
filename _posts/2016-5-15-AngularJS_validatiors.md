---
title: Кратко о валидаторах в AngularJS
tags: AngularJS JavaScript
redirect_from: "/AngularJS_validatiors/"
---

Очень часто, при разработке экранных форм (будь то форма регистрации, отправки обратной связи или ввода платежных реквизитов) право ввода информации, корректность которой нам просто необходимо затем проверить перед её дальнейшей обработкой, отдаётся в руки конечного пользователя.

Сам процесс проверки корректности введённой информации (соответствия формату накладываемых на неё ограничений) называется валидацией, а каждое из ограничений- валидатор.

## Примеры валидаторов в повседневной жизни

Примеров валидаторов огромное множество: это и соответствие формату адреса электронной почты, требования к длине строки или наличию/отсутствию в ней заданных символов.

По умолчанию angularjs (на момент 1.5.5) присутствуют 4 директивы, выполняющие роль валидаторов:

1. ngRequired - требование к обязательности заполнения;
2. ngMaxlength - требование к максимальной длине;
3. ngMinlength - требование к минимальной длине;
4. ngPattern - требование к соответствию формата, задаваемого с помощью регулярного выражения.

В большинстве случаев их хватает,- наиболее функциональной из них является ngPattern, благодаря которой кол-во требований ограничивается лишь вашей фантазией и средствами описания регулярных выражений.

## Когда хочется чего-нибудь эдакого

В тех исключительных случаях, когда нам необходимо написать свой собственный валидатор, мы можем это сделать с помощью определения собственной директивы.

Пускай мы хотим чтобы значение являлось числом, которое делится на 2 и на 7 без остатка.

### До версии 1.3

До выхода angularjs версии 1.3 мы можем сделать это следующим образом, добавив нашу логику проверки в массивы `$parsers` и `$formatters`, возвращая из функции либо `undefined`, либо корректное значение (это является одной из причин, почему не удастся установить некорректное начальное значение с помощью `ngModel` и почему я отказался от возврата `undefined`).

Так же нам потребуется вызывать функцию `$setValidity()`, передавая в неё наименование валидатора и результат проверки для дальнейшей обработки (например, отображения ошибок).

[**Jsfiddle demo**](https://jsfiddle.net/pnsdbzLr/1/):

```javascript
function numericDirective() {
  return {
    require: "ngModel",
    link: function (scope, elem, attr, ctrl) {
      var numericRegexp = /^\d+$/;

      var validator = function (viewValue) {
        var isValid =
          numericRegexp.test(viewValue) &&
          viewValue % 2 === 0 &&
          viewValue % 7 === 0;

        ctrl.$setValidity("numericOnly", isValid);

        // we should always return value and don't return undefined
        // otherwise we won't set invalid value as default ( e.g. = 'q1' )
        return viewValue;
      };

      ctrl.$parsers.push(validator);
      ctrl.$formatters.push(validator);
    },
  };
}
```

### После версии 1.3 (синхронно)

Начиная с версии angularjs 1.3 в нашем распоряжении появляется отдельный объект (контейнер) для валидаторов `$validators`, наименование поля которого является именем валидатора, а из проверяющей функции требуется лишь вернуть true/false.

К тому же, больше нет необходимости вызывать функцию `$setValidity()`.

[**Jsfiddle demo**](https://jsfiddle.net/qfjn0bc4/1/):

```javascript
function numericDirective() {
  return {
    require: "ngModel",
    link: function (scope, elem, attr, ctrl) {
      var numericRegexp = /^\d+$/;

      var validator = function (viewValue) {
        return (
          numericRegexp.test(viewValue) &&
          viewValue % 2 === 0 &&
          viewValue % 7 === 0
        );
      };

      ctrl.$validators.numericOnly = function (modelValue, viewValue) {
        return validator(viewValue);
      };
    },
  };
}
```

### После версии 1.3 (асинхронно)

Так же, одной из очень интересных возможностей для валидации, является асинхронная валидация реализованная с помощью promise’ов и доступная, благодаря добавлению объекта `$asyncValidators` и полю `$pending` у формы, которое сигнализирует о выполнении процесса валидации в текущий момент времени.

[**Jsfiddle demo**](https://jsfiddle.net/r08bhggj/):

```javascript
function numericDirective($q, $timeout) {
  return {
    require: "ngModel",
    link: function (scope, elem, attr, ctrl) {
      var numericRegexp = /^\d+$/;

      var validator = function (viewValue) {
        var isValid =
          numericRegexp.test(viewValue) &&
          viewValue % 2 === 0 &&
          viewValue % 7 === 0;

        return $q(function (resolve, reject) {
          return $timeout(function () {
            return isValid ? resolve("Valid") : reject("Invalid");
          }, 1500);
        });
      };

      ctrl.$asyncValidators.numericOnly = function (modelValue, viewValue) {
        return validator(viewValue);
      };
    },
  };
}
```

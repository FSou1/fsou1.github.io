---
title: Class, extends and super in JavaScript
tags: JavaScript
redirect_from: "/Classes_in_javascript/"
---

Classes were introduced to JavaScript in ECMAScript 2015. Even though it was about 5 years ago, I still want to drop a quick note about this feature.

**N.B.** JavaScript classes **does not** introduce a new object-oriented inheritance model. All in all they are primarily syntactical sugar over existing prototype-based inheritance.

To declare a class we can use the `class` keyword:

```
class Person {
 constructor(name, age) {
  this.name = name;
  this.age = age;
 }
}
```

The body of a class is executed in [strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode).

The `constructor` method is used for creating and initializing an object of a class type. Only one method with the name `constructor` is allowed (otherwise a `SyntaxError` will be thrown).

To declare a derived class we should use the `extends` keyword. Derived classes can use the `super` keyword to call the `constructor` of the super (base) class:

```
class Person {
 constructor(name, age) {
  this.name = name;
  this.age = age;
 }
}

class Employee extends Person {
 constructor(name, age, employer) {
  super(name, age);
  this.employer = employer;
 }
}
```

The main advantage of inheritance is code re-usability and standards. Nevertheless, beware of its improper use since it may lead to [God object ](https://en.wikipedia.org/wiki/God_object) and [code smells](https://en.wikipedia.org/wiki/Code_smell).

Another use of the `super` keyword is to call corresponding methods of super (base) class:

```
class Person {
 constructor(name, age) {
  this.name = name;
  this.age = age;
 }

 greet() {
  console.log(`Hey, my name is ${this.name}`);
 }
}

class Employee extends Person {
 constructor(name, age, employer) {
  super(name, age);
  this.employer = employer;
 }

 greet() {
  super.greet();
  console.log(`I'm working at ${this.employer}`);
 }
}

const john = new Employee('John', 29, 'Google');
john.greet();

// Hey, my name is John
// I'm working at Google
```

Fin.

---
title: TypeScript types cheat sheet
tags: TypeScript
redirect_from: "/TypeScript_types_cheat_sheet/"
---

This post is a collection of the available TypeScript types, examples of their usual use, and JavaScript outputs.

**Update history**:

- 13 Feb 2020 - Added `void` type (thx to [AngularBeginner](https://www.reddit.com/r/typescript/comments/f33on2/just_blogged_typescript_types_cheat_sheet/fhgvczq?utm_source=share&utm_medium=web2x));
- 13 Feb 2020 - Added `unknown` type (thx to [andra_nl](https://www.reddit.com/r/typescript/comments/f33on2/just_blogged_typescript_types_cheat_sheet/fhgsax6?utm_source=share&utm_medium=web2x));
- 8 Mar 2020 - Added the const enum section;

# Boolean

Nothing special, just `true` and `false`:

**TypeScript**:

```typescript
let isDone: boolean = true;
```

**JavaScript**:

```javascript
let isDone = true;
```

> Note: `Boolean` with an upper case B is different from `boolean` with a lower case b. Upper case `Boolean` is an object type whereas lower case `boolean` is a primitive type. It is always recommended to use `boolean`.

# Number

4 ways to declare a number:

**TypeScript**:

```typescript
let decimal: number = 6;
let hex: number = 0xf00d;
let binary: number = 0b0101;
let octal: number = 0o744;
```

**JavaScript**:

```javascript
let decimal = 6;
let hex = 0xf00d;
let binary = 0b0101;
let octal = 0o744;
```

> Note: All numbers in TypeScript are floating point values

# String

Same to `boolean`, pretty much as-is:

**TypeScript**:

```typescript
let color: string = "blue";
let template = `Sky is ${color}`;
```

**JavaScript**:

```javascript
let color = "blue";
let template = `Sky is ${color}`;
```

# Array

There are two ways to declare an array:

**TypeScript**:

```typescript
let list: number[] = [1, 2, 3];
let array: Array<number> = [1, 2, 3];
```

**JavaScript**:

```javascript
let list = [1, 2, 3];
let array = [1, 2, 3];
```

> Note: Both ways produce the same JavaScript output

You can also declare an array that contains elements of different data types:

**TypeScript**:

```typescript
let values: (string | number)[] = ["Apple", 2, "Orange", 3, 4, "Banana"];
// or
let values: Array<string | number> = ["Apple", 2, "Orange", 3, 4, "Banana"];
```

**JavaScript**:

```javascript
let values = ["Apple", 2, "Orange", 3, 4, "Banana"];
// or
let values = ["Apple", 2, "Orange", 3, 4, "Banana"];
```

# Tuple

Tuple is a type that allows you to express an array with a fixed number of elements whose types are known:

**TypeScript**:

```typescript
let x: [string, number];
x = ["hello", 10];
let str = x[0];
let num = x[1];
```

**JavaScript**:

```javascript
let x;
x = ["hello", 10];
let str = x[0];
let num = x[1];
```

# Enum

## Numeric

Numeric enum allows you to use and specify `numeric` values:

**TypeScript**:

```typescript
enum Color {
  Red = 1,
  Green = 2,
}
let c: Color = Color.Green;
```

**JavaScript**:

```javascript
var Color;
(function (Color) {
  Color[(Color["Red"] = 1)] = "Red";
  Color[(Color["Green"] = 2)] = "Green";
})(Color || (Color = {}));
let c = Color.Green;
```

Look at this elegant approach:

```javascript
Color[(Color["Red"] = 1)] = "Red";
```

Since the `Color` is the object, this line is responsible for producing the following output:

```javascript
{
  1: "Red"
  2: "Green"
  Red: 1
  Green: 2
}
```

Eventually, it allows us to access the enum both ways: either `Color.Red` or `Color[1]`.

### Get name

Two ways to get the enum's key:

**TypeScript**:

```typescript
let cs: string = Color[0] || Color[Color.Green]; // Green
```

**JavaScript**:

```javascript
let cs = Color[0] || Color[Color.Green]; // Green
```

> Note: Works with numeric enums only

### Duplicates

I don't know why but you can use a single value for multiple enum keys:

**TypeScript**:

```typescript
enum Vehicle {
  Car = 1,
  Plane = 1,
}
let vs: Vehicle = Vehicle.Car; // 1
let vss: Vehicle = Vehicle.Plane; // 1
```

**JavaScript**:

```javascript
var Vehicle;
(function (Vehicle) {
  Vehicle[(Vehicle["Car"] = 1)] = "Car";
  Vehicle[(Vehicle["Plane"] = 1)] = "Plane";
})(Vehicle || (Vehicle = {}));
let vs = Vehicle.Car; // 1
let vss = Vehicle.Plane; // 1
```

In this case you'll end up with the following output:

```javascript
{
  1: "Plane"
  Car: 1
  Plane: 1
}
```

> Note: You can get the same value by multiple keys but not vice versa

### Const

Const enums are completely removed during compilation. Const enum mebers are inlined:

**TypeScript**:

```typescript
const enum Directions {
  Up,
  Down,
  Left,
  Right,
}
let directions = [
  Directions.Up,
  Directions.Down,
  Directions.Left,
  Directions.Right,
];
```

**JavaScript**:

```javascript
"use strict";
let directions = [0 /* Up */, 1 /* Down */, 2 /* Left */, 3 /* Right */];
```

## String

String enums allow you to specify `string` values but the output is a bit different:

**TypeScript**:

```typescript
enum Sex {
  Male = "MALE",
  Female = "FEMALE",
}
```

**JavaScript**:

```javascript
var Sex;
(function (Sex) {
  Sex["Male"] = "MALE";
  Sex["Female"] = "FEMALE";
})(Sex || (Sex = {}));
```

The output object won't have `MALE` and `FEMALE` properties, so there is only one way to access values:

```typescript
let s: Sex = Sex.Male; // Male
let ss: Sex = Sex["MALE"] || Sex[Sex.Male]; // Error: Property MALE doesn't exist on type Sex
```

## Heterogeneous

Again, the use case isn't clear for me, but you can define enums with values of various types:

**TypeScript**:

```typescript
enum Status {
  Started = "STARTED",
  Progress = 1,
  Done,
}
```

**JavaScript**:

```javascript
var Status;
(function (Status) {
  Status["Started"] = "STARTED";
  Status[(Status["Progress"] = 1)] = "Progress";
  Status[(Status["Done"] = 2)] = "Done";
})(Status || (Status = {}));
```

> Status.Done has the value `2` which was calculated automatically

# Any

TypeScript has type-checking and compile-time checks. And `any` is extremely helpful when we don't have prior knowledge about the type of some variables:

**TypeScript**:

```typescript
let notSure: any = 4;
notSure = "maybe a string";
notSure = false;
```

**JavaScript**:

```javascript
let notSure = 4;
notSure = "maybe a string";
notSure = false;
```

Why not have an array of any values? Absolutely not a problem:

**TypeScript**:

```typescript
let notSureList: any[] = [1, "free", true];
```

**JavaScript**:

```javascript
let notSureList = [1, "free", true];
```

# Unknown

The unknown type is very similar to `any` but much less permissive. On the one hand `unknown` variables can hold any values (e.g. `true`, `[]`, `Math.random`, `undefined`, `new Error()`). On the other hand TypeScript won't let you to access type specific properties and operations without explicit and prior type checking:

**TypeScript**:

```typescript
// unknown
function format(value: unknown): any {
  // return value.trim();    // Error
  if (typeof value === "string") {
    return value.trim(); // OK
  }

  // return value.length; // Error
  if (value instanceof Array) {
    return value.length; // OK
  }
}
```

**JavaScript**:

```javascript
function format(value) {
  // return value.trim();    // Error
  if (typeof value === "string") {
    return value.trim(); // OK
  }

  // return value.length; // Error
  if (value instanceof Array) {
    return value.length; // OK
  }
}
```

> Both `typeof` and `instanceof` operators are valid for type checking

# Null & undefined

Null and undefined are separate types in TypeScript. They're not extremely useful on their own, but still can assist you as a part of union type declarations:

**TypeScript**:

```typescript
function search(term: string | null | undefined) {}
```

**JavaScript**:

```javascript
function search(term) {}
```

# Void

Void type is used to declare functions that don't return any meaningful value (besides `undefined` and `null` (if `--strictNullChecks` is disabled)). Anyway, you can even declare void variables:

**TypeScript**:

```typescript
function log(): void {
  return undefined;
}
const l: void = 1; // Error: Type '1' is not assignable to type 'void'
const ll: void = log(); // undefined
```

**JavaScript**:

```javascript
function log() {
  return undefined;
}
const l = 1; // Error: Type '1' is not assignable to type 'void'
const ll = log(); // undefined
```

# Never

The last piece in puzzle is the type `never`. It could be used as the return type of a function that always throws an exception or never returns a value.

**TypeScript**:

```typescript
function err(msg: string): never {
  throw new Error(msg);
}
function fail(): never {
  return err("Failed");
}
function infLoop(): never {
  while (true) {}
}
```

**JavaScript**:

```javascript
function err(msg) {
  throw new Error(msg);
}
function fail() {
  return err("Failed");
}
function infLoop() {
  while (true) {}
}
```

Reference:

1. [TypeScript playground](http://www.typescriptlang.org/play/?ssl=1&ssc=1&pln=68&pc=53#code/PTAECEHtIGwUwIYDsBQ8AuoCWBnAIpEnAFygBG08yoAvKOgE4CucA3CiiKAHJMC2ZOAzRxMAEzgBjLHwQxSSfoIa1QANnYZQACzgAPBUqGqADHoBmJk2M2jyWJAgYBPQwON0TZEwEZftzEhJdDk3ZVNIAHYAFmj2TjAAZUYHAHMRTElYSAZSHBSkVNUAIjIYFmKA+jg+AAcYBHQ4VQADRIBrZ2wcUAASAG8smByAXxb4rgBBBgYEZwzQGFx0MKEAbQBdVTWfABpQACZ9gGYNqqdZ11Bpy4AeRXcGAD5tvcOTs44uABUmergFgZQGt8gw0vsHspPnptsVdDBhsV9n5PlpQao9GsTKi7A8MTtPglQABRPEAChwMn+EP4Qiwkn2qVxCD4cAAlCg4HiAMLZFT9UAAJTgYlUbwA4gw4FzVAdQCMFpJSLzhio6CqcgA6SXSpBVSQ4PIFIrqvlYrYAHwtoA1DDWtu1Uq5Wy4Oq5XzApP4oApVPg+1BaQ5XO9iX0oAFAFk5M06MVI5MADLEpGgABiNRjJTTxITyeK8oWhtAYZhdFLmuj8FAXCrALRxdLqlLa3jSZTlutLYrdZdnpmOQ9JPJOBZzQAbnIWMG8QA1ODaenVgXcpxi-YABQaRDFha04+L88XkmrdCPS7gmtXDFYoDv95rYB8CwPh4XF9U55Pl63yDYD64Z8iS9PgfV0JoGEgJkiEgJgcBnUMQnQOCIxLEIGCaUU6AAckSb5JkFb5iTwbDN0g1IpRwHo6DeAgdwVIlkHmLQkEgdBEiYKVSCY1Q4hQVj2M42NQGw2RnEEUAEFAQNCmw9gBI4qVVHMOQcDYIcbjmUBIHMSSkGY3E2MUuBE2Wbj9M2V59mw8wnVI+hmDgDYh0UBF9iYJAJHMBwRQWJhSA8ryfKw0BArgbyiBsBYkEMBFVFcmAJjAIhxyEFBzA84IsEIUAhAYMk+BwVIjTBQo2QUOBUv5FAH3QbRIIAd1AIgmuJAd8sK1I2XYBUMqQLKcpUrAYDJcrmsq4x+hq+8pWQhgkFymYyWwtMEGGkVsO6lBesy9BsoWhxzETaBalGiqqojaa7waxdqzJRhp1QhUGK4dBnFqZoECooQ9sIItIFZWdzK6OM6twbpJOk41Kn+wHEy5VI6tWNUfVuGTUieHAAbgWc2U1eBCjqqosbhhG6oOZHVApbHZ0knp0bxgnEe0VggA);
2. [Basic types](https://www.typescriptlang.org/docs/handbook/basic-types.html);
3. [Enums in TypeScript](https://www.tutorialsteacher.com/typescript/typescript-enum);
4. [Boolean in TypeScript](https://www.tutorialsteacher.com/typescript/typescript-boolean);
5. [Boolean in Javascript and TypeScript](https://fettblog.eu/boolean-in-javascript-and-typescript/);
6. [The unknown Type in TypeScript](https://mariusschulz.com/blog/the-unknown-type-in-typescript);

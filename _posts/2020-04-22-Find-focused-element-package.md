---
title: find-focused-element package is available
tags: TypeScript npm
---

It's a short post about a new npm package I've just uploaded.

## TL;DR

- [Stackblitz - sample](https://stackblitz.com/edit/angular-uj2nvt)
- [GitHub - source code](https://github.com/FSou1/find-focused-element)
- [Npmjs](https://www.npmjs.com/package/find-focused-element)

## Why

While developing a chrome extension I needed an easy way to access a currently focused element.

## Quick Start

Add it to your project:

```bash
npm install --save find-focused-element
```

Import using ES Modules:

```js
import findFocusedElem from "find-focused-element";
```

Or as a CommonJS:

```js
const findFocusedElem = require("find-focused-element");
```

Use:

```js
const elem = findFocusedElem(window.document);
```

## Browser Support

The library has been tested in:

- Latest Edge, Firefox, Chrome, Opera, Safari (Mac)
- iOS 11 Safari
- IE 8-11

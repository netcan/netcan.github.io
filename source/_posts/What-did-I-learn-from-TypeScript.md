---
title: What did I learn from TypeScript
toc: true
original: true
date: 2023-05-09 22:12:07
tags:
- TypeScript
categories:
- 编程
---

## Preface
Thanks to the emergence of large language model AI, I'm confident in my ability to write posts in English fluently and to learn new skills with ease.

## why am I learning TypeScript?
Javascript is a widely-used programming language that can do all sorts of cool stuff. However, sometimes its flexible nature can make things confusing and lead to writing error-prone code. One of the main reasons for this is that it's a dynamically-typed language. So, when you stumble upon an object returned from a function, it might not be immediately clear what you can do with it.

TypeScript, the savior of JS, has arrived to fix all of its problems. Because who needs dynamic typing anyway? As its name implies, TypeScript adds a type system to JavaScript.

And to make things even more fun, since browsers and Node.js can only run JS code, TypeScript needs a compiler to transform it into JS, just like C++ compiler transforms code into binary. Because who doesn't love adding an extra step to the process? At least the extra compile time can help you catch some type errors before running your code.

## TypeScript vs C++
C++ is one of my favorite programming language, it has the most powerful type system in the land and truly fascinating compile-time metaprogramming features. Who needs a debugger when the only tool you need to interact with the compiler is typing?

In C++, you can use types to generate code, why bother with runtime costs when you can just move them to compile-time?

TypeScript's type system is pretty impressive, but it's no match for C++'s metaprogramming wizardry. Sure, TypeScript can do introspection like checking if a type has a field, filtering out fields, adding fields, and generating fields based on other fields.

TypeScript is a superset of JS come with a type system. However, its type system is limited to the 8 types that correspond to the results of the `typeof` operator in JS, which include number/bigint, string, boolean, object, function, undefined, symbol. it's important to note that array type is considered to be of the object type.

While TypeScript provides the ability to define interfaces, classes, union types, enums, type alias, and more, it can be challenging to check the type of custom-defined types, because `typeof` operator is limited to the 8 basic types. if you want to check for a specific shape type, such as a circle or rectangle, you may need to define custom fields like `kind` to check for thoese shape types.

```ts
interface Circle {
  kind: "circle";
  radius: number;
}
 
interface Square {
  kind: "square";
  sideLength: number;
}
 
type Shape = Circle | Square;
function getArea(shape: Shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
  }
}
```

The `interface` keyword is used to declare an interface type. TypeScript doesn’t use “types on the left”-style declarations like `int x;`, type annotations will always go after the things be typed, such as `radius: number`, it declare a radius field with number type. it's worth noting that values can also be used as types, such as `kind: "circle"`. Even thought the `Circle` and `Square` interfaces have the same `kind` field, they are considered to be different types.
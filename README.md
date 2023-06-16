<!-- @format -->

# Decorator Metadata

**Stage**: 3

**Spec Text**: https://github.com/pzuraq/ecma262/pull/10

This proposal seeks to extend the [Decorators](https://github.com/tc39/proposal-decorators)
proposal by adding the ability for decorators to associate _metadata_ with the
value being decorated.

## Overview

Decorators are functions that allow users to metaprogram by wrapping and
replacing existing values. This allows them to solve a number of use cases quite
well, such as memoization, reactivity, method binding, and more. However, there
are a number of use cases which require code _external_ to the decorator and
decorated class to be able to introspect and understand what decorations were
applied, including:

- Validation
- Serialization
- Web component definition
- Dependency injection
- Declarative routing/application structure
- ...and more

In previous iterations of the decorators proposal, all decorators had access to
the class prototype, allowing them to associate metadata directly via a
`WeakMap` by using the class as a key. This is no longer possible in the most
recent version, however, as decorators only have access to the value they are
_directly_ decorating (e.g. method decorators have access to the method, field
decorators have access to the field, etc).

This proposal extends decorators by providing a metadata _object_, which can be
used either to directly store metadata, or as a WeakMap key. This object is
provided via the decorator's context argument, and is then accessible via the
`Symbol.metadata` property on the class definition after decoration.

## Detailed Design

The overall decorator signature will be updated to the following:

```ts
type Decorator = (value: Input, context: {
  kind: string;
  name: string | symbol;
  access: {
    get?(): unknown;
    set?(value: unknown): void;
  };
  isPrivate?: boolean;
  isStatic?: boolean;
  addInitializer?(initializer: () => void): void;
+ metadata?: Record<string | number | symbol, unknown>;
}) => Output | void;
```

The new `metadata` property is a plain JavaScript object. The same object is
passed to every decorator applied to a class or any of its elements. After the
class has been fully defined, it is assigned to the `Symbol.metadata` property
of the class.

An example usage might look like:

```js
function meta(key, value) {
  return (_, context) => {
    context.metadata[key] = value;
  };
}

@meta('a', 'x')
class C {
  @meta('b', 'y')
  m() {}
}

C[Symbol.metadata].a; // 'x'
C[Symbol.metadata].b; // 'y'
```

### Inheritance

If the decorated class has a parent class, then the prototype of the `metadata`
object is set to the metadata object of the superclass. This allows metadata to
be inherited in a natural way, taking advantage of shadowing by default,
mirroring class inheritance. For example:

```js
function meta(key, value) {
  return (_, context) => {
    context.metadata[key] = value;
  };
}

@meta('a', 'x')
class C {
  @meta('b', 'y')
  m() {}
}

C[Symbol.metadata].a; // 'x'
C[Symbol.metadata].b; // 'y'

class D extends C {
  @meta('b', 'z')
  m() {}
}

D[Symbol.metadata].a; // 'x'
D[Symbol.metadata].b; // 'z'
```

In addition, metadata from the parent can be read during decoration, so it can
be modified or extended by children rather than overriding it.

```ts
function appendMeta(key, value) {
  return (_, context) => {
    // NOTE: be sure to copy, not mutate
    const existing = context.metadata[key] ?? [];
    context.metadata[key] = [...existing, value];
  };
}

@appendMeta('a', 'x')
class C {}

@appendMeta('a', 'z')
class D extends C {}

C[Symbol.metadata].a; // ['x']
D[Symbol.metadata].a; // ['x', 'z']
```

### Private Metadata

In addition to public metadata placed directly on the metadata object, the
object can be used as a key in a `WeakMap` if the decorator author does not want
to share their metadata.

```ts
const PRIVATE_METADATA = new WeakMap();

function meta(key, value) {
  return (_, context) => {
    let metadata = PRIVATE_METADATA.get(context.metadata);

    if (!metadata) {
      metadata = {};
      PRIVATE_METADATA.set(context.metadata, metadata);
    }

    metadata[key] = value;
  };
}

@meta('a', 'x')
class C {
  @meta('b', 'y')
  m() {}
}

PRIVATE_METADATA.get(C[Symbol.metadata]).a; // 'x'
PRIVATE_METADATA.get(C[Symbol.metadata]).b; // 'y'
```

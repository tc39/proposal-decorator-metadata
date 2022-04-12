<!-- @format -->

# Decorator Metadata

**Stage**: 2

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

This proposal extends decorators by providing a value to use as a key to
associate metadata with. This key is then accessible via the
`Symbol.metadataKey` property on the class definition.

## Detailed Design

The overall decorator signature will be updated to the following:

```ts
interface MetadataKey {
  parent: MetadataKey | null;
}

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
+ metadataKey?: MetadataKey;
+ class?: {
+   metadataKey: MetadataKey;
+   name: string;
+ }
}) => Output | void;
```

Two new values are introduced, `metadataKey` and `class`.

### `metadataKey`

`metadataKey` is present for any _tangible_ decoratable value, specifically:

- Classes
- Class methods
- Class accessors and auto-accessors

It is not present for class fields because they have no tangible value (e.g.
there is nothing to associate the metadata with, directly or indirectly).
`metadataKey` is then set on the decorated value once decoration has completed:

```js
const METADATA = new WeakMap();

function meta(value) {
  return (_, context) => {
    METADATA.set(context.metadataKey, value);
  };
}

@meta('a')
class C {
  @meta('b')
  m() {}
}

METADATA.get(C[Symbol.metadataKey]); // 'a'
METADATA.get(C.m[Symbol.metadataKey]); // 'b'
```

This allows metadata to be associated directly with the decorated value.

### `class`

The `class` object is available for all _class element_ decorators, including
fields. The `class` object contains two values:

1. The `metadataKey` for the class itself
2. The name of the class

This allows decorators for class elements to associate metadata with the class.
For method decorators, this can simplify certain flows. For class fields, since
they have no tangible value to associate metadata with, the class metadata key
is the only way to store their metadata.

```js
const METADATA = new WeakMap();
const CLASS = Symbol();

function meta(value) {
  return (_, context) => {
    const metadataKey = context.class?.metadataKey ?? context.metadataKey;
    const metadataName = context.kind === 'class' ? CLASS : context.name;

    let meta = METADATA.get(metadataKey);

    if (meta === undefined) {
      meta = new Map();
      METADATA.set(metadataKey, meta);
    }

    meta.set(metadataName, value);
  };
}

@meta('a')
class C {
  @meta('b')
  foo;

  @meta('c')
  get bar() {}

  @meta('d')
  baz() {}
}

// Accessing the metadata
const meta = METADATA.get(C[Symbol.metadataKey]);

meta.get(CLASS); // 'a';
meta.get('foo'); // 'b';
meta.get('bar'); // 'c';
meta.get('baz'); // 'd';
```

### `parent`

Metadata keys also have a `parent` property. This is set to the value of
`Symbol.metadataKey` on the prototype of the value being decorated.

```js
const METADATA = new WeakMap();

function meta(value) {
  return (_, context) => {
    const classMetaKey = context.class.metadataKey;
    const existingValue = METADATA.get(classMetaKey.parent) ?? 0;

    METADATA.set(classMetaKey, existingValue + value);
  };
}

class C {
  @meta(1)
  foo;
}

class D extends C {
  @meta(2)
  foo;
}

// Accessing the metadata
METADATA.get(C[Symbol.metadataKey]); // 3
```

## Examples

Todo

Babel plugin for static and runtime type checking using Flow and tcomb.

**Tools**

[Flow](https://flowtype.org/) is a static type checker for JavaScript.

[tcomb](https://github.com/gcanti/tcomb) is a library for Node.js and the browser which allows you to check the types of JavaScript values at runtime with a simple and concise syntax. It's great for Domain Driven Design and for adding safety to your internal code.

**Runtime type checking (tcomb), why?**

- you don't want or you can't use `Flow`
- you want refinement types
- you want to validate the IO boundary (for example API payloads)
- you want to enforce immutability
- you want to leverage the runtime type introspection provided by `tcomb`'s types

**Static type checking (Flow)**

`babel-plugin-tcomb` is `Flow` compatible, this means that you can run them side by side, statically checking your code with `Flow` and let `tcomb` catching the remaining bugs at runtime.

# Setup

First, install via npm.

```sh
npm install --save tcomb
npm install --save-dev babel-plugin-tcomb
```

Then, in your babel configuration (usually in your `.babelrc` file), add (at least) the following plugins:

```js
{
  "plugins" : [
    "syntax-flow",
    "tcomb",
    "transform-flow-strip-types"
  ]
}
```

# How it works

> **Important**. `tcomb` must be `require`able

**Example 1**. Type checking functions.

```js
function sum(a: number, b: number) {
  return a + b
}
```

compiles to:

```js
import t from 'tcomb'

function sum(a, b) {
  _assert(a, t.Number, 'a') // <= runtime type checking
  _assert(b, t.Number, 'b') // <= runtime type checking

  return a +b
}
```

**Example 2**. Defining domain models.

```js
interface Person {
  name: string;
  surname: ?string
}
```

compiles to:

```js
import t from 'tcomb'

var Person = t.interface({
  name: t.String,
  surname: t.maybe(t.String)
}, 'Person');
```

## Defining refinements (*)

In order to define [refinement types](https://github.com/gcanti/tcomb/blob/master/docs/API.md#the-refinement-combinator) you can use the `$Refinement` type providing a predicate:

```js
import type { $Refinement } from 'tcomb'

// define you predicate...
const isInteger = n => n % 2 === 0

// ...and pass it to the suitable intersection type involving the $Refinement type
type Integer = number & $Refinement<typeof isInteger>;

function foo(n: Integer) {
  return n
}

foo(2)   // flow ok, tcomb ok
foo(2.1) // flow ok, tcomb throws [tcomb] Invalid value 2.1 supplied to n: Integer
foo('a') // flow throws, tcomb throws
```

## Runtime type introspection (*)

Check out the [meta object](https://github.com/gcanti/tcomb/blob/master/docs/API.md#the-meta-object) in the tcomb documentation.

```js
import type { $Reify } from 'tcomb'

type Person = { name: string };

const ReifiedPerson = (({}: any): $Reify<Person>)
console.log(ReifiedPerson.meta) // => { kind: 'interface', props: ... }
```

> (*) these are considered (inevitable and useful) hacks

## Validating (at runtime) the IO boundary using typecasts

```js
type User = { name: string };

export function loadUser(userId: string): Promise<User> {
  return axios.get('...').then(p => (p: User)) // <= type cast
}
```

## Recursive types

Just add a `// recursive` comment on top:

```js
// recursive
type Path = {
  node: Node,
  parentPath: Path
};
```

## Type-checking React

Using [tcomb-react](https://github.com/gcanti/tcomb-react):

```js
// @flow

import React from 'react'
import ReactDOM from 'react-dom'
import { props } from 'tcomb-react'

type Props = {
  name: string
};

@props(Props)
class Hello extends React.Component<void, Props, void> {
  render() {
    return <div>Hello {this.props.name}</div>
  }
}


ReactDOM.render(<Hello />, document.getElementById('app'))
```

Flow will throw:

```
index.js:12
 12: class Hello extends React.Component<void, Props, void> {
                                               ^^^^^ property `name`. Property not found in
 19: ReactDOM.render(<Hello />, document.getElementById('app'))
                     ^^^^^^^^^ props of React element `Hello`
```

while `tcomb-react` will warn:

```
Warning: Failed propType: [tcomb] Invalid prop "name" supplied to Hello, should be a String.

Detected errors (1):

  1. Invalid value undefined supplied to String
```

Additional babel configuration:

```js
{
  "presets": ["es2015", "react"],
  "plugins" : [
    "tcomb",
    "transform-decorators-legacy"
  ]
}
```

# Caveats

- `tcomb` must be `require`able
- generics are not handled (`Flow`'s responsability)

# Plugin config

- `skipAsserts: boolean = false` removes the asserts but keeps the models
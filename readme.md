# Preact and Typescript

> Some simple examples demonstrating how Typescript and Preact can work together. :heart:

## Table of Contents

1. [Setup](#setup)
	1. [Install Dependencies](#install-dependencies)
	2. [Setup tsconfig](#tsconfigjson)
	3. [Setup webpack config](#webpackconfigjs)
2. [Stateful Components](#stateful-components)
3. [Functional Components](#functional-components)
4. [Components With Children](#components-with-children)
5. [Higher Order Components (HOC)](#higher-order-components-hoc)
6. [Extending HTML Attributes](#extending-html-attributes)
7. [Rendering](#rendering)
8. [Custom Elements / Web Components](#custom-elements--web-components)

## Setup

### Install Dependencies

```bash
$ npm install --save preact typescript webpack ts-loader
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "sourceMap": true,
    "module": "commonjs",
    "target": "es5",
    "jsx": "react",
    "jsxFactory": "h"
  },
  "include": [
    "./src/**/*.tsx",
    "./src/**/*.ts"
  ]
}
```

It's important to note the usage of `jsx: "react"` and `jsxFactory: "h"`. There are [several options available](https://www.typescriptlang.org/docs/handbook/jsx.html#basic-usage) for `jsx` that determine how Typescript outputs JSX syntax. By default without `jsxFactory: "h"`, Typescript will convert JSX tags to `React.createElement(...)` which won't work for Preact. Preact instead uses `h` as its JSX pragma, which you can read more about at [WTF is JSX](https://jasonformat.com/wtf-is-jsx/).

### webpack.config.js

```
var path = require('path');

module.exports = {
  devtool: 'source-map',
  entry: './src/app',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'app.js'
  },
  resolve: {
    extensions: ['.ts', '.tsx']
  },
  module: {
    loaders: [
      {
        test: /\.tsx?$/,
        exclude: /node_modules/,
        loaders: ['ts-loader']
      }
    ]
  }
}
```

> TODO: document webpack config

## Stateful Components

Stateful components are Preact components that use any combination of state and/or [lifecycle methods](https://preactjs.com/guide/api-reference#lifecycle-methods).

```jsx
import { h, Component } from 'preact';

export interface Props {
  value: string,
  optionalValue?: string
}

export interface State {
  useOptional: boolean
}

export default class StatefulComponent extends Component<Props, State> {

  constructor() {
    super();
    this.state = {
      useOptional: false
    };
  }

  componentWillMount() {
    fetch('/some/api/')
      .then((response: any) => response.json())
      .then(({ useOptional }) => this.setState({ useOptional }));
  }

  render({ value, optionalValue }: Props, { useOptional }: State) {
    return (
      <div>{value} {useOptional ? optionalValue : null}</div>
    );
  }

}
```

### Example Usage

```jsx
<StatefulComponent /> // throws error, property "value" is missing
<StatefulComponent value="foo" /> // ok
<StatefulComponent value="foo" optionalValue={true} /> // throws error, value "true" is not assignable to type string
```

## Functional Components

Functional components are components do not use state and are simple javascript functions accepting props and returns a single JSX/VDOM element.

```jsx
import { h } from 'preact';

export interface Props {
  value: string,
  optionalValue?: string
}

export default function SomeFunctionalComponent({ value, optionalValue }: Props) {
  return (
    <div>
      {value} {optionalValue}
    </div>
  );
}
```

### Example Usage

```jsx
<SomeFunctionalComponent /> // throws error, property "value" is missing
<SomeFunctionalComponent value="foo" /> // ok
<SomeFunctionalComponent value="foo" optionalValue={true} /> // throws error, value "true" is not assignable to type string
```

You can also use named variables with `FunctionalComponent<Props>`.

```jsx
import { h, FunctionalComponent } from 'preact';

export interface Props {
  value: string,
  optionalValue?: string
}

export const SomeFunctionalComponent: FunctionalComponent<Props> = ({ value, optionalValue }: Props) => {
  return (
    <div>
      {value} {optionalValue}
    </div>
  );
};
```

## Components with Children

It's likely at some point you will want to nest elements, and with Typescript you can validate that any children props are valid JSX elements.

```jsx
import { h } from 'preact';

export interface Props {
  children?: JSX.Element[]
}

export function A() {
  return <div></div>;
}

export function B() {
  return 'foo';
}

export default function ComponentWithChildren({ children }: Props) {
  return (
    <div>
      {children}
    </div>
  )
}
```

### Example Usage

```jsx
<ComponentWithChildren /> // ok

// ok
<ComponentWithChildren>
  <span></span>
</ComponentWithChildren>

// ok
<ComponentWithChildren>
  <A />
</ComponentWithChildren>

// throws error, not a valid JSX Element
<ComponentWithChildren>
  <B />
</ComponentWithChildren>
```

## Higher Order Components (HOC)

Using Higher Order Components (HOC) allows for certain component logic to be reused and is a natural pattern for compositional components. An HOC is simply a function that takes a component and returns that component.

```jsx
// app/hoc.tsx

import { h, AnyComponent, Component } from 'preact';

export interface Props {
  email: string
}

export interface State {
  firstName: string,
  lastName: string
}

export default function HOC<P extends Props>(SomeComponent: AnyComponent<any, any>) {
  return class extends Component<P, State> {
    componentDidMount() {
      let { email } = this.props;
      fetch(`/user/${email}`)
        .then((response: any) => response.json())
        .then(({ firstName, lastName }) => this.setState({ firstName, lastName }))
    }

    render(props, state) {
      return <SomeComponent {...props} {...state} />;
    }
  }
}
```

Then, you can wrap your components with your HOC simplifying view logic to potentially only *props*.


```jsx
// app/component.tsx

import { h } from 'preact';
import HOC from './hoc';

export interface Props {
  firstName: string,
  lastName: string,
  email: string
}

function FunctionalUserComponent({ firstName, lastName, email }: Props) {
  return <div>{firstName} {lastName}: { email }</div>
}

export default HOC<{ email: string }>(FunctionalUserComponent);
```

### Example Usage

```jsx
// import FunctionalUserComponent from './app/component'

<FunctionalUserComponent /> // throws error, property "email" is missing
<FunctionalUserComponent email="foo@bar.com" /> // ok, first and last names are resolved by the HOC
```

## Extending HTML Attributes

If you have a component that passes down unknown HTML attributes using, you can extend `JSX.HtmlAttributes` to allow any valid HTML attribute for your component.

```jsx
import { h, Component } from 'preact';

export interface Props {
  value?: string
}

export default class ComponentWithHtmlAttributes extends Component<JSX.HtmlAttributes & Props, any> {
  render({ value, ...otherProps }: Props, {}: any) {
    return (
      <div {...otherProps}>{value}</div>
    );
  }
}
```

### Example Usage

```jsx
<ComponentWithHtmlAttributes /> // ok
<ComponentWithHtmlAttributes class="foo" /> // ok, valid html attribute
<ComponentWithHtmlAttributes foo="bar" /> // throws error, invalid html attribute
```

## Rendering

Rendering components requires an `Element`. The second argument will append your component to your node:

```jsx
import { h, render } from 'preact';
import MyComponent from './MyComponent';

render(<MyComponent />, document.body as Element);
```

Additionally, you can choose to replace a specific node by specifying a third argument:

```jsx
import { h, render } from 'preact';
import MyComponent from './MyComponent';

const node = document.getElementById('root') as Element;
render(<MyComponent />, node, node.firstElementChild as Element);
```

## Custom Elements / Web Components

With vanilla JSX in preact, you can create any element with whatever name you want and it'll get transpiled to `h('my-element-name', ...)`. However Typescript is more opinionated and will complain if an imported component or HTML element does not exist with that name. Normally, this is what you would want with Preact components but custom elements may or may not be imported into your JSX files.

For Typescript there is a special interface [`JSX.IntrinsicElements`](https://www.typescriptlang.org/docs/handbook/jsx.html#intrinsic-elements) available where you can define additional elements for Typescript to include. As a bonus, you can define any custom attributes as properties gaining additional type safety for custom elements in JSX!

> typings.d.ts

```jsx
export interface MyCustomElementProps {
  value: string,
  optionalValue?: string
}

declare module JSX {
  interface IntrinsicElements {
    "my-custom-element": MyCustomElementProps
  }
}
```

### Example Usage


```jsx
<my-custom-element /> // throws error, property "value" is missing
<my-custom-element value="foo" /> // ok
<my-custom-element value="foo" optionalValue={true} /> // throws error, value "true" is not assignable to type string
```

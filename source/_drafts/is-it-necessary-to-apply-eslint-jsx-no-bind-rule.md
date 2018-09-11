---
title: Is It Necessary to Apply ESLint jsx-no-bind Rule?
tags: [javascript, react, eslint]
categories: Programming
---

When using [ESLint React plugin][1], you may find a rule called [`jsx-no-bind`][2]. In short, it prevents you from using `.bind` or arrow function in a JSX prop. For instance, ESLint will complain about the arrow function in the `onClick` prop.

```javascript
class List extends React.Component {
  render() {
    return (
      <ul>
        {this.state.items.map(item => (
          <li key={item.id} onClick={() => { alert(item.id) }}>{item.text}</li>
        ))}
      </ul>
    )
  }
}
```

There're two reasons why this rule is introduced. First, a new function will be created on every `render` call, which may increase the frequency of garbage collection. Second, it will disable the pure rendering process, i.e. when you're using a `PureComponent`, or implement the `shouldComponentUpdate` method by yourself with identity comparison, a new function object in the props will cause unnecessary re-render of the component.

But some people argue that these two reasons are not solid enough to enforce this rule on all projects, especially when the solutions will introduce more codes and decrease readability. In [Airbnb ESLint preset][3], the team only bans the usage of `.bind`, but allows arrow function in both props and refs. I did some googling, and was convinced that this rule is not quite necessary. Someone says it's premature optimization, and you should measure before you optimize. I agree with that. In the following sections, I will illustrate how arrow function would affect the pure component, what solutions we can use, and talk a little bit about React rendering internals.

<!-- more -->

## Different Types of React Component

The regular way to create a React component is to extend the `React.Component` class and implement the `render` method. There is also a built-in `React.PureComponent`, which implements the life-cycle method `shouldComponentUpdate` for you. In regular component, this method will always return `true`, indicating that React should call `render` whenever the props or states change. `PureComponent`, on the other hand, does a shallow identity comparison for the props and states to see whether this component should be re-rendered. The following two components behave the same:

```javascript
class PureChild extends React.PureComponent {
  render() {
    return (
      <div>{this.props.message}</div>
    )
  }
}

class RegularChild extends React.Component {
  shouldComponentUpdate(nextProps, nextStates) {
    return this.props.message !== nextProps.message
  }

  render() {
    return (
      <div>{this.props.message}</div>
    )
  }
}
```

When their parent component is re-rendering, they will both check the message content in props and decide whether they should render again. The comparison rule is quite simple in pure component, it iterates the props and states object, compare each others' members with `===` equality check. In JavaScript, only primitive types and the very same object will pass this test, for example:

```javascript
1 === 1
'hello world' === 'hello world'
[] !== []
(() => {}) !== (() => {})
```

Clearly, arrow functions will fail the test and cause pure component to re-render every time its parent is re-rendered. On the other side, if you do not use pure component, or do props and states check on your own, enabling the `jsx-no-bind` rule is plain unnecessary.

Recently another kind of component has become popular, the stateless functional component (SFC). These components render results solely depend on their props, so they are like functions that take inputs and produce steady outputs. But under the hood, they are just regular components, i.e. they do not implement `shouldComponentUpdate`, and you can not implement by yourself either.

```javascript
const StatelessChild = (props) => {
  return (
    <div>{props.message}</div>
  )
}
```

## How to Fix `jsx-no-bind` Error

Arrow functions are usually used in event handlers. If we use normal functions or class methods, `this` keyword is not bound to the current instance, it is `undefined`. By using `.bind` or arrow function, we can access other class methods through `this`. To fix the `jsx-no-bind` error while still keeping the handler function bound, we can either bind it in constructor, or use the experimental class property syntax, which can be transformed by [Babel][6]. More information can be found in React [official document][5], and here is the [gist][7] of different solutions.

```javascript
export default class NoArgument extends React.Component {
  constructor() {
    this.handleClickBoundA = this.handleClickUnbound.bind(this)
    this.handleClickBoundC = () => { this.setState() }
  }
  handleClickUnbound() { /* "this" is undefined */ }
  handleClickBoundB = () => { this.setState() }
  render() {
    return (
      <div>
        Error: jsx-no-bind
        <button onClick={() => { this.setState() }}>ArrowA</button>
        <button onClick={() => { this.handleClickUnbound() }}>ArrowB</button>
        <button onClick={this.handleClickUnbound.bind(this)}>Bind</button>
        No error:
        <button onClick={this.handleClickBoundA}>BoundA</button>
        <button onClick={this.handleClickBoundB}>BoundB</button>
        <button onClick={this.handleClickBoundC}>BoundC</button>
      </div>
    )
  }
}
```

For handlers that require extra arguments, e.g. a list of clickable items, things will be a little tricky. There're two possible solutions, one is to create separate component for the item, and pass handler function and argument as props.

```javascript
class Item extends React.PureComponent {
  handleClick = () => { this.props.onClick(this.props.item.id) }
  render() {
    return (
      <li onClick={this.handleClick}>{this.props.item.text}</li>
    )
  }
}

export default class ListSeparate extends React.Component {
  handleClick = (itemId) => { alert(itemId) }
  render() {
    return (
      <ul>
        {this.props.items.map(item => (
          <Item key={item.id} item={item} onClick={this.handleClick} />
        ))}
      </ul>
    )
  }
}
```

separation of concerns, but add a lot of codes, and make them hard to understand.

## Virtual DOM and Reconciliation

[reconciliation][4]

* virtual dom vs dom
* React event handling


## References

* https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md
* https://cdb.reacttraining.com/react-inline-functions-and-performance-bdff784f5578
* https://reactjs.org/docs/optimizing-performance.html
* https://levelup.gitconnected.com/how-exactly-does-react-handles-events-71e8b5e359f2


[1]: https://github.com/yannickcr/eslint-plugin-react
[2]: https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md
[3]: https://github.com/airbnb/javascript/blob/eslint-config-airbnb-v17.1.0/packages/eslint-config-airbnb/rules/react.js#L93
[4]: https://reactjs.org/docs/reconciliation.html
[5]: https://reactjs.org/docs/handling-events.html
[6]: https://babeljs.io/docs/plugins/transform-class-properties/
[7]: https://github.com/jizhang/jsx-no-bind/blob/master/src/components/NoArgument.js


* https://github.com/airbnb/javascript/issues/801

That was my previous understanding, but since React has a default shouldComponentUpdate implementation of return true, and since most event handlers are added to a global delegation pool and not attached directly to DOM elements, this isn't actually a performance concern even though it seems like it should be.

* https://github.com/facebook/react/issues/7379#issuecomment-270117281

If something is slow then try using PureComponent and see if it helps.
If nothing is slow then don’t bother optimizing it.

There is no sure way to tell which is going to be slower: reconciliation or shallow comparisons everywhere. In pathological cases (shallow equality checks often failing) PureComponents can be slower. So the only rule of thumb is to only add one when you know you have a perf problem and you can verify adding it solves that problem. Does this help?

* https://github.com/airbnb/javascript/pull/782

First, as I have recently learned, the only advantage with the constructor-binding is that .bind is slow - creating new functions is not.
Second, performance should always be a secondary concern to code clarity and readability - something would have to be very slow to trump readability concerns, and I've been convinced that creating non-bound functions as event handlers (not as props, however, just as bottom-level event handlers) is not a performance hit due to the design of React.

Thus this is mostly a question of subjective readability, I think.

https://maarten.mulders.tk/blog/2017/07/no-bind-or-arrow-in-jsx-props-why-how.html
https://medium.com/@esamatti/react-js-pure-render-performance-anti-pattern-fb88c101332f
https://reactjs.org/docs/faq-functions.html#example-passing-params-using-data-attributes

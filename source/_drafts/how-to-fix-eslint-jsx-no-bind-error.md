---
title: How to Fix ESLint jsx-no-bind Error
tags: [javascript, react, eslint]
categories: Programming
---

When using [ESLint React plugin][1], you may find a rule called [`jsx-no-bind`][2]. In short, it prevents you from using `.bind` or arrow function in a JSX prop. For instance, ESLint will complain about the arrow function in the `onClick` prop.

```javascript
class List extends React.Component {
  render() {
    return (
      <Item onClick={() => { console.log('click') }} />
    )
  }
}
```

There're two reasons why this rule is introduced. First, a new function will be created on every `render` call, which may increase the frequency of garbage collection. Second, it will disable the pure rendering process, i.e. when you're using a `PureComponent`, or implement the `shouldComponentUpdate` method by yourself with identity comparison, a new function object in the props will cause unnecessary re-render of the component.

But some people argue that these two reasons are not solid enough to enforce this rule on all projects, especially when the solutions will introduce more codes and decrease readability. In [Airbnb ESLint preset][3], the team only bans the usage of `.bind`, but allows arrow function in both props and refs. I did some googling, and was convinced that this rule is not quite necessary. Someone says it's premature optimization, and you should measure before you optimize. I agree with that. However, digging into this question is a good chance to learn React, and I will note down what I have collected in the following article.

<!-- more -->

## Different Types of React Component

The regular way to create a React component is to extend the `React.Component` class and implement the `render` method. There is also a built-in `React.PureComponent`, which implements the life-cycle method `shouldComponentUpdate` for you. In regular component, this method will always return `true`, indicating that React should call `render` whenever the props or states change. `PureComponent`, on the other hand, does a shallow identity comparison for the props and states to see whether this component should be re-rendered. The following two components behaves the same:

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

* no argument
    * bind in constructor
    * class property
* with argument
    * wrap to child component
    * dataset


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

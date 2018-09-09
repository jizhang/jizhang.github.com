---
title: jsx-no-bind
tags: [javascript, react, eslint]
categories: Programming
---

* what the rule checks
* why such rule
    * create function object on every render call
    * affect pure component rendering
* how to solve
    * no argument
        * bind in constructor
        * class property
    * with argument
        * wrap to child component
        * dataset
* airbnb rule
    * arrow function is not slow, bind is though
    * not so much pure component
    * virtual dom vs dom
    * React event handling


## References

* https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md

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

* https://stackoverflow.com/a/43968902/1030720

not an anti-pattern. Don't think about performance impact until you actually find a performance problem.

https://maarten.mulders.tk/blog/2017/07/no-bind-or-arrow-in-jsx-props-why-how.html
https://cdb.reacttraining.com/react-inline-functions-and-performance-bdff784f5578
https://medium.com/@esamatti/react-js-pure-render-performance-anti-pattern-fb88c101332f
https://reactjs.org/docs/reconciliation.html
https://reactjs.org/docs/optimizing-performance.html
https://reactjs.org/docs/faq-functions.html#example-passing-params-using-data-attributes
https://levelup.gitconnected.com/how-exactly-does-react-handles-events-71e8b5e359f2

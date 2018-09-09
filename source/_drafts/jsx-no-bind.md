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
    * virtual dom vs dom
    * React event handling


## References

* https://github.com/yannickcr/eslint-plugin-react/blob/master/docs/rules/jsx-no-bind.md

https://maarten.mulders.tk/blog/2017/07/no-bind-or-arrow-in-jsx-props-why-how.html

https://github.com/palantir/tslint-react/issues/96#issuecomment-412999164
https://stackoverflow.com/a/43968902/1030720
https://cdb.reacttraining.com/react-inline-functions-and-performance-bdff784f5578
https://github.com/airbnb/javascript/pull/782
https://medium.com/@esamatti/react-js-pure-render-performance-anti-pattern-fb88c101332f


https://reactjs.org/docs/reconciliation.html
https://reactjs.org/docs/optimizing-performance.html

https://reactjs.org/docs/faq-functions.html#example-passing-params-using-data-attributes

https://www.youtube.com/watch?v=ZCuYPiUIONs&t=1267s

https://levelup.gitconnected.com/how-exactly-does-react-handles-events-71e8b5e359f2
https://www.youtube.com/watch?v=dRo_egw7tBc

https://github.com/airbnb/javascript/issues/801

---
author: "Nantian Duan"
date: 2018-06-12
linktitle: Javascript 中的缓存
title: Javascript 中的缓存
---

## 斐波那契数列

现在我们来看一个老生常谈的问题--计算斐波那契数列的某一项的值。首先我们用递归来实现：

```javascript
const fibonacci = n => {
  if (n === 0 || n === 1) return 1;
  return fibonacci(n - 1) + fibonacci(n - 2);
};
```

很简单是吧？试试计算 fibonacci(100)。你的电脑很可能会卡死。为什么会这样呢？当你计算 f(n) 是，会先计算 f(n-1) 和 f(n-2)，而计算 f(n-1) 的时候，
会先计算 f(n-2) 和 f(n-3)。注意到了吗？ f(n-2) 被重复计算了。实际上该程序的时间复杂度是 O(2^n)，也就是说当 n = 100 的时候，足足需要计算 2^100 = 1.26e+30 次！

然后要优化这个算法，也很简单。很自然的就想到的就是避免重复计算，我们使用一个闭包（closure）缓存我们的计算结果：

```javascript
const fibonacci = (() => {
  const memo = {}; // 计算过的值缓存在这里

  const func = n => {
    if (memo[n] === undefined) {
      if (n === 0 || n === 1) {
        memo[n] = 1;
      } else {
        memo[n] = func(n - 1) + func(n - 2);
      }
    }
    return memo[n];
  };

  return func;
})();
```
返回的 func 和 memo 关联起来。在脱离了 memo 的作用域后，依然能通过 func 访问 memo。现在我们在 memo 中缓存了第 N 项的计算结果，
这样我们就避免了大量的重复计算。不难看出，现在的时间复杂度是 O(N)。

## memoize

现在我们将上述函数抽象出来，作为一个通用的能够缓存函数计算结果的函数。代码如下：

```javascript
const memoize = fn => {
  const memo = {};
  return (...args) => {
    const memoId = JSON.stringify(args); // 序列化函数参数
    memo[memoId] = memo[memoId] || fn.apply(fn, args);
    return memo[memoId];
  };
};
```

现在我们可以这样写 fibonacci 函数:

```javascript
const fibonacci = memoize(n => {
  if (n === 1 || n === 2) return 1;
  return fibonacci(n - 1) + fibonacci(n - 2);
});
```

现在通过 memoize 函数，自动完成了计算结果的缓存。

## 局限性

首先我们通过 `JSON.stringify(args)` 将参数序列化为字符串作为缓存的索引。但是如果输入参数不能被序列化，那么 memoize 函数将不会起作用。

其次我们将调用结果都缓存到了 memo 对象中，所以已缓存的结果会一直占用着内存，而不会被清除。

最后要注意的是缓存只对纯函数有效，即对于相同的输入参数，一定会得到相同的输出结果。例如：

```javascript
let bar = 1;

const foo = (baz) => {
  return baz + bar;
}

foo(1);
bar++;
foo(1); // 得到了错误的结果
```

## 基于 WeakMap 的缓存

weakmap 相比 object 有两个特点，一是只能使用 object 作用键，不接受基本类型。二是 WeakMap 持有是对象的“弱引用”，在没其他地方有对这个对象的引用的时候，将会进行垃圾回收。例子：

```javascript
let john = { name: "John" };

let weakMap = new WeakMap();
weakMap.set(john, "...");

john = null;

// john 会从内存中移除
```

现在我们基于 WeakMap 做 memoizee:

```javascript
const memoize = (fn) => {
  const memo = new WeakMap();
  return (obj) => {
    if (!memo.get(obj)) {
      const result = fn(obj);
      memo.set(obj, result);
    }

    return memo.get(obj);
  };
};
```

检测缓存是否命中：

```javascript
const keys = memoize((obj) => {
  return Object.keys(obj);
});

const foo = { bar: 1 };

const key1 = keys(foo);

const key2 = keys(foo);

console.log(key1 === key2); // true
```

## memoize-one

有些时候，我们仅仅只需要缓存最后一次调用的参数和结果。比如解决在 React 的重复渲染问题。

[memoize-one](https://github.com/alexreardon/memoize-one) 就是用来做这个的，它的代码很精简：

```javascript
import areInputsEqual from './are-inputs-equal';

// Using ReadonlyArray<T> rather than readonly T as it works with TS v3
export type EqualityFn = (newArgs: any[], lastArgs: any[]) => boolean;

function memoizeOne<
  // Need to use 'any' rather than 'unknown' here as it has
  // The correct Generic narrowing behaviour.
  ResultFn extends (this: any, ...newArgs: any[]) => ReturnType<ResultFn>
>(resultFn: ResultFn, isEqual: EqualityFn = areInputsEqual): ResultFn {
  let lastThis: unknown;
  let lastArgs: unknown[] = [];
  let lastResult: ReturnType<ResultFn>;
  let calledOnce: boolean = false;

  // breaking cache when context (this) or arguments change
  function memoized(this: unknown, ...newArgs: unknown[]): ReturnType<ResultFn> {
    if (calledOnce && lastThis === this && isEqual(newArgs, lastArgs)) {
      return lastResult;
    }

    // Throwing during an assignment aborts the assignment: https://codepen.io/alexreardon/pen/RYKoaz
    // Doing the lastResult assignment first so that if it throws
    // nothing will be overwritten
    lastResult = resultFn.apply(this, newArgs);
    calledOnce = true;
    lastThis = this;
    lastArgs = newArgs;
    return lastResult;
  }

  return memoized as ResultFn;
}

// default export
export default memoizeOne;
// named export
export { memoizeOne };
```

## React memo

[React.memo](https://reactjs.org/docs/react-api.html#reactmemo) 用于 React 中的函数式组件，对于相同的 props，会跳过这次渲染，重用最后一次渲染的结果。本质上其实就是 memoizeOne。

## reselect

[reselect](https://github.com/reduxjs/reselect) 用于缓存 Redux 的 selector。实际上也是缓存最后一次调用结果。默认的 memoize 函数如下：

```javascript
export function defaultMemoize(func, equalityCheck = defaultEqualityCheck) {
  let lastArgs = null
  let lastResult = null
  // we reference arguments instead of spreading them for performance reasons
  return function () {
    if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
      // apply arguments instead of spreading for performance.
      lastResult = func.apply(null, arguments)
    }

    lastArgs = arguments
    return lastResult
  }
}
```

> - [WeakMap](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakMap)
> - [reactmemo](https://reactjs.org/docs/react-api.html#reactmemo)
> - [reselect](https://github.com/reduxjs/reselect)
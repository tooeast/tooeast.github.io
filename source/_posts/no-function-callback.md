---
title: 不要将函数当作回调使用
date: 2021-05-28 15:26:09
tags: [JS,笔记]
---

> 原作者：[Jake Archibald](https://link.zhihu.com/?target=https%3A//jakearchibald.com/)
> 
> 原文链接：[https://jakearchibald.com/2021/function-callback-risks/](https://link.zhihu.com/?target=https%3A//jakearchibald.com/2021/function-callback-risks/)
> 
> 译者：[当我是树吧](https://www.zhihu.com/people/zhu-kun-20-67)、[塔萨达尔](https://www.zhihu.com/people/wang-jiang-bo-29)

这似乎是一个正在卷土重来的旧模式：

```javascript
// 把数字转化成千分位分隔的字符串。
import { toReadableNumber } from 'some-library';
const readableNumbers = someNumbers.map(toReadableNumber);
```

```toReadableNumber``` 的实现是这样的：

```javascript
export function toReadableNumber(num) {
  // 把数字转化成千分位分隔的字符串。
  // 例如 10000000 转化成 '10,000,000'
}
```

这样看起来一切正常，但是如果```some-library```做了升级，结果可能就不一样了。因为```toReadableNumber```一开始可能就不是被设计用来作为```array.map```的参数的。

问题出在这里：

```js
// 我们这样写:
const readableNumbers = someNumbers.map(toReadableNumber);
// 我们以为的效果:
const readableNumbers = someNumbers.map((n) => toReadableNumber(n));
// 实际的效果:
const readableNumbers = someNumbers.map((item, index, arr) =>
  toReadableNumber(item, index, arr),
);
```

可以看到，除了```item```参数，还多余的传递了索引和数组本身给```toReadableNumber```。当```toReadableNumber```只有一个参数的时候这没什么问题，但是当发生如下修改时：

```js
export function toReadableNumber(num, base = 10) {
  // 把数字转化成千分位分隔的字符串。
  // 传入一个参数 base，默认值是 10
}
```

开发人员已经尽力适配了老代码。虽然为```toReadableNumber```增加了一个参数，但是指定了默认值，这样做本身是没有任何问题的。但谁曾想在某些代码中```toReadableNumber```已经被传了三个参数！

以上，由于```toReadableNumber```不是专门设计作为```array.map```的回调，所以更安全靠谱的做法是写一个适用的函数，然后单独调用```toReadableNumber```：

```js
const readableNumbers = someNumbers.map((n) => toReadableNumber(n));
```

这样做的好处是当```toReadableNumber```再增加参数，也不会造成代码错误。

### Web 平台提供的函数也会有相同的问题

最近看到这样一段代码：

```js
// A promise for the next frame:
const nextFrame = () => new Promise(requestAnimationFrame);
```

等效于：

```js
const nextFrame = () =>
  new Promise((resolve, reject) => requestAnimationFrame(resolve, reject));
```

之所以现在没有问题，是因为```requestAnimationFrame```只接受一个参数。如果将来```requestAnimationFrame```增加了额外的参数，所有进行了这项升级的浏览器在运行上述代码的时候都可能崩溃。

这个例子很好地反应了这种模式是如何出错的：

```js
const parsedInts = ['-10', '0', '10', '20', '30'].map(parseInt);
```

如果面试中遇到这样的问题，建议你翻翻白眼直接离场。但我还是说一下吧，答案是```[-10, NaN, 2, 6, 12]```，因为 [parseInt](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseInt) 有第二个参数。

## 对象参数也有相同的问题

Chrome 90 将允许你使用一个`AbortSignal`来删除一个事件监听器，这意味着一个单独的`AbortSignal`可以用来删除事件监听器、[取消请求](https://link.zhihu.com/?target=https%3A//developers.google.com/web/updates/2017/09/abortable-fetch)、以及做其他任何支持信号的事情。

```js
const controller = new AbortController();
const { signal } = controller;

el.addEventListener('mousemove', callback, { signal });
el.addEventListener('pointermove', callback, { signal });
el.addEventListener('touchmove', callback, { signal });

// 移除所有监听
controller.abort();
```

我们期望的写法是这样的：

```js
const controller = new AbortController();
const { signal } = controller;
el.addEventListener(name, callback, { signal });
```

但是有人这样写：

```js
const controller = new AbortController();
el.addEventListener(name, callback, controller);
```

和前面的例子一样，现在这样写没问题，不代表以后不会出问题。

```AbortController```并不是被设计用来作为```addEventListener```的参数。之所以现在可以运行，是因为 [AbortController](https://dom.spec.whatwg.org/#abortcontroller) 和 [addEventListener](https://link.zhihu.com/?target=https%3A//dom.spec.whatwg.org/%23dictdef-addeventlisteneroptions)有且只有一个同名属性，那就是```signal```。

如果说将来`AbortController`增加一个`controller.capture(otherController)`方法，那么结果就会失控，你的监听器的行为就改变了。因为 `addEventListener` 会把 `capture` 视为一个真值，`capture` 对 `addEventListener` 来说是一个有效的选项（译者注：`addEventListener`的第三个参数`options`是一个对象，包含一个名为`capture`的布尔值字段）。

正如那个回调的例子，最好创建一个对象专门来做 `addEventListener` 的第三个参数`options`。

```js
const controller = new AbortController();
const options = { signal: controller.signal };
el.addEventListener(name, callback, options);
// 或者这样
const { signal } = controller;
el.addEventListener(name, callback, { signal });
```

综上所述，小心作为回调使用的函数，以及作为选项使用的对象，除非它们就是专为此设计的。我不知道有哪一条 linting 规则可以捕捉到它。（注：好像[这个规则](https://github.com/sindresorhus/eslint-plugin-unicorn/blob/main/docs/rules/no-array-callback-reference.md)可以捕获一些情况，感谢 [James Ross](https://twitter.com/CherryJimbo/status/1355190401037180931) !）

## TypeScript 并不能解决这个问题

> 注：当我第一次发布文章的时候，我就说了 TypeScript 不能防止这些问题，但是还是有些人在 Twitter 上告诉我说，”用 TypeScript 就行了“，所以我们就好好唠唠这个事情。

TypeScript [这样写](https://www.typescriptlang.org/play?ts=4.2.0-beta#code/GYVwdgxgLglg9mABAgpgQQE4HMAUBDbARgC5EBnKDGMLASkQG8AoRRCBMuAGxQDou4uAlkK0A3EwC+TJqky4A5AAsUXAQoA0iBQHc4GLgBMF4oA)会报错：

```ts
function oneArg(arg1: string) {
  console.log(arg1);
}

oneArg('hello', 'world');
//              ^^^^^^^
// Expected 1 arguments, but got 2.
```

但是[这么写](https://www.typescriptlang.org/play?ts=4.2.0-beta#code/GYVwdgxgLglg9mABAgpgQQE4HMAUBDbARgC5EBnKDGMLASkQG8AoRRCBMuAGxQDou4uAlkK0A3EwC+TJqEiwEiKAHc4mLAGE8XLgCM8EANY4Iu0viKkKVGgBpEwgExXK1OogC8APkQA3ODAAJvTMrKY4AOQAFig6cBH2EaoYXIER4lIyKmrYWjr6Rjio6uJAA)就不报错：

```ts
function twoArgCallback(cb: (arg1: string, arg2: string) => void) {
  cb('hello', 'world');
}

twoArgCallback(oneArg);
```

。。。虽然结果是一样的（译者注：都是 `oneArg` 方法接收了两个参数）。

因此 TypeScript [这么写](https://www.typescriptlang.org/play?ts=4.2.0-beta&ssl=7&ssc=57&pln=1&pc=1#code/GYVwdgxgLglg9mABFOAlApgQwCaYEYA26AciALZ7oBOAFGOQFyL0XUCUTAzlFTGAOaIA3gChEiAPQTEGKCCpIWiTJ0TdeAxH2WIAFuUxIqWXIXSJgcKmQB0YydICiggIwAGD57eIyMfrqhESgg4MnMAcncAGk8Yj3D7YzkFRHDwgG4RAF8RERCwbkRjHHwiUlYqVQBeRABtFyjEACZGgGYAXRsyTAAHGhQMErNyyio2dKA)也不报错：

```ts
function toReadableNumber(num: number): string {
  // 把数字转化成千分位分隔的字符串。
  // 例如 10000000 转化成 '10,000,000'
  return '';
}

const readableNumbers = [1, 2, 3].map(toReadableNumber);
```

如果 `toReadableNumber` 添加一个 _`string`_ 类型的参数，[TypeScript 会报错](https://www.typescriptlang.org/play?ts=4.2.0-beta&ssl=7&ssc=57&pln=1&pc=1#code/GYVwdgxgLglg9mABFOAlApgQwCaYEYA26AciALZ7oBOAFGOQFyL0XUA0iA7gBaZToA3akwDOUKjDABzAJSjxkqYgDeAKESIA9JsQYoIKkhaJMIxGInTEkk4m7lMSKllyF0iYHCpkAdOq06AKJKAIwADBGRYYhkMFLcUIiUEHBk7gDk4WyR2RHp-s76hojp6QDcqgC+qqopYGKIzjj4RKSsVGYAvIgA2iEcAEwcAMwAuj5kmAAONCgYzW5tlFQyZUA)，但是这个检查对我们这个例子来说没有用。我们新增的参数类型是 _`number`_， [它符合类型约束](https://www.typescriptlang.org/play?ts=4.2.0-beta#code/GYVwdgxgLglg9mABFOAlApgQwCaYEYA26AciALZ7oBOAFGOQFyL0XUA0iA7gBaZToA3akxaUqASiYBnKFRhgA5ogDeAKESIA9JsQYoIKkhaJMUxDLmLE8k4m7lMSKllyF0iYHCpkAdOq06AKJKAIwADBGRYYhkMArcUIiUEHBk7gDk4WyR2RHp-s76hojp6QDcqgC+qqopYDKIzjj4RKSsVGYAvIgA2iEcAEwcAMwAuj5kmAAONCgYzW5tYuJlQA)。

对于```requestAnimationFrame```来说情况会更加糟糕，只要使用新版本浏览器的时候就会出错，和项目版本无关。此外，TypeScript DOM 类型往往落后于浏览器几个月。

尽管我是 TypeScript 的粉丝，此博客是使用 TypeScript 构建的，但它不能解决此问题，而且可能不应该解决。

从这一点来说，大部分其他类型语言表现得和 TypeScript 都不相同，并且[不允许以这种方式进行回调](https://dartpad.dev/342c8d1bb4779bd1ff10610bb3e9ac30)。但是 TypeScript 是[有意为之](https://github.com/Microsoft/TypeScript/wiki/FAQ#why-are-functions-with-fewer-parameters-assignable-to-functions-that-take-more-parameters)，否则下面的代码将无法运行，因为被传入的回调函数被传入了多余的参数。

```ts
const numbers = [1, 2, 3];
const doubledNumbers = numbers.map((n) => n * 2);
```

在 JavaScript 中这是非常常见的做法，并且非常安全。所以 TypeScript 这样做也是情理之中。

问题是“这个函数是不是用来做 `map` 的回调的”，对于 JavaScript 来说，类型并不能真正解决问题。相反，我好奇 JS 是不是应该在传入多余参数的时候抛出错误，确实，这将'保留'额外的参数位置以便之后的升级。但在现有功能上直接改造是不现实的，这样会造成兼容性的问题，但是我们可以现在增加控制台的警告。我提出过这样的想法，但是并没有多少人感兴趣。

另外当涉及选项对象时，事情会变得有些棘手：

```ts
interface Options {
  reverse?: boolean;
}

function whatever({ reverse = false }: Options = {}) {
  console.log(reverse);
}
```

按照上面的说法，如果传递给```whatever```的对象具有```reverse```之外的属性，则 API 应该发出警告。我们看下面的示例：

```ts
whatever({ reverse: true });
```

我们传入的是一个 `Object` 的实例，它天然就有额外的属性，像`toString`，`constructor`，`valueOf`，`hasOwnProperty`等。所以要求属性都是“自有”属性（这不是它在运行时的工作方式）似乎过于严格，所以对于 `Object`自身的属性或许应该考虑放宽一些限制。

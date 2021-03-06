---
title: JS正则表达式学习二
date: 2018-11-23 18:11:40
tags: [正则表达式,笔记]
---
先贴一下老姚文章的[地址](https://juejin.im/post/5965943ff265da6c30653879)。

> 上次主要记录了字符匹配的部分，接下来学习位置匹配的部分

#### 1. 位置是什么，如何匹配

位置其实很好理解，就是相邻的字符之间的位置。比如常见的最开头和最结尾。在 es5 中共有六个锚字符，也就是表示位置的字符：```^ $ \b \B (?=p) (?!p)```。其中```^```和```$```在多行匹配时，表示每行的开头和结尾。

```\b```和```\B```分别表示单词边界和非单词边界。即单词之间的位置和单词内部每个元素间的位置。

```(?=p)```表示 ```p```前面的位置。相对的，```(?!p)```表示非```p```前面的位置。比如说如下：
```javascript
var result = "hello".replace(/(?=l)/g, '#');
console.log(result); 
// => "he#l#lo"

var result = "hello".replace(/(?!l)/g, '#');
console.log(result); 
// => "#h#ell#o#"
```

#### 2. 例子

* 数字千分位分隔符

实际开发中，经常会有需要添加数字千分位的需求。例如```12345678``` 变成 ```12,345,678```。代码如下：

```javascript
var result = "12345678".replace(/(?!^)(?=(\d{3})+$)/g, ',')
console.log(result); 
// => "12,345,678"
```

这个正则表示，至少出现一次三个数字连续出现的时候，则在三个数字前面的位置添加一个逗号。最前面的```(?!^)```表示这个位置不能是开头，否则会出现 ```,123,456,789``` 的情况。

* 密码校验问题

我们都知道，用户输入密码的时候是很有必要做检验的。很多平台就会规定密码必须包含数字大小写字母特殊符等，而且有长度限制。

比如光考虑长度和组成问题的话，会有``` /^[0-9A-Za-z]{6,12}$/```。这很明显，表示包含数字或者大小写字母的6到12位的字符串。但是这种情况不能判断是否包含某种元素，如不能判定是否包含数字，所以又了接下来的升级版。

```/(?=.*[0-9])^[0-9A-Za-z]{6,12}$/``` 这个在之前的基础上增加了必须包含数字的校验。其中```(?=.*[0-9])```中的 ```.*``` 表示任意字符，所以可以解释成有一个不管前面是啥，反正包含数字的串，如 ```ABC1```，就会通过匹配。还有 ```^```表示开头的位置。所以其实这两个表示的是同一个位置。但是通常的要求是同时包含多种字符，接下来继续升级。

```/(?=.*[0-9])(?=.*[a-z])^[0-9A-Za-z]{6,12}$/```。这样就表示必须同时包含数字和小写字母。做了两次位置的校验。接下来是终极的答案。

```javascript
var reg = /((?=.*[0-9])(?=.*[a-z])|(?=.*[0-9])(?=.*[A-Z])|(?=.*[a-z])(?=.*[A-Z]))^[0-9A-Za-z]{6,12}$/;
console.log( reg.test("1234567") ); // false 全是数字
console.log( reg.test("abcdef") ); // false 全是小写字母
console.log( reg.test("ABCDEFGH") ); // false 全是大写字母
console.log( reg.test("ab23C") ); // false 不足6位
console.log( reg.test("ABCDEF234") ); // true 大写字母和数字
console.log( reg.test("abcdEF234") ); // true 三者都有
```
以上表示这个密码必须符合下列三种情况之一：同时包含数字和小写字母、同时包含数字和大写字母、同时包含小写字母和大写字母。符合任意一种就会通过。对于这种情况下，我们也可以使用```(?!p)```。因为要求三种字符（数字、小写字母、大写字母）至少出现两种，则意味着不能光有一种字符，所以还有如下写法。

```javascript
var reg = /(?!^[0-9]{6,12}$)(?!^[a-z]{6,12}$)(?!^[A-Z]{6,12}$)^[0-9A-Za-z]{6,12}$/;
console.log( reg.test("1234567") ); // false 全是数字
console.log( reg.test("abcdef") ); // false 全是小写字母
console.log( reg.test("ABCDEFGH") ); // false 全是大写字母
console.log( reg.test("ab23C") ); // false 不足6位
console.log( reg.test("ABCDEF234") ); // true 大写字母和数字
console.log( reg.test("abcdEF234") ); // true 三者都有
```

其中 ```(?!^[0-9]{6,12}$)``` 表示不能全是数字，其他同理。

> 学习到这里其实脑袋是有点懵的，关于位置的匹配本来使用的就不多。幸亏看到了这篇干货文章，好好学习了！

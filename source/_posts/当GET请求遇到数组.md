---
title: 当GET请求遇到数组
date: 2021-04-12 17:24:20
tags: [前端,HTTP]
---
 ### 前言

 我们日常开发中，经常会遇到要在请求中携带数组的情况。但是由于encode，数据都会被转化为字符串传输。我们来探究这种情况下数组该如何转换，有哪些形式来转换。当然，不管哪种形式的传输，都需要服务器接口支持对应的格式。
 
 > 一般情况下不建议用 GET 传输数组，因为无法保证数组长度，可能会超出get长度限制。在确保小数据量的情况下，我们可以讨论。

### 形式

通常情况下，我们会把数组转换成别的形式，比如说转化成逗号分隔的字符串或者直接转化成 json 来传输。

```javascript
arr = [1, 2, 3];

// 转化成逗号分隔
?arr=1,2,3

// 转化成json
?arr=[1,2,3]
```

这两种形式是比较常用的转化形式，通过服务器同学的解码，可以重新得到数组。但是除了这两种形式，还有哪些形式吗？

通过学习，我们了解到，常常被使用的还有如下几种形式：

```javascript
arr = [1, 2, 3];

// 带 []
?arr[]=1&arr[]=2&arr[]=3

// 带 []，且有下标的
?arr[0]=1&arr[1]=2&arr[2]=3

// 重复出现
?arr=1&arr=2&arr=3
```

### 实现

这时候是不是有个疑惑，那我们每次传参之前都得手动转一遍，岂不是很麻烦？

其实 qs 就自带了这个功能。

```javascript
qs.stringify({ a: ['b', 'c'] }, { arrayFormat: 'indices' })
// 'a[0]=b&a[1]=c'
qs.stringify({ a: ['b', 'c'] }, { arrayFormat: 'brackets' })
// 'a[]=b&a[]=c'
qs.stringify({ a: ['b', 'c'] }, { arrayFormat: 'repeat' })
// 'a=b&a=c'
qs.stringify({ a: ['b', 'c'] }, { arrayFormat: 'comma' })
// 'a=b,c'
```

我们也可以自己来实现一遍

```javascript
function formatArrayFunc(key, array, format) {
  switch(format) {
    case 'brackets':
      return bracketsArray(key, array);

    case 'indices':
      return indicesArray(key, array);

    case 'repeat':
      return repeatArray(key, array);

    case 'comma':
      return commaArray(key, array);

    default:
      return `${key}=${JSON.stringify(array)}`;
  }
}


function stringifyParams(params, formatArray = 'comma') {
  let str = [];

  Object.keys(params).forEach(key => {
    if(Array.isArray(params[key])) {
      str.push(formatArrayFunc(key, params[key], formatArray));
    }
    else {
      str.push(`${String(key)}=${String(params[key])}`);
    }
  });

  return '?' + str.join('&');
}

function bracketsArray(key, array) {
  return array.map(item => {
    return `${key}[]=${item}`
  }).join('&');
}

function indicesArray(key, array) {
  return array.map((item, index) => {
    return `${key}[${index}]=${item}`
  }).join('&');
}

function repeatArray(key, array) {
  return array.map(item => {
    return `${key}=${item}`
  }).join('&');
}

function commaArray(key, array) {
  return `${key}=${array.join(',')}`;
}

const params = {
  name: 'lzd',
  list: ['a', 'b', 'c'],
  isvip: true
}

console.log(stringifyParams(params, 'indices'));
// ?name=lzd&list[0]=a&list[1]=b&list[2]=c&isvip=true

console.log(stringifyParams(params, 'brackets'));
// ?name=lzd&list[]=a&list[]=b&list[]=c&isvip=true

console.log(stringifyParams(params, 'repeat'));
// ?name=lzd&list=a&list=b&list=c&isvip=true

console.log(stringifyParams(params, 'comma'));
// ?name=lzd&list=a,b,c&isvip=true
```

### 接收

以上我们讨论的数组的转化形式，都需要服务器做相应的处理，可以事先就约定好形式。有的形式需要接口中处理，如逗号分隔形式，json形式，有的形式个别框架会自动转换，如 brackets、indices。

我们可以实现一个接收各种形式数组转化，并且转化成真数组的方法。

> 我们主要针对 indices、brackets、repeat 三种形式，因为这个明确是数组形式。但是 comma 和 逗号分隔，其本来形式并不一定是数组。所以需要针对特点场景来做具体处理，这里不做处理

```javascript
function parseParam(url) {
  const paramsUri = /.*\?(.*)$/.exec(url)[1];

  // 没匹配到直接返回空对象
  if(!paramsUri) return {};

  const keyValues = paramsUri.split('&');

  const result = {}
  keyValues.forEach(item => {
    // 检查是否带有的等号
    if(/=/.test(item)) {
      const split = item.split('=');

      // 是否带有 []，带的话，肯定是数组
      if(/\[(\d?)\]/.test(split[0])) {
        const array = (/(.*)\[(\d?)\]/.exec(split[0]));

        // 检查是 list[]=1&list[]=2 类型 还是 list[1]=1&list[2]=2
        if(array[2] === '') {
          
          if(!result[array[1]]) {
            result[array[1]] = [];
          }
          result[array[1]].push(split[1])
        }
        else {
          if(!result[array[1]]) {
            result[array[1]] = [];
          }
          result[array[1]][array[2]] = split[1]
        }
      }
      // 没有 []，不一定不是数组，可能是 repeat 形式的
      else {
        if(result[split[0]]) {
          if(Array.isArray(result[split[0]])) {
            result[split[0]].push(split[1])
          }
          else {
            result[split[0]] = [result[split[0]], split[1]]
          }
        }
        else {
          result[split[0]] = split[1]
        }
      }
    }
    else {
      // 没带等号的，默认值为 true
      result[item] = true;
    }
  })

  return result;
}

const url = "baidu.com?testname=yiqihaochidian&list[0]=lzd&list[1]=la&list[2]=lml&qs[]=1&qs[]=2&isvip&business=changba&user=lzd&user=liao&user=ys";

console.log(parseParam(url));

// {
//   testname: 'yiqihaochidian',
//   list: [ 'lzd', 'la', 'lml' ],
//   qs: [ '1', '2' ],
//   isvip: true,
//   business: 'changba',
//   user: [ 'lzd', 'liao', 'ys' ]
// }
```


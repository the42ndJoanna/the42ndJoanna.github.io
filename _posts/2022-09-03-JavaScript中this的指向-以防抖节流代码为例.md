---
title: JavaScript中this的指向：以防抖节流代码为例
published: true
---

# 前情提要

防抖/节流代码通常会传入两个参数，一个是最终调用的函数，一个是 delay 时间，在参考网络代码时，发现有些代码实现会用`apply`为传入的函数绑定`this`，但有些情况又不需要，遂打算梳理一下箭头函数与非箭头函数中 this 的指向。

<br />

一些众所周知的总结（通俗版，就不使用“词法环境”这种不能一目了然的词了）：

不是箭头函数时，函数内的`this`指向**实际**调用了这个函数的某个对象。

是箭头函数时，函数内的`this`指向函数**所在的作用域指向的对象**。

<br />

接下来就以防抖函数为例（节流函数同理），具体情况具体分析。本文使用“自顶向下”的分析结构，先从`debounce`绑定浏览器事件开始说起，再说`debounce`的参数，最后说`debounce`返回的函数。

<br />
<br />

# 防抖函数绑定`document`事件时的`this`

先抛开防抖函数内部不谈，从防抖函数绑定`document`事件开始讲起。

<br />

下列写法，在绑定时已经调用了`debounce`。其实，真正绑定 onclick 事件的是`debounce`被调用后`return`的函数，而`debounce`已经先在全局范围内被调用了。所以，当`onclick`事件触发时，在`debounce`内部，`this`指向`window`：

```jsx
function debounce(fn, delay) {
// 在这一层的this是window
  return ...
}

document.getElementById("button").onclick = debounce()
```

<br />

下列写法，在绑定时没有**调用**`debounce`，而是直接把`debounce`绑定到 button 元素上。所以，当`onclick`事件触发时，在`debounce`内部，`this`指向 button 元素：

```jsx
function debounce(fn, delay) {
// 在这一层的this是button元素
  return ...
}

document.getElementById("button").onclick = debounce
```

这就是“`this`指向**实际**调用了这个函数的某个对象”的意义。

<br />
<br />

# 如果把防抖函数改成匿名函数或箭头函数…

无论匿名与否，只要不是箭头函数，`this`绑定的都是 button 元素，即实际调用了该函数的对象：

```jsx
// 匿名函数
document.getElementById("button").onclick = function () {
  console.log(this); // this为button元素
};

// 非匿名函数
document.getElementById("button").onclick = function debounce() {
  console.log(this); // this为button元素
};
```

<br />

但如果是箭头函数，无论匿名与否，`this`绑定的都是`window`，即函数所在作用域指向的对象：

```jsx
// 此时箭头函数的作用域都是全局

// 匿名箭头函数
document.getElementById("button").onclick = () => {
  console.log(this); //this为window
};

// 非匿名箭头函数
const callMe = () => {
  console.log(this); //this为window
};

document.getElementById("button").onclick = callMe;
```

<br />
<br />

# `fn`参数传入防抖函数，`fn`中的`this`

在防抖函数中，通常都有一个`fn`参数，接下来梳理`fn`中`this`的指向。

<br />

`fn`不是箭头函数时，无论匿名与否，`this`指向的都是`window`：

```jsx
document.getElementById("button").onclick = debounce(function callMe() {
  console.log(this); // this指向window
}, 200);

document.getElementById("button").onclick = debounce(function () {
  console.log(this); // this指向window
}, 200);
```

这时，为什么“实际调用了函数”的对象是`window`呢？笔者也好奇。随后搜索到了一篇相关博客：[作为参数的函数中上下文（this）的问题](https://www.zjy.name/context-of-function-as-parameter/)

根据这位作者得到的结论：**如果函数没有直接被某个对象调用（比如，作为参数传入），那么 this 指代的上下文环境是全局对象（严格模式下是 undefined）。**

经过实验，该结论应该是正确的，时间有限，这里暂且不再深入。

<br />

所以，`fn`是箭头函数时，无论匿名与否，`this`指向的也都是`window`：

```jsx
// 匿名箭头函数
document.getElementById("button").onclick = debounce(() => {
  console.log(this); //this为window
}, 200);

// 非匿名箭头函数
const callMe = () => {
  console.log(this); //this为window
};

document.getElementById("button").onclick = debounce(callMe, 200);
```

<br />
<br />

# 防抖函数`return`的函数中，`this`指向何处

`return`的函数不是箭头函数时，`this`指向 button，`arguments`是 button 元素的`event`，是否匿名对其没有影响：

```jsx
// debounce内部实现略去，只展示“返回一个函数”

// 返回非匿名函数
function debounce(fn, wait) {
  return function returnMe() {
    console.log(this); // this指向button
    console.log(arguments); // arguments为button元素的event
  };
}

// 返回匿名函数
function debounce(fn, wait) {
  return function () {
    console.log(this); // this指向button
    console.log(arguments); // arguments为button元素的event
  };
}

document.getElementById("button").onclick = debounce(() => {}, 200);
```

这也再次印证了“`this`指向**实际**调用了这个函数的某个对象”，在上方的例子中，实际调用`returnMe`函数的对象是 button 元素。

<br />

`return`的函数是箭头函数时，`this`指向`window`，`arguments`是`debounce`传入的两个参数`fn`和`delay`，是否匿名对其没有影响：

```jsx
// debounce内部实现略去，只展示“返回一个函数”

// 返回非匿名函数
function debounce(fn, delay) {
  const returnMe = () => {
    console.log(this); // this指向window
    console.log(arguments); // arguments是fn和delay
  };
  return returnMe;
}

// 返回匿名函数
function debounce(fn, delay) {
  return () => {
    console.log(this); // this指向window
    console.log(arguments); // arguments是fn和delay
  };
}

document.getElementById("button").onclick = debounce(() => {}, 200);
```

上例中，`reutrnMe`函数所在的作用域是`debounce`函数中，`debounce`函数指向的`this`是`window`，再次印证了箭头函数“`this`指向函数**所在的作用域指向的对象**”。

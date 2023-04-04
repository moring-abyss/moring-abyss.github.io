---
title: 笔记：从零实现一个简单的vue-1
---

Vue 是一个声明式的 UI 框架，简单来理解什么是声明式，即为你声明你要什么，不需要关心内部具体的实现，只需要得到结果。
例如：这段代码在 vue 中表示一个绑定了点击事件的 div，不需要关心事件是如何绑定的

```javascript
<script setup>
// 函数
function handler() {
  console.log("click")
}
</script>

<template>
  <div @click="handler"></div>
</template>
```

而在命令式编程中，需要编写具体的事件绑定逻辑

```html
<div></div>

document.querySelector("div").addEventListener("click", function() {
console.log("click") })
```

Vue 只需要开发者描述真实的 DOM 节点树结构，隐藏了内部具体的实现。

我们可以创建一个 JavaScript 对象来描述一个 DOM 元素结构，一个 DOM 节点的元素结构包含：

1. 标签名：div span a 等
2. 属性：如 name id class 等
3. 事件：click hover keydown 等
4. 层级结构：父子节点、兄弟节点

转换为代码：

```javascript
const vNode = {
  // 标签名称
  tag: "div",
  // 标签属性
  props: {
    onClick: handler,
  },
  // 没有子节点
  children: [],
};
```

在理解了声明式 UI 编程后，我们需要实现一个编译器来把 DOM 元素的声明式代码转换为具体的 JavaScript 对象

编译原理中的编译过程包含：词法分析、语法分析、语义分析、中间代码生成、优化、目标代码生成等步骤
将要实现的 Vue 模版编译器包含了两个个部分：

1. parser-将模版字符串解析为模版 AST（词法分析、语法分析、语义分析）
2. transformer-将模版 AST 转换为 JavaScript AST（生成代码）

# 实现tokenize
首先是词法分析部分：
> ps: 此为简化版，不处理单闭合标签、各类异常情况以及复杂的 vue 语法等，类似<></> 、 \<tagName />
> 词法分析中，解析器会逐个读取字符，根据词法规则切分成一个个词法记号，即 Token

```html
<div>这是一段html代码</div>
```

这一段代码可以切分为 3 个词法单元：

1. 开始标签：<div>
2. 文本节点：这是一段 html 代码
3. 结束标签：</div>

这里定义一个有限状态机，通过不同状态的迁移来切分
可以参考 whatwg 关于<a href="https://html.spec.whatwg.org/multipage/parsing.html#parse-state" target="_blank" style="color: blue;text-decoration: underline;">html 解析器状态</a>的定义简化定义我们需要的各种状态：

```javascript
const State = {
  initial: 1, // 初始状态
  tagOpen: 2, // 标签开始状态
  tagName: 3, // 标签名称状态
  text: 4, // 文本状态
  tagEnd: 5, // 结束标签状态
  tagEndName: 6, // 结束标签名称状态
};
```

词法规则有了，状态定义好了，开始实现 tokenize

```javascript
const State = {
  initial: 1, // 初始状态
  tagOpen: 2, // 标签开始状态
  tagName: 3, // 标签名称状态
  text: 4, // 文本状态
  tagEnd: 5, // 结束标签状态
  tagEndName: 6, // 结束标签名称状态
};
/**
 * @param str 模版字符串
 * @return 返回Token数组
 */
function tokenize(str) {
  // 设置当前状态为初始状态
  let currentState = State.initial;
  // 保存token的数组
  const tokens = [];
  // 缓存当前读取的字符
  const chars = [];
  // 当模版字符串读取完毕后退出循环
  while (str) {
    // 读取第一个字符
    const char = str[0];
    // 匹配状态机当前处于哪一个状态
    switch (currentState) {
      case State.initial:
        break;
      case State.tagOpen:
        break;
      case State.tagName:
        break;
      case State.text:
        break;
      case State.tagEnd:
        break;
      case State.tagEndName:
        break;
    }
  }
  // 最后返回token数组
  return tokens;
}
```

这里已经大致缕清了思路，只需要按不同的状态去处理就行了。

> 状态机当前处于初始状态

```javascript
if (char === '<') {
  // 状态机切换到标签开始状态
  currentState = State.tagOpen
} else {
  // 初始状态下碰到其他字符一律当作文本节点处理
  // 1. 切换到文本状态
  currentState = State.text
  // 2. 将当前读取字符缓存
  chars.push(char)
}
// 消费掉字符
str = str.slice(1)
break;
```

> 状态机当前处于标签开始状态

```javascript
if (char === "/") {
  // 切换到结束标签状态
  currentState = State.tagEnd;
} else {
  // 1. 切换到标签名称状态
  currentState = State.tagName;
  // 2. 缓存当前读取字符
  chars.push(char);
}
// 消费掉该字符
str = str.slice(1);
```

> 状态机当前处于标签名称状态

```javascript
if (char === ">") {
  // 1. 遇到字符>，说明标签闭合，切换到初始状态
  currentState = State.initial;
  // 2. 把之前缓存的字符拼接并创建为一个token
  tokens.push({
    type: "text",
    name: chars.join(""),
  });
  // 3. chars已经被消费了，清空掉
  chars.length = 0;
} else {
  // 继续缓存当前字符
  chars.push(char);
}
// 消费掉该字符
str = str.slice(1);
```

> 状态机当前处于文本状态

```javascript
if (char === "<") {
  // 1. 遇到<，切换到标签开始状态
  currentState = State.tagOpen;
  // 2. 从文本状态 迁移 到标签开始状态时，表示文本节点字符已收集完毕，创建文本token
  tokens.push({
    type: "text",
    content: chars.join(""),
  });
  // 3. chars已经被消费了，清空掉
  chars.length = 0;
} else {
  // 缓存当前字符
  chars.push(char);
}
// 消费掉该字符
str = str.slice(1);
```

> 状态机当前处于标签结束状态

```javascript
// 1. 切换到结束标签名称状态
currentState = State.tagEndName;
// 2. 缓存当前字符
chars.push(char);
// 3. 消费掉该字符
str = str.slice(1);
```

> 状态机当前处于结束标签名称状态

```javascript
if (char === ">") {
  // 1. 标签闭合，切换到初始状态
  currentState = State.initial;
  // 2. 保存标签名称
  tokens.push({
    type: "tagEnd",
    name: chars.join(""),
  });
  // 3. chars已经被消费了，清空掉
  chars.length = 0;
} else {
  // 缓存当前字符
  chars.push(char);
}
// 消费掉该字符
str = str.slice(1);
```

> 完整代码

```javascript
const State = {
  initial: 1, // 初始状态
  tagOpen: 2, // 标签开始状态
  tagName: 3, // 标签名称状态
  text: 4, // 文本状态
  tagEnd: 5, // 结束标签状态
  tagEndName: 6, // 结束标签名称状态
};
/**
 * @param str 模版字符串
 * @return 返回Token数组
 */
function tokenize(str) {
  // 设置当前状态为初始状态
  let currentState = State.initial;
  // 保存token的数组
  const tokens = [];
  // 缓存当前读取的字符
  const chars = [];
  // 当模版字符串读取完毕后退出循环
  while (str) {
    // 读取第一个字符
    const char = str[0];
    // 匹配状态机当前处于哪一个状态
    switch (currentState) {
      case State.initial:
        // 状态机当前处于初始状态
        if (char === "<") {
          // 1. 状态机切换到标签开始状态
          currentState = State.tagOpen;
        } else {
          // 初始状态下碰到其他字符一律当作文本节点处理
          // 1. 切换到文本状态
          currentState = State.text;
          // 2. 将当前读取字符缓存
          chars.push(char);
        }
        // 消费掉该字符
        str = str.slice(1);
        break;
      case State.tagOpen:
        // 状态机当前处于标签开始状态
        if (char === "/") {
          // 切换到结束标签状态
          currentState = State.tagEnd;
        } else {
          // 1. 切换到标签名称状态
          currentState = State.tagName;
          // 2. 缓存当前读取字符
          chars.push(char);
        }
        // 消费掉该字符
        str = str.slice(1);
        break;
      case State.tagName:
        // 状态机当前处于标签名称状态
        if (char === ">") {
          // 1. 遇到字符>，说明标签闭合，切换到初始状态
          currentState = State.initial;
          // 2. 把之前缓存的字符拼接并创建为一个token
          tokens.push({
            type: "tag",
            name: chars.join(""),
          });
          // 3. chars已经被消费了，清空掉
          chars.length = 0;
        } else {
          // 继续缓存当前字符
          chars.push(char);
        }
        // 消费掉该字符
        str = str.slice(1);
        break;
      case State.text:
        // 状态机当前处于文本状态
        if (char === "<") {
          // 1. 遇到<，切换到标签开始状态
          currentState = State.tagOpen;
          // 2. 从文本状态 迁移 到标签开始状态时，表示文本节点字符已收集完毕，创建文本token
          tokens.push({
            type: "text",
            content: chars.join(""),
          });
          // 3. chars已经被消费了，清空掉
          chars.length = 0;
        } else {
          // 缓存当前字符
          chars.push(char);
        }
        // 消费掉该字符
        str = str.slice(1);
        break;
      case State.tagEnd:
        // 状态机当前处于标签结束状态
        // 1. 切换到结束标签名称状态
        currentState = State.tagEndName;
        // 2. 缓存当前字符
        chars.push(char);
        // 3. 消费掉该字符
        str = str.slice(1);
        break;
      case State.tagEndName:
        // 状态机当前处于结束标签名称状态
        if (char === ">") {
          // 1. 标签闭合，切换到初始状态
          currentState = State.initial;
          // 2. 保存标签名称
          tokens.push({
            type: "tagEnd",
            name: chars.join(""),
          });
          // 3. chars已经被消费了，清空掉
          chars.length = 0;
        } else {
          // 缓存当前字符
          chars.push(char);
        }
        // 消费掉该字符
        str = str.slice(1);
        break;
    }
  }
  // 最后返回token数组
  return tokens;
}
```

当模版字符串为<div>这是一段html代码</div>，执行 tokenize 后最终结果为

```json
[
  {"type":"tag","content":"div"},
  {"tag":"text","content":"这是一段html代码"},
  {"type":"tagEnd","content":"div"}
]
```

# 实现parse
根据tokenize解析出的token来构建AST树
构建树的过程需要遍历token数组，从第一个token开始直到处理完所有的token。

以刚刚的示例：
开始标签(div) -> 文本(这是一段html代码) -> 结束标签(div)

处理逻辑按顺序为：
创建div元素节点 -> 创建文本节点，并作为div的子节点插入到父节点中 -> 处理完毕

如果要解析的元素存在多层节点时，子节点如何找到正确的父节点
例如：
```javascript
const tokens = tokenize("<div>父节点文本<span>子节点文本</span></div>");
```
```json
[
  ({ type: "tag", name: "div" },
  { type: "text", content: "父节点文本" },
  { type: "tag", name: "span" },
  { type: "text", content: "子节点文本" },
  { type: "tagEnd", name: "span" },
  { type: "tagEnd", name: "div" }),
]
```
我们可以用数组来模拟栈，通过后进先出的原理来实现，运行逻辑：
创建stack数组，遍历tokens
第一次循环：开始标签div，创建div节点，并推入stack中，此时stack = [div]
第二次循环：文本，创建文本节点，从stack中获取最后一个元素（顶部元素），将该节点作为子节点插入到div中
第三次循环：开始标签span，创建span节点，并推入stack中，此时stack = [div, span]
第四次循环：文本，创建文本节点，从stack中获取最后一个元素（顶部元素），将该节点作为子节点插入到span中
第五次循环：结束标签span，从stack中弹出尾部节点，此时stack = [div]
第六次循环：结束标签div，从stack中弹出尾部节点，此时stack = []

ok,思路清楚了之后我们来写实现的代码
```javascript
/**
 * 根据模版字符串解析AST
 * @param str 模版字符串
 */
function parse(str) {
  // 解析token
  const tokens = tokenize(str);
  // 创建根节点
  const root = {
    type: "root",
    children: [],
  };
  // 创建 elementStack 栈，起初只有 Root 根节点
  const elementStack = [root]
  // 遍历tokens
  while (tokens.length) {
    const token = tokens[0];
    // 获取当前栈顶节点作为父节点 parent
    const parent = elementStack[elementStack.length - 1]
    switch (token.type) {
      case "tag":
        // 开始标签则创建 Element 类型的AST节点
        const elementNode = {
          type: "Element",
          tag: token.name,
          children: []
        };
        // 将其添加到父级节点的 children 中
        parent.children.push(elementNode);
        // 将当前节点压入栈
        elementStack.push(elementNode);
        break;
      case "text":
        // 创建文本节点
        const textNode = {
          type: "Text",
          content: token.content
        }
        // 将其插入到父节点中
        parent.children.push(textNode)
        break;
      case "tagEnd":
        // 遇到结束标签，弹出栈顶元素
        elementStack.pop();
        break;
    }
    // 消费掉token
    tokens.shift()
  }
  return root
}
```

最后生成的AST结构如下：
```json
{
    "type": "root", 
    "children": [
        {
            "type": "Element", 
            "tag": "div", 
            "children": [
                {
                    "type": "Text", 
                    "content": "父节点文本"
                }, 
                {
                    "type": "Element", 
                    "tag": "span", 
                    "children": [
                        {
                            "type": "Text", 
                            "content": "子节点文本"
                        }
                    ]
                }
            ]
        }
    ]
}
```

github代码地址：<a href="https://github.com/moring-abyss/mini-vue" target="_blank" style="color: blue;text-decoration: underline;">mini-vue</a>
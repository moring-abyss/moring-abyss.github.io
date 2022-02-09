---
title: 编译原理学习笔记
tags: 编译原理
---
> 简单的加减乘除计算器

<a href="https://moring-abyss.github.io/example/3.html" target="_blank">完整演示地址</a>

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <input id="int" /><button id="btn">calc</button>
    <script>
      /**
       * 词法分析类
       */
      class SimpleLexer {
        // 所有枚举状态
        static enum = {
          INITIAL: "init",
          PLUS: "plus", // +
          MINUS: "minus", // -
          STAR: "star", // *
          DIVIDE: "divide", // /
          NUMBER: "number",
        };
        read(code) {
          this.state = SimpleLexer.enum.INITIAL;
          this.tokenText = "";
          this.tokenList = [];
          let ch;
          Array.prototype.forEach.call(code, (char) => {
            ch = char;
            this.tokenize(char);
          });
          if (this.tokenText.length > 0) {
            this.initToken(ch);
          }
        }
        tokenize(char) {
          // 词法分析
          switch (this.state) {
            case SimpleLexer.enum.INITIAL:
              // 重新确定后续状态
              this.state = this.initToken(char, true);
              break;
            case SimpleLexer.enum.PLUS:
            case SimpleLexer.enum.MINUS:
            case SimpleLexer.enum.STAR:
            case SimpleLexer.enum.DIVIDE:
              // 退出当前状态
              this.state = this.initToken(char);
              break;
            case SimpleLexer.enum.NUMBER:
              if (this.getTokenType(char) === SimpleLexer.enum.NUMBER) {
                // 保持当前状态
                this.tokenText += char;
              } else {
                // 退出当前状态
                this.state = this.initToken(char);
              }
              break;
          }
        }
        initToken(char) {
          if (this.tokenText.length > 0) {
            this.tokenList.push({
              text: this.tokenText,
              type: this.getTokenType(this.tokenText),
            });
            this.tokenText = "";
          }
          this.tokenText = char;
          return this.getTokenType(char);
        }
        getTokenType(char, init) {
          // 获取token类型
          switch (char) {
            case "+":
              return SimpleLexer.enum.PLUS;
            case "-":
              return SimpleLexer.enum.MINUS;
            case "*":
              return SimpleLexer.enum.STAR;
            case "/":
              return SimpleLexer.enum.DIVIDE;
            default:
              if (!isNaN(char)) {
                return SimpleLexer.enum.NUMBER;
              }
              if (init) {
                return SimpleLexer.enum.INITIAL;
              }
              throw new Error("程序编译异常");
          }
        }
      }
      /**
       * AST节点
       * @params type 类型
       * @params text 文本
       * @params parent 父节点
       * @params children 子节点
       */
      class ASTNode {
        static enum = {
          Programm: "Programm", //程序入口，根节点
          IntDeclaration: "IntDeclaration", //整型变量声明
          ExpressionStmt: "ExpressionStmt", //表达式语句，即表达式后面跟个分号
          AssignmentStmt: "AssignmentStmt", //赋值语句

          Primary: "Primary", //基础表达式
          Multiplicative: "Multiplicative", //乘法表达式
          Additive: "Additive", //加法表达式

          Identifier: "Identifier", //标识符
          IntLiteral: "IntLiteral", //整型字面量
        };
        constructor(type, text) {
          this.type = type;
          this.text = text;
          this.parent = null;
          this.children = [];
        }

        addChild(child) {
          this.children.push(child);
          child.parent = this;
        }
      }
      /**
       * 语法分析
       */
      class SimpleCalculator {
        parse(tokens) {
          // 创建ast根节点
          let astRootNode = new ASTNode(ASTNode.enum.Programm, "Calculator");
          let child = this.additive(tokens);
          if (child != null) {
            astRootNode.addChild(child);
          }
          return astRootNode;
        }
        /**
         * 语法解析-加法表达式
         */
        additive(tokens) {
          let child1 = this.multiplicative(tokens);
          let node = child1;
          let token = tokens[0];
          if (child1 && token) {
            if (
              token.type === SimpleLexer.enum.PLUS ||
              token.type === SimpleLexer.enum.MINUS
            ) {
              token = tokens.shift();
              let child2 = this.additive(tokens);
              if (child2) {
                node = new ASTNode(ASTNode.enum.Additive, token.text);
                node.addChild(child1);
                node.addChild(child2);
              } else {
                throw new Error(
                  "invalid additive expression, expecting the right part."
                );
              }
            }
          }
          return node;
        }
        /**
         * 语法解析-乘法表达式
         */
        multiplicative(tokens) {
          let child1 = this.primary(tokens);
          let node = child1;
          let token = tokens[0];
          if (child1 && token) {
            // 匹配完一个基础表达式后如果token还有值则继续匹配,否则直接返回基础表达式(在这个简单的计算器中也就是表示数值的ast节点)
            if (
              token.type === SimpleLexer.enum.STAR ||
              token.type === SimpleLexer.enum.DIVIDE
            ) {
              token = tokens.shift();
              let child2 = this.multiplicative(tokens);
              if (child2) {
                // 顺利匹配成功，返回乘法表达式的ast节点
                node = new ASTNode(ASTNode.enum.Multiplicative, token.text);
                node.addChild(child1);
                node.addChild(child2);
              } else {
                // 没有匹配到对应运算符，这里就是解析异常了，简单理解为 1 *，缺失表达式右边的部分
                throw new Error(
                  "invalid multiplicative expression, expecting the right part."
                );
              }
            }
          }
          return node;
        }
        /**
         * 语法解析-基础表达式
         * 这里只做数值判断，类似括号，标识符等不考虑
         */
        primary(tokens) {
          let node = null;
          // 预读token
          let token = tokens[0];
          if (token) {
            if (token.type === SimpleLexer.enum.NUMBER) {
              // 如果是数值
              token = tokens.shift();
              node = new ASTNode(ASTNode.enum.IntLiteral, token.text);
            }
          }
          return node;
        }
      }

      /**
       * 求值
       */
      function evaluate(node) {
        let result = 0;
        let child1;
        let child2;
        let value1;
        let value2;
        switch (node.type) {
          case ASTNode.enum.Programm:
            node.children.forEach((child) => {
              result = evaluate(child);
            });
            break;
          case ASTNode.enum.Additive:
            child1 = node.children[0];
            value1 = evaluate(child1);
            child2 = node.children[1];
            value2 = evaluate(child2);
            if (node.text === "+") {
              result = value1 + value2;
            } else {
              result = value1 - value2;
            }
            break;
          case ASTNode.enum.Multiplicative:
            child1 = node.children[0];
            value1 = evaluate(child1);
            child2 = node.children[1];
            value2 = evaluate(child2);
            if (node.text === "*") {
              result = value1 * value2;
            } else {
              result = value1 / value2;
            }
            break;
          case ASTNode.enum.IntLiteral:
            result = parseInt(node.text);
          default:
        }
        // 展示计算过程
        if(value1 && value2) {
          console.log(`${value1}${node.text}${value2}=${result}`)
        }
        return result;
      }

      const intEle = document.getElementById("int");
      const btnEle = document.getElementById("btn");
      const simpleLexer = new SimpleLexer();
      const simpleCalculator = new SimpleCalculator();
      btnEle.addEventListener("click", function (e) {
        const code = intEle.value;
        // 词法分析
        simpleLexer.read(code);
        // 语法分析
        let astNode = simpleCalculator.parse(simpleLexer.tokenList);
        // 计算
        const result = evaluate(astNode);
        console.log(result);
      });
    </script>
  </body>
</html>

```

### 左递归
当一个文法规则推导出来的句型中包含以自身为最左符号的句型时，称之为左递归。
> add ::= add + mul | mul
由于产生式的第一个元素为自身，程序会无限的递归下去。

通过伪代码举例，该程序会无限递归
```javascript
function assert(bool) {
  return assert(bool) && bool
}
```

### 结合性
同样优先级的运算符是从左到右计算还是从右到左计算叫做结合性

当前additive函数的算法使用了右递归的写法
> add ::= mul | mul + add
从计算过程的打印中可以看出右递归导致了右结合

### 如何解决左递归
add有两个产生式
1. add ::= add + mul
2. add ::= mul
如果只用1来推导add，推导过程无法终结，形成add + mul + mul... + mul这样的序列
要终结推导过程就必然要使用到产生式2，且在最后一步使用，因此产生式可以改为
> add ::= mul add'
add'产生式必须满足 + mul + mul.. + mul的序列
> add' ::= + mul add'
又因为序列的长度有限，所以需要一个产生式ε(表示空集)来终结这个推导过程
> add' ::= + mul add' | ε
add'的模式用EBNF写法改写
> add' ::= ('+' mul)*
最终的add推导出的产生式
> add ::= mul ('+' mul)*

改写上述的加法表达式解析的算法后的代码：
```javascript
additive(tokens) {
  let child1 = this.multiplicative(tokens);
  let node = child1;
  if(child1) {
    while(true) {
      let token = tokens[0]
      if(token && (
        token.type === SimpleLexer.enum.PLUS ||
        token.type === SimpleLexer.enum.MINUS
      )) {
        token = tokens.shift()
        let child2 = this.multiplicative(tokens)
        node = new ASTNode(ASTNode.enum.Additive, token.text);
        node.addChild(child1);
        node.addChild(child2);
        child1 = node
      } else {
        break;
      }
    }
  }
  return node;
}
```

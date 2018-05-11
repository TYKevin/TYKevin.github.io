---
title: JavaScript 高级程序设计笔记（二）- JS 基本概念(1)
date: 2018-05-10 20:20:00
tags:
    - JavaScript
    - JavaScript 高级程序设计
---

在这一章，会跟大家阐述 ECMAScript 所定义的语法、数据类型。(ps. 流控制和函数会在下一章说明)。

## 语法
* ES 中的一切（变量名、函数名和操作符）都大小写敏感
* 标识符
    * 第一个字符：支持 字母、下划线(_) 和 $
    * 其他支持 字母、下划线(_)、$ 或 数字
    * 驼峰命名
    * 禁用关键字、保留字、true、false 和 null
* 严格模式（strict mode）
    * ES5 引入, 为 JS 定义一种不同的解析与执行模型
    * 对于不安全操作会抛出错误
    * 兼容性：支持 IE10+、Firefox 4+、Safari 5.1+、Opera 12+ 和 Chrome
```javascript
   /**
    * 在 JS 顶部添加；
    * 此行为编译指示，告知切换为严格模式；
    * 此为不破坏 ES3 语法而特意为之；
    */
   'use strict'
    // 函数顶部添加 为此函数切换为严格模式
    function doSomeThing () {
        'use strict'
        // 函数内容
    }
```

* 关键字和保留字
    * 关键字
        ```javascript
        break、do、instanceof、typeof
        case、if、else、new、var
        try、catch、finally、return、void
        continue、for、switch、while
        debugger、function
        this、with、default、throw、delete、in
        ```
    * 保留字
        ```javascript
        export、static、super、class、extends
        float、char、boolean、int、enum、short
        byte、log、final、native、synchronized
        package、throws、interface、abstract
        const、goto、private、transient
        debugger、implements、protected、volatile
        double、import、public、eval、arguments
        ```
        其实在我的理解上来看，看这个语言接下来会有怎么样的发展，在保留字里就可以得到一个大概的预测。

## 变量
```javascript
var message; // 定义变量，此时值为 undefined
var success = true; // 定义并初始化变量

function test () {
    var testVar = 'helloVar';
    testGlobalVar = 'hello'; // 全局变量，但是不推荐这么做，会造成可维护性极差
}
test();
console.log(testGlobalVar); // 输出 hello
testGlobalVar = 'helloJS'; // 在严格模式下会报 ReferenceError 错误
console.log(testVar); // 错误，因为 var testVar 定义为局部变量
```
> ps. 给未经声明的变量赋值，在严格模式下会抛出 ReferenceError 错误。

## 数据类型
ES 定义的7种数据类型：
> 6种基本数据类型：Undefined、Null、Boolean、Number、String、Symbol(ES6 中定义的)
> 1种复杂数据类型：Object（有一组无序的KV组成）

### typeof 操作符
typeof 操作符用于检测变量的数据类型，操作符的结果返回字符串。
> 'undefined' -> 这个值未定义
> 'boolean' -> 布尔值
> 'string' -> 字符串
> 'number' -> 数值
> 'object' -> 对象或 null
> 'function' -> 函数
> 'symbol' -> 符号类型

```javascript
var message = 'smt.'
console.log(typeof (message)) // 'string'
console.log(typeof message) // 'string', typeof 为操作符，所以括号可以省略
console.log(typeof 95) // 'number'

// hacker
console.log(typeof null) // 返回 'object'
```

### Undefined
Undefined 类型只有一个值，即 undefined。在声明变量但未对其初始化时，这个变量的值就是 undefined。
```javascript
var message
// var age
console.log(message) // 'undefined'
console.log(age) // 报错， 因为 age 并未声明

// 对于 类型为 undefined 的变量，只能进行 typeof 操作
console.log(typeof message) // undefined
console.log(typeof age) // 也返回 undefined，这是个神奇的地方，即使未声明，也返回undefined
```

### Null
Null 类型只有一个值，就是null，表示空指针对象，这也正式 typeof null 返回 'object' 的原因。如果定义的变量准备用来存放对象，最好初始化为null，这样可以判断此变量是否被赋值对象引用。

undefined 派生自 null 值，所以：
`console.log(null == undefined) // 输出为 true`

> 对于 undefined，没有必要在初始化时进行显式赋值
> 而对于 null，如果准备存储的类型为对象，则应该去赋值。

### Boolean
Boolean 有 true 和 false 两个值（区分大小写）。

```JavaScript
var message = 'hello world'
if (message) {
    console.log('Value is true')
}
```

上面这段代码的结构，相信大家平时也经常用到，这背后究竟发生了什么，会让 message 返回 true？

原来，Boolean 的值与每个数据类型通过`Boolean(var)` 都可以有等价的转换。
> Boolean ->  true false
> String -> true: 任何非空字符串; false: 空字符串;
> Number -> true: 任何非零的数字; false: 0 和 NaN;
> Object -> true: 任何对象; false: null;
> Undefined -> true: (没有对应的转换值); false: undefined;

所以，在上面那个例子中，其实message 自动转换为了 true
```javascript
var message = 'hello world'
var messageAsBoolean = Boolean(message) // 这一步 JS 帮我们搞定了
if (messageAsBoolean) {
  console.log('Value is true')
}
```

### Number
Number 类型表示整数和浮点数，其使用的为 IEEE754 格式。

#### 声明 十/八/十六 进制数值：不同的数值字面量格式
Number 类型同时支持通过不同的字面量意义 来声明十进制、八进制和十六进制。在运算时，都会转成十进制数值来进行算数计算。
```javascript
// 十进制
var intNumber = 55 // 整数，56

// 八进制：第一位必须是 0，然后是八进制数字序列（0～7）
var octalNum1 = 070 // 56
var octalNum2 = 079 // **无效的八进制**，这里虽然以0开头，但是超出了八进制范围
var octalNum3 = 08  // **无效的八进制**

// 十六进制：前两位必须是 0x，后跟任何十六进制数字
var hexNum1 = 0xA   // 10
var hexNum2 = 0x1f  // 31
```

#### 声明浮点数值
浮点数相必大家都比较熟悉了，这里将 ES 中几个注意点罗列出来。
##### 声明
```javascript
var floatNum1 = 1.1 // 普通声明 1.1
var floatNum2 = 0.1 // 普通声明 0.1
var floatNum3 = .1  // 0.1, 支持声明小数点前没有整数的形式（不推荐）
var floatNum4 = 3.125e7 // 31250000, 科学计数法声明
```

##### 浮点数精度
浮点数最高精度为17位小数，但是需要注意的是，算数计算时，精确度远远不如整数。

```javascript
0.1 + 0.2 == 0.3 // 返回 false
0.05 + 0.25 == 0.3 // 返回 true
```

0.1 + 0.2 的结果是 0.30000000000000004，而 0.05 + 0.25 却等于 0.3，这是因为采用了 IEEE754 数值格式，所以在浮点数计算时会产生舍入误差的问题。

#### 数值范围
取得 ES 中最小数：`Number.MIN_VALUE`
取得 ES 中最大数：`Number.MAX_VALUE`

如果超出这个阈值，则会将数值转换为 Infinity（正无穷） 或 -Infinity（负无穷）

判断 是否超出范围：
```javascript
// 以下语句，返回 false，因为已超出正常数值范围
isFinite(Number.MAX_VALUE + Number.MAX_VALUE)
```

#### NaN 数值
NaN，即非数值（Not a Number），表示一个本来要返回数值却为返回数值的情况（容错）。例如，`3 * 'abc'` 就会返回 NaN。

有两个特点：
* 任何涉及 NaN 的操作都会返回 NaN。
* NaN 与任何值 都不相等，包括 NaN 本身，`NaN == NaN` 返回 `false`。

所以，就有了isNaN() 函数, 来确定这个参数是否为 NaN：
```javascript
isNaN(NaN) // true
isNaN(10) // false，10 是一个数
isNaN('10') // false，可以被转换为 10
isNaN('abc') // true，不可以转换为数值
isNaN(true) // false，可以转换为数值 1
```
> `isNaN()` 同样也适用于 Object 类型，首先调用对象的 `valueOf()`，确定是否可以转换为数值，如果不能会再次调用 `toString()` 来判断返回值。

#### 数值转换
可以通过 `Number()`、`parseInt()`、`parseFloat()` 将非数值转换为数值。

##### `Number()` 转换规则
* Boolean 值 -> true：1；false：0；
* null -> 返回零
* undefined  -> 返回 NaN
* 字符串：
    * 只包含数字：返回十进制数，忽略前导零，例如 `Number('011')` 返回 11
    * 有效浮点数: 返回对应的浮点数
    * 有效十六进制：返回 相同大小的十进制整数值
    * 空字符串：返回 0
    * 其他的格式：返回 NaN
* 对象格式：首先调用对象的 `valueOf()`，确定是否可以转换为数值，如果不能会再次调用 `toString()` 来判断返回值。

我们来看几个例子：
```javascript
Number('Hello') // NaN
Number('') // 0
Number('0000011') // 11
Number(true) // 1
```

因为 `Number()` 转换规则十分复杂而且不太合理，所以出现了 `parseInt()`。

##### `parseInt()` 转换规则
* 忽略前置空格字符，找到第一个有效字符，不是数字或-，直接返回 NaN
* 如果是有效字符，则继续解析后续直至遇到非数字字符。
* 可识别字符串中 十进制、十六进制和八进制，八进制识别在 ES5 中被废弃。

依然，我们看几个例子：
```javascript
parseInt('188js') // 188
parseInt('') // NaN
parseInt('0xA') // 十六进制识别，10
parseInt(22.5) // 22
parseInt('070') // ES3 中返回 56，ES5 中返回 70
parseInt(070) // ES3 和 ES5 都返回56
parseInt('70') // 70
```

为了解决进制转换混乱的问题，可以添加第二个参数来指定转换的进制，例如
```javascript
parseInt('0xAF', 16) // 175
parseInt('AF', 16) // 175, 如果添加第二个参数，不带 0x 也是可以的
parseInt('070', 8) // 56，指定了转换进制，则会将 '070' 转换为 56
```
> 建议，在转换时都指定进制，以保证结果正确。

##### `parseFloat()` 转换规则
与 `parseInt()` 类似，不过有几点不同的地方：
* 字符串中只有 第一个小数点有效
* 始终忽略前导的零
* 十六进制数 永远返回 0
* 直解析十进制的有效数，所以不能指定转换进制
* 可以解析 科学计数法 表示的数

OK，为了加深印象，我们还是再来写几个例子：
```javascript
parseFloat('188js') // 188 (整数)
parseFloat('0xA') // 0
parseFloat('22.5') // 22.5
parseFloat('22.3.3') // 22.3
parseFloat('0908.5') // 908.5
parseFloat(3.125e7) // 31250000
```

### String
String 类型用来定义字符串。以 "" 或 '' 来表示。String 类型支持转义序列。

#### 转换为字符串
有两种方式：`toString()` 和 `String()`

##### `toString()`
返回相应值的字符串表现。Number、Boolean、Object、String 类型都有toString()方法。需要注意的是 null 和 undefined 是没有这个方法的。

###### 输出 二/八/十六进制
一般情况下, `toString()` 方法不必传递参数。但是对于Number对象我们可以通过传入 **输出数值的基数**，来更改返回值的进制，举个例子：
```javascript
var num = 10
num.toString() // '10'
num.toString(2) // '1010'
num.toString(8) // '12'
num.toString(10) // '10'

// 那如果传入的是字符，会是怎样？
var str = '10'
str.toString(2) // '10', 当然是没有效果咯
```

> 在不知道转换的值是否为 null 或 undefined 时，使用 `toString()` 会造成异常。所以，就有了 `String()`

##### `String()`
`String()` 可以将任何类型的值转换为字符串。规则如下：
* 如果值有 `toString()` 则会调用该方法并返回结果
* 如果值为 null，返回 'null'
* 如果值为 undefined，返回 'undefined'

还是来举几个例子：
```javascript
var value1 = 10
var value2 = true
var value3 = null
var value4

String(value1) // '10'
String(value2) // 'true'
String(value3) // 'null'
String(value4) // 'undefined'
```

### Object
ES 中的对象就是一组数据和功能的集合。和 Java 一样，Object 类型是所有他的实例的基础。

```javascript
// 定义 Object 类型
var obj  = new Object()
var obj1 = new Object //【不推荐】如果构造函数没有参数，则可以省略括号
```

Object 具有的属性与方法，同样他存在于所有的对象和实例中。
* Constructor：用于创建当前对象的函数。
* hasOwnProperty(proertyName): 用于检查给定的属性在当前的对象实例中是否存在。proertyName为 String 类型。
* isPrototypeOf(object): 用于检查传入的对象是否为另一个对象的原型。
* propertyIsEnumerable(proertyName): 检查给定的属性是否能够使用 for-in 语句来枚举。
* toLocaleString(): 返回对象的字符串表示，该字符串与执行环境的地区对应。
* toString(): 这个在上文有介绍哈，返回对象的字符串表示。
* valueOf(): 返回对象的String、Number 或 Boolean 表示。通常和 toString() 返回值相同。

### Symbol 类型
Symbol 类型是ES6 引入的一种新的原始数据类型。通过 `Symbol()` 声明的类型表示一种独一无二的值。这里我简单介绍一下Symbol类型，预先介绍一下，下一个系列 会计划 去介绍 ES6 中的一些特性，那时会专门开一篇文章，去详细介绍 symbol 类型。

类型定义：
```javascript
let symbolType = Symbol()
typeof symbolType // symbol
```

因为 Symbol 类型，表示独一无二的值，所以，symbol 的变量互不相等。这里先简要介绍一下哈，更多的解析 会在下一个系列放出～～～

-----
关于语法和变量 就为大家介绍到这里，大家也可以扫码关注我的微信公众号，可以及时收到推送，让你更及时的看到新文章，让我们一起成为更好的开发者～
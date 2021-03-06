# JavaScript核心原理解析

## delete 0：JavaScript中到底有什么是可以销毁的

**习惯中的“引用”**

早期的 JavaScript 在推广时，采用传统的数据类型分类即值类型和引用类型，并且，所谓值类型中的字符串是按照引用来赋值和传递引用（而不是传递值）的。

但是什么是引用类型呢？在这件事上，JavaScript 偷了个懒，它强行定义了“Object 和 Function 就是引用类型”。这样一来，引用类型和值类型就给开发人员讲清楚了，对象和函数呢，也就可以理解了：它们按引用来传递和使用。

绝大多数情况下，这样解释是可行的。但是到了 delete 运算这里，就不行。因为这样一来，`delete 0` 就是删除一个值，而 `delete x` 就既可能是删除一个值，也可能是删除一个引用。然而，当时 JavaScript 又同时约定：那些在 global 对象上声明的属性，就“等同于”全局变量。于是，这就带来了第三个问题：`delete x` 还可能是删除一个 global 对象上的属性。而它在执行这个操作的时候，看起来却像是一个全局变量（的名字）。

**到底在删除什么？**

```js
delete obj.x
```

`obj.x`既不是之前说过的引用类型，也不是之前说过的值类型，它与`typeof(x)`识别的所有类型都无关。因为，它是一个表达式。所以，`delete` 这个操作的正式语法设计并不是“删除某个东西”，而是“删除一个表达式的结果”：

**表达式的结果是什么？**

在 JavaScript 中，有两个东西可以被执行并存在执行结果值（Result），包括语句和表达式。比如你用`eval()`来执行一个字符串，那么事实上，你执行的是一个语句，并返回了语句的值；而如果你使用一对括号来强制一个表达式执行，那么这个括号运算得到的，就是这个表达式的值。表达式的值，在 ECMAScript 的规范中，称为“引用”。

```js
delete 0
```

事实上是在说：JavaScript 将 0 视为一个表达式，并尝试删除它的求值结果。

所以，现在这里的 0，其实不是值（Value）类型的数据，而是一个表达式运算的结果值（Result）。而在进一步的删除操作之前，JavaScript 需要检测这个 Result 的类型：

- 如果它是值，则按照传统的 JavaScript 的约定返回 true；
- 如果它是一个引用，那么对该引用进行分析，以决定如何操作。

这个检测过程说明，ECMAScript 约定：任何表达式计算的结果（Result）要么是一个值，要么是一个引用。并且需要留意的是，在这个描述中，所谓对象，其实也是值。准确地说，是“非引用类型”。

**总结**

- delete 运算符尝试删除值数据时，会返回 true，用于表示没有错误（Error）。
- delete 0 的本质是删除一个表达式的值（Result）。
- delete x 与上述的区别只在于 Result 是一个引用（Reference）。
- delete 其实只能删除一种引用，即对象的成员（Property）。

所以，只有在`delete x`等值于`delete obj.x`时 delete 才会有执行意义。例如`with (obj) ...`语句中的 `delete x`，以及全局属性 `global.x`。

## var x = y = 100：声明语句与语法改变了JavaScript语言核心性质

**声明**

严格意义上讲，JavaScript 只有变量和常量两种标识符，六条声明语句中：

- `let x` 声明变量 x。不可在赋值之前读。
- `const x` 声明常量 x。不可写。
- `var x` 声明变量 x。在赋值之前可读取到 undefined 值。
- `function x` 声明变量 x。该变量指向一个函数。
- `class x` 声明变量 x。该变量指向一个类（该类的作用域内部是处理严格模式的）。
- `import …` 导入标识符并作为常量（可以有多种声明标识符的模式和方法）。

除了这六个语句之外，还有两个语句有潜在的声明标识符的能力，不过它们并不是严格意义上的声明语句（声明只是它们的语法效果）。这两个语句是指：

- `for (var|let|constx…) …` for 语句有多种语法来声明一个或多个标识符，用作循环变量。
- `try … catch (x) …` catch 子句可以声明一个或多个标识符，用作异常对象变量。

总的来说，除上述的语法，用户是没有其它方式来在当前的代码上下文中“声明”出一个标识符来的。所有的“声明”：

> - 都意味着 JavaScript 将可以通过“静态”语法分析发现那些声明的标识符；
> - 标识符对应的变量 / 常量“一定”会在用户代码执行前就已经被创建在作用域中。

**发生了什么？**

回到讨论的这行代码`var x = y = 100`，在这行代码中，等号的右边是一个表达式`y = 100`，它发生了一次“向不存在的变量赋值”，所以它隐式地声明了一个全局变量y，并赋值为 100。
而一个赋值表达式操作的本身也是有“结果（Result）”的，它是右操作数的值。注意，这里是“值”而非“引用”，例如下面的测试中的a将是一个函数，而不是带着“this 对象”信息的方法：

```js
// 调用 obj.f() 时将检测 this 是不是原始的 obj
> obj = { f: function() { return this === obj } };
// false，表明赋值表达式的“结果 (result)”只是右侧操作数的值，即函数 f
> (a = obj.f)();
false
```

我们讲述了整个语句的过程，也就是说，由于“y = 100”的结果值是 100，所以该值将作为初始值赋值“变量 x”。并且，从语义上来说，这是变量“x”的初始绑定。

之所以强调这一点，是因为相同的分析过程也可以用在 const 声明上，而 const 声明是只有一次绑定的，常量的初始绑定也是通过“执行赋值过程”来实现的。

**总结**

- var 等声明语句总是在变量作用域（变量表）或词法作用域中静态地声明一个或多个标识符。
- 全局变量的管理方式决定了“向一个不存在的变量赋值”所导致的变量泄露是不可避免的。
- 动态添加的“var 声明”是可以删除的，这是唯一能操作 varNames 列表的方式（不过它并不存在多少实用意义）。
- 变量声明在引擎的处理上被分成两个部分：一部分是静态的、基于标识符的词法分析和管理，它总是在相应上下文的环境构建时作为名字创建的；另一部分是表达式执行过程，是对上述名字的赋值，这个过程也称为绑定。
- 标题里的这行代码中，x 和 y 是两个不同的东西，前者是声明的名字，后者是一个赋值过程可能创建的变量名。

## a.x = a = {n:2}：一道被无数人无数次地解释过的经典面试题

```js
var a = {n:1}
a.x = a = {n:2}
```

上面发生了两次赋值，第一次赋值发生于“a = {n: 2}”，它覆盖了“原始的变量a；第二次赋值发生于被”a.x”引用暂存的“原始的变量a”。

这里给出一段简单的代码，来复现这个现场，以便看清这个结果。例如：

```js
// 声明“原始的变量 a”
var a = {n:1};
// 使它的属性表冻结（不能再添加属性）
Object.freeze(a);
try {
  // 本节的示例代码
  a.x = a = {n:2};
}
catch (x) {
  // 异常发生，说明第二次赋值“a.x = ...”中操作的`a`正是原始的变量 a
  console.log('第二次赋值导致异常.');
}
// 第一次赋值是成功的
console.log(a.n); //
```

第二次赋值操作中，将尝试向“原始的变量a”添加一个属性“a.x“，且如果它没有冻结的话，属性“a.x”会指向第一次赋值的结果。

- 有一个新的a产生，它覆盖了原始的变量a，它的值是{n:2}；
- 最左侧的“a.x”的计算结果中的“原始的变量a”在引用传递的过程中丢失了，且“a.x”被同时丢弃。

所以，第二次赋值操作“a.x = …”实际是无意义的。因为它所操作的对象，也就是“原始的变量a”被废弃了。但是，如果有其它的东西，如变量、属性或者闭包等，持有了这个“原始的变量a”，那么上面的代码的影响仍然是可见的。

> 事实上，由于 JavaScript 中支持属性读写器，因此向“a.x”置值的行为总是可能存在“某种执行效果”，而与“a”对象是否被覆盖或丢弃无关。

```js
var a = {n:1}, ref = a;
a.x = a = {n:2};
console.log(a.x); // --> undefined
console.log(ref.x); // {n:2}
```

阅读 JQuery 代码的过程中发现了这一使用模式：

```js
elemData = {}
...
elemData.events = elemData = function(){};
elemData.events  = {};
```

意味着给旧的变量添加一个指向新变量的属性。因此，一个链表是可以像下面这样来创建的：

```js
var i = 10, root = {index: "NONE"}, node = root;
// 创建链表
while (i > 0) {
  node.next = node = new Object;
  node.index = i--;  // 这里可以开始给新 node 添加成员
}
// 测试
node = root;
while (node = node.next) {
  console.log(node.index);
}
```

## export default function() {}：你无法导出一个匿名函数表达式

- export ...语句通常是按它的词法声明来创建的标识符的，例如export var x = ...就意味着在当前模块环境中创建的是一个变量，并可以修改等等。但是当它被导入时，在import语句所在的模块中却是一个常量，因此总是不可写的。
- 由于export default ...没有显式地约定名字“default（或default）”应该按let/const/var的哪一种来创建，因此 JavaScript 缺省将它创建成一个普通的变量（var），但即使是在当前模块环境中，它事实上也是不可写的，因为你无法访问一个命名为“default”的变量——它不是一个合法的标识符。
- 所谓匿名函数，仅仅是当它直接作为操作数（而不是具有上述“匿名函数定义”的语法结构）时，才是真正匿名的，例如：

  ```js
  console.log((function(){}).name));  // ""
  ```

- 由于类表达式（包括匿名类表达式）在本质上就是函数，因此它作为 default 导出时的性质与上面所讨论的是一致的。
- 导出项（的名字）总是作为词法声明被声明在当前模块作用域中的，这意味着它不可删除，且不可重复导出。亦即是说即使是用var x...来声明，这个x也是在 _lexicalNames_ 中，而不是在 _varNames_ 中。
- 所谓“某个名字表”，对于 export 来说是模块的导出表，对于 import 来说就是名字空间（名字空间是用户代码可以操作的组件，它映射自内部的模块导入名字表）。不过，如果用户代码不使用“import * as …”的语法来创建这个名字空间，那么该名字表就只存在于 JavaScript 的词法分析过程中，而不会（或并不必要）创建它在运行期的实例。这也是我一直用“某个名字表”来称呼它的原因，它并不总是以实体形式存在的。
- 没有模块会导出（传统意义上的）main()，因为 ECMAScript 为了维护模块的静态语义，而把执行过程及其入口的定义丢回给了引擎或宿主本身。

**Q&A**

- `export default function(){}`。这个语法本身没有任何的问题。但是他看似导出一个匿名函数表达式。其实他真正导出的是一个具有名字的函数，名字的default。
- ESModule 根据 import 构建依赖树，所以在代码运行前名字就是已经存在于上下文，然后在运行模块最顶层代码，给名字绑定值，就出现了‘变量提升’的效果。

## for (let x of [1,2,3]) ...：for循环并不比使用函数递归节省开销

绝大多数 JavaScript 语句都并没有自己的块级作用域。从语言设计的原则上来看，越少作用域的执行环境调度效率也就越高，执行时的性能也就越好。

基于这个原则，switch语句被设计为有且仅有一个作用域，无论它有多少个 case 语句，其实都是运行在一个块级作用域环境中的。例如：

```js
var x = 100, c = 'a';
switch (c) {
  case 'a':
    console.log(x); // ReferenceError
    break;
  case 'b':
    let x = 200;
    break;
}
```

在这个例子中，switch 语句内是无法访问到外部变量x的，即便声明变量x的分支case 'b'永远都执行不到。这是因为所有分支都处在同一个块级作用域中，所以任意分支的声明都会给该作用域添加这个标识符，从而覆盖了全局的变量x。

一些简单的、显而易见的块级作用域包括：

```js
// 例 1
try {
  // 作用域 1
}
catch (e) { // 表达式 e 位于作用域 2
  // 作用域 2
}
finally {
  // 作用域 3
}
// 例 2
//（注：没有使用大括号）
with (x) /* 作用域 1 */; // <- 这里存在一个块级作用域
// 例 3, 块语句
{
  // 作用域 1
  // 除了这三个语句和“一个特例”之外，所有其它的语句都是没有块级作用域的。例如`if`条件语句的几种常见书写形式：
if (x) {
  ...
}
// or
if (x) {
  ...
} else {
  ...
}
```

这些语法中的“块级作用域”都是一对大括号表示的“块语句”自带的，与上面的“例 3”是一样的，而与if语句本身无关。

**循环语句中的块**

并不是所有的循环语句都有自己的块级作用域，例如 `while` 和 `do…while` 语句就没有。而且，也不是所有 for 语句都有块级作用域。在 JavaScript 中，有且仅有：

```js
for (<let/const>…) …
```

这个语法有自己的块级作用域。当然，这也包括相同设计的`for await`和`for .. of/in ..`。例如：

```js
for await (<let/const>x of …) …
for (<let/const>x … in …) …
for (<let/const>x … of …) …
```

语句`for (<const/let> x ...) ...`语法中的标识符x是一个词法名字，应该由for语句为它创建一个（块级的）词法作用域来管理之。

**第二个作用域**

```js
var x = 100;
for (let x = 102; x < 105; x++)
  console.log('value:', x);  // 显示“value: 102~104”
console.log('outer:', x); // 显示“outer: 100”
```

因为for语句的这个块级作用域的存在，导致循环体内访问了一个局部的x值（循环变量），而外部的（outer）变量x是不受影响的。

那么在循环体内是否需要一个新的块级作用域呢？这取决于在语言设计上是否支持如下代码：

```js
for (let x = 102; x < 105; x++)
  let x = 200;
```

也就是说，如果循环体（单个语句）允许支持新的变量声明，那么为了避免它影响到循环变量，就必须为它再提供另一个块级作用域。很有趣的是，在这里，JavaScript 是不允许声明新的变量的。上述的示例会抛出一个异常，提示“单语句不支持词法声明”：

```bash
SyntaxError: Lexical declaration cannot appear in a single-statement context
```

**for 循环的代价**

```js
for (let i in x)
  setTimeout(()=>console.log(i), 1000);
```

这个例子创建了一些定时器。当定时器被触发时，函数会通过它的闭包（这些闭包处于 loopEnv 的子级环境中）来回溯，并试图再次找到那个标识符i。然而，当定时器触发时，整个 for 迭代有可能都已经结束了。这种情况下，要么上面的 forEnv 已经没有了、被销毁了，要么它即使存在，那个i的值也已经变成了最后一次迭代的终值。

所以，要想使上面的代码符合预期，这个 loopEnv 就必须是“随每次迭代变化的”。也就是说，需要为每次迭代都创建一个新的作用域副本，这称为迭代环境（iterationEnv)。因此，每次迭代在实际上都并不是运行在 loopEnv 中，而是运行在该次迭代自有的 iterationEnv 中。

也就是说，在语法上这里只需要两个“块级作用域”，而实际运行时却需要为其中的第二个块级作用域创建无数个副本。

这就是 for 语句中使用“let/const”这种块级作用域声明所需要付出的代价。

**总结**

- 当在这样的 for 循环中添加块语句时，块语句在每个迭代中都会都会创建一次它自己的**块级作用域副本**。这个循环体越大，支持的层次越多，那么这个环境的创建也就越频繁，代价越高昂。
- 无论用户代码是否直接引用 loopEnv 中的循环变量，这个过程都是会发生的。这是因为 JavaScript 允许动态的 eval()，所以引擎并不能依据代码文本静态地分析出循环体（ForBody）中是否引用哪些循环变量。
- **“循环与函数递归在语义上等价”**。所以在事实上，上述这种 for 循环并不比使用函数递归节省开销。在函数调用中，这里的循环变量通常都是通过函数参数传递来处理的。因而，那些支持“let/const”的 for 语句，本质上也就与“在函数参数界面中传递循环控制变量的递归过程”完全等价，并且在开销上也是完全一样的。
- 因为每一次函数调用其实都会创建一个新的闭包——也就是函数的作用域的一个副本。

**Q&A**

- for只要写大括号就代表着块级作用域。所以只要写大括号，不管用let 还是 var，一定是会创建相应循环数量的块级作用域的。
- 如果不用大括号，在for中使用了let，也会创建相应循环数量的块级作用域。也就是说，可以提高性能的唯一情况只有（符合业务逻辑的情况下），循环体是单行语句就不使用大括号且for中使用var。
- 因为单语句没有块级作用域，而词法声明是不可覆盖的，单语句后面的词法声明会存在潜在的冲突。

## x: break x; 搞懂如何在循环外使用break，方知语句执行真解

1. “GOTO 语句是有害的”。1972 年图灵奖得主——迪杰斯特拉（Edsger Wybe Dijkstra, 1968）
2. 很多新的语句或语法被设计出来用来替代 GOTO 的效果的，但考虑到 GOTO 的失败以及无与伦比的破坏性，这些新语法都被设计为功能受限的了。
3. 任何的一种 GOTO 带来的都是对“顺序执行”过程的中断以及现场的破坏，所以也都存在相应的执行现场回收的机制。
4. 有两种中断语句，它们的语义和应用场景都不相同。
5. 语句有返回值。
6. 在顺序执行时，当语句返回 Empty 的时候，不会改写既有的其他语句的返回值。
7. 标题中的代码，是一个“最小化的 break 语句示例”。

## `${1}`：详解JavaScript中特殊的可执行结构

模板字面量本身是一个特殊的可执行结构，但是它调动了包括引用、求值、标识符绑定、内部可执行结构存储，以及执行函数调用在内的全部能力。这是 JavaScript 厘清了所有基础的可执行结构之后，才在语法层面将它们融汇如一的结果。

**总结**

1. 标题中的代码称为模板字面量，是一种可执行结构。JavaScript 中有许多类似的可执行结构，它们通常要用固定的逻辑、在确定的场景下交付 JavaScript 的一些核心语法的能力。
2. 与参数表和赋值模板有相似的地方，模板字面量也是将它的形式规格（Formal）作为可执行结构来保存的。
3. 只是参数表与赋值模板关注的是名字，因此存储的是“名字（lhs）”与“名字的值（rhs）的取值方法”之间的关系，执行的结果是 argArray 或在当前作用域中绑定的名字等。
4. 而模板字面量关注的是值，它存储的是“结果”与“结果的计算过程”之间的关系。由于模板字面量的执行结果是一个字符串，所以当它作为值来读取时，就会激活它的运算求值过程，并返回一个字符串值。
5. 模板字面量与所有其它字面量（能作为引用）相似，它也可以作为引用。
   1. ```js
      1=1
      ```
   2. “1=1”包括了“1”作为引用和值（lhs 和 rhs）的两种形式，在语法上是成立的。
   3. ```
       foo`${1}`
       ```
   4. 所以上面这行代码在语法上也是成立的。因为在这个表达式中，${1}使用的不是模板字面量的值，而是它的一个“（类似于引用的）结构”。
6. “模板字面量调用（TemplateLiteral Call）”是唯一一个会使用模板字面量的引用形态（并且也没有直接引用它的内部结构）的操作。这种引用形态的模板字面量也被称为“标签模板（Tagged Templates）”，主要包括模板的位置和那些可计算的标签的信息。例如：
   1. ```
      > var x = 1;
      > foo = (...args) => console.log(...args);
      > foo`${x}`
      [ '', '' ] 1
      ```
   2. 模板字面量的内部结构中，主要包括将模板多段截开的一个数组，原始的模板文本（raw）等等。在引擎处理模板时，只会将该模板解析一次，并将这些信息作为一个可执行结构缓存起来（以避免多次解析降低性能），此后将只使用该缓存的一个引用。当它作为字面量被取值时，JavaScript 会在当前上下文中计算各个分段中的表达式，并将表达式的结果值填回到模板从而拼接成一个结果值，最后返回给用户。

## x => x：函数式语言的核心抽象：函数与表达式的同一性

1. 传入参数的过程执行于函数之外，例如`f(a=100)；`绑定参数的过程执行于函数（的闭包）之内，例如`function foo(x=100) ..`。
2. `x=>x`在函数界面的两端都是值操作，也就是说 input/output 的都是数据的值，而不是引用。
3. 参数有两种初始化方法，它们根本的区别在于绑定初值的方式不同。
4. 闭包是函数在运行期的一个实例。

## (...x)：不是表达式、语句、函数，但它却能执行

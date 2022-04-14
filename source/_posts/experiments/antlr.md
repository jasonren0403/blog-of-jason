---
title: ANTLR语言识别工具 
date: 2019-05-20 17:11:01 
tags:
  - "ANTLR"
  - "编译原理"
categories:
  - ["实验","编译原理"]

---

## 实验目的

1. 了解ANTLR语言识别工具，安装ANTLR环境，熟悉ANTLR的使用。
2. 掌握使用ANTLR构建的语法分析器和树解析器分析语言，通过ANTLR实现语法分析过程。

<!-- more -->

## 实验条件

1. Windows 10 专业版
2. JDK 1.8版本（64位，版本号为1.8.0_202）
3. ANTLR环境——ANTLR 4

## 实验内容

### 安装ANTLR

个人所用的集成开发环境中提供ANTLR插件的安装，故可直接进入插件下载功能中下载该插件。运行时环境配置过程和安装过程不再赘述。 {% asset_img 1.png %} 安装以后，下方工具栏会多出两个窗口：ANTLR
Preview和Tool Output。 {% grouppicture 2-2 %} {% asset_img 2.png "ANTLR Preivew窗口" %} {% asset_img 3.png "Tool Output窗口"
%} {% endgrouppicture %} Preview中可以显示Parse tree，从而省去了在控制台中输入`-gui`命令的麻烦。

### 使用ANTLR实现实数的四则运算

{% note info %}

#### 要求

创建ANTLR语法文件`expr.g4`，解析如赋值语句`a = 5.5` 和表达式`a+4*3÷2-1` 这样的语句。

- 要验证所编写的文法是否正确，可使用TestRig测试语法，在CMD中输入<code>-gui</code>可直观展示其语法分析树。
- 创建expr文件（如test.expr，输入一个表达式）检验程序是否能够正确运行和计算四则算式。 {% endnote %}

1. 文法设计

   {% codeblock %} E -> E OP E|VAR = E|E'; E'-> D; OP: [+|-|*|/]; D:[0-9]+; VAR:[a-zA-Z]+; {% endcodeblock %}

   由此编写的`expr.g4`文件如下：（还考虑了一些换行及空格因素）

   {% codeblock expr.g4 lang:g4 %} grammar expr; exprs : setExpr | calcExpr ; setExpr : id=ID '=' num=NUMBER ; calcExpr:
   calcExpr op=(MUL | DIV) calcExpr | calcExpr op=(ADD | SUB) calcExpr | factor ; factor: (sign=(ADD | SUB))? num=NUMBER
   // 计算因子可以是一个正数或负数 | id=ID // 计算因子可以是一个变量 ; WS : [ \t\n\r]+ -> skip ; ID : [a-zA-Z]+ ; NUMBER : [0-9]+('.'([0-9]+)?)?
   ; ADD : '+' ; SUB : '-' ; MUL : '*' ; DIV : '/' ; {% endcodeblock %}

2. 运行结果

{% tabs results, 1 %}
<!-- tab 赋值语句 -->
控制台工具显示如下 {% asset_img res1-1.png %}

Main函数如下 {% codeblock lang:java mark:2 %} public static void main(String[] args){ String input_string = "a=5.5";
InputStream is = System.in; try{ ANTLRInputStream input = new ANTLRInputStream(input_string); exprLexer lexer = new
exprLexer(input); CommonTokenStream tokens = new CommonTokenStream(lexer); exprParser parser = new exprParser(tokens);
ParseTree tree = parser.exprs(); Visitor visitor = new Visitor(); System.out.println(visitor.visit(tree)); } catch(
IOException ioe){ ioe.printStackTrace(); } } {% endcodeblock %}

运行结果如下 {% asset_img res1-2.png "在visitSetExpr时触发了其中的打印操作，外层返回值为null" %}
<!-- endtab -->
<!-- tab 不完整的式子 -->
{% note %} 本次测试的式子为`a+4*3/2-1`
{% endnote %}

控制台工具显示如下 {% asset_img res2-1.png %}

Main函数如下 {% codeblock lang:java mark:2 %} public static void main(String[] args){ String input_string = "a+4*3/2-1";
InputStream is = System.in; try{ ANTLRInputStream input = new ANTLRInputStream(input_string); exprLexer lexer = new
exprLexer(input); CommonTokenStream tokens = new CommonTokenStream(lexer); exprParser parser = new exprParser(tokens);
ParseTree tree = parser.exprs(); Visitor visitor = new Visitor(); System.out.println(visitor.visit(tree)); } catch(
IOException ioe){ ioe.printStackTrace(); } } {% endcodeblock %}

{% asset_img res2-2.png "为了方便，在每个函数的visit函数外层加了一个控制台打印输出" %} 从输出来看，对语法树的遍历过程几乎遍及了句子的每一个结点。由于最后得到的句型是一个类似于<code>a+<
数字></code>的格式，所以它输出<code>NaN</code>，与预期一致。

{% note warning %}
`a`是与前一个赋值语句分开输入的，所以它并没有被定义值 {% endnote %}

<!-- endtab -->
<!-- tab 一个正常的运算式 -->
{% note %} 本次测试的式子为`1+4.5*2-3/3`
{% endnote %}

Main函数代码类似，控制台工具显示如下 {% asset_img res3-1.png "分析树的结构与预期一致" %}

代码输出结果如下 {% asset_img res3-2.png %} 得到了正确的输出结果，证明我们写的语法解析是正确的。

<!-- endtab -->
<!-- tab 异常处理 -->

1. 针对除0错误，程序可以顺利显示出`Infinity`，没有中断程序 {% asset_img res4-1.png %}
2. 输入过长也不会中断程序 {% asset_img res4-2.png %}
3. 符号错误需要单独处理 {% asset_img res4-3.png "报错：不被识别的字符（本例输入了分号），需要一个ID，NUMBER，加号或减号" %}

{% note warning %} 这说明符号错误需要单独处理，java中可以使用try-catch来完成这项工作。除此之外，也可从前端角度考虑，提前检验用户输入。 {% endnote %}
<!-- endtab -->
{% endtabs %}

## 总结

这个解析过程是怎么实现的呢？

* 在生成ANTLR识别文件时，实际上生成了很多个java和token源文件。 {% asset_img end1.png %}

{% asset_img end2.png %}

* 有两个tokens文件和六个java文件，`Listener`和`Visitor`是两个接口，`Listener`中定义了一些退出或进入一个表达式中所进行的语法动作，`Visitor`
  中定义了遍历语法树中项目的返回动作，一个语法项对应一个`visit`函数。具体由`xxBaseListener`或`xxBaseVisitor`
  来实现。默认状态下，所有的实现都是空的，我们可以扩展`xxBase[Listener|Visitor]`类，自己定义语法动作。本实验中，只需要实现`Visitor`的内容即可，不过由于上下文的关系，我们需要自建一个`Context`
  类，负责上下文内容的保存和识别。

* 重写`visit`函数：

```java
public <你需要的返回值> visit<A>(xxParser.<A>Context ctx)
```

* .tokens文件中将语法常量对应了一个整数数值，方便常量的建立。.interp文件则记录了字面值。
* Parser中，对每一个语法变量，生成了很多上下文相关的方法（xxContext）。其中的某些常量在前面所述的.tokens文件中可以找到。
* Lexer中，则存储了字面量及其相关方法，某些内容可在.interp文件中找到。
* 而所有开头的xx来自于语法文件第一行的grammar xx，即为语法名。
* `Visitor`实现完成后，在主函数中建立`visitor`对象，调用`visit`语法树方法，将用户输入作为参数传入，其返回值即为语法分析结果。

{% codeblock visitor.java lang:java %} public class Visitor extends exprBaseVisitor<Double> { @Override public Double
visitExprs(exprParser.ExprsContext ctx) { //System.out.println("visit exprs"); return visit(ctx.getChild(0)); }
@Override public Double visitSetExpr(exprParser.SetExprContext ctx) { Context.getInstance().setContext(ctx.id.getText(),
ctx.num.getText()); System.out.println("Assign operation in current line:"+ctx.id.getText()+" = "+ctx.num.getText());
return null; } @Override public Double visitCalcExpr(exprParser.CalcExprContext ctx) { //System.out.println("visit calc
expr"); int cc = ctx.getChildCount(); if (cc == 3) { switch (ctx.op.getType()) { case exprParser.ADD:
return visit(ctx.calcExpr(0)) + visit(ctx.calcExpr(1)); case exprParser.SUB:
return visit(ctx.calcExpr(0)) - visit(ctx.calcExpr(1)); case exprParser.MUL:
return visit(ctx.calcExpr(0)) * visit(ctx.calcExpr(1)); case exprParser.DIV:
return visit(ctx.calcExpr(0)) / visit(ctx.calcExpr(1)); } } else if (cc == 1) { return visit(ctx.getChild(0)); } throw
new RuntimeException(); } @Override public Double visitFactor(exprParser.FactorContext ctx) { //System.out.println("
visit factor"); int cc = ctx.getChildCount(); if (cc == 3) { return visit(ctx.getChild(1)); } else if (cc == 2) { if (
ctx.sign.getType() == exprParser.ADD)
return Double.valueOf(ctx.getChild(1).getText()); if (ctx.sign.getType() == exprParser.SUB)
return -1 * Double.valueOf(ctx.getChild(1).getText()); } else if (cc == 1) { if (ctx.num != null)
return Double.valueOf(ctx.getChild(0).getText()); if (ctx.id != null)
return Context.getInstance().getValue(ctx.id.getText()); } throw new RuntimeException(); } } {% endcodeblock %}

* 目前实现的版本中，可以正确地进行运算，但还没有实现连续赋值功能，也没有考虑到多语句的分隔及分开解析，例如输入`"a=1 a+1"`后不能得到`2`，可能需要修改`Context`类和`Visitor`
  类中的代码，使其可以记录上下文信息，将`a`的值存入暂存区。

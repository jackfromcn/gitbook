# Flowchart 基本语法

[Flowchart](http://flowchart.js.org/) 是一种通过文本方式描述的流程图，通过它你仅需几行代码即可在文档中绘制流程图。



## 语法简介

Flowchart 的语法来源于[ js-sequence-diagrams](https://bramp.github.io/js-sequence-diagrams/) ，定义流程图分为下边两步。



### 定义流程图元素

```
name=>type: content:>url
```



**name:** 元素的名称

**type：类型，包括如下类别**：

- start
- end
- operation
- subroutine
- condition
- inputoutput
- parallel

**content：**文本内容

**url：**连接



**示例：**

```
st=>start: Start:>http://www.google.com[blank]
e=>end:>http://www.google.com
op1=>operation: My Operation
sub1=>subroutine: My Subroutine
cond=>condition: Yes or No?:>http://www.google.com
io=>inputoutput: catch something...
para=>parallel: parallel tasks
```



### 连接元素

使用 `->` 符号即可：



```
cond(yes)->io->e
cond(no)->op1->e
```



## 操作入口

插入卡片 -> 选择文本绘图  -> Flowchart



![img](./1555402834257-9fa0ddac-dabb-469f-a4fd-55a1c58abcdd.png)



## 示例

### 示例 1

**效果：**

![img](./a424a89a007b92ef219250a04765a1da.svg)



**代码：**

```
st=>start: Start:>http://www.google.com[blank]
e=>end:>http://www.google.com
op1=>operation: My Operation
sub1=>subroutine: My Subroutine
cond=>condition: Yes
or No?:>http://www.google.com
io=>inputoutput: catch something...
para=>parallel: parallel tasks

st->op1->cond
cond(yes)->io->e
cond(no)->para
para(path1, bottom)->sub1(right)->op1
para(path2, top)->op1
```



### 示例 2

**效果：**

![img](./db6eeec143e95d44de3574b38fa8a366.svg)



**代码：**

```
st=>start: Start|past:>http://www.google.com[blank]
e=>end: End|future:>http://www.google.com
op1=>operation: My Operation|past
op2=>operation: Stuff|current
sub1=>subroutine: My Subroutine|invalid
cond=>condition: Yes
or No?|approved:>http://www.google.com
c2=>condition: Good idea|rejected
io=>inputoutput: catch something...|future

st->op1(right)->cond
cond(yes, right)->c2
cond(no)->sub1(left)->op1
c2(yes)->io->e
c2(no)->op2->e
```



### 示例 3

**效果：**

![img](./8cc91ef5701876de59ea23c31888c1d2.svg)



**代码：**

```
st=>start: Improve your
l10n process!
e=>end: Continue to have fun!:>https://youtu.be/YQryHo1iHb8[blank]
op1=>operation: Go to locize.com:>https://locize.com[blank]
sub1=>subroutine: Read the awesomeness
cond(align-next=no)=>condition: Interested to
getting started?
io=>inputoutput: Register:>https://www.locize.io/register[blank]
sub2=>subroutine: Read about improving
your localization workflow
or another source:>https://medium.com/@adrai/8-signs-you-should-improve-your-localization-process-3dc075d53998[blank]
op2=>operation: Login:>https://www.locize.io/login[blank]
cond2=>condition: valid password?
cond3=>condition: reset password?
op3=>operation: send email
sub3=>subroutine: Create a demo project
sub4=>subroutine: Start your real project
io2=>inputoutput: Subscribe

st->op1->sub1->cond
cond(yes)->io->op2->cond2
cond2(no)->cond3
cond3(no,bottom)->op2
cond3(yes)->op3
op3(right)->op2
cond2(yes)->sub3
sub3->sub4->io2->e
cond(no)->sub2(right)->op1

st@>op1({"stroke":"Red"})@>sub1({"stroke":"Red"})@>cond({"stroke":"Red"})@>io({"stroke":"Red"})@>op2({"stroke":"Red"})@>cond2({"stroke":"Red"})@>sub3({"stroke":"Red"})@>sub4({"stroke":"Red"})@>io2({"stroke":"Red"})@>e({"stroke":"Red","stroke-width":6,"arrow-end":"classic-wide-long"})
```



## 参考资料

<https://flowchart.js.org/>

<https://bramp.github.io/js-sequence-diagrams/>

<https://blog.csdn.net/ethmery/article/details/50670294>
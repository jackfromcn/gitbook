# SpEL 表达式

>  前言：Spring表达式语言(简称SpEL)是一种与JSP2的EL功能类似的表达式语言，它可以在运行时查询和操作对象图。与JSP2的EL相比，SpEL功能更加强大，它甚至支持方法调用和基本字符串模板函数。SpEL可以独立于Spring容器使用——只是当成简单的表达式语言来使用；也可以在Annotation或XML配置中使用SpEL，这样可以充分利用SpEL简化Spring的Bean配置。

## 一、使用Expression接口进行表达式求值

  Spring的SpEL可以单独使用，可以使用SpEL对表达式计算、求值。SpEL主要提供了如下三个接口(位于org.springframework.expression包)：

**ExpressionParser**：该接口的实例负责解析一个SpEL表达式，返回一个Expression对象

**Expression**：该接口的实例代表一个表达式

**EvaluationContext**：代表计算表达式值的上下文，当SpEL表达式中含有变量时，程序将需要使用该API来计算表达式的值
  Expression实例代表一个表达式，它包含了如下方法用于计算，得到表达式的值：



### `getValue()` 方法重载

`Object getValue()`：计算表达式的值
`<T> T getValue(Class<T> desiredResultType)`：计算表达式的值，而且尝试将该表达式的值当成desiredResultType类型处理
`Object getValue(EvaluationContext context)`：使用指定的EvaluationContext来计算表达式的值（其中EvaluationContext相当于容器包含着Expression表达式中所需要用到的值，如果是根对象，那么连对象名称都可以省略直接使用）
`<T> T getValue(EvaluationContext context,Class<T> desiredResultType)`：使用指定的EvaluationContext来计算表达式的值，而且尝试将该表达式的值当成desiredResultType类型处理
`Object getValue(Object rootObject)`：以rootObject作为表达式的root对象来计算表达式的值
`<T> T getValue(Object rootObject,Class<T> desiredResultType)`：以rootObject作为表达式的root对象来计算表达式的值，而且尝试将该表达式的值当成desiredResultType类型处理。

  上面程序中使用ExpressionParser多次解析了不同类型的表达式，ExpressionParser调用parseExpression()方法将返回一个Expression实例（表达式对象）。程序调用Expression对象的getValue()方法即可获取该表达式的值。

  EvaluationContext代表SpEL计算表达式的“上下文”，这个Context对象可以包含多个对象，但只有一个root(根)对象。

注：EvaluationContext的作用类似于OGNL中的StackContext，EvaluationContext 可以包含多个对象，但只能有一个root对象。当表达式中包含变量时，SpEL就会根据EvaluationContext中变量的值对表达式进行计算。
  为了往EvaluationContext里放入对象（SpEL称之为变量），可以调用该方法的如下方法：

setVariable(String name,Object value)：向EvaluationContext中放入value对象，该对象名为name
  **为了在SpEL访问EvaluationContext中指定对象，因采用与OGNL类似的格式：#name**

注意：该接口中没有定义设置root对象的方法，我们需要使用实现EvaluationContext接口的实现类StandardEvaluationContext来设置root对象，如下：

setRootObject(Object rootObject)
  在SpEL中访问root对象的属性时，可以省略root对象前缀。当然使用Expression对象计算表达式的值时，也可以直接指定root对象，如上面示例中的代码：

exp.getValue(person, String.class) //以person对象为root对象计算表达式的值

## 二、 Bean定义中的表达式语言支持

  SpEL的一个重要作用就是扩展Spring容器的功能，允许在Bean定义中使用SpEL。在XML文件和Annotation中都可以使用SpEL。在XML和Annotation中使用SpEL时，在表达式外边增加#{}包围即可。
示例如下：

```xml
<!-- 使用util.properties加载指定资源文件 -->
<util:properties id="confTest"
	location="classpath:test_zh_CN.properties"/>
<!--
配置setName()的参数时，在表达式中调

用方法
配置setAxe()的参数时，在表达式中创建对象
配置调用setBooks()的参数时，在表达式中访问其他Bean的属性 -->
<bean id="author" class="org.crazyit.app.service.impl.Author"
	p:name="#{T(java.lang.Math).random()}"
	p:axe="#{new org.crazyit.app.service.impl.SteelAxe()}"
	p:books="#{ {confTest.a , confTest.b} }"/>
```

  上面使用到了util命名空间，并且也利用到了SpEL的类型运算符T()。



## 三、SpEL语法

  虽然SpEL在功能上大致与JSP2的EL类似，但SpEL比JSP2的EL更强大。

### 3.1 直接量表达式

  直接量表达式是SpEL中最简单的表达式，直接量表达式就是在表达式中使用Java语言支持的直接量，包括字符串、日期、数值、boolean值和null，示例如下：

```java
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'");
String message = (String) exp.getValue();//亦或exp.getValue(String.class)
```

### 3.2 在表达式中创建数组

  SpEL表达式直接支持使用静态初始化、动态初始化两种语法来创建数组，示例如下：

```java
//创建一个数组
Expression exp = parser.parseExpression("new String[]{'java','spring'});
```


### 3.3 在表达式中创建List集合

  SpEL直接使用如下语法来创建List集合：`{ele1,ele2,ele3...}`



### 3.4 在表达式中访问List、Map等集合元素

  为了在SpEL中访问List集合的元素，可以使用如下语法格式：`list[index]`

  为了在SpEL中访问Map集合的元素，可以使用如下语法格式：`map[key]`

示例如下：

```java
// 创建一个ExpressionParser对象，用于解析表达式
ExpressionParser parser = new SpelExpressionParser();
//创建一个List
List<String> list=new ArrayList<>();
list.add("Java");
list.add("Spring");
//创建一个Map
Map<String,Double> map=new HashMap<>();
map.put("key_1", 25.0);
map.put("key_2", 60.0);
//创建一个EvaluationContext对象，作为SpEL解析变量的上下文
EvaluationContext  ctx=new StandardEvaluationContext();
//设置两个变量
ctx.setVariable("myList", list);
ctx.setVariable("myMap", map);
//访问List集合的第二个元素
String tempString=parser.parseExpression("#myList[1]").getValue(ctx,String.class);
System.out.println("List集合的第二个元素为："+tempString);
//访问Map集合的指定元素
Double tempDouble=parser.parseExpression("#myMap[key_2]").getValue(ctx, Double.class);
System.out.println("Map集合中key为key_2的值为："+tempDouble);
```


### 3.5 调用方法

  在SpEL中调用方法与在Java代码中调用方法没有任何区别，示例如下：

```java
//调用String对象的substring()方法
String temp=parser.parseExpression("'HelloWorld'.substring(2,5)").getValue(String.class);
System.out.println(temp);
```


### 3.6 算术、比较、逻辑、赋值、三目等运算符

  与JSP2 EL类似的是SpEL同样支持这些运算符，值得指出的是SpEL中使用赋值运算符的功能比较强大，这种赋值可以直接改变表达式所引用的实际对象。示例如下：

```java
// 创建一个ExpressionParser对象，用于解析表达式
ExpressionParser parser = new SpelExpressionParser();
// 创建一个List
List<String> list = new ArrayList<>();
list.add("Java");
list.add("Spring");
// 创建一个EvaluationContext对象，作为SpEL解析变量的上下文
EvaluationContext ctx = new StandardEvaluationContext();
// 设置一个变量
ctx.setVariable("myList", list);
// 对集合的第一个元素进行赋值
parser.parseExpression("#myList[0]='我爱你中国'").getValue(ctx);
// 下面测试输出
System.out.println("List更改后的第一个元素的值为：" + list.get(0));
// 使用三目运算符
System.out.println(parser.parseExpression("#myList.size()>3 ? 'myList长度大于3':'myList长度不大于于3'").getValue(ctx));
```



### 3.7 类型运算符

  SpEL提供了一个特殊的运算符：**T()**，这个运算符用于告诉SpEL将该运算符内的字符串当成“类”处理，避免Spring对其进行其他解析。尤其是调用某个类的静态方法时，T()运算符尤其有用。示例如下：

```java
System.out.println(parser.parseExpression("T(java.lang.Math).random()").getValue());
System.out.println(parser.parseExpression("T(System).getProperty('os.name')").getValue());
```


注意：T()运算符使用java.lang包下的类时可以省略包名，但使用其他包下的所有类时应使用全限定类名。



### 3.8 调用构造器

  SpEL允许在表达式中直接使用new来调用构造器，这种调用可以创建一个Java对象。



### 3.9 变量

  SpEL允许通过EvaluationContext来使用变量，该对象包含了一个setVariable(String name,Object value)方法，该方法用于设置一个变量(同样可以是一个对象)。一旦在EvaluationContext中设置了变量，就可以在SpEL中通过#name来访问该变量。示例前面已经罗列了很多，不过在SpEL中有两个特殊的变量：

- **#this**：引用SpEL当前正在计算的对象

- **#root**：引用SpEL的EvaluationContext的root对象



### 3.10 自定义函数

  SpEL允许开发者开发自定义函数，所谓自定义函数，也就是为Java方法重新起个名字而已。通过StandardEvaluationContext的如下方法可在SpEL中注册自定义函数：`registerFunction(String name, Method method)`
注意：SpEL自定义函数的作用不大，因为SpEL本身已经允许在表达式语言中调用方法，因此将方法重新定义的自定义函数的意义不大。



### 3.11Elvis运算符

  Elvis运算符只是三目运算符的特殊写法。例如对如下三目运算符写法：`name != null ? name : "newval"`，其中name变量写了两次，因此比较繁琐。SpEL允许将上面写法简化为：`name?:"newVal"`



### 3.12 安全导航操作

  在SpEL中调用很可能导致NullPointerException（空指针异常），如果调用对象的属性本身就是空，那么调用该对象的属性的属性自然就引发异常，为了避免这种情况，SpEL支持如下用法：foo?.bar，其中foo是假设根对象的属性，然后调用foo的bar字段，这种方式不会引发空指针异常。

```java
//使用安全操作，将输出null
System.out.println("安全操作结果："+parser.parseExpression("#foo?.bar").getValue());
//不适用安全操作，将引发异常
System.out.println("安全操作结果："+parser.parseExpression("#foo.bar").getValue());
```


### 3.13 集合选择

  SpEL允许直接对集合进行选择操作，这种选择操作可以根据指定表达式对集合元素进行筛选，只有符合条件的额集合元素才会被选择出来。SpEL集合选择的语法格式如下：`collection.?[condition_expr]`

示例如下：

```java
parser.parseExpression("#myList.?[length()>7]").getValue(ctx);
parser.parseExpression("#myMap.?[value()>80]").getValue(ctx);
```


  当操作List集合时，condition_expr中访问的每个属性、方法都是以集合元素为主调的；当操作Map集合时，需要显式地用key引用Map Entry的key，用value引用Map Entry的value。



### 3.14 集合投影

  SpEL允许对集合进行投影运算，这种投影运算将依次迭代每个集合元素，迭代时将根据指定表达式对集合元素进行计算得到一个新的结果，依次将每个结果收集成新的集合，这个新的集合将作为投影运算的结果，SpEL投影运算的语法格式为：`collection.![condition_expr]`

  condition_expr是一个根据集合元素定义的表达式，上面的SpEL会把collection集合中的元素依次传入condition_expr，每个元素得到一个新的结果，所以计算出来的结果所组成的新结果就是该表达式的返回值。示例如下：

```java
// 创建一个ExpressionParser对象，用于解析表达式
ExpressionParser parser = new SpelExpressionParser();
// 创建一个List
List<String> list = new ArrayList<>();
list.add("Java");
list.add("Spring");
// 创建一个EvaluationContext对象，作为SpEL解析变量的上下文
EvaluationContext ctx = new StandardEvaluationContext();
// 设置一个变量
ctx.setVariable("myList", list);
System.out.println(parser.parseExpression("#myList.![length()]").getValue(ctx));
```


  如果List中存在的是一个类的实例对象的集合，name可以使用属性名，将所有对象的该属性的值作为新结果返回。



### 3.15 表达式模板

  表达式模板有点类似于带占位符的国际化消息。表达式模板的本质是对“直接量表达式”的扩展，它允许在“直接量表达式”中插入一个或多个#{expr}，#{expr}将会被动态计算出来，示例如下：

```java
// 创建一个ExpressionParser对象，用于解析表达式
ExpressionParser parser = new SpelExpressionParser();
Person p1=new Person(1, "小明", 162);
Person p2=new Person(2, "小红", 182);
Expression expr =parser.parseExpression("我的名字是#{name}，身高是#{height}",new TemplateParserContext());
//将使用p1对象的name、height填充上面表达式模板中的#{}
System.out.println(expr.getValue(p1));
//将使用p2对象的name、height填充上面表达式模板中的#{}
System.out.println(expr.getValue(p2));
```


  使用ExpressionParser解析字符串模板是需要传入TemplateParserContext参数，改参数实现了ParserContext接口，它用于为表达式解析传入一些额外的信息，例如TemplateParserContext指定解析时需要计算#{和}之间的值。



## 使用SpEL表达式中的运算符

SpEL提供了多种运算符。

| 类型       | 运算符                                       |
| ---------- | -------------------------------------------- |
| 关系       | <，>，<=，>=，==，!=，lt，gt，le，ge，eq，ne |
| 算术       | +，- ，* ，/，%，^                           |
| 逻辑       | &&，                                         |
| 条件       | ?: (ternary)，?: (elvis)                     |
| 正则表达式 | matches                                      |
| 其他类型   | ?.，?[…]，![…]，^[…]，$[…]                   |

 

参考资料：

- 《轻量级JavaEE企业应用实战 第四版》
- [**SpEL 表达式**](https://blog.csdn.net/fanxiaobin577328725/article/details/68942967)
- [**spring之SpEL表达式**](https://blog.csdn.net/ya_1249463314/article/details/68484422)


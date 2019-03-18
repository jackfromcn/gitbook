### Integer是对象,方法参数传递时，在方法体内修改，为何没有修改其调用方的值

时间：2019年03月13日18:12:21

今天下午突然想了一个很基础，但又不理解的问题。

`Integer` 本身也是对象，在通过方法参数传递时，应该传递的是引用啊，在方法中修改其值，是不是也会修改调用方的值。于是进行尝试。代码如下：<br/>

```java
@Test
public void test() {
    Integer i = new Integer(3000);
    modify(i);
    System.out.println(i);
}

public void modify(Integer i) {
    i = new Integer(10);
    System.out.println(i);
}
```

输出结果如下：<br/>

```bash
10
3000
```



几番查询后，比较满意的答案如下。

![image-20190313181047380](/Users/wencheng/Desktop/gitbook/gitbook/日常工作中遇到的问题/image-20190313181047380.png)



结论：

方法参数列表也有自己的方法栈，即，变量也会分配在栈中，变量只想 `Integer` 对象，之后该变量重新指向了另一个 `Integer` 对象，因此方法参数传递时，方法栈中修改的值，不会影响到调用方的值。


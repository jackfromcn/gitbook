# 类型描述工具——TypeDescriptor2

### 1. 构建ResolvableType对象
ResolvableType提供了静态for方法进行构建,下面我们以代码来示例

#### 1.1.forClass
```java
    @SuppressWarnings("serial")
    static class ExtendsList extends ArrayList<CharSequence> {
    }

    @Test
    public void forClass() throws Exception {
        ResolvableType type = ResolvableType.forClass(ExtendsList.class);
        assertThat(type.getType(), equalTo((Type) ExtendsList.class));
    }
```

#### 1.2.forField
```java
    @Test
    public void forField() throws Exception {
        Field field = Fields.class.getField("charSequenceList");
        ResolvableType type = ResolvableType.forField(field);
        assertThat(type.getType(), equalTo(field.getGenericType()));
    }
```

#### 1.3 forMethodParameter
forMethodParameter还有很多变种
##### 1.3.1 forConstructorParameter
##### 1.3.2 forMethodReturnType
```java
    @Test
    public void forMethodParameter() throws Exception {
        Method method = Methods.class.getMethod("charSequenceParameter", List.class);
        MethodParameter methodParameter = MethodParameter.forMethodOrConstructor(method, 0);
        ResolvableType type = ResolvableType.forMethodParameter(methodParameter);
        assertThat(type.getType(), equalTo(method.getGenericParameterTypes()[0]));
    }
```

### 2. 获取Type类型

#### 2.1 getType方法
获取自身类型
```java
    @Test
    public void forClassTest() throws Exception {
        ResolvableType type = ResolvableType.forClass(ExtendsList.class);
        assertThat(type.getType(), equalTo((Type) ExtendsList.class));
        assertThat(type.getRawClass(), equalTo(ExtendsList.class));
    }
```

#### 2.2 getSuperType方法
```java
    @Test
    public void superTypeTest() throws Exception {
        ResolvableType type = ResolvableType.forType(ExtendsList.class);
        ResolvableType superType=type.getSuperType();
        System.out.println(superType.getType());
    }
```
```bash
输出结果:
java.util.ArrayList<java.lang.CharSequence>
```

#### 2.3 getRawClass
当Type为ParameterizedType时有效
```java
    @Test
    public void getRawClassTest() throws Exception {
        ResolvableType type = ResolvableType.forType(ExtendsList.class);
        ResolvableType superType=type.getSuperType();
        System.out.println(superType.getRawClass());
    }
```
```bash
输出结果:
java.util.ArrayList
```

#### 2.4 as方法
上面的方式可以将ExtendsList以as的方式转换一下,向上取接口或父类
```java
    @Test
    public void asTest() throws Exception {
        ResolvableType type = ResolvableType.forType(ExtendsList.class);
        ResolvableType listType=type.as(ArrayList.class);
        System.out.println(listType.getType());
        System.out.println(listType.getRawClass());
    }
```
```bash
输出结果:
java.util.ArrayList<java.lang.CharSequence>
class java.util.ArrayList
```

#### 2.5 getGeneric方法
这时候看起来使用应该是比较简单了
```java
    @Test
    public void getGenericTest() throws Exception {
        ResolvableType type = ResolvableType.forType(ExtendsList.class);
        ResolvableType listType=type.as(ArrayList.class);
        System.out.println(listType.getGeneric().getType());
    }
```

#### 2.6 resolveGeneric方法
更加简化的一种方法
```java
    @Test
    public void resolveGenericTest() throws Exception {
        ResolvableType type = ResolvableType.forType(ExtendsList.class);
        ResolvableType listType=type.as(ArrayList.class);
        System.out.println(listType.resolveGeneric());
    }
```

#### 2.7 getComponentType方法
返回数组类型
```java
    @Test
    public void getComponentTypeForClassArray() throws Exception {
        Field field = Fields.class.getField("arrayClassType");
        ResolvableType type = ResolvableType.forField(field);
        assertThat(type.isArray(), equalTo(true));
        assertThat(type.getComponentType().getType(),
                equalTo((Type) ((Class) field.getGenericType()).getComponentType()));
    }

    @Test
    public void getComponentTypeForGenericArrayType() throws Exception {
        ResolvableType type = ResolvableType.forField(Fields.class.getField("genericArrayType"));
        assertThat(type.isArray(), equalTo(true));
        assertThat(type.getComponentType().getType(),
                equalTo(((GenericArrayType) type.getType()).getGenericComponentType()));
    }
```

### 参考
http://jinnianshilongnian.iteye.com/blog/1993608
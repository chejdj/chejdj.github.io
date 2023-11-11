---
layout: post
title: Gson序列化Kotlin数据类型默认值失效
date: 2023-11-11 17:00:00
categories: 
- Java
tags:
- Java
---  

### 背景
最近开发的时候遇到一个问题，服务端返回的Json中如果没有某个字段的时候，我需要设置一个默认的值，但是给这个数据类设置默认值1的时候，Json解析之后还是返回了0。 但是项目里面其他数据类设置默认值是有效的。
### 样例
```
data class A(val author:String = "", val age:Int=1)

data class B(val author:String?, val age:Int =1)
```
项目中唯一的不同就是 A的构造函数author设置了，B没有设置。当服务端没有返回age字段的时候，A序列化age具有默认值，B的age 默认值为0， 与预期不符合
### 源码解析
Gson序列化，最终都是通过Gson.fromJson进行序列化的
```
public <T> T fromJson(JsonReader reader, Type typeOfT) throws JsonIOException, JsonSyntaxException {
    boolean isEmpty = true;
    boolean oldLenient = reader.isLenient();
    reader.setLenient(true);
    try {
      reader.peek();
      isEmpty = false;
      TypeToken<T> typeToken = (TypeToken<T>) TypeToken.get(typeOfT);
      TypeAdapter<T> typeAdapter = getAdapter(typeToken); //获取类型构造器
      T object = typeAdapter.read(reader); //构造类
      return object;
    } catch (EOFException e) {
      /*
       * For compatibility with JSON 1.5 and earlier, we return null for empty
       * documents instead of throwing.
       */
      if (isEmpty) {
        return null;
      }
      throw new JsonSyntaxException(e);
    } catch (IllegalStateException e) {
      throw new JsonSyntaxException(e);
    } catch (IOException e) {
      // TODO(inder): Figure out whether it is indeed right to rethrow this as JsonSyntaxException
      throw new JsonSyntaxException(e);
    } catch (AssertionError e) {
      AssertionError error = new AssertionError("AssertionError (GSON " + GsonBuildConfig.VERSION + "): " + e.getMessage());
      error.initCause(e);
      throw error;
    } finally {
      reader.setLenient(oldLenient);
    }
  }
```
关键的代码就是，获取TypeAdapter然后通过这个TypeAdapter#read进行构造(这里注册了很多TypeAdapter包括我们自己自定义)
通过断点我们可以知道调用`ReflectiveTypeAdapterFactory` 这个类
```
@Override public T read(JsonReader in) throws IOException {
      if (in.peek() == JsonToken.NULL) {
        in.nextNull();
        return null;
      }

      T instance = constructor.construct(); //关键
     //.....
}
```
constructor.construct()跟踪到ConstructorConstructor 方法
```
// 该类是否已经继承InstanceCreator,外部提供构造函数
final InstanceCreator<T> typeCreator = (InstanceCreator<T>) instanceCreators.get(type);
    if (typeCreator != null) {
      return new ObjectConstructor<T>() {
        @Override public T construct() {
          return typeCreator.createInstance(type);
        }
      };
    }

    // 基础类型构造int,String,char等
    @SuppressWarnings("unchecked") // types must agree
    final InstanceCreator<T> rawTypeCreator =
        (InstanceCreator<T>) instanceCreators.get(rawType);
    if (rawTypeCreator != null) {
      return new ObjectConstructor<T>() {
        @Override public T construct() {
          return rawTypeCreator.createInstance(type);
        }
      };
    } 

    // 默认构造函数
    ObjectConstructor<T> defaultConstructor = newDefaultConstructor(rawType);
    if (defaultConstructor != null) {
      return defaultConstructor;
    }

    //一些队列构造Set,List,Map等
    ObjectConstructor<T> defaultImplementation = newDefaultImplementationConstructor(type, rawType);
    if (defaultImplementation != null) {
      return defaultImplementation;
    }
 
    // 非安全构造
    return newUnsafeAllocator(type, rawType);
```
也就是说Gson构造对象有下面优先级
1.该类自身扩展的构造方法，需要实现 InstanceCreator 接口  
2.基础类型构建
3.默认构造函数
4.队列Set，List,Map等特殊构造
5.非安全构造，兜底使用
其中默认构造函数取的是空参数构造方法
```
private <T> ObjectConstructor<T> newDefaultConstructor(Class<? super T> rawType) {
    try {
      final Constructor<? super T> constructor = rawType.getDeclaredConstructor();
     //....
     //....
}
```
**unsafe 方式去构造对象，会绕过构造函数，只会在堆中去分配一个对象实例**
[unsafe类介绍](https://www.cnblogs.com/trunks2008/p/14720811.html)
### 原因定位
我们再回过来头看A和B的差异，原因就是A都默认给了参数，Kotlin(反编译成Java)会生成默认的无参构造函数，构造A对象的时候就会调用A的构造，所以赋值操作也会执行。但是B的构造函数，则没有默认的无参构造函数，所以只能通过unsafe方式构造对象，绕过了构造函数，不会执行赋初值逻辑，默认值只能是基础类型的初值
### 问题解决方法
本质: 尽量走构造方法，而不是走unsafe构建
1. 构造函数所有变量设置默认值
```
data class B(val author:String="", val age:Int =1)
```
2.不复写构造函数，使用无参构造函数
```
data class B{
  val author:String?
  val age:Int = 1
}
```

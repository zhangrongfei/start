#痛点

在java编码过程中，大家碰到的最多的异常是什么，我相信必然这货`NullPointerException`必然是排行第一的。那我们在平时编码中，有各种编码规范与其相关，比如时时的判断`null`，方法禁止返回`null`等，例如
```
public void bindUserToRole(User user) {
    if (user != null) {
        String roleId = user.getRoleId();
        if (roleId != null) {
            Role role = roleDao.findOne(roleId);
            if (role != null) {
                role.setUserId(user.getUserId());
                roleDao.save(role);
            }
        }
    }
}
```
或者
```
public String bindUserToRole(User user) {
    if (user == null) {
        return;
    }

    String roleId = user.getRoleId();
    if (roleId == null) {
        return;
    }

    Role = roleDao.findOne(roleId);
    if (role != null) {
        role.setUserId(user.getUserId());
        roleDao.save(role);
    }
}
```
为了防止`NullPointerException`，好好的代码写成这个鸟样，或许下面的会看上去比较顺眼一点，但是大体还是一样的。
其实我们有一种更为优雅的方式来完成上面的功能，如下
```
Optional<String> roleOpt = Optional.ofNullable(user).map(User::getRoleId);
if(roleOpt.isPresent()){
    ....
}
```
这样，我们仅需要对我们关心的做一次校验，省却了前面的一系列的检验操作。


#Optional的引入
基于上述的一些原因，在JDK8中，引入了一个新的类`java.util.Optional`，来避免这类问题的处理。
先看下类的说明
>A container object which may or may not contain a non-null value.If a value is present, `isPresent()`will return `true` and`get()` will return the value.

这里面说明这是一个可以包含`null`或者非`null`的容器，其最基本的两个操作就是`isPresent()`和`get()`，基本用起来就是这样
```
if( oneOptional.isPresent() ){
     String s = oneOptional.get();
    ....
}
```
首先我们看下这个类中包含哪些方法，如下

![image.png](https://upload-images.jianshu.io/upload_images/4840092-6e587f7f6167d8b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
首先，其有一个成员变量和一个定义的常量如下
```
   private static final Optional<?> EMPTY = new Optional<>();
   private final T value;
```
其中`value`表示其封装的真实的对象，`EMPTY`是定义的一个表示空的常量。构造方法很很简单，两个常见的私有构造方法`Optional()`和`Optional(T)`，其中不带参的默认设置`value`为`null`，带参的构造函数，不允许传`null`。

`Optional`类包含3个静态方法生成`Optional`对象，分别为
* `Optional<T> empty()`
生成一个空`Optional`对象，其`value`为`null`
* `Optional<T> of(T value)`
调用`Optional(T)`构造方法，其`value`不允许为`null`
* `Optional<T> ofNullable(T value)`
与`Optional<T> of(T value)`的差别是其传入的`value`允许为`null`

下面简单介绍一下里面的各个方法和一些简单的使用示例
* `get()`
返回`value`，若为`null`则抛出异常` NoSuchElementException`
* `isPresent()`
判断当前的`value`是否为`null`
* `ifPresent(Consumer<? super T> consumer)`
该方法支持传入一个`Consumer`对象，当`value`不为`null`的时候调用，为`null`则不做任何操作，示例如下
```
    public void testIfPresent(){
        Optional<String> optional=Optional.of("zh");
        optional.ifPresent(new Consumer<String>() {
            @Override
            public void accept(String s) {
                System.out.println(s);
            }
        });
    }
```
配合`lambda`使用，代码更为简洁
```
    public void testIfPresent(){
        Optional<String> optional=Optional.of("zh");
        optional.ifPresent(s -> System.out.println(s));
    }
```
* `filter(Predicate<? super T> predicate)`
顾名思义，这个方法的作用就是filter（过滤），该方法用于过滤一个`Optional`对象，通过传入的`Predicate`对象中的一个`test`方法，例如
```
        Optional<String> optional=Optional.of("aa");
        Optional<String> optionalResult=optional.filter(s -> s.startsWith("a"));
        System.out.println(optionalResult);
```
输出结果是
```
Optional[aa]
```
由于其返回仍是一个`Optional`对象，我们可以有如下较为优雅的写法
```
Optional<String> optionalResult=optional.filter(s -> s.startsWith("a"))
                .filter(s -> s.length()==2)
                .filter(s -> s.endsWith("a"));
```
* `map(Function<? super T, ? extends U> mapper)`
此方法支持一个`Function`参数，在这个`Function`里面可以对这个`Optional`对象做一些操作或者改变，其返回值仍为一个`Optional`类型，例如
```
Optional<String> optional=Optional.of("aa");
        Optional<Integer> result= optional.map(s -> s.toUpperCase())
                .map(s->s.length());
```
输出结果为
```
Optional[2]
```
`map`提供一种优雅的流式的方式来替代先前繁杂的`if`判断和数据处理

* `flatMap(Function<? super T, Optional<U>> mapper)`
和`map`类似，只不过需要我们手动将方法的返回，封装成`Optional`对象，如
```
Optional<Integer> result= optional.flatMap(s -> Optional.of(s.toUpperCase()))
                .flatMap(s -> Optional.of(s.length()));
```
其余功能与`map`一致 。
  
* `orElse(T other)`
这个方法较为简单，参数为一个默认值，即若`value`为`null`，则返回默认值
```
        Optional<String> optional=Optional.of("aa");
        System.out.println(optional.orElse("bb"));
        Optional<String> optional1=Optional.ofNullable(null);
        System.out.println(optional1.orElse("bb"));
```
输出为
```
aa
bb
```
* `orElseGet(Supplier<? extends T> other)`
提供一个`Supplier`入参，改`Supplier`提供一个默认值，例如
```
    Optional<String> optional=Optional.ofNullable(null);
    System.out.println(optional.orElseGet(() -> "aaa"));
```
* `orElseThrow(Supplier<? extends X> exceptionSupplier)`
顾名思义，若为`null`则抛出异常
```
        Optional<String> optional=Optional.ofNullable(null);
        System.out.println(optional.orElseThrow(() -> new RuntimeException()));
```

#使用场合注意
>Reports calls to java.util.Optional.get() without first checking with a isPresent() call if a value is available. If the Optional does not contain a value, get() will throw an exception. 

Optional.get() 前不事先用 isPresent() 检查值是否可用. 假如 Optional 不包含一个值, get() 将会抛出一个异常
>Reports any uses of java.util.Optional<T>, java.util.OptionalDouble, java.util.OptionalInt, java.util.OptionalLong or com.google.common.base.Optional as the type for a field or a parameter. Optional was designed to provide a limited mechanism for library method return types where there needed to be a clear way to represent “no result”. Using a field with type java.util.Optional is also problematic if the class needs to be Serializable, which java.util.Optional is not

使用任何像 Optional 的类型作为字段或方法参数都是不可取的. Optional 只设计为类库方法的, 可明确表示可能无值情况下的返回类型. Optional 类型不可被序列化, 用作字段类型会出问题的

#借鉴
*   [https://lw900925.github.io/java/java8-optional.html](https://lw900925.github.io/java/java8-optional.html)
*   http://www.importnew.com/22060.html




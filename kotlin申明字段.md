### 在kotlin中如何声明public属性？在kotlin中，属性的默认修饰为public？

#### 1、使用val

```kotlin
val name1: String = "name1"
```

反编译后的Java代码：

```java
@NotNull
private static final String name1 = "name1";

@NotNull
public static final String getName1() {
  return name1;
}
```

##### name1的实际类型为：private static final，私有静态常量

#### 2、使用var

```kotlin
var name2: String? = "name2"
```

反编译后的Java代码：

```java
@Nullable
private static String name2 = "name2";

@Nullable
public static final String getName2() {
  return name2;
}

public static final void setName2(@Nullable String var0) {
  name2 = var0;
}
```

##### name2的实际类型为：private static，私有静态变量

#### 3、使用lateinit var

```kotlin
lateinit var name3: String
```

反编译后的Java代码：

```java
public static String name3;

@NotNull
public static final String getName3() {
  String var10000 = name3;
  if (var10000 == null) {
     Intrinsics.throwUninitializedPropertyAccessException("name3");
  }

  return var10000;
}

public static final void setName3(@NotNull String var0) {
  Intrinsics.checkNotNullParameter(var0, "<set-?>");
  name3 = var0;
}
```

##### name3的实际类型为：public static，公开静态变量

#### 总结：

1. 如何声明public属性？

   在kotlin中如果想要声明真正的public类型，lateinit var

   val 和 var声明的实际上都是private类型

2. 属性的默认修饰为public？

   只是会生成对应的public的get和set方法

---
layout:     post
title:      Kotlin 泛型中的 in 和 out
subtitle:   与 Java 泛型对比
date:       2020-06-01
author:     JH
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Kotlin
    - Java
---

### in & out

#### out (协变)
如果泛型类只将泛型类型作为函数的返回(输出)，那么使用 out 可以称之为生产类/接口，它主要是用来生产指定的泛型对象

```kotlin
// 子类泛型对象可以赋值给父类泛型对象用 out
interface Production<out T> {
    fun produce(): T
}
```
可以称其为 production class/interface，因为其主要是产生（produce）指定泛型对象。

#### in (逆变)
如果泛型类只将泛型类型作为函数的入参(输入)，那么使用 in 可以称之为消费类/接口，它主要是用来消费指定的泛型对象

```kotlin
interface Consumer<in T> {
// 父类泛型对象可以赋值给子类泛型对象用 in
fun consume(item: T)
}
```
可以称其为 consumer class/interface，因为其主要是消费指定泛型对象。

#### 例子说明
假设有一个汉堡（burger）对象，它是一种快餐，当然更是一种食物。

```kotlin
open class Food
open class FastFood : Food() 
class Burger : FastFood()
```

**汉堡生产者**
```kotlin
class FoodStore : Producer<Food> {
    override fun product(): Food {
        println("Produce food")
        return Food()
    }
}

class FastFoodStore : Producer<FastFood> {
    override fun product(): FastFood {
        println("Produce fastFood")
        return FastFood()
    }
}

class BurgerStore : Producer<Burger> {
    override fun product(): Burger {
        println("Produce burger")
        return Burger()
    }
}
```

现在，对于生产类可以这样赋值：

```kotlin
val p1: Producer<Food> = FoodStore()
val p2: Producer<Food> = FastFoodStore()
val p3: Producer<Food> = BurgerStore()
```

很显然，汉堡/快餐商店是属于食品商店

**因此，对于 out 泛型，可以使用子类泛型的对象赋值给父类泛型的对象**

而，如果像下面这样反过来使用子类 - Burger 泛型，就会出现错误。
因为快餐（Fastfood）和食品（Food）商店不仅仅提供汉堡（Burger）

```kotlin
val production1 : Production<Burger> = FoodStore()  // Error
val production2 : Production<Burger> = FastFoodStore()  // Error
```

**汉堡消费者**
```kotlin
class Everybody : Consumer<Food> {
    override fun consume(item: Food) {
        println("Eating food")
    }
}

class ModernPeople : Consumer<FastFood> {
    override fun consume(item: FastFood) {
        println("Eating fastFood")
    }
}

class American : Consumer<Burger> {
    override fun consume(item: Burger) {
        println("Eating burger")
    }
}
```

现在，我们能够将 Everybody, ModernPeople 和 American 都指定给汉堡消费者（Consumer<Burger>）：

```kotlin
val c1: Consumer<Burger> = American()
val c2: Consumer<Burger> = ModernPeople()
val c3: Consumer<Burger> = Everybody()
```

很显然这里汉堡的消费者不仅是美国人，更是现代人、所有人

**因此，对于 in 泛型，可以将使用父类泛型的对象赋值给使用子类泛型的对象**

同样，如果这里反过来使用父类 - Food 泛型，就会报错：
```kotlin
val consumer2 : Consumer<Food> = ModernPeople()  // Error
val consumer3 : Consumer<Food> = American()  // Error
```
根据以上的内容，我们还可以这样来理解什么时候用 in 和 out：

父类泛型对象可以赋值给子类泛型对象，用 in
子类泛型对象可以赋值给父类泛型对象，用 out

### Java 泛型中的 extend & super
由上述介绍，可以联想对比 Java 中的 extend 和 super

    <? extends T>和<? super T>是Java泛型中的“通配符（Wildcards）”和“边界（Bounds）”的概念。
    
    <? extends T>：是指 “上界通配符（Upper Bounds Wildcards）”

    <? super T>：是指 “下界通配符（Lower Bounds Wildcards）”

提前定义主类Plate：
```java
class Plate<T>{
    private T item;
    public Plate(T t){item=t;}
    public void set(T t){item=t;}
    public T get(){return item;}
}
```

#### extend 上界通配符
```java
Plate<？ extends Fruit>
```
一个能放水果以及一切是水果派生类的盘子，有Plate<Fruit>、 Plate<Apple>、Plate<Banana>、Plate<Red Apple>等

上界<? extends T>不能往里存，只能往外取。原因是编译器只知道容器内是Fruit或者它的派生类，但具体是什么类型不知道。

因此，往盘子里放东西的set()方法失效，但取东西get()方法还有效：

```java
Plate<? extends Fruit> p=new Plate<Apple>(new Apple());
	
//不能存入任何元素
p.set(new Fruit());    //Error
p.set(new Apple());    //Error

//读取出来的东西只能存放在Fruit或它的基类里。
Fruit newFruit1=p.get();
Object newFruit2=p.get();
Apple newFruit3=p.get();    //Error
```


#### super 下界通配符
```java
Plate<? super Fruit>
```
一个能放水果以及一切是水果基类的盘子，只有 Plate<Food>、Plate<Fruit>

下界<? super T>不影响往里存，但往外取只能放在Object对象里。因为下界规定了元素的最小粒 度的下限，实际上是放松了容器元素的类型控制。既然元素是Fruit的基类，那往里存粒度比Fruit 小的都可以。但往外读取元素就费劲了，只有所有类的基类Object对象才能装下。但这样的话， 元素的类型信息就全部丢失。

因此，从盘子里取东西的get()方法部分失效，只能存放到Object对象里，set()方法正常。

```java
Plate<? super Fruit> p=new Plate<Fruit>(new Fruit());

//存入元素正常
p.set(new Fruit());
p.set(new Apple());

//读取出来的东西只能存放在Object类里。
Apple newFruit3=p.get();    //Error
Fruit newFruit1=p.get();    //Error
Object newFruit2=p.get();
```

**根据以上内容，频繁往外读取内容的，适合用上界Extends。经常往里插入的，适合用下界Super，即是PECS（Producer Extends Consumer Super）原则**

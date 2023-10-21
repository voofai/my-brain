---
date: 2022-01-22
share: true
tags:
  - java
slug: bigdecimal
---

`BigDecimal` это класс языка Java для представления чисел с плавающей запятой. И если вы уверены, что `1.8 == 1.80`, то вас может ждать сюрприз. 
<!-- more -->
Метод `BigDecimal.equals` может отработать не так как вы ожидали. Цитата из javadoc:

>Compares this BigDecimal with the specified Object for equality. Unlike compareTo, this method considers two BigDecimal objects equal only if they are equal in value and scale (thus 2.0 is not equal to 2.00 when compared by this method).

Таким образом сравнивается не только сами значения, но и количество знаков после запятой, даже если это нули. Если вы хотите, чтобы при сравнении точность не играла такой роли, то используйте метод `compareTo`. На практике это выглядит так:

```java
var a = new BigDecimal("0.8");
var b = new BigDecimal("0.80");

System.out.println(a.equals(b));            // вывод: false
System.out.println(a.compareTo(b) == 0);    // вывод: true
```

В этом смысле забавно выглядит, например, код на Kotlin, в котором, в отличие от Java, есть перегрузка операторов:

```kotlin
val a = BigDecimal("2.0")
val b = BigDecimal("2.00")

a == b  // false
a > b  // false
a < b  // false
a.compareTo(b) == 0  // true
```

из остальных jvm языков Scala ведет себя также как Kotlin и там `==` определяется как вызов метода `equals`. А вот groovy старается быть ближе к народу и там `==` ссылается на `compareTo`:

```groovy
def a = new BigDecimal("2.0")
def b = new BigDecimal("2.00")

println a == b // true
```

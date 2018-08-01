---
title: Java meta-annotations
date: 2018-08-01 12:53:08
tags:
    - JavaBasic
---

## `@Retention`
`@Retention` annotation specifies how the marked annotation is stored:
（用于指明注解怎样存储
* __RetentionPolicy.SOURCE__ – The marked annotation is retained only in the source level and is ignored by the compiler.
（仅存储在源代码中，编译成class文件后将不再保存该注解
* __RetentionPolicy.CLASS__ – The marked annotation is retained by the compiler at compile time, but is ignored by the Java Virtual Machine (JVM).
（存储在源代码和class文件中，在JVM中运行时不再保存该注解
* __RetentionPolicy.RUNTIME__ – The marked annotation is retained by the JVM so it can be used by the runtime environment.
（在源代码、class文件和JVM运行时期都将保存该注解

## `@Documented`
`@Documented` annotation indicates that whenever the specified annotation is used those elements should be documented using the Javadoc tool. (By default, annotations are not included in Javadoc.)
（用于指明该注解是否要生成文档 [example](https://www.developer.com/java/other/article.php/10936_3556176_3/An-Introduction-to-Java-Annotations.htm)

## `@Target`
`@Target` annotation marks another annotation to restrict what kind of Java elements the annotation can be applied to. A target annotation specifies one of the following element types as its value:
（用于指明注解可以应用到哪一种Java元素类型上
* __ElementType.ANNOTATION_TYPE__ can be applied to an annotation type.（应用在注解
* __ElementType.CONSTRUCTOR__ can be applied to a constructor.（应用在构造器
* __ElementType.FIELD__ can be applied to a field or property.（应用在域或属性
* __ElementType.LOCAL_VARIABLE__ can be applied to a local variable.（应用在局部变量
* __ElementType.METHOD__ can be applied to a method-level annotation.（应用在方法
* __ElementType.PACKAGE__ can be applied to a package declaration.（应用在包
* __ElementType.PARAMETER__ can be applied to the parameters of a method.（应用在方法参数
* __ElementType.TYPE__ can be applied to any element of a class.（应用在任何地方

## `@Inherited `
`@Inherited` annotation indicates that the annotation type can be inherited from the super class. (This is not true by default.) When the user queries the annotation type and the class has no annotation for this type, the class' superclass is queried for the annotation type. This annotation applies only to class declarations.
（用于指明注解能否被继承，默认是不能被继承的，[example](https://stackoverflow.com/questions/23973107/how-to-use-inherited-annotation-in-java)

## `@Repeatable`
`@Repeatable` annotation, introduced in Java SE 8, indicates that the marked annotation can be applied more than once to the same declaration or type use. For more information, see Repeating Annotations.
（Java8新特性，用于指明注解能否在同一个地方多次使用，[example](https://docs.oracle.com/javase/tutorial/java/annotations/repeating.html)

笔记来源： [Annotations](https://docs.oracle.com/javase/tutorial/java/annotations/predefined.html)
---
title: Spring的自定义注解
date: 2018-03-10 15:12:19
tags: [注解,Java,Spring]
categories: Java
---
# 元注解
元注解是指注解的注解。Java的元注解有四种，分别是@Documented、@Retention、@Target和@Inherited。
## @Documented
> Indicates that annotations with a type are to be documented by javadoc and similar tools by default.

## @Retention
> Indicates how long annotations with the annotated type are to be retained.

@Retention决定注解的保留时间，它有一个RetentionPolicy类型的value成员变量，默认值是RetentionPolicy.CLASS。枚举类RetentionPolicy有三个值：
* SOURCE
	> Annotations are to be discarded by the compiler.
* CLASS
	> Annotations are to be recorded in the class file by the compiler but need not be retained by the VM at run time.  This is the default behavior.
* RUNTIME
	> Annotations are to be recorded in the class file by the compiler and retained by the VM at run time, so they may be read reflectively.

## @Target
> Indicates the contexts in which an annotation type is applicable.

## Inherited
> Indicates that an annotation type is automatically inherited. 

# 自定义注解

待续。。。


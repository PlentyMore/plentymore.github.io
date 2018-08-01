---
title: Java Exceptions
date: 2018-08-01 14:41:06
tags:
    - JavaBasic
---

## The Three Kinds of Exceptions
* 已检查异常
>The first kind of exception is the checked exception. These are exceptional conditions that a well-written application should anticipate and recover from. For example, suppose an application prompts a user for an input file name, then opens the file by passing the name to the constructor for java.io.FileReader. Normally, the user provides the name of an existing, readable file, so the construction of the FileReader object succeeds, and the execution of the application proceeds normally. But sometimes the user supplies the name of a nonexistent file, and the constructor throws java.io.FileNotFoundException. A well-written program will catch this exception and notify the user of the mistake, possibly prompting for a corrected file name.

>Checked exceptions are subject to the Catch or Specify Requirement. All exceptions are checked exceptions, except for those indicated by Error, RuntimeException, and their subclasses.
* 错误
>The second kind of exception is the error. These are exceptional conditions that are external to the application, and that the application usually cannot anticipate or recover from. For example, suppose that an application successfully opens a file for input, but is unable to read the file because of a hardware or system malfunction. The unsuccessful read will throw java.io.IOError. An application might choose to catch this exception, in order to notify the user of the problem — but it also might make sense for the program to print a stack trace and exit.

>Errors are not subject to the Catch or Specify Requirement. Errors are those exceptions indicated by Error and its subclasses.
* 运行时异常
>The third kind of exception is the runtime exception. These are exceptional conditions that are internal to the application, and that the application usually cannot anticipate or recover from. These usually indicate programming bugs, such as logic errors or improper use of an API. For example, consider the application described previously that passes a file name to the constructor for FileReader. If a logic error causes a null to be passed to the constructor, the constructor will throw NullPointerException. The application can catch this exception, but it probably makes more sense to eliminate the bug that caused the exception to occur.

>Runtime exceptions are not subject to the Catch or Specify Requirement. Runtime exceptions are those indicated by RuntimeException and its subclasses.

>Errors and runtime exceptions are collectively known as unchecked exceptions.

## 异常类型选择
* 能够从异常中恢复，则使用已检查异常，若不能恢复，则使用运行时异常
* 用运行时异常表示编程错误（具体参考Effective Java

笔记来源： 
1. [Java Exception](https://docs.oracle.com/javase/tutorial/essential/exceptions/definition.html)
2. Effective Java 
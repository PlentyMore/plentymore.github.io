---
title: Java Basic
date: 2018-08-01 12:49:55
tags:
    - JavaBasic
---

# Variables
## Terminology

### "field" and "variable"
#### 4 kinds of variables:
* Instance Variables (Non-Static Fields)（实例变量
* Class Variables (Static Fields)（类/静态变量
* Local Variables（本地/局部变量 
* Parameters （参数
>If we are talking about "fields in general" (excluding local variables and parameters), we may simply say "fields". If the discussion applies to "all of the above", we may simply say "variables". If the context calls for a distinction, we will use specific terms (static field, local variables, etc.) as appropriate. You may also occasionally see the term "member" used as well. A type's fields, methods, and nested types are collectively called its members.

域(fields)不包括局部变量和参数，变量(variables)包括全部(fields, local variables, parameters)

## Primitive Data Type
Data Type | Default Value(for fields) | Size | Min | Max
-------------- | ------------------------------ | ---- | --- | ---
byte | 0 | 8-bit| -128 | 127
short | 0 | 16-bit | -32,768 | 32767
int | 0 | 32-bit |  -2^31 |  -2^31 -1
long | 0L | 64-bit | -2^63 |  -2^63 -1
float | 0.0f | 32-bit | [link](https://docs.oracle.com/javase/specs/jls/se7/html/jls-4.html#jls-4.2.3) | [link](https://docs.oracle.com/javase/specs/jls/se7/html/jls-4.html#jls-4.2.3)
double | 0.0d | 64-bit | above link | above link 
boolean | false |  not precisely defined. | false | true
char | '\u0000' | 16-bit | '\u0000' | '\uffff'

## 在数字字面值中使用下划线
>In Java SE 7 and later, any number of underscore characters (_) can appear anywhere between digits in a numerical literal. This feature enables you, for example. to separate groups of digits in numeric literals, which can improve the readability of your code.

>For instance, if your code contains numbers with many digits, you can use an underscore character to separate digits in groups of three, similar to how you would use a punctuation mark like a comma, or a space, as a separator.

>The following example shows other ways you can use the underscore in numeric literals:
```angular2html
long creditCardNumber = 1234_5678_9012_3456L;
long socialSecurityNumber = 999_99_9999L;
float pi =  3.14_15F;
long hexBytes = 0xFF_EC_DE_5E;
long hexWords = 0xCAFE_BABE;
long maxLong = 0x7fff_ffff_ffff_ffffL;
byte nybbles = 0b0010_0101;
long bytes = 0b11010010_01101001_10010100_10010010;
```

>You can place underscores only between digits; you **cannot** place underscores in the following places:
>* At the beginning or end of a number
>* Adjacent to a decimal point in a floating point literal
>* Prior to an F or L suffix
>* In positions where a string of digits is expected

>The following examples demonstrate valid and invalid underscore placements (which are highlighted) in numeric literals:


// **Invalid: cannot put underscores**

__// adjacent to a decimal point__

float pi1 = 3_.1415F;

// **Invalid: cannot put underscores**
 
// **adjacent to a decimal point**

float pi2 = 3._1415F;

// **Invalid: cannot put underscores**
 
// **prior to an L suffix**

long socialSecurityNumber1 = 999_99_9999_L;


// OK (decimal literal)

int x1 = 5_2;

// **Invalid: cannot put underscores**

// **At the end of a literal**

int x2 = 52_;

// OK (decimal literal)

int x3 = 5_______2;


// **Invalid: cannot put underscores**

// **in the 0x radix prefix**

int x4 = 0_x52;

// **Invalid: cannot put underscores**

// **at the beginning of a number**

int x5 = 0x_52;

// OK (hexadecimal literal)

int x6 = 0x5_2;
 
// **Invalid: cannot put underscores**

// **at the end of a number**

int x7 = 0x52_;


# Operators
**Operator Precedence**

优先级相同的运算符，除了分配运算符（=）以外，都算从左往右运算

Operators | Precendence
------------- | ------------------
postfix | expr++&nbsp;expr\-\-  
unary | ++expr&nbsp;\-\-expr&nbsp;+expr&nbsp;-expr&nbsp;~&nbsp;!
multiplicative | *&nbsp;/&nbsp;%
additive | +&nbsp;-
shift | <<&nbsp;>>&nbsp;>>>
relational | <&nbsp;>&nbsp;<=&nbsp;>=&nbsp;instanceof
equality | ==&nbsp;!=
bitwise AND | &
bitwise exclusive OR | ^
bitwise inclusive OR | &#124;
logical AND | &&
logical OR | &#124;&#124;
ternary | ? :
assignment | =&nbsp;+=&nbsp;-=&nbsp;*=&nbsp;/=&nbsp;%=&nbsp;&=&nbsp;^=&nbsp;<<=&nbsp;>>=&nbsp;>>>=&nbsp;&#124;=

笔记来源： [Language Basics](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/index.html)


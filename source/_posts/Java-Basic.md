---
title: Java Basic
date: 2018-08-01 12:49:55
tags:
    - JavaBasic
---

# Variables
## Terminology
>If we are talking about "fields in general" (excluding local variables and parameters), we may simply say "fields". If the discussion applies to "all of the above", we may simply say "variables". If the context calls for a distinction, we will use specific terms (static field, local variables, etc.) as appropriate. You may also occasionally see the term "member" used as well. A type's fields, methods, and nested types are collectively called its members.

域(fields)不包括局部变量和参数，变量(variables)包括全部(fields, local variables, parameters)

## Primitive Data Type
Data Type | Default Value(for fields)
-------------- | ------------------------------
byte | 0
short | 0
int | 0
long | 0L
float | 0.0f
double | 0.0d
boolean | false
char | '\u0000'

# Operators
**Operator Precedence**

Operators | Precendence
------------- | ------------------
postfix | expr++ expr--
unary | ++expr --expr +expr -expr ~ !
multiplicative | * / %
additive | + -
shift | << >> >>>
relational | < > <= >= instanceof
equality | == !=
bitwise AND | &
bitwise exclusive OR | ^
bitwise inclusive OR | \|
logical AND | &&
logical OR | \|\|
ternary | ? :
assignment | = += -= *= /= %= &= ^= |= <<= >>= >>>=


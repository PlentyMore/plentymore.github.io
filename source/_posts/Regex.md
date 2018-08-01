---
title: Regex
date: 2018-08-01 13:44:38
tags:
    - JavaBasic
---

## Character Classes


Construct | Description
-- | --
[abc] | a, b, or c (simple class)（a,b或c
[^abc] | Any character except a, b, or c (negation)（除a,b或c以外的任意字符
[a-zA-Z] | a through z, or A through Z, inclusive (range)（a-z或A-Z，包括a,z,A,Z
[a-d[m-p]] | a through d, or m through p: [a-dm-p] (union)（并集，a-d或者e-f之间的所有字符
[a-z&&[def]] | d, e, or f (intersection)（交集，a-z中与d,e或f相同的字符
[a-z&&[^bc]] | a through z, except for b and c: [ad-z] (subtraction)（a-z中除b和c以外的字符
[a-z&&[^m-p]] | a through z, and not m through p: [a-lq-z] (subtraction)（a-z中除m-p以外的字符

## Predefined Character Classes

Construct | Description
-- | --
. | Any character (may or may not match line terminators)（任意字符，可能匹配也可能不匹配行结束符
\d | A digit: [0-9]（数字
\D | A non-digit: [^0-9]（非数字
\s | A whitespace character: [ \t\n\x0B\f\r]（空白符
\S | A non-whitespace character: [^\s]（非空白符
\w | A word character: [a-zA-Z_0-9]（数字，字母和下划线
\W | A non-word character: [^\w]（非数字，非字母和非下划线的字符

>In the table above, each construct in the left-hand column is shorthand for the character class in the right-hand column. For example, \d means a range of digits (0-9), and \w means a word character (any lowercase letter, any uppercase letter, the underscore character, or any digit). Use the predefined classes whenever possible. They make your code easier to read and eliminate errors introduced by malformed character classes.

>Constructs beginning with a backslash are called escaped constructs. We previewed escaped constructs in the String Literals section where we mentioned the use of backslash and \Q and \E for quotation. If you are using an escaped construct within a string literal, you must precede the backslash with another backslash for the string to compile. For example:（以反斜杆'\\'开头的为转义字符
`private final String REGEX = "\\d"; // a single digit`

>In this example \d is the regular expression; the extra backslash is required for the code to compile. The test harness reads the expressions directly from the Console, however, so the extra backslash is unnecessary.
## Quantifiers

Greedy | Reluctant | Possessive | Meaning
-- | -- | -- | --
X? | X?? | X?+ | X, once or not at all（匹配0或1次
X* | X*? | X*+ | X, zero or more times（匹配0次以上
X+ | X+? | X++ | X, one or more times（匹配1此以上
X{n} | X{n}? | X{n}+ | X, exactly n times（匹配n次，n为具体数字
X{n,} | X{n,}? | X{n,}+ | X, at least n times（匹配至少n次
X{n,m} | X{n,m}? | X{n,m}+ | X, at least n but not more than m times（n-m次
### Differences Among Greedy, Reluctant, and Possessive Quantifiers

>There are subtle differences among greedy, reluctant, and possessive quantifiers.

>Greedy quantifiers are considered "greedy" because they force the matcher to read in, or eat, the entire input string prior to attempting the first match. If the first match attempt (the entire input string) fails, the matcher backs off the input string by one character and tries again, repeating the process until a match is found or there are no more characters left to back off from. Depending on the quantifier used in the expression, the last thing it will try matching against is 1 or 0 characters.

>The reluctant quantifiers, however, take the opposite approach: They start at the beginning of the input string, then reluctantly eat one character at a time looking for a match. The last thing they try is the entire input string.

>Finally, the possessive quantifiers always eat the entire input string, trying once (and only once) for a match. Unlike the greedy quantifiers, possessive quantifiers never back off, even if doing so would allow the overall match to succeed.

>To illustrate, consider the input string xfooxxxxxxfoo.

 ```
Enter your regex: .*foo  // greedy quantifier
Enter input string to search: xfooxxxxxxfoo
I found the text "xfooxxxxxxfoo" starting at index 0 and ending at index 13.

Enter your regex: .*?foo  // reluctant quantifier
Enter input string to search: xfooxxxxxxfoo
I found the text "xfoo" starting at index 0 and ending at index 4.
I found the text "xxxxxxfoo" starting at index 4 and ending at index 13.

Enter your regex: .*+foo // possessive quantifier
Enter input string to search: xfooxxxxxxfoo
No match found.
```
>The first example uses the greedy quantifier .* to find "anything", zero or more times, followed by the letters "f" "o" "o". Because the quantifier is greedy, the .* portion of the expression first eats the entire input string. At this point, the overall expression cannot succeed, because the last three letters ("f" "o" "o") have already been consumed. So the matcher slowly backs off one letter at a time until the rightmost occurrence of "foo" has been regurgitated, at which point the match succeeds and the search ends.

>The second example, however, is reluctant, so it starts by first consuming "nothing". Because "foo" doesn't appear at the beginning of the string, it's forced to swallow the first letter (an "x"), which triggers the first match at 0 and 4. Our test harness continues the process until the input string is exhausted. It finds another match at 4 and 13.

>The third example fails to find a match because the quantifier is possessive. In this case, the entire input string is consumed by .*+, leaving nothing left over to satisfy the "foo" at the end of the expression. Use a possessive quantifier for situations where you want to seize all of something without ever backing off; it will outperform the equivalent greedy quantifier in cases where the match is not immediately found.

## Boundary Matchers

Boundary Construct | Description
-- | --
^ | The beginning of a line（行开头
$ | The end of a line（行结尾
\b | A word boundary（单词边界
\B | A non-word boundary（非单词边界
\A | The beginning of the input（输入的起始位置
\G | The end of the previous match（上次匹配的结尾处
\Z | The end of the input but for the final terminator, if any
\z | The end of the input（输入的结束位置

## Embedded Flag Expressions

Constant | Equivalent Embedded Flag Expression
-- | --
Pattern.CANON_EQ | None
Pattern.CASE_INSENSITIVE | (?i)
Pattern.COMMENTS | (?x)
Pattern.MULTILINE | (?m)
Pattern.DOTALL | (?s)
Pattern.LITERAL | None
Pattern.UNICODE_CASE | (?u)
Pattern.UNIX_LINES | (?d)

笔记来源： [Regular Expressions](https://docs.oracle.com/javase/tutorial/essential/regex/index.html)



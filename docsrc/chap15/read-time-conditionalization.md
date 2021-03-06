# `*FEATURES*` and Read-Time Conditionalization（`*FEATURES*` 和读取期条件化）

Before you can implement this API in a library that will run correctly
on multiple Common Lisp implementations, I need to show you the
mechanism for writing implementation-specific code.

在能够实现这个可在多个 Common Lisp 实现上正确运行的库的
API 之前，我需要首先介绍编写实现相关代码的手法。

While most of the code you write can be "portable" in the sense that
it will run the same on any conforming Common Lisp implementation, you
may occasionally need to rely on implementation-specific functionality
or to write slightly different bits of code for different
implementations. To allow you to do so without totally destroying the
portability of your code, Common Lisp provides a mechanism, called
read-time conditionalization, that allows you to conditionally include
code based on various features such as what implementation it's being
run in.

“在任何符合标准的 Common Lisp 实现上正确运行的代码都将产生相同的行为”，从这个意义上说，尽管你所编写的多数代码都是
“可移植的”，但你可能偶尔需要依赖于实现相关的功能，或者为不同实现编写稍有差别的代码。为了使你在不完全破坏代码可移植性的情况下做到这点，Common
Lisp 提供了一种称为读取期条件化的机制，从而允许你有条件地包含基于当前运行的实现等各种特性的代码。

The mechanism consists of a variable `*FEATURES*` and two extra bits of
syntax understood by the Lisp reader. `*FEATURES*` is a list of symbols;
each symbol represents a "feature" that's present in the
implementation or on the underlying platform. These symbols are then
used in feature expressions that evaluate to true or false depending
on whether the symbols in the expression are present in
`*FEATURES*`. The simplest feature expression is a single symbol; the
expression is true if the symbol is in `*FEATURES*` and false if it
isn't. Other feature expressions are boolean expressions built out of
**NOT**, **AND**, and **OR** operators. For instance, if you wanted to
conditionalize some code to be included only if the features foo and
bar were present, you could write the feature expression (and foo
bar).

该机制由一个变量 `*FEATURES*` 和两个被 Lisp
读取器理解的附加语法构成。`*FEATURES*`
是一个符号的列表，每个符号代表存在于当前实现或底层平台的一个
“特性”。这些符号随后用在特性表达式中，根据表达式中的符号是否存在于 `*FEATURES*`
求值为成真或假。最简单的特性表达式是单个符号，当符号在 `*FEATURES*`
中时该表达式为真，否则为假。其他的特性表达式是构造在
**NOT**、**AND** 和 **OR**
操作符上的布尔表达式。例如，如果要条件化某些代码使其只有当特性 `foo`
和 `bar` 存在时才被包含，那么可以将特性表达式写成 `(and foo bar)`。

The reader uses feature expressions in conjunction with two bits of
syntax, `#+` and `#-`. When the reader sees either of these bits of
syntax, it first reads a feature expression and then evaluates it as I
just described. When a feature expression following a `#+` is true, the
reader reads the next expression normally. Otherwise it skips the next
expression, treating it as whitespace. `#-` works the same way except it
reads the form if the feature expression is false and skips it if it's
true.

读取器将特性表达式与两个语法标记 `#+` 和 `#-`
配合使用。当读取器看到任何一个这样的语法时，它首先读取特性表达式并按照我刚刚描述的方式求值。当跟在
`#+` 之后的特性表达式为真时，读取器会正常读取下一个表达式；否则它会跳过下一个表达式，将它作为空白对待。`#-`
以相同的方式工作，只是它在特性表达式为假时才读取后面的形式，而在特性表达式为真时跳过它。

The initial value of `*FEATURES*` is implementation dependent, and what
functionality is implied by the presence of any given symbol is
likewise defined by the implementation. However, all implementations
include at least one symbol that indicates what implementation it
is. For instance, Allegro Common Lisp includes the symbol `:allegro`,
CLISP includes `:clisp`, SBCL includes `:sbcl`, and CMUCL includes
`:cmu`. To avoid dependencies on packages that may or may not exist in
different implementations, the symbols in `*FEATURES*` are usually
keywords, and the reader binds `*PACKAGE*` to the **KEYWORD** package while
reading feature expressions. Thus, a name with no package
qualification will be read as a keyword symbol. So, you could write a
function that behaves slightly differently in each of the
implementations just mentioned like this:

`*FEATURES*`
的初始值是实现相关的，并且任何给定符号的存在所代表的功能也同样是由实现定义的。尽管如此，所有的实现都包含至少一个符号来指示当前是什么实现。例如，Allegro
Common Lisp 带有符号 `:allegro`，CLISP 带有 `:clisp`， SBCL
带有 `:sbcl`，而 CMUCL 带有 `:cmu`。为了避免依赖于在不同实践中可能不存在的包，`*FEATURES*`
中的符号通常是关键字，并且读取器在读取特性表达式时将 `*PACKAGE*`
绑定到 `KEYWORD`
包上。这样，不带包限定符的名字将被读取成关键字符号。因此，你可以像下面这样编写一个在前面提到的每个实现中行为稍有不同的函数：

```lisp
(defun foo ()
  #+allegro (do-one-thing)
  #+sbcl (do-another-thing)
  #+clisp (something-else)
  #+cmu (yet-another-version)
  #-(or allegro sbcl clisp cmu) (error "Not implemented"))
```

In Allegro that code will be read as if it had been written like this:

在 Allegro 中读取上述代码，会好像代码原本就写成下面这样：

```lisp
(defun foo ()
  (do-one-thing))
```

while in SBCL the reader will read this:

而在 SBCL 中读取器将读到下面的内容：

```lisp
(defun foo ()
  (do-another-thing))
```

while in an implementation other than one of the ones specifically
conditionalized, it will read this:

而在一个不属于上述特定条件化实现的平台上，它将被读取成下面这样：

```lisp
(defun foo ()
  (error "Not implemented"))
```

Because the conditionalization happens in the reader, the compiler
doesn't even see expressions that are skipped. This means you pay no
runtime cost for having different versions for different
implementations. Also, when the reader skips conditionalized
expressions, it doesn't bother interning symbols, so the skipped
expressions can safely contain symbols from packages that may not
exist in other implementations.

因为条件化过程发生在读取器中，编译器根本无法看到被跳过的表达式， 这意味着你不会为不同实现的不同版本付出任何运行时代价。另外，当读取器跳过条件化的表达式时，它不会
intern 其中的符号，因此被跳过的表达式可以安全地包含在其他实现中可能不存在的包中的符号。

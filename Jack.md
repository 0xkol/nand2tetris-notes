# Nand2Tetris - Jack (Chapters 9, 10, 11)

This document describes the *Jack* programming language, as well as the process of compiling *Jack* to VM language.

- [Nand2Tetris - Jack (Chapters 9, 10, 11)](#nand2tetris---jack-chapters-9-10-11)
- [Jack Language Grammar](#jack-language-grammar)
    - [Lexical Elements](#lexical-elements)
    - [Program Structure](#program-structure)
    - [Statements](#statements)
    - [Expressions](#expressions)
- [Compilation](#compilation)
    - [Symbol Table](#symbol-table)
    - [Translating Expressions](#translating-expressions)
    - [Flow of Control](#flow-of-control)
    - [Object Construction](#object-construction)
    - [Object Manipulation](#object-manipulation)
- [Standard Jack Mapping Over the VM](#standard-jack-mapping-over-the-vm)
    - [Files and Subroutines](#files-and-subroutines)
    - [Variables](#variables)
    - [Compiling Subroutines](#compiling-subroutines)
    - [Special OS Services](#special-os-services)

# Jack Language Grammar

## Lexical Elements

The Jack language includes five types of terminal elements (tokens):
- `keyword`: '**class**' | '**constructor**' | '**function**' | '**method**' | '**field**' | '**static**' | '**var**' | '**int**' | '**char**' | '**boolean**' | '**void**' | '**true**' | '**false**' | '**null**' | '**this**' | '**let**' | '**do**' | '**if**' | '**else**' | '**while**' | '**return**'
- `symbol`: '**{**' | '**}**' | '**(**' | '**)**' | '**[**' | '**]**' | '**.**' | '**,**' | '**;**' | '**+**' | '**-**' | '**\***' | '**/**' | '**&**' | '**|**' | '**<**' | '**>**' | '**=**' | '**~**'
- `integerConstant`: A decimal number in the range 0..32767
- `stringConstant`: '**"**' A sequence of Unicode characters not including double quote or newline '**"**'
- `identifier`: A sequence of letters, digits, and underscore ('_') not starting with a digit.

## Program Structure

A Jack program is a collection of classes, each appearing in a separate file. The compilation unit is a class. A class is a sequence of tokens structured according to the following context free grammar:
- `class`: '**class**' `className` '**{**' `classVarDec`\* `subroutineDec`\* '**}**'
- `classVarDec`: ('**field**' | '**static**') `type` `identifier` ('**,**' `identifier`)\* '**;**'
- `subroutineDec`: ('**constructor**' | '**method**' | '**function**') ('**void**' | `type`) `identifier` '**(**' `parameterList` '**)**' `subroutineBody`
- `%type%`: '**int**' | '**char**' | '**boolean**' | `identifier`
- `parameterList`: (`type` `identifier` ('**,**' `type` `identifier`)\*)?
- `subroutineBody`: '**{**' `varDec`\* `statements` '**}**'
- `varDec`: '**var**' `type` `identifier` ('**,**' `identifier`)\* '**;**'

## Statements

- `statements`: (`letStatement` | `ifStatement` | `whileStatement` | `doStatement` | `returnStatement`)\*
- `letStatement`: '**let**' `identifier`('**[**' `expression` '**]**')? '**=**' `expression` '**;**'
- `ifStatement`: '**if**' '**(**' `expression` '**)**' '**{**' `statements` '**}**' ('**else**' '**{**' `statements` '**}**')?
- `whileStatement`: '**while**' '**(**' `expression` '**)**' '**{**' `statements` '**}**'
- `doStatement`: '**do**' `subroutineCall` '**;**'
- `returnStatement`: '**return**' `expression` '**;**'

## Expressions

- `expression`: `term` (`op` `term`)\*
- `term`: `integerConstant` | `stringConstant` | `keywordConstant` | `identifier` | `identifier` '**[**' `expression` '**]**' | `subroutineCall` | `unaryOp` `term` | '**(**' `expression` '**)**'
- `%subroutineCall%`: `identifier` '**(**' `expressionList` '**)**' | `identifier` '**.**' `identifier` '**(**' `expressionList` '**)**'
- `expressionList`: (`expression` ('**,**' `expression`)\*)?
- `%op%`: '**+**' | '**-**' | '**\***' | '**/**' | '**&**' | '**|**' | '**<**' | '**>**' | '**=**'
- `%unaryOp%`: '**-**' | '**~**'
- `%keywordConstant%`: '**true**' | '**false**' | '**null**' | '**this**'

> Note: the non-terminals that are surrounded by '%' are simply for convenience of grammar reader/writer and does not appear in the exported XML of the syntax analyzer.

# Compilation

## Symbol Table

**Symbol table** is data structure created and maintained by compilers in order to store information about the occurrence of identifiers in the program.

For Jack compilation we construct a two symbol tables:
- Class-level symbol table
- Subroutine-level symbol table

The structure of the symbol table (including example):

| id | type | kind  | number |
|:---|:-----|:------|:-------|
| x  | int  | local | 0      |

When building a symbol table for a '**method**' subroutine, the symbol table is first populated with:

| id   | type        | kind     | number |
|:-----|:------------|:---------|:-------|
| this | *className* | argument | 0      |

Symbol table is populated when the following productions are matched by the parser:
- `classVarDec` - for static and fields
- `varDec` - for local variables
- `subroutineDec` - for arguments

When an `identifier` is encountered else-where in the program, the symbol table is consulted, in the following order:
1. **subroutine-level** symbol table
2. **class-level** symbol table
3. If not found, throw an error

## Translating Expressions

The *Jack* grammar specifies:

`expression`: `term` (`op` `term`)\*

In translation, I found it useful to transform this production to:

`expression`: `term` (`op` `expression`)?

Now it is really easy to translate expression:

1. emit code for `term`
2. If exists, emit code for `expression`
3. If exists, emit code for `op`

### Translate `term`

- For **integer constant** who's value is `n`, emit `push constant n`
- For **string constant** emit:
  ```
  push constant <str-length>
  call String.new 1
  push constant <first-char>
  call String.appendChar 1

  /* This works because "call String.new" leaves the newly created string object onto the stack, and String.appendChar is already a "method" subroutine that requires that the first argument will be the object that is manipulated */

  ...

  push constant <last-char>
  call String.appendChar 1

  /* This works because String.appendChar leaves on the stack a reference to the string object that was processed */
  ```
- For **keyword constant** use the following mapping:
    - "this" :-> `push pointer 0`
    - "true" :-> `push constant 0` followed by `not` (essetially push -1).
    - "false", "null" :-> `push constant 0`
- For an **identifier**, just emit: `push <varKind> <varNum>`
- For **unary term**, emit the `term` followed by the unary operator.
- For **nested expression** (expression surrounded by parentheses), emit the expression.
- For **subroutine call**:
    - `id(...)`
        - This term is allowed only inside a `method` body.
        - We emit `push pointer 0` to push the currently processed object.
        - We find the enclosed class name and invoke the method specifying number of arguments as defined + 1 (for the object itself).
    - `id1.id2(...)`:
        - if `id1` is *found on the symbol table*, we assume it is a variable pointing to an initialized object, and thus we (1) retrieve it's class type (2) push `id1` onto the stack (3) call `id2` method with number of arguments defined + 1 (for the object itself).
        - if `id2` is *not found on the symbol table*, we assume it is a class name, and `id2` is treated as `function` or `constructor`. (1) `id1` is served as class name (2) push arguments as is (3) call `id2` with the number of arguments defined.
- For **array expression** of the form `arr[expr]`:
  ```
  push expr // emit code which pushes "expr" onto the stack
  push arr
  add
  pop pointer 1 // anchor the THAT segment
  push that 0   // push the first value of the THAT segment
  ```

## Flow of Control

### If statement

```
+-------------------+                   +---------------------------+ 
| if (expression) { |                   | // emit expression        |
|     statements1   |    compilation    | if-goto IF_TRUE           |
| } else {          | +---------------> |                           |
|     statements2   |                   | // emit statements2       |
| }                 |                   | goto IF_END               |
+-------------------+                   |                           |
                                        | label IF_TRUE             |
                                        | // emit statements1       |
                                        |                           |
                                        | label IF_END              |
                                        | // rest of the code here  |
                                        +---------------------------+
                                                
```

Example (Assuming `x` is the first local variable):

```
+-------------------+                   +-------------------------------------------------+ 
| if (x > 0) {      |                   | push local 0                                    |
|     return x;     |    compilation    | push constant 0                                 |
| } else {          | +---------------> | gt                                              |
|     return -x;    |                   | if-goto IF_TRUE     // if (x > 0) goto IF_TRUE  |
| }                 |                   |                                                 |
+-------------------+                   | push local 0                                    |
                                        | neg                                             |
                                        | return              // return -x                |
                                        | goto IF_END                                     |
                                        |                                                 |
                                        | label IF_TRUE                                   |
                                        | push local 0                                    |
                                        | return              // return x                 |
                                        |                                                 |
                                        | label IF_END                                    |
                                        | // rest of the code goes here                   |
                                        +-------------------------------------------------+
```

### While Statement

```
+---------------------+                 +--------------------------------+
| while(expression) { |  compilation    | label WHILE_EXPR               |
|     statements      | +------------>  | // emit code for expression    |
| }                   |                 | not                            |
+---------------------+                 | if-goto WHILE_END              |
                                        |                                |
                                        | // emit statements here        |
                                        |                                |
                                        | goto WHILE_EXPR                |
                                        |                                |
                                        | label WHILE_END                |
                                        | // rest of code goes here      |
                                        +--------------------------------+
```

Example:

```
+---------------------+                 +-------------------------------+
| while (x > 0) {     |  compilation    | label WHILE_EXPR              |
|     let x = x - 1;  | +------------>  | push local 0                  |
| }                   |                 | push constant 0               |
+---------------------+                 | gt                            |
                                        | not                           |
                                        | if-goto WHILE_END             |
                                        |                               |
                                        | push local 0                  |
                                        | push constant 1               |
                                        | sub                           |
                                        | pop local 0                   |
                                        |                               |
                                        | goto WHILE_EXPR               |
                                        |                               |
                                        | label WHILE_END               |
                                        | // rest of code goes here     |
                                        +-------------------------------+
```

### Void Methods

Consider the following subroutine call:

```
do Output.printInt(0);
```

We translate the subroutine call as explained in [Translating Expressions](#translating-expressions), but the return value should be tossed away. Therefore we write after the subroutine call evaluation:
```
// ...code that handles subroutine call evaluation

pop temp 0

// rest of the code goes here...
```

### Array Assignment

We want to translate an expression of the form:
```
let arr[expr1] = expr2;
```

Note that `expr2` can be complex and include another array access term.

The translation is as follows:
```
// Compute the address that we should modify
push expr1
push arr
add

// Compute the expression that we need to evaluate and store the result in temp. var
push expr2
pop temp 0

// The top of the stack contain the location of the memory cell we need to write to
pop pointer 1 // anchor THAT segment
push temp 0
pop that 0
```

Another strategy, without using the "temp" segment:
```
push expr2

push expr1
push arr
add

pop pointer 1

pop that 0
```

## Object Construction

Let `Foo` be a class with the following prototype:
```
class Foo {
    // field and static declarations

    constructor Foo new(...) { ... }

    // methods declarations
}
```

A programmer who wants to use a `Foo` object typically must initialize it:
```
var Foo f;
let f = Foo.new(...);
```

The contract is:
- The constructor should **allocate** space for all the fields of the class `Foo`.
- The constructor should **return** the base address of the allocated memory.

With that in mind, the code segment above is handled as if it was ordinary subroutine call.

A definition of a constructor should be translated to VM code as follows:
```
// Count the number of fields declared and allocate enough memory for them
push constant <fields-num>
call Memory.alloc 1

// Set the THIS pointer to the newly allocated base address
pop pointer 0

// Rest of constructor code goes here

// At some point, a "return this;" statement must occur (part of the contract)
push pointer 0
return
```

## Object Manipulation

The first argument sent to a *Jack* `method` is the object which we want to manipulate.

The first operation in the translated method should be anchoring the THIS pointer:
```
push argument 0 // This first argument is the object base address
pop pointer 0   // We anchor the THIS segment to the current object

// rest of method code goes here
```

# Standard Jack Mapping Over the VM

## Files and Subroutines

- Each file `filename.jack` is compiled into a VM file `filename.vm`
- Each subroutine `subName` in file `filename.jack` is compiled into a VM function `filename.subName`
- A Jack `constructor` or `function` with *k* arguments is compiled into a VM function that operates on *k* arguments.
- A Jack `method` with *k* arguments is compiled into a VM function that operates on *k+1* arguments (the first argument is the object which need to be manipulated).

## Variables

- **Local variables** - mapped on the virtual segment `local`.
- **Argument variables** - mapped on the virtual segment `argument`.
- **Static variables** - mapped on the virtual memory segment `static` of the compiled `.vm` class file.
- **Field variables**:
    - `pointer 0` has been anchored to the current object's base address at the beginning of the method execution (this stage is performed automatically by the compiler)
    - the *i*-th field of this object is mapped to `this i`

## Compiling Subroutines

- When compiling a Jack method, the compiled VM code must set the base of the `this` segment to `argument 0`.
- When compiling a Jack constructor, the compiled VM code must allocate a memory block for the new object, and then set the base of the segment `this` to the new object's base address. Moreover, the compiled VM code must return the object's base address to the caller.
- When compiling a void method, the compiled VM code must return the value `constant 0`.

## Special OS Services

- Multiplication is handled using the OS function `Math.multiply(x, y)`
- Division is handled using the OS function `Math.divide(x, y)`
- String constants are created using the OS constructor `String.new(length)`
- String assignments like x="str.." are handled using a series of calls to `String.appendChar(c)`
- Object construction requires allocating space for the new object using the OS function `Memory.alloc(size)`
- Object destruction is handled by the OS function `Memory.deAlloc(o)`, usually wrapped inside a `dispose` method.
# APLang - VM Specs

Index:

[TOC]

## Modules and Packages

## Class File

## Bytecode and Operations

### Single Operand

#### Code Flow

CALL, RETURN, IF, GOTO
PPPP MXXX XXXX XXXX
P: Instruction Id
M: Virtual(0)/Static(1)
X: Index in the Reference Pool
Stack:
  Virtual:
    ..., object ref, [arg1, [arg2, ...]] -> ...
  Static:
    ..., [arg1, [arg2, ...]] -> ...


#### Unary and Conversion

PPPP AABB 0000 0000

00 - Signed Integer
01 - Unsigned Integer
10 - Floating Point
11 - String

Stack:
  ..., X -> ..., Z

#### Math, Comparations and Equality

PPPP XXXX AABB 0000
Stack:
  ..., A, B -> ..., C

X can have the following values:

```
0000 OP_BIN : +
0001 OP_BIN : -
0010 OP_BIN : *
0011 OP_BIN : **
0100 OP_BIN : /
0101 OP_BIN : %

0110 OP_BIN : |
0111 OP_BIN : &
1000 OP_BIN : ^
1001 OP_BIN : >>
1010 OP_BIN : <<
1011 OP_BIN : >>>

1100 OP_LOG : ||
1101 OP_LOG : &&

1110 OP_EQ  : ==
1111 OP_EQ  : !=
```

#### Registers/Local Variables and Fields

LOAD, STORE

PPPP MFXX XXXX XXXX
P: Instruction Id
M: Mode : Load(0)/Store(1)
F: Source/Destination: Local Variable/Register(0) or Field(1)
X:
  When Local Var: Index in Register Pool
  When Field: Index in the Reference Pool


#### Stack

PUSH, POP, DUP, SWAP

#### Miscellaneous

NOP, CONSTANT

### Overview

CALL, RETURN, IF, GOTO
CONV
MATH
GET, PUT
LOAD, STORE
PUSH, POP, DUP, SWAP
NOP, CONSTANT

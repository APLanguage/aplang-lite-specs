# APLang - VM Specs

Index:

[TOC]

## Modules and Packages

## Class File

```
File {
  u4             magic;
  u2             constant_pool_count;
  cp_info        constant_pool[constant_pool_count];
  u2             reference_pool_count;
  ref_info       reference_pool[reference_pool_count];
  u2             fields_count;
  u2             methods_count;
  u2             classes_count;
  field_info     fields[fields_count];
  method_info    methods[methods_count];
  class_info     classes[classes_count];
}
```
```
class_info {
  u1             name_len;
  char           name[name_len];
  u2             super_count;
  u2             fields_count;
  u2             methods_count;
  u2             classes_count;
  class_ref      supers[super_count];
  field_info     fields[fields_count];
  method_info    methods[methods_count];
  class_info     classes[classes_count];
}
```
```
field_info {
  u1             name_len;
  char           name[name_len];
  type           type;
  u4             code_length;
  u1             code[code_length]
}
```
```
method_info {
  u1             name_len
  char           name[name_len]
  type           return_type;
  u1             parameter_count;
  type           parameters[parameter_count];
  u4             code_length;
  u1             code[code_length]
}
```

## Bytecode and Operations

### Code Flow

#### CALL
PPPP MDWW XXXX XXXX
P: Instruction Id
M: Virtual(0)/Static(1)
X: Index in the Reference Pool
WW: Amount of next bytes which be also used for index (up to 3, total up to 4).
D: If result will be ignored/poped.
Stack:
  Virtual:
    ..., object ref, [arg1, [arg2, ...]] -> ...
  Static:
    ..., [arg1, [arg2, ...]] -> ...  

#### RETURN

#### IF

PPPP CCCW XXXX XXXX
P: Instruction Id
C: Condition
X: Signed 8bit byte offset
W: If set, next byte is 
```
001 ==
010 !=
011 >
100 <
101 >=
110 <=
111 boolean
```
> Note: If pops the value(s).

#### GOTO

PPPP WXXX XXXX XXXX
P: Instruction Id
X: Signed 12bit byte offset

### Unary and Conversion

PPPP M--- ---- ----
P: Instruction Id
M: Mode: Unary(0)/Conversion(1)
Unary:
  PPPP 0CSD ---- ----
  C: Operation
    0 - Negation
    1 - Bitwise Negation
  S: Source: Stack(0)/LocVar(1)
  D: Destination: Stack(0)/LocVar(1)
  W: Wide, if set 1 an next byte will determine the index of the LocVar
  S == 0 && D == 0:
    PPPP 0C00 ---- ----
  S != D:
    PPPP 0CSD WXXX XXXX
    X: Index of the LocVar
  S == 1 && D == 1:
    PPPP 0C11 WXXX WXXX
    A: Index of the Source LocVar
    B: Index of the Destination LocVar
    If W set for both, first next byte is for source and second next byte will be for destination

> If S is stack, then the operand will be popped from the stack, if D is stack, the result will be pushed onto the stack.

Conversion:
  PPPP 1AAA BBBT WXXX
  A/B:
    000 - Signed byte
    001 - Unsigned byte
    010 - Signed Integer
    011 - Unsigned Integer
    100 - Signed Big Int
    101 - Unsigned Big Int
    110 - Floating Point
    111 - Big Floating Point
  T: Target: Stack(0)/LocVar(1)
  If T set, W determines if the next byte is used for the index or XXX

### Math, Comparations and Equality

PPPP XXXX XXAB TCCC
A: First operand: Stack(0)/LocVar(1)
B: Second operand: Stack(0)/LocVar(1)
T: Target: Stack(0)/LocVar(1)
The Index byte is next in the byte stream if bit set, in the order A, B then T.
C: Type (both operands must be same type): Same as A/B in Conversion, above.

X can have the following values:

```
00000 OP_MTH : +
00001 OP_MTH : -
00010 OP_MTH : *
00011 OP_MTH : **
00100 OP_MTH : /
00101 OP_MTH : %

01000 OP_BIT : |
01001 OP_BIT : &
01010 OP_BIT : ^
01011 OP_BIT : >>
01100 OP_BIT : <<
01101 OP_BIT : >>>

10000 OP_LOG : ||
10001 OP_LOG : &&
10010 OP_EQ  : ==
10011 OP_EQ  : !=

11000 OP_CMP : <
11001 OP_CMP : <=
11010 OP_CMP : >
11011 OP_CMP : >=
```

### Registers/Local Variables and Fields

LOAD, STORE

PPPP MTWW XXXX XXXX
P: Instruction Id
M: Mode : Load(0)/Store(1)
T: Source/Destination: Local Variable/Register(0) or Reference Pool(1)
WW: Amount of next bytes which be also used for index (up to 3, total up to 4).
X:
  When Local Var: Index in Register Pool
  When Field: Index in the Reference Pool


### Stack

PUSH, POP, DUP, SWAP

POP:
  PPPP AABB
  P: Instruction Id
  A: First stack entry to pop
  B: Second stack entry to pop
  Note: To pop only 1 entry → A == B

PUSH (for primitive Types, otherwise use LOAD):
  PPPP DXXX
  PPPP: Instruction Id
  D: If to push twice
  XXX: Type
    000 - Signed byte
    001 - Unsigned byte
    010 - Signed Integer
    011 - Unsigned Integer
    100 - Signed Big Int
    101 - Unsigned Big Int
    110 - Floating Point
    111 - Big Floating Point
  Next bytes in stream represent the value for this type.

DUP:

### Miscellaneous

NOP
PPPP ----
----: Should be ignored by the VM: debuggers or tools can use these bits for general purpose.

## Overview

NOP
0000
CALL, RETURN, IF,   GOTO
0001  0010    0011  0100
CONV
0101
MATH
0110
LOAD, STORE
0111  1000
PUSH, POP, DUP, SWAP
1001  1010 1011 1100

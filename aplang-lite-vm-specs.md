# APLang - VM Specs

Index:

[TOC]

## Modules and Packages

## VM
  - 2^8  (256)    Registers per Frame
  - 2^16 (65,536) Bytes per Code block (method/field init)

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
```
type {

}
```

## Bytecode and Operations

### Code Flow

#### CALL (Methods & Fields)
PPPP MDW- XXXX XXXX
P: Instruction Id
-: General purpose bit, VM must ignore them.
M: Virtual(0)/Static(1)
X: Index in the Reference Pool
W: Amount of next byte extends the index to 16-bit using next byte.
D: If result will be ignored/poped.
Stack:
  Virtual:
    ..., object ref, [arg1, [arg2, ...]] -> ..., [value]
  Static:
    ..., [arg1, [arg2, ...]] -> ..., [value]

#### RETURN
PPPP T--- XXXX XXXX
-: General purpose bits, VM must ignore them.
If the function returns a value:
  T: Stack(0)/Register(1)
  X: If T set, this is the index of the register to return.
If T == 0 or function has no return value, X can be used for general purpose and must be ignored by the VM.

#### IF

PPPP CCCW XXXX XXXX
P: Instruction Id
C: Condition
X: Signed 8-16bit location.
W: If set, next byte is 
```
1 0 -1
0 0 0 = NOP
0 0 1 = <
0 1 0 = ==
0 1 1 = <=
1 0 0 = >
1 0 1 = !=
1 1 0 = >=
1 1 1 = unconditional jump
```
> Note: If pops the value.

### Unary

#### Inversion
PPPP M--- ---- ----
P: Instruction Id
M: Mode: Unary(0)/Conversion(1)
PPPP 0SD- ---- ----
S: Source: Stack(0)/LocVar(1)
D: Destination: Stack(0)/LocVar(1)
S == 0 && D == 0:
  PPPP 000-
S != D:
  PPPP 0SDW XXXX XXXX
  X: Index of the LocVar
  W: Wide, if set, the next byte will extends the Index to 16bit index.
S == 1 && D == 1:
  PPPP 011W XXXX WXXX
  A: Index of the Source LocVar
  B: Index of the Destination LocVar
  If W set for both, first next byte is for source and second next byte will be for destination

> If S is stack, then the operand will be popped from the stack, if D is stack, the result will be pushed onto the stack.

#### Conversion

PPPP AAAT BBBW XXXX
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
If T set, W determines if the index is extended to 12 bit

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

PPPP |  
-----------------------
0000 | ----           NOP
0001 | MDWW XXXX XXXX CALL
0010 | ---- ---- ---- RETU
0011 | --WW XXXX XXXX GOTO
0100 | M--- ---- ---- INV
0101 | AAAT BBBW XXXX CONV
0110 | XXXX XXAB TCCC MATH
0111 | MTWW XXXX XXXX LORE
1000 | DXXX           PUSH
1001 | AABB           POP
1010 | ----           DUP
1011 | ----           SWAP


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
  u1             info;
 [ 
  u4             code_length;
  u1             code[code_length]
  
  // or
  
  u4             index; // Constant Pool
  
  // or
  
  u1, u4, u8,
  i1, i4, i8,
  float, double  value;
 ]
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
  u1             type_id; 
  [ // if object
    u1             path_len
    char           path[path_len]   
  ]
}
```
type_id:
  0 - Signed byte
  1 - Unsigned byte
  2 - Signed Integer
  3 - Unsigned Integer
  4 - Signed Big Int
  5 - Unsigned Big Int
  6 - Floating Point
  7 - Big Floating Point
  8 - Boolean
  9 - Object
```
ref_info {
  u1               ref_id;
  u1               name_len
  char             name[name_len]
  [ u2             class_ref; // when class-attribute or classmethod ]
  [ // when method
    type           return_type;
    u1             parameter_count;
    type           parameters[parameter_count];
  ]
}
```
ref_id:
  0 - classpath
  1 - class-attribute
  2 - class-method
  3 - global-field
  4 - global-method
```
cp_info {
  u1               constant_type;
  u4               bytes_len;
  u1               bytes[bytes_len];
}
```
constant_type:
  0 - string
  1 - encoded object

## Bytecode and Operations

### Code Flow

#### CALL (Methods & Fields)
PPPP SDW- XXXX XXXX
P: Instruction Id
-: General purpose bit, VM must ignore them.
S: Virtual(0)/Static(1)
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
W: If set, next byte expands the index from 8 to 16 bit.
```
1 0 -1
0 0 0 = NOP // TODO: find purpose
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
PPPP T-DD XXXX XXXX
P: Instruction Id
T: Target: Stack(0)/Register(1)
DD:
  00 - byte
  01 - int
  10 - big int
  11 - boolean // a boolean is a byte which has either 1 (true) or 0 (false) as value.

#### Conversion

PPPP TAAA BBB- ----
A/B:
  000 - Signed byte
  001 - Unsigned byte
  010 - Signed Integer
  011 - Unsigned Integer
  100 - Signed Big Int
  101 - Unsigned Big Int
  110 - Floating Point
  111 - Big Floating Point
T: Target: Stack(0)/Register(1)

### Math, Comparations and Equality

PPPP XXXX XXAB TCCC
A: First operand: Stack(0)/LocVar(1)
B: Second operand: Stack(0)/LocVar(1)
T: Target: Stack(0)/LocVar(1)
The Index byte is next in the byte stream if bit set, in the order A, B then T.
C: Type (both operands must be same type): Same as A/B in Conversion, above.

X can have the following values:

```
// can be applied to ints & floats
00000 OP_MTH : +
00001 OP_MTH : -
00010 OP_MTH : *
00011 OP_MTH : **
00100 OP_MTH : /
// can only be applied to ints
00101 OP_MTH : %
01011 OP_BIT : >>
01100 OP_BIT : <<
01101 OP_BIT : >>>
// can be applied to ints
01000 OP_BIT : |
01001 OP_BIT : &
01010 OP_BIT : ^
// can only be applied to ubyte (boolean are ubytes)
10000 OP_LOG : ||
10001 OP_LOG : &&
10010 OP_EQ  : ==
10011 OP_EQ  : !=
// can be applied to ints & floats
11000 OP_CMP : <
11001 OP_CMP : <=
11010 OP_CMP : >
11011 OP_CMP : >=
```

### Load from/Store to Register

LOAD, STORE

PPPP M--- XXXX XXXX
P: Instruction Id
M: Mode : Load(0)/Store(1)
X: Index
Load: Takes from the register and push on the stack.
Store: Takes from the stack and puts into a register

### Stack

PUSH, POP, DUP, SWAP

POP:
  PPPP XXXX
  P: Instruction Id
  XXXX + 1: Bytes to pop from stack.

PUSH (for primitive Types, otherwise use LOAD):
  PPPP XXXX
  PPPP: Instruction Id
  D: If to push twice
  XXX + 1: Bytes to push on the stack. 
  Next bytes will be taken.

DUP:
  PPPP XXXX
  PPPP: Instruction Id
  XXXX + 1: Bytes to dup from the stack.

  ..., bytes[XXXX + 1] -> ..., bytes[XXXX + 1], bytes[XXXX + 1]

SWAP:
  PPPP XXXX
  PPPP: Instruction Id
  XXXX + 1: Bytes to swap in stack.

  ..., a_bytes[XXXX + 1], b_bytes[XXXX + 1] -> ..., b_bytes[XXXX + 1], a_bytes[XXXX + 1]

### Miscellaneous

NOP
PPPP ----
----: Should be ignored by the VM: debuggers or tools can use these bits for general purpose.

## Overview

PPPP |  
-----------------------
0000 | ----           NOP
0001 | SDW- XXXX XXXX CALL
0010 | ---- ---- ---- RETU
0011 | CCCW XXXX XXXX IF
0100 | T-DD XXXX XXXX INV
0101 | AAAT BBBW XXXX CONV
0110 | XXXX XXAB TCCC MATH
0111 | M--- XXXX XXXX LORE
1000 | XXXX           PUSH
1001 | XXXX           POP
1010 | XXXX           DUP
1011 | XXXX           SWAP


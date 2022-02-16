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
  field_info     fields[fields_count];
  u2             methods_count;
  method_info    methods[methods_count];
  u2             classes_count;
  class_info     classes[classes_count];
}
```
```
ref_info {
  u1             refinfobyte;
  [u2            parent;]
  u1             name_len;
  char           name[name_len];
  [
    Package {}
    Class {}
    Field {
      [u2        type_ref;]
    }
    Method{
      [u2        ret_type_ref;]
      u1         parameter_count;
      u1-3         parameters[parameter_count];
    }
  ]
}
```
```
refinfobyte DDDD RPTT {
  DDDD 4type (Field: Type, Method: Return type)
  R {
    0 - Package
    0 - Class
    1 - Field
    0/1 - Method
  }
  P {
    0 - No Parent
    1 - Has Parent
  }
  TT {
    00 - Package
    01 - Class
    10 - Field
    11 - Method
  }
}
```
```
class_info {
  u1             name_len;
  char           name[name_len];
  u1             super_count;
  u2             supers[super_count];
  u1             fields_count;
  field_info     fields[fields_count];
  u1             methods_count;
  method_info    methods[methods_count];
  u1             classes_count;
  class_info     classes[classes_count];
}
```
```
finfobyte DDDD --VV {
  DDDD - 4type
  --
  VV [
    00 - code
    01 - constant_pool_index
    10 - direct value
  ]
}

field_info {
  u1             finfobyte;
  u1             name_len;
  char           name[name_len];
  [u2            type]
 [ 
  u4             code_length;
  u1             code[code_length]
  
  // or
  
  u2             index; // Constant Pool
  
  // or
  
  u1, u4, u8,
  i1, i4, i8,
  float, double  value;
 ]
}
```
```
minfobyte DDDD R--- {
  DDDD - 4type
  R { has return }
  ---
}

method_info {
  u1             minfobyte;
  u1             name_len
  char           name[name_len]
  [u2            return_type;]
  u1             parameter_count;
  u1-3           parameters[parameter_count];
  u4             code_length;
  u1             code[code_length]
}
```

```
4type [ 
  0000 - objref
  0001 - i8
  0010 - i16
  0011 - i32
  0100 - i64
  0101 - f32
  0110 - f64
]
```
```
cp_info {
  u1               constant_type;
  [u4               bytes_len;]
  u1               bytes[bytes_len];
}
constant_type:
  000 - int 8 bit
  001 - int 16 bit
  010 - int 32 bit
  011 - int 64 bit
  100 - float
  101 - double
  110 - UTF8
```

## Bytecode and Operations

### Code Flow

#### CALL
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

PPPP WCCC XXXX XXXX
P: Instruction Id
C: Condition
X: Signed 8-16bit offset.
W: If set, next byte expands the index from 8 to 16 bit.
```
1 0 -1
0 0 0 = unconditional absolute jump
0 0 1 = <
0 1 0 = ==
0 1 1 = <=
1 0 0 = >
1 0 1 = !=
1 1 0 = >=
1 1 1 = unconditional relative jump
```
> Note: If pops the value.

### Unary

#### Inversion, Negation
PPPP MDDD
P: Instruction Id
M: Mode: Inversion(0)/Negation(1)
X: Register-Index
DDD:
  000 - 8 bit int
  001 - 16 bit int
  010 - 32 bit int
  011 - 64 bit int
  100 - 32 bit float
  101 - 64 bit float
  110 - boolean // a boolean is a byte which has either 1 (true) or 0 (false) as value, effectivly (~b & 0b1).

#### Conversion

PPPP T--- AAAA BBBB
A/B:
  0000 - U8
  0001 - U16
  0010 - U32
  0011 - U64
  0100 - I8
  0101 - I16
  0110 - I32
  0111 - I64
  1000 - Float
  1001 - Double
T: Target: Stack(0)/Register(1)
If T set, next byte will be the register index

### Math, Comparations and Equality
PPPP ABTC CCCX XXXX
A: First operand: Stack(0)/LocVar(1)
B: Second operand: Stack(0)/LocVar(1)
T: Target: Stack(0)/LocVar(1)
The Index byte is next in the byte stream if bit set, in the order A, B then T.
C: Type (both operands must be same type):
  0000 - U8
  0001 - U16
  0010 - U32
  0011 - U64
  0100 - I8
  0101 - I16
  0110 - I32
  0111 - I64
  1000 - Float
  1001 - Double

X can have the following values:

```
// can only be applied to ints & floats
00000 OP_MTH : +
00001 OP_MTH : -
00010 OP_MTH : *
00011 OP_MTH : **
00100 OP_MTH : /
00101 OP-MTH : //
// can only be applied to ints
00110 OP_MTH : %
01000 OP_BIT : |
01001 OP_BIT : &
01010 OP_BIT : ^
01011 OP_BIT : >>
01100 OP_BIT : >>>
01101 OP_BIT : <<
// can only be applied to ubyte (boolean are ubytes)
10000 OP_LOG : ||
10001 OP_LOG : &&
10010 OP_EQ  : ==
10011 OP_EQ  : !=
// can only be applied to ints & floats
11000 OP_CMP : <
11001 OP_CMP : <=
11010 OP_CMP : >
11011 OP_CMP : >=
11100 OP_CMP : <=>
```

### Load from/Store to Register

LOAD, STORE

PPPP M--- XXXX XXXX
P: Instruction Id
M: Mode : Load(0)/Store(1)
X: Index
Load: Takes from the register and push on the stack.
Store: Takes from the stack and puts into a register

### Get from/Put to field

GET, PUT

PPPP MWRV XXXX XXXX
P: Instruction Id
M: Mode : Get(0)/Put(1)
W: If set, next byte expands the index from 8 to 16 bit.
R: Target: Stack(0)/Register(1), if set, next byte (after index) decide register index.
V: Reversed: If this Field is virtual and this is a PUT, the obj ref will be poped first, then the actual value
X: Index
Load: Takes from the field and push on the stack/register.
Store: Takes from the stack/register and puts into the field.

### Stack

POP, DUP, SWAP

POP:
  PPPP XXXX
  P: Instruction Id
  XXXX + 1: entries to pop from stack.

DUP:
  PPPP AABB
  PPPP: Instruction Id
  AA + 1: Entries to dup from the top of the stack.
  BB: How deep under the top. (0 -> dup on top)

SWAP:
  PPPP ----
  PPPP: Instruction Id

### Miscellaneous

NOP
PPPP ----
----: Should be ignored by the VM: debuggers or tools can use these bits for general purpose.

DICT : Direct Int Constant
PPPP TTDD
TT - Target: Stack(0)/Register(1)/Field(2)/Field-Wide(3)
DD -
  00 - 8 bit
  01 - 16 bit
  10 - 32 bit
  11 - 64 bit

DFCT : Direct Float Constant
PPPP TT-D
TT - Target: Stack(0)/Register(1)/Field(2)/Field-Wide(3)
D -
  0 - 32 bit float
  1 - 64 bit float

ICST : Indirect Constant - pushes the value in the constant pool onto the stack
PPPP TT-W
TT - Target: Stack(0)/Register(1)/Field(2)/Field-Wide(3)
W - one byte more for the index

OOP
PPPP W-AA XXXX XXXX
A: Cast(0)/Check(1)/CheckNot(2)
W: Wide: If it should be a 16bit index

On check, it pushes the result on the stack

## Overview  

PPPP |  
-----------------------  
0000 | ----           NOP   
0001 | SDW- XXXX XXXX CALL  
0010 | T--- XXXX XXXX RETU  
0011 | WCCC XXXX XXXX IF    
0100 | T-DD           INV   
0101 | T--- AAAA BBBB CONV  
0110 | ABTC CCCX XXXX MATH  
0111 | M--- XXXX XXXX LORE  
1000 | MWRV XXXX XXXX GUT   
1001 | W--A XXXX XXXX OOP   
1010 | XXXX           POP   
1011 | AABB           DUP   
1100 | ----           SWAP  
1101 | TTDD           DICT  
1110 | TT-D           DFST  
1111 | TT-W           ICST  

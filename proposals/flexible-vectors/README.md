# Concrete model for flexible vectors

## Terms

- (WASM) compiler: A compiler reads a program source code (eg: C) and translates it into WASM bytecode.
- target architecture: the architecture where the WASM bytecode is ran.
- WASM engine: An engine reads WASM bytecode and translates it into native instructions for the architecture it is currently running on, and runs them.
- element width: size in bits of the element (`float` is 32-bit wide).
- SIMD width: size in bits of the hardware SIMD registers.
- SIMD length: number of elements that can fit within a hardware SIMD register (SIMD width / element width).
- lane: element within a vector

## General goal

The core of the flexible vectors proposal is to a have a single bytecode that can target multiple architectures with different SIMD width.
This should be as efficient as possible (ie: it should run as fast as possible).
In order to do that, flexible vectors should abstract the SIMD width away.

List of potential targets:

| ISA           | SIMD width   |
|:--------------|:-------------|
| SSE - SSE4.2  | 128          |
| AVX - AVX2    | 256          |
| AVX512 -      | 512          |
| Altivec - VSX | 128          |
| Neon          | 128          |
| SVE           | 128 -  2048  |
| Risc-V V      | 128 - 65536? |

I propose that SIMD width is accessible by WASM bytecode, and have the following property: SIMD width is a runtime constant, and thus cannot change during the execution of the program.
Therefore, all vectors manipulated by the program have the exact same width.
Smaller vectors are handled using masks.
As a consequence, a WASM compiler cannot assume any particular value, but can assume it will not change.
A WASM engine could optimize the bytecode for the target architecture (constant folding) where the actual SIMD width is known.


We could impose more constraints on the SIMD width:

- It should be a multiple of 128 bits.
- It is a power of 2? (might be problematic to target SVE)
- It cannot be wider than 2048? (might be problematic to target Risc-V V)

## SIMD types

Vector types:

- `vec.v8`: vector of 8-bit elements
- `vec.v16`: vector of 16-bit elements
- `vec.v32`: vector of 32-bit elements
- `vec.v64`: vector of 64-bit elements
- `vec.v128`: vector of 128-bit elements

Mask types:

- `vec.m8`: mask for a vector of 8-bit elements
- `vec.m16`: mask for a vector of 16-bit elements
- `vec.m32`: mask for a vector of 32-bit elements
- `vec.m64`: mask for a vector of 64-bit elements
- `vec.m128`: mask for a vector of 128-bit elements

Vector types can be interpreted in multiple ways:

| vector type | interpretations                                    |
|:------------|:---------------------------------------------------|
| `vec.v8`    | `vec.i8`                                           |
| `vec.v16`   | `vec.i16`                                          |
| `vec.v32`   | `vec.i32`, `vec.f32`                               |
| `vec.v64`   | `vec.i64`, `vec.f64`                               |
| `vec.v128`  | `vec.v8x16`, `vec.v16x8`, `vec.v32x4`, `vec.v64x2` |

Mask types are not vector types, and their actual representation differ depending on the target architecture.
In particular, `vec.m8` and `vec.m16` might have different architectural sizes (or actually be the same).
In a similar fashion, `vec.v8` and `vec.m8` might also have different architectural sizes (or actually be the same).

## Immediate LaneIdx operands

As the vector length is not known at compile time, it does not make much sense to specify lane indices as immediates.

Therefore, the type `i32` will be used to specify a lane index.
The range of a lane index is no further constrained, and is intepreted modulo the actual vector length.

If we impose an upper bound on the SIMD width, we can further constraint the range of lane indices.
For instance, a maximum width of 2048 would allow to store a lane index in a 8-bit integer.
Similarily, a maximum width of 524288 would allow to store a lane index in a 16-bit integer.
The would not change the instruction encoding as they all take runtime lane indices which are always `i32`.

## Special Operations

### Vector length

Returns the number of elements.
Returned value will always be the same during the whole execution of the program.

- `vec.v8.length -> i32`
- `vec.v16.length -> i32`
- `vec.v32.length -> i32`
- `vec.v64.length -> i32`
- `vec.v128.length -> i32`

### Mask count

Returns the number of active lanes of a `mask`

- `vec.m8.count(m: vec.m8) -> i32`
- `vec.m16.count(m: vec.m16) -> i32`
- `vec.m32.count(m: vec.m32) -> i32`
- `vec.m64.count(m: vec.m64) -> i32`
- `vec.m128.count(m: vec.m128) -> i32`

```python
def mask.S.count(m):
    result = 0
    for i in range(mask.S.length):
      if m[i]:
          result += 1
    return result
```

### Mask all active

Returns a mask where all lanes are active

- `vec.m8.all -> vec.m8`
- `vec.m16.all -> vec.m16`
- `vec.m32.all -> vec.m32`
- `vec.m64.all -> vec.m64`
- `vec.m128.all -> vec.m128`

```python
def mask.S.all():
    result = mask.S.New()
    for i in range(mask.S.length):
        result[i] = 1
    return result
```

### Mask None active

Returns a mask where no lane is active

- `vec.m8.none -> vec.m8`
- `vec.m16.none -> vec.m16`
- `vec.m32.none -> vec.m32`
- `vec.m64.none -> vec.m64`
- `vec.m128.none -> vec.m128`

```python
def mask.S.none():
    result = mask.S.New()
    for i in range(mask.S.length):
        result[i] = 0
    return result
```

### Mask index CMP

Returns a mask whose active lanes satisfy `(x + laneIdx) CMP n`
CMP one of the following: `eq`, `ne`, `lt`, `le`, `gt`, `ge`

The addition and the comparison are done signed, with infinite precision.

- `vec.m8.index_CMP(x: i32, n: i32) -> vec.m8`
- `vec.m16.index_CMP(x: i32, n: i32) -> vec.m16`
- `vec.m32.index_CMP(x: i32, n: i32) -> vec.m32`
- `vec.m64.index_CMP(x: i32, n: i32) -> vec.m64`
- `vec.m128.index_CMP(x: i32, n: i32) -> vec.m128`

```python
def vec.S.index_CMP(x, n):
    result = vec.S.New()
    for i in range(vec.S.length):
        if (x + i) CMP n:
            result[i] = 1
        else:
            result[i] = 0
    return result
```

### Mask index first

Returns the index of the first active lane.
If there is no active lane, the length of the vector is returned.

- `vec.m8.index_first(m: vec.m8) -> i32`
- `vec.m16.index_first(m: vec.m16) -> i32`
- `vec.m32.index_first(m: vec.m32) -> i32`
- `vec.m64.index_first(m: vec.m64) -> i32`
- `vec.m128.index_first(m: vec.m128) -> i32`

```python
def vec.S.index_first(m):
    for i in range(vec.S.length):
        if m[i]:
            return i
    return vec.S.length
```

### Mask index last

Returns the index of the last active lane.
If there is no active lane, -1 is returned.

- `vec.m8.index_last(m: vec.m8) -> i32`
- `vec.m16.index_last(m: vec.m16) -> i32`
- `vec.m32.index_last(m: vec.m32) -> i32`
- `vec.m64.index_last(m: vec.m64) -> i32`
- `vec.m128.index_last(m: vec.m128) -> i32`

```python
def vec.S.index_last(m):
    idx = -1
    for i in range(vec.S.length):
        if m[i]:
            idx = i
    return idx
```

### Mask first

Returns a mask with a single active lane that corresponds to the first active lane of the input.
If there is no active lanes, then an empty mask is returned.

- `vec.m8.first(m: vec.m8) -> vec.m8`
- `vec.m16.first(m: vec.m16) -> vec.m16`
- `vec.m32.first(m: vec.m32) -> vec.m32`
- `vec.m64.first(m: vec.m64) -> vec.m64`
- `vec.m128.first(m: vec.m128) -> vec.m128`

```python
def vec.S.first(m):
    result = mask.S.New()
    for i in range(vec.S.length):
        if m[i]:
            result[i] = 1
            break
    return result
```

### Mask last

Returns a mask with a single active lane that corresponds to the last active lane of the input.
If there is no active lanes, then an empty mask is returned.

- `vec.m8.last(m: vec.m8) -> vec.m8`
- `vec.m16.last(m: vec.m16) -> vec.m16`
- `vec.m32.last(m: vec.m32) -> vec.m32`
- `vec.m64.last(m: vec.m64) -> vec.m64`
- `vec.m128.last(m: vec.m128) -> vec.m128`

```python
def vec.S.last(m):
    result = mask.S.New()
    idx = -1
    for i in range(vec.S.length):
        if m[i]:
            idx = i
    if idx >= 0:
        result[idx] = 1
    return result
```


## Memory operations

### Vector load

- `vec.v8.load(a: memarg) -> vec.v8`
- `vec.v16.load(a: memarg) -> vec.v16`
- `vec.v32.load(a: memarg) -> vec.v32`
- `vec.v64.load(a: memarg) -> vec.v64`
- `vec.v128.load(a: memarg) -> vec.v128`

### Vector load mask zero

Inactive elements are set to `0`

- `vec.v8.load_mz(m: vec.m8, a: memarg) -> vec.v8`
- `vec.v16.load_mz(m: vec.m16, a: memarg) -> vec.v16`
- `vec.v32.load_mz(m: vec.m32, a: memarg) -> vec.v32`
- `vec.v64.load_mz(m: vec.m64, a: memarg) -> vec.v64`
- `vec.v128.load_mz(m: vec.m128, a: memarg) -> vec.v128`

### Vector load mask undefined

Inactive elements have undefined values

- `vec.v8.load_mx(m: vec.m8, a: memarg) -> vec.v8`
- `vec.v16.load_mx(m: vec.m16, a: memarg) -> vec.v16`
- `vec.v32.load_mx(m: vec.m32, a: memarg) -> vec.v32`
- `vec.v64.load_mx(m: vec.m64, a: memarg) -> vec.v64`
- `vec.v128.load_mx(m: vec.m128, a: memarg) -> vec.v128`

### Vector load splat

- `vec.v8.load_splat(a: memarg) -> vec.v8`
- `vec.v16.load_splat(a: memarg) -> vec.v16`
- `vec.v32.load_splat(a: memarg) -> vec.v32`
- `vec.v64.load_splat(a: memarg) -> vec.v64`
- `vec.v128.load_splat(a: memarg) -> vec.v128`

### Vector store

- `vec.v8.store(a: memarg, v: vec.v8)`
- `vec.v16.store(a: memarg, v: vec.v16)`
- `vec.v32.store(a: memarg, v: vec.v32)`
- `vec.v64.store(a: memarg, v: vec.v64)`
- `vec.v128.store(a: memarg, v: vec.v128)`

### Vector store mask

Inactive elements are not stored

- `vec.v8.m_store(m: vec.m8, a: memarg, v: vec.v8)`
- `vec.v16.m_store(m: vec.m16, a: memarg, v: vec.v16)`
- `vec.v32.m_store(m: vec.m32, a: memarg, v: vec.v32)`
- `vec.v64.m_store(m: vec.m64, a: memarg, v: vec.v64)`
- `vec.v128.m_store(m: vec.m128, a: memarg, v: vec.v128)`

## Lane operations

### Splat scalar

For `vec.i8.splat` and `vec.i16.splat`, `x` is truncated to 8 and 16 bits respectively.

- `vec.i8.splat(x: i32) -> vec.v8`
- `vec.i16.splat(x: i32) -> vec.v16`
- `vec.i32.splat(x: i32) -> vec.v32`
- `vec.f32.splat(x: f32) -> vec.v32`
- `vec.i64.splat(x: i64) -> vec.v64`
- `vec.f64.splat(x: f64) -> vec.v64`
- `vec.v128.splat(x: v128) -> vec.v128`

### Extract lane

`idx` is interpreted modulo the length of the vector.

- `vec.s8.extract_lane(v: vec.v8, idx: i32) -> i32`
- `vec.u8.extract_lane(v: vec.v8, idx: i32) -> i32`
- `vec.s16.extract_lane(v: vec.v16, idx: i32) -> i32`
- `vec.u16.extract_lane(v: vec.v16, idx: i32) -> i32`
- `vec.i32.extract_lane(v: vec.v32, idx: i32) -> i32`
- `vec.f32.extract_lane(v: vec.v32, idx: i32) -> f32`
- `vec.i64.extract_lane(v: vec.v64, idx: i32) -> i64`
- `vec.f64.extract_lane(v: vec.v64, idx: i32) -> f64`
- `vec.v128.extract_lane(v: vec.v128, idx: i32) -> v128`

### Replace lane

`idx` is interpreted modulo the length of the vector.

- `vec.i8.extract_lane(v: vec.v8, idx: i32, x: i32) -> vec.v8`
- `vec.i16.extract_lane(v: vec.v16, idx: i32, x: i32) -> vec.v16`
- `vec.i32.extract_lane(v: vec.v32, idx: i32, x: i32) -> vec.v32`
- `vec.f32.extract_lane(v: vec.v32, idx: i32, x: f32) -> vec.v32`
- `vec.i64.extract_lane(v: vec.v64, idx: i32, x: i64) -> vec.v64`
- `vec.f64.extract_lane(v: vec.v64, idx: i32, x: f64) -> vec.v64`
- `vec.v128.extract_lane(v: vec.v128, idx: i32, x: v128) -> vec.v128`

### Load lane

Loads a single lane into existing vector.
`idx` is interpreted modulo the length of the vector.

- `vec.v8.load_lane(a: memarg, v: vec.v8, idx: i32) -> vec.v8`
- `vec.v16.load_lane(a: memarg, v: vec.v16, idx: i32) -> vec.v16`
- `vec.v32.load_lane(a: memarg, v: vec.v32, idx: i32) -> vec.v32`
- `vec.v64.load_lane(a: memarg, v: vec.v64, idx: i32) -> vec.v64`
- `vec.v128.load_lane(a: memarg, v: vec.v128, idx: i32) -> vec.v128`

### Store lane

Stores a single lane from vector
`idx` is interpreted modulo the length of the vector.

- `vec.v8.store_lane(a: memarg, v: vec.v8, idx: i32)`
- `vec.v16.store_lane(a: memarg, v: vec.v16, idx: i32)`
- `vec.v32.store_lane(a: memarg, v: vec.v32, idx: i32)`
- `vec.v64.store_lane(a: memarg, v: vec.v64, idx: i32)`
- `vec.v128.store_lane(a: memarg, v: vec.v128, idx: i32)`


## Unary Arithmetic operators

UNOP designates any unary operator (eg: neg, not)

### UNOP

- `vec.v8.UNOP(a: vec.v8) -> vec.v8`
- `vec.v16.UNOP(a: vec.v16) -> vec.v16`
- `vec.v32.UNOP(a: vec.v32) -> vec.v32`
- `vec.v64.UNOP(a: vec.v64) -> vec.v64`
- `vec.v128.UNOP(a: vec.v128) -> vec.v128`
- `vec.m8.UNOP(a: vec.m8) -> vec.m8`
- `vec.m16.UNOP(a: vec.m16) -> vec.m16`
- `vec.m32.UNOP(a: vec.m32) -> vec.m32`
- `vec.m64.UNOP(a: vec.m64) -> vec.m64`
- `vec.m128.UNOP(a: vec.m128) -> vec.m128`

Note:

> - Masks only support bitwise operations.

### UNOP mask zero

Inactive lanes are set to zero.

- `vec.v8.UNOP_mz(m: vec.m8, a: vec.v8) -> vec.v8`
- `vec.v16.UNOP_mz(m: vec.m16, a: vec.v16) -> vec.v16`
- `vec.v32.UNOP_mz(m: vec.m32, a: vec.v32) -> vec.v32`
- `vec.v64.UNOP_mz(m: vec.m64, a: vec.v64) -> vec.v64`
- `vec.v128.UNOP_mz(m: vec.m128, a: vec.v128) -> vec.v128`
- `vec.m8.UNOP_mz(m: vec.m8, a: vec.m8) -> vec.m8`
- `vec.m16.UNOP_mz(m: vec.m16, a: vec.m16) -> vec.m16`
- `vec.m32.UNOP_mz(m: vec.m32, a: vec.m32) -> vec.m32`
- `vec.m64.UNOP_mz(m: vec.m64, a: vec.m64) -> vec.m64`
- `vec.m128.UNOP_mz(m: vec.m128, a: vec.m128) -> vec.m128`

Note:

> - Masks only support bitwise operations.

### UNOP mask merge

Inactive lanes are left untouched.

- `vec.v8.UNOP_mm(m: vec.m8, a: vec.v8) -> vec.v8`
- `vec.v16.UNOP_mm(m: vec.m16, a: vec.v16) -> vec.v16`
- `vec.v32.UNOP_mm(m: vec.m32, a: vec.v32) -> vec.v32`
- `vec.v64.UNOP_mm(m: vec.m64, a: vec.v64) -> vec.v64`
- `vec.v128.UNOP_mm(m: vec.m128, a: vec.v128) -> vec.v128`
- `vec.m8.UNOP_mm(m: vec.m8, a: vec.m8) -> vec.m8`
- `vec.m16.UNOP_mm(m: vec.m16, a: vec.m16) -> vec.m16`
- `vec.m32.UNOP_mm(m: vec.m32, a: vec.m32) -> vec.m32`
- `vec.m64.UNOP_mm(m: vec.m64, a: vec.m64) -> vec.m64`
- `vec.m128.UNOP_mm(m: vec.m128, a: vec.m128) -> vec.m128`

Note:

> - Masks only support bitwise operations.

### UNOP mask undefined

Inactive lanes are undefined.

- `vec.v8.UNOP_mx(m: vec.m8, a: vec.v8) -> vec.v8`
- `vec.v16.UNOP_mx(m: vec.m16, a: vec.v16) -> vec.v16`
- `vec.v32.UNOP_mx(m: vec.m32, a: vec.v32) -> vec.v32`
- `vec.v64.UNOP_mx(m: vec.m64, a: vec.v64) -> vec.v64`
- `vec.v128.UNOP_mx(m: vec.m128, a: vec.v128) -> vec.v128`

## Binary Arithmetic operators

BINOP designates any binary operator that is not a comparison (eg: add, sub, rsub, mul, div, rdiv, and, or, xor...)

### Select mask

Selects active elements from `a` and inactive elements from `b`.

- `vec.v8.select(m: vec.m8, a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.select(m: vec.m16, a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.select(m: vec.m32, a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.select(m: vec.m64, a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.select(m: vec.m128, a: vec.v128, b: vec.v128) -> vec.v128`

### BINOP

- `vec.v8.BINOP(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.BINOP(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.BINOP(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.BINOP(a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.BINOP(a: vec.v128, b: vec.v128) -> vec.v128`
- `vec.m8.BINOP(a: vec.m8, b: vec.m8) -> vec.m8`
- `vec.m16.BINOP(a: vec.m16, b: vec.m16) -> vec.m16`
- `vec.m32.BINOP(a: vec.m32, b: vec.m32) -> vec.m32`
- `vec.m64.BINOP(a: vec.m64, b: vec.m64) -> vec.m64`
- `vec.m128.BINOP(a: vec.m128, b: vec.m128) -> vec.m128`

Note:

> - Masks only support bitwise operations.

### BINOP mask zero

Inactive elements are set to zero.

- `vec.v8.BINOP_mz(m: vec.m8, a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.BINOP_mz(m: vec.m16, a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.BINOP_mz(m: vec.m32, a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.BINOP_mz(m: vec.m64, a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.BINOP_mz(m: vec.m128, a: vec.v128, b: vec.v128) -> vec.v128`
- `vec.m8.BINOP_mz(m: vec.m8, a: vec.m8, b: vec.m8) -> vec.m8`
- `vec.m16.BINOP_mz(m: vec.m16, a: vec.m16, b: vec.m16) -> vec.m16`
- `vec.m32.BINOP_mz(m: vec.m32, a: vec.m32, b: vec.m32) -> vec.m32`
- `vec.m64.BINOP_mz(m: vec.m64, a: vec.m64, b: vec.m64) -> vec.m64`
- `vec.m128.BINOP_mz(m: vec.m128, a: vec.m128, b: vec.m128) -> vec.m128`

Note:

> - Masks only support bitwise operations.

### BINOP mask merge

Inactive elements are forwarded from `a`.

- `vec.v8.BINOP_mm(m: vec.m8, a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.BINOP_mm(m: vec.m16, a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.BINOP_mm(m: vec.m32, a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.BINOP_mm(m: vec.m64, a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.BINOP_mm(m: vec.m128, a: vec.v128, b: vec.v128) -> vec.v128`
- `vec.m8.BINOP_mm(m: vec.m8, a: vec.m8, b: vec.m8) -> vec.m8`
- `vec.m16.BINOP_mm(m: vec.m16, a: vec.m16, b: vec.m16) -> vec.m16`
- `vec.m32.BINOP_mm(m: vec.m32, a: vec.m32, b: vec.m32) -> vec.m32`
- `vec.m64.BINOP_mm(m: vec.m64, a: vec.m64, b: vec.m64) -> vec.m64`
- `vec.m128.BINOP_mm(m: vec.m128, a: vec.m128, b: vec.m128) -> vec.m128`

Note:

> - Masks only support bitwise operations.

### BINOP mask undefined

Inactive elements are undefined.

- `vec.v8.BINOP_mx(m: vec.m8, a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.BINOP_mx(m: vec.m16, a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.BINOP_mx(m: vec.m32, a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.BINOP_mx(m: vec.m64, a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.BINOP_mx(m: vec.m128, a: vec.v128, b: vec.v128) -> vec.v128`


## Comparisons

CMP designates any comparison operator (eg: `eq_u`, `ne_s`, `lt_f`, `le_s`, `gt_f`, `ge_u`)

### CMP

- `vec.m8.CMP(a: vec.v8, b: vec.v8) -> vec.m8`
- `vec.m16.CMP(a: vec.v16, b: vec.v16) -> vec.m16`
- `vec.m32.CMP(a: vec.v32, b: vec.v32) -> vec.m32`
- `vec.m64.CMP(a: vec.v64, b: vec.v64) -> vec.m64`

```python
def vec.S.CMP(a, b):
    result = mask.S.New()
    for i in range(mask.S.length):
        if a[i] CMP b[i]:
            result[i] = 1
        else:
            result[i] = 0
    return result
```

### CMP mask

Inactive elements are set to `0`.

- `vec.m8.CMP_m(m: vec.m8, a: vec.v8, b: vec.v8) -> vec.m8`
- `vec.m16.CMP_m(m: vec.m16, a: vec.v16, b: vec.v16) -> vec.m16`
- `vec.m32.CMP_m(m: vec.m32, a: vec.v32, b: vec.v32) -> vec.m32`
- `vec.m64.CMP_m(m: vec.m64, a: vec.v64, b: vec.v64) -> vec.m64`

```python
def vec.S.CMP_m(m, a, b):
    result = mask.S.New()
    for i in range(mask.S.length):
        if m[i] and a[i] CMP b[i]:
            result[i] = 1
        else:
            result[i] = 0
    return result
```

### Sign to mask

For each lane, the mask lane is set to `1` if the element is negative (floats are not interpreted), and `0` otherwise.

- `vec.m8.sign(a: vec.v8) -> vec.m8`
- `vec.m16.sign(a: vec.v16) -> vec.m16`
- `vec.m32.sign(a: vec.v32) -> vec.m32`
- `vec.m64.sign(a: vec.v64) -> vec.m64`

## Inter-lane operations

### LUT1 zero

Gets elements from `a` located at the index specified by `idx`.
Elements whose index is out of bounds are set to `0`.

- `vec.v8.lut1_z(idx: vec.v8, a: vec.v8) -> vec.v8`
- `vec.v16.lut1_z(idx: vec.v16, a: vec.v16) -> vec.v16`
- `vec.v32.lut1_z(idx: vec.v32, a: vec.v32) -> vec.v32`
- `vec.v64.lut1_z(idx: vec.v64, a: vec.v64) -> vec.v64`
- `vec.v128.lut1_z(idx: vec.v128, a: vec.v128) -> vec.v128`

```python
def vec.S.lut1_z(idx, a):
    result = vec.S.New()
    for i in range(vec.S.length):
        if idx[i] < vec.S.length:
            result[i] = a[idx[i]]
        else:
            result[i] = 0
    return result
```

### LUT1 merge

Gets elements from `a` located at the index specified by `idx`.
Elements whose index is out of bounds are taken from `fallback`.

- `vec.v8.lut1_m(idx: vec.v8, a: vec.v8, fallback: vec.v8) -> vec.v8`
- `vec.v16.lut1_m(idx: vec.v16, a: vec.v16, fallback: vec.v16) -> vec.v16`
- `vec.v32.lut1_m(idx: vec.v32, a: vec.v32, fallback: vec.v32) -> vec.v32`
- `vec.v64.lut1_m(idx: vec.v64, a: vec.v64, fallback: vec.v64) -> vec.v64`

```python
def vec.S.lut1_m(idx, a, fallback):
    result = vec.S.New()
    for i in range(vec.S.length):
        if idx[i] < vec.S.length:
            result[i] = a[idx[i]]
        else:
            result[i] = fallback[i]
    return result
```

### LUT2 zero

Gets elements from `a` and `b` located at the index specified by `idx`.
If the index is lower than length, elements are taken from `a`, if index is between length and 2 * length, elements are taken from `b`.
Elements whose index is out of bounds are set to `0`.

- `vec.v8.lut2_z(idx: vec.v8, a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.lut2_z(idx: vec.v16, a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.lut2_z(idx: vec.v32, a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.lut2_z(idx: vec.v64, a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.lut2_z(idx: vec.v128, a: vec.v128, b: vec.v128) -> vec.v128`

```python
def vec.S.lut2_z(idx, a):
    result = vec.S.New()
    for i in range(vec.S.length):
        if idx[i] < vec.S.length:
            result[i] = a[idx[i]]
        elif idx[i] < 2*vec.S.length:
            result[i] = b[idx[i] - vec.S.length]
        else:
            result[i] = 0
    return result
```
### LUT2 merge

Gets elements from `a` and `b` located at the index specified by `idx`.
If the index is lower than length, elements are taken from `a`, if index is between length and 2 * length, elements are taken from `b`.
Elements whose index is out of bounds are taken from fallback.

- `vec.v8.lut2_m(idx: vec.v8, a: vec.v8, b: vec.v8, fallback: vec.v8) -> vec.v8`
- `vec.v16.lut2_m(idx: vec.v16, a: vec.v16, b: vec.v16, fallback: vec.v16) -> vec.v16`
- `vec.v32.lut2_m(idx: vec.v32, a: vec.v32, b: vec.v32, fallback: vec.v32) -> vec.v32`
- `vec.v64.lut2_m(idx: vec.v64, a: vec.v64, b: vec.v64, fallback: vec.v64) -> vec.v64`
- `vec.v128.lut2_m(idx: vec.v128, a: vec.v128, b: vec.v128, fallback: vec.v128) -> vec.v128`

```python
def vec.S.lut2_m(idx, a, b, fallback):
    result = vec.S.New()
    for i in range(vec.S.length):
        if idx[i] < vec.S.length:
            result[i] = a[idx[i]]
        elif idx[i] < 2*vec.S.length:
            result[i] = b[idx[i] - vec.S.length]
        else:
            result[i] = fallback[i]
    return result
```

### V128 shuffle

Applies shuffle to each v128 of the vector.

- `vec.i8x16.shuffle(a: vec.v128, b: vec.v128, imm: ImmLaneIdx32[16]) -> vec.v128`

```python
def vec.i8x16.shuffle(a, b, imm):
    result = vec.v128.New()
    for i in range(vec.v128.length):
        result[i] = i8x16.shuffle(a[i], b[i], imm)
    return result
```

### V128 swizzle

Applies swizzle to each v128 of the vector.

- `vec.i8x16.swizzle(a: vec.v128, s: vec.v128) -> vec.v128`

```python
def vec.i8x16.swizzle(idx, a, s):
    result = vec.v128.New()
    for i in range(vec.v128.length):
        result[i] = i8x16.swizzle(a[i], s[i], imm)
    return result
```

### Splat lane

Gets a single lane from vector and broadcast it to the entire vector.
`idx` is interpreted modulo the cardinal of the vector.

- `vec.v8.splat_lane(v: vec.v8, idx: i32) -> vec.v8`
- `vec.v16.splat_lane(v: vec.v16, idx: i32) -> vec.v16`
- `vec.v32.splat_lane(v: vec.v32, idx: i32) -> vec.v32`
- `vec.v64.splat_lane(v: vec.v64, idx: i32) -> vec.v64`
- `vec.v128.splat_lane(v: vec.v128, idx: i32) -> vec.v128`

```python
def vec.S.splat_lane(v, imm):
    idx = idx % vec.S.length
    result = vec.S.New()
    for i in range(vec.S.length):
        result[i] = v[idx]
    return result
```

### Concat

Copies elements from vector `a` from first active element to last active element.
Inner inactive elements are also copied.
The remaining elements are set from the first elements from `b`.

- `vec.v8.concat(m: vec.m8, a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.concat(m: vec.m16, a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.concat(m: vec.m32, a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.concat(m: vec.m64, a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.concat(m: vec.m128, a: vec.v128, b: vec.v128) -> vec.v128`


```python
def vec.S.concat(m, a, b):
    begin = -1
    end = -1
    for i in range(vec.S.length):
        if m[i]:
            end = i + 1
            if begin < 0:
                begin = i

    result = vec.S.New()
    i = 0
    for j in range(begin, end):
        result[i] = a[j]
        i += 1
    for j in range(0, vec.S.length - i):
        result[i] = b[j]
        i += 1
    return result
```

### Lane shift

Concats the 2 input vector to form a single double-width vector.
Shifts this double-width vector by `n` lane to the right (to LSB).
Extracts the lower half of the shifted vector.
`n` is interpreted modulo the length of the vector.


- `vec.v8.lane_shift(a: vec.v8, b: vec.v8, n: i32) -> vec.v8`
- `vec.v16.lane_shift(a: vec.v16, b: vec.v16, n: i32) -> vec.v16`
- `vec.v32.lane_shift(a: vec.v32, b: vec.v32, n: i32) -> vec.v32`
- `vec.v64.lane_shift(a: vec.v64, b: vec.v64, n: i32) -> vec.v64`
- `vec.v128.lane_shift(a: vec.v128, b: vec.v128, n: i32) -> vec.v128`

```python
def vec.S.lane_shift(a, b, n):
    result = vec.S.New()
    n = n % vec.S.length
    for i in range(0, vec.S.length - n):
        result[i] = a[i + n]
    for i in range(vec.S.length - n, vec.S.length):
        result[i] = b[i - (vec.S.length - n)]
    return result
```

### Interleave even

Extracts even elements from both input and interleaves them.

- `vec.v8.interleave_even(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.interleave_even(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.interleave_even(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.interleave_even(a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.interleave_even(a: vec.v128, b: vec.v128) -> vec.v128`
- `vec.m8.interleave_even(a: vec.m8, b: vec.m8) -> vec.m8`
- `vec.m16.interleave_even(a: vec.m16, b: vec.m16) -> vec.m16`
- `vec.m32.interleave_even(a: vec.m32, b: vec.m32) -> vec.m32`
- `vec.m64.interleave_even(a: vec.m64, b: vec.m64) -> vec.m64`
- `vec.m128.interleave_even(a: vec.m128, b: vec.m128) -> vec.m128`


```python
def vec.S.interleave_even(a, b):
    result = vec.S.New()
    for i in range(vec.S.length/2):
        result[2*i] = a[2*i]
        result[2*i + 1] = b[2*i]
    return result
```

Note:

> - can be implemented with `TRN1` on Neon/SVE

### Interleave odd

Extracts odd elements from both input and interleaves them.

- `vec.v8.interleave_odd(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.interleave_odd(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.interleave_odd(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.interleave_odd(a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.interleave_odd(a: vec.v128, b: vec.v128) -> vec.v128`
- `vec.m8.interleave_odd(a: vec.m8, b: vec.m8) -> vec.m8`
- `vec.m16.interleave_odd(a: vec.m16, b: vec.m16) -> vec.m16`
- `vec.m32.interleave_odd(a: vec.m32, b: vec.m32) -> vec.m32`
- `vec.m64.interleave_odd(a: vec.m64, b: vec.m64) -> vec.m64`
- `vec.m128.interleave_odd(a: vec.m128, b: vec.m128) -> vec.m128`


```python
def vec.S.interleave_odd(a, b):
    result = vec.S.New()
    for i in range(vec.S.length/2):
        result[2*i] = a[2*i+1]
        result[2*i + 1] = b[2*i+1]
    return result
```

Note:

> - can be implemented with `TRN2` on Neon/SVE

### Concat even

Extracts even elements from both input and concatenate them.

- `vec.v8.concat_even(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.concat_even(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.concat_even(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.concat_even(a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.concat_even(a: vec.v128, b: vec.v128) -> vec.v128`
- `vec.m8.concat_even(a: vec.m8, b: vec.m8) -> vec.m8`
- `vec.m16.concat_even(a: vec.m16, b: vec.m16) -> vec.m16`
- `vec.m32.concat_even(a: vec.m32, b: vec.m32) -> vec.m32`
- `vec.m64.concat_even(a: vec.m64, b: vec.m64) -> vec.m64`
- `vec.m128.concat_even(a: vec.m128, b: vec.m128) -> vec.m128`


```python
def vec.S.concat_even(a, b):
    result = vec.S.New()
    
    for i in range(vec.S.length/2):
        result[i] = a[2*i]
    for i in range(vec.S.length/2):
        result[i + vec.S.length/2] = b[2*i]
    return result
```

Note:

> - can be implemented with `UZP1` on Neon/SVE
> - Wrapping narrowing integer conversions could be implemented with this function

### Concat odd

Extracts odd elements from both input and concatenate them.

- `vec.v8.concat_odd(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.concat_odd(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.concat_odd(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.concat_odd(a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.concat_odd(a: vec.v128, b: vec.v128) -> vec.v128`
- `vec.m8.concat_odd(a: vec.m8, b: vec.m8) -> vec.m8`
- `vec.m16.concat_odd(a: vec.m16, b: vec.m16) -> vec.m16`
- `vec.m32.concat_odd(a: vec.m32, b: vec.m32) -> vec.m32`
- `vec.m64.concat_odd(a: vec.m64, b: vec.m64) -> vec.m64`
- `vec.m128.concat_odd(a: vec.m128, b: vec.m128) -> vec.m128`


```python
def vec.S.concat_odd(a, b):
    result = vec.S.New()
    
    for i in range(vec.S.length/2):
        result[i] = a[2*i+1]
    for i in range(vec.S.length/2):
        result[i + vec.S.length/2] = b[2*i+1]
    return result
```

Note:

> - can be implemented with `UZP2` on Neon/SVE

### Interleave low

Extracts the lower half of both input and interleaves their elements.

- `vec.v8.interleave_low(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.interleave_low(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.interleave_low(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.interleave_low(a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.interleave_low(a: vec.v128, b: vec.v128) -> vec.v128`
- `vec.m8.interleave_low(a: vec.m8, b: vec.m8) -> vec.m8`
- `vec.m16.interleave_low(a: vec.m16, b: vec.m16) -> vec.m16`
- `vec.m32.interleave_low(a: vec.m32, b: vec.m32) -> vec.m32`
- `vec.m64.interleave_low(a: vec.m64, b: vec.m64) -> vec.m64`
- `vec.m128.interleave_low(a: vec.m128, b: vec.m128) -> vec.m128`


```python
def vec.S.interleave_low(a, b):
    result = vec.S.New()
    for i in range(vec.S.length/2):
        result[2*i] = a[i]
        result[2*i + 1] = b[i]
    return result
```

Note:

> - can be implemented with `ZIP1` on Neon/SVE

### Interleave high

Extracts the higher half of both input and interleaves their elements.

- `vec.v8.interleave_high(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.v16.interleave_high(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.v32.interleave_high(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.v64.interleave_high(a: vec.v64, b: vec.v64) -> vec.v64`
- `vec.v128.interleave_high(a: vec.v128, b: vec.v128) -> vec.v128`
- `vec.m8.interleave_high(a: vec.m8, b: vec.m8) -> vec.m8`
- `vec.m16.interleave_high(a: vec.m16, b: vec.m16) -> vec.m16`
- `vec.m32.interleave_high(a: vec.m32, b: vec.m32) -> vec.m32`
- `vec.m64.interleave_high(a: vec.m64, b: vec.m64) -> vec.m64`
- `vec.m128.interleave_high(a: vec.m128, b: vec.m128) -> vec.m128`


```python
def vec.S.interleave_high(a, b):
    result = vec.S.New()
    for i in range(vec.S.length/2):
        result[2*i] = a[i + vec.S.length/2]
        result[2*i + 1] = b[i + vec.S.length/2]
    return result
```

Note:

> - can be implemented with `ZIP2` on Neon/SVE

## Conversions

### Narrowing conversions

Converts each elements of both inputs to narrower types using saturation, and concats them.

- `vec.i8.narrow_i16_u(a: vec.v16, b: vec.v16) -> vec.v8`
- `vec.i8.narrow_i16_s(a: vec.v16, b: vec.v16) -> vec.v8`
- `vec.i16.narrow_i32_u(a: vec.v32, b: vec.v32) -> vec.v16`
- `vec.i16.narrow_i32_s(a: vec.v32, b: vec.v32) -> vec.v16`
- `vec.i32.narrow_i64_u(a: vec.v64, b: vec.v64) -> vec.v32`
- `vec.i32.narrow_i64_s(a: vec.v64, b: vec.v64) -> vec.v32`

### Mask narrowing

Returns a `mask` for a narrower type with the same active lanes

- `vec.m8.narrow_m16(a: vec.m16, b: vec.m16) -> vec.m8`
- `vec.m16.narrow_m32(a: vec.m32, b: vec.m32) -> vec.m16`
- `vec.m32.narrow_m64(a: vec.m64, b: vec.m64) -> vec.m32`
- `vec.m64.narrow_m128(a: vec.m128, b: vec.m128) -> vec.m64`

```python
def mask.S.narrow(a, b):
    result = mask.S.New()
    for i in range(mask.S.length/2):
        result[i] = a[i]
    for i in range(mask.S.length/2):
        result[i + mask.S.length/2] = a[i]
    return result
```

### Widening conversions

- `vec.i16.widen_low_i8_u(a: vec.v8) -> vec.v16`
- `vec.i16.widen_low_i8_s(a: vec.v8) -> vec.v16`
- `vec.i32.widen_low_i16_u(a: vec.v8) -> vec.v32`
- `vec.i32.widen_low_i16_s(a: vec.v8) -> vec.v32`
- `vec.i64.widen_low_i32_u(a: vec.v8) -> vec.v64`
- `vec.i64.widen_low_i32_s(a: vec.v8) -> vec.v64`
- `vec.i16.widen_high_i8_u(a: vec.v8) -> vec.v16`
- `vec.i16.widen_high_i8_s(a: vec.v8) -> vec.v16`
- `vec.i32.widen_high_i16_u(a: vec.v8) -> vec.v32`
- `vec.i32.widen_high_i16_s(a: vec.v8) -> vec.v32`
- `vec.i64.widen_high_i32_u(a: vec.v8) -> vec.v64`
- `vec.i64.widen_high_i32_s(a: vec.v8) -> vec.v64`

### Mask widening

Returns a `mask` for a wider type with the same active lanes as the lower/higher part of the original `mask`

- `vec.m16.widen_low_m8(m: vec.m8) -> vec.m16`
- `vec.m32.widen_low_m16(m: vec.m16) -> vec.m32`
- `vec.m64.widen_low_m32(m: vec.m32) -> vec.m64`
- `vec.m128.widen_low_m64(m: vec.m64) -> vec.m128`
- `vec.m16.widen_high_m8(m: vec.m8) -> vec.m16`
- `vec.m32.widen_high_m16(m: vec.m16) -> vec.m32`
- `vec.m64.widen_high_m32(m: vec.m32) -> vec.m64`
- `vec.m128.widen_high_m64(m: vec.m64) -> vec.m128`

```python
def vec.S.widen_low_T(m):
    result = vec.S.New()
    for i in range(vec.S.length):
        result[i] = m[i]
    return result
```

```python
def mask.S.widen_high_T(m):
    result = vec.S.New()
    for i in range(vec.S.length):
        result[i] = m[i + vec.S.length/2]
    return result
```

### Floating point promotion

- `vec.f64.promote_low_f32(a: vec.v32) -> vec.v64`
- `vec.f64.promote_high_f32(a: vec.v32) -> vec.v64`

### Floating point demotion

- `vec.f32.demote_f64(a: vec.v64, b: vec.v64) -> vec.v32`

### Integer to single-precision floating point

- `vec.f32.convert_i32_s(a: vec.v32) -> vec.v32`
- `vec.f32.convert_i32_u(a: vec.v32) -> vec.v32`
- `vec.f32.convert_i64_s(a: vec.v64, b: vec.v64) -> vec.v32`
- `vec.f32.convert_i64_u(a: vec.v64, b: vec.v64) -> vec.v32`

### Integer to double-precision floating point

- `vec.f64.convert_low_i32_s(a: vec.v32) -> vec.v64`
- `vec.f64.convert_low_i32_u(a: vec.v32) -> vec.v64`
- `vec.f64.convert_high_i32_s(a: vec.v32) -> vec.v64`
- `vec.f64.convert_high_i32_u(a: vec.v32) -> vec.v64`
- `vec.f64.convert_i64_s(a: vec.v64) -> vec.v64`
- `vec.f64.convert_i64_u(a: vec.v64) -> vec.v64`

### single precision floating point to integer with saturation

- `vec.i32.trunc_sat_f32_s(a: vec.v32) -> vec.v32`
- `vec.i32.trunc_sat_f32_u(a: vec.v32) -> vec.v32`
- `vec.i64.trunc_sat_low_f32_s(a: vec.v32) -> vec.v64`
- `vec.i64.trunc_sat_low_f32_u(a: vec.v32) -> vec.v64`
- `vec.i64.trunc_sat_high_f32_s(a: vec.v32) -> vec.v64`
- `vec.i64.trunc_sat_high_f32_u(a: vec.v32) -> vec.v64`

### double precision floating point to integer with saturation

- `vec.i32.trunc_sat_f64_s(a: vec.v64, b: vec.v64) -> vec.v32`
- `vec.i32.trunc_sat_f64_u(a: vec.v64, b: vec.v64) -> vec.v32`
- `vec.i64.trunc_sat_f64_s(a: vec.v64) -> vec.v64`
- `vec.i64.trunc_sat_f64_u(a: vec.v64) -> vec.v64`

### Reinterpret casts

- `vec.v8.cast_v16(a: vec.v16) -> vec.v8`
- `vec.v8.cast_v32(a: vec.v32) -> vec.v8`
- `vec.v8.cast_v64(a: vec.v64) -> vec.v8`
- `vec.v8.cast_v128(a: vec.v128) -> vec.v8`
- `vec.v16.cast_v8(a: vec.v8) -> vec.v16`
- `vec.v16.cast_v32(a: vec.v32) -> vec.v16`
- `vec.v16.cast_v64(a: vec.v64) -> vec.v16`
- `vec.v16.cast_v128(a: vec.v128) -> vec.v16`
- `vec.v32.cast_v8(a: vec.v8) -> vec.v32`
- `vec.v32.cast_v16(a: vec.v16) -> vec.v32`
- `vec.v32.cast_v64(a: vec.v64) -> vec.v32`
- `vec.v32.cast_v128(a: vec.v128) -> vec.v32`
- `vec.v64.cast_v8(a: vec.v8) -> vec.v64`
- `vec.v64.cast_v16(a: vec.v16) -> vec.v64`
- `vec.v64.cast_v32(a: vec.v32) -> vec.v64`
- `vec.v64.cast_v128(a: vec.v128) -> vec.v64`
- `vec.v128.cast_v8(a: vec.v8) -> vec.v128`
- `vec.v128.cast_v16(a: vec.v16) -> vec.v128`
- `vec.v128.cast_v32(a: vec.v32) -> vec.v128`
- `vec.v128.cast_v64(a: vec.v64) -> vec.v128`

### Mask cast

- `vec.m8.cast_m16(m: vec.m16) -> vec.m8`
- `vec.m8.cast_m32(m: vec.m32) -> vec.m8`
- `vec.m8.cast_m64(m: vec.m64) -> vec.m8`
- `vec.m8.cast_m128(m: vec.m128) -> vec.m8`
- `vec.m16.cast_m8(m: vec.m8) -> vec.m16`
- `vec.m16.cast_m32(m: vec.m32) -> vec.m16`
- `vec.m16.cast_m64(m: vec.m64) -> vec.m16`
- `vec.m16.cast_m128(m: vec.m128) -> vec.m16`
- `vec.m32.cast_m8(m: vec.m8) -> vec.m32`
- `vec.m32.cast_m16(m: vec.m16) -> vec.m32`
- `vec.m32.cast_m64(m: vec.m64) -> vec.m32`
- `vec.m32.cast_m128(m: vec.m128) -> vec.m32`
- `vec.m64.cast_m8(m: vec.m8) -> vec.m64`
- `vec.m64.cast_m16(m: vec.m16) -> vec.m64`
- `vec.m64.cast_m32(m: vec.m32) -> vec.m64`
- `vec.m64.cast_m128(m: vec.m128) -> vec.m64`
- `vec.m128.cast_m8(m: vec.m8) -> vec.m128`
- `vec.m128.cast_m16(m: vec.m16) -> vec.m128`
- `vec.m128.cast_m32(m: vec.m32) -> vec.m128`
- `vec.m128.cast_m64(m: vec.m64) -> vec.m128`


```python
def vec.S.cast_T(m):
    result = vec.S.New()
    if vec.T.length < vec.S.length:
        d = vec.S.length / vec.T.length
        for i in range(vec.T.length):
            for j in range(d):
            result[i*d + j] = m[i]
    else:
        d = vec.T.length / vec.S.length
        for i in range(vec.S.length):
            result[i] = m[i * d]
    return result
```

### Mask to vec

Active lanes are set to `-1` (all one bits), and inactive lanes are set to `0`.

- `vec.v8.convert_m8(m: vec.m8) -> vec.v8`
- `vec.v16.convert_m16(m: vec.m16) -> vec.v16`
- `vec.v32.convert_m32(m: vec.m32) -> vec.v32`
- `vec.v64.convert_m64(m: vec.m64) -> vec.v64`
- `vec.v128.convert_m128(m: vec.m128) -> vec.v128`

## Test masks

### Test none

Returns `1` if and only if all lanes are inactive.
Returns `0` otherwise.

- `vec.m8.test_none(m: vec.m8) -> i32`
- `vec.m16.test_none(m: vec.m16) -> i32`
- `vec.m32.test_none(m: vec.m32) -> i32`
- `vec.m64.test_none(m: vec.m64) -> i32`
- `vec.m128.test_none(m: vec.m128) -> i32`

### Test any

Returns `1` if and only if there is at least one active lane.
Returns `0` otherwise.

- `vec.m8.test_any(m: vec.m8) -> i32`
- `vec.m16.test_any(m: vec.m16) -> i32`
- `vec.m32.test_any(m: vec.m32) -> i32`
- `vec.m64.test_any(m: vec.m64) -> i32`
- `vec.m128.test_any(m: vec.m128) -> i32`

### Test all

Returns `1` if and only if all lanes are active.
Returns `0` otherwise.

- `vec.m8.test_all(m: vec.m8) -> i32`
- `vec.m16.test_all(m: vec.m16) -> i32`
- `vec.m32.test_all(m: vec.m32) -> i32`
- `vec.m64.test_all(m: vec.m64) -> i32`
- `vec.m128.test_all(m: vec.m128) -> i32`

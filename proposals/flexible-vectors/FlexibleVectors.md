Flexible vectors overview
=========================

The goal of this proposal is to provide flexible vector instructions for
WebAssembly as a way to bridge the gap between existing SIMD instruction sets
available on various platforms. More specifically, this proposal aims to enable
better use processing capabilities of existing SIMD hardware and bring
performance of vector operaions available in WebAssembly closer to native.
`simd128` proposal already identified operations that would commonly work on
platforms that are important to WebAssembly, this proposal is attempting to
extend the same operations to work with variable vector lengths.

## Types

Proposal introduces the following vector types:

- `vec.i8` : 8-bit integer lanes
- `vec.i16`: 16-bit integer lanes
- `vec.i32`: 32-bit integer lanes
- `vec.i64`: 64-bit integer lanes
- `vec.f32`: single precision floating point lanes
- `vec.f64`: double precision floating point lanes

### Lane division interpretation

In semantic pseudocode `S` is the particular vector type, `S.LaneBits` is the
size of the lane in bits, `S.Lanes` is the number of lanes, which is dynamic.

|    S      | S.LaneBits |
|-----------|-----------:|
| `vec.i8`  |          8 |
| `vec.i16` |         16 |
| `vec.i32` |         32 |
| `vec.f32` |         32 |
| `vec.i64` |         64 |
| `vec.f64` |         64 |

### Restrictions

Lane values are intended to be handled exactly like in `simd128` proposal, with
the following differences applying to overall types:

- Runtime sets maximum vector length for every type
- Number of lanes is set separately for different lane sizes
- Vectors with different lane size are not immediately interoperable

## Immediate operands

_TBD_ value range, depends on instruction encoding.

- `ImmLaneIdxV8`: lane index for 8-bit lanes
- `ImmLaneIdxV16`: lane index for 16-bit lanes
- `ImmLaneIdxV32`: lane index for 32-bit lanes
- `ImmLaneIdxV64`: lane index for 64-bit lanes

## Operations

Completely new operations introduced in this proposal are the operations that
provide interface to vector length.

### Vector length

Querying length of supported vector:

- `vec.i8.length -> i32`
- `vec.i16.length -> i32`
- `vec.i32.length -> i32`
- `vec.i64.length -> i32`
- `vec.f32.length -> i32`
- `vec.f64.length -> i32`

### Constructing vector values

Create vector with identical lanes:

- `vec.i8.splat(x:i32) -> vec.i8`
- `vec.i16.splat(x:i32) -> vec.i16`
- `vec.i32.splat(x:i32) -> vec.i32`
- `vec.i64.splat(x:i64) -> vec.i64`
- `vec.f32.splat(x:f32) -> vec.f32`
- `vec.f64.splat(x:f64) -> vec.f64`

Construct vector with `x` replicated to all lanes:

```python
def S.splat(x):
    result = S.New()
    for i in range(S.Lanes):
        result[i] = x
    return result
```

### Accessing lanes

#### Extract lane as a scalar

- `vec.i8.extract_lane_s(a: vec.i8, imm: ImmLaneIdxV8) -> i32`
- `vec.i8.extract_lane_u(a: vec.i8, imm: ImmLaneIdxV8) -> i32`
- `vec.i16.extract_lane_s(a: vec.i16, imm: ImmLaneIdxV16) -> i32`
- `vec.i16.extract_lane_u(a: vec.i16, imm: ImmLaneIdxV16) -> i32`
- `vec.i32.extract_lane(a: vec.i32, imm: ImmLaneIdxV32) -> i32`
- `vec.i64.extract_lane(a: vec.i64, imm: ImmLaneIdxV64) -> i64`
- `vec.f32.extract_lane(a: vec.f32, imm: ImmLaneIdxV32) -> f32`
- `vec.f64.extract_lane(a: vec.f64, imm: ImmLaneIdxV64) -> f64`

Extract the scalar value of lane specified in the immediate mode operand `imm`
in `a`. The `{interpretation}.extract_lane{_s}{_u}` instructions are encoded
with one immediate byte providing the index of the lane to extract.

```python
def S.extract_lane(a, i):
    return a[i]
```

The `_s` and `_u` variants will sign-extend or zero-extend the lane value to
`i32` respectively.

#### Replace lane value

- `vec.i8.replace_lane(a: vec.i8, imm: ImmLaneIdxV8, x: i32) -> vec.i8`
- `vec.i16.replace_lane(a: vec.i16, imm: ImmLaneIdxV16, x: i32) -> vec.i16`
- `vec.i32.replace_lane(a: vec.i32, imm: ImmLaneIdxV32, x: i32) -> vec.i32`
- `vec.i64.replace_lane(a: vec.i64, imm: ImmLaneIdxV64, x: i64) -> vec.i64`
- `vec.f32.replace_lane(a: vec.f32, imm: ImmLaneIdxV32, x: f32) -> vec.f32`
- `vec.f64.replace_lane(a: vec.f64, imm: ImmLaneIdxV64, x: f64) -> vec.f64`

Return a new vector with lanes identical to `a`, except for the lane specified
in the immediate mode operand `imm` which has the value `x`. The
`{interpretation}.replace_lane` instructions are encoded with an immediate byte 
providing the index of the lane the value of which is to be replaced.

```python
def S.replace_lane(a, i, x):
    result = S.New()
    for j in range(S.Lanes):
        result[j] = a[j]
    result[i] = x
    return result
```

The input lane value, `x`, is interpreted the same way as for the splat
instructions. For the `i8` and `i16` lanes, the high bits of `x` are ignored.


### Integer arithmetic

Wrapping integer arithmetic discards the high bits of the result.

```python
def S.Reduce(x):
    bitmask = (1 << S.LaneBits) - 1
    return x & bitmask
```

Integer division operation is omitted to be compatible with 128-bit SIMD.

#### Integer addition

- `vec.i8.add(a: vec.i8, b: vec.i8) -> vec.i8`
- `vec.i16.add(a: vec.i16, b: vec.i16) -> vec.i16`
- `vec.i32.add(a: vec.i32, b: vec.i32) -> vec.i32`
- `vec.i64.add(a: vec.i64, b: vec.i64) -> vec.i64`

Lane-wise wrapping integer addition:

```python
def S.add(a, b):
    def add(x, y):
        return S.Reduce(x + y)
    return S.lanewise_binary(add, a, b)
```

#### Integer subtraction

- `vec.i8.sub(a: vec.i8, b: vec.i8) -> vec.i8`
- `vec.i16.sub(a: vec.i16, b: vec.i16) -> vec.i16`
- `vec.i32.sub(a: vec.i32, b: vec.i32) -> vec.i32`
- `vec.i64.sub(a: vec.i64, b: vec.i64) -> vec.i64`

Lane-wise wrapping integer subtraction:

```python
def S.sub(a, b):
    def sub(x, y):
        return S.Reduce(x - y)
    return S.lanewise_binary(sub, a, b)
```

#### Integer multiplication

- `vec.i8.mul(a: vec.i8, b: vec.i8) -> vec.i8`
- `vec.i16.mul(a: vec.i16, b: vec.i16) -> vec.i16`
- `vec.i32.mul(a: vec.i32, b: vec.i32) -> vec.i32`
- `vec.i64.mul(a: vec.i64, b: vec.i64) -> vec.i64`

Lane-wise wrapping integer multiplication:

```python
def S.mul(a, b):
    def mul(x, y):
        return S.Reduce(x * y)
    return S.lanewise_binary(mul, a, b)
```

#### Integer negation

- `vec.i8.neg(a: vec.i8, b: vec.i8) -> vec.i8`
- `vec.i16.neg(a: vec.i16, b: vec.i16) -> vec.i16`
- `vec.i32.neg(a: vec.i32, b: vec.i32) -> vec.i32`
- `vec.i64.neg(a: vec.i64, b: vec.i64) -> vec.i64`

Lane-wise wrapping integer negation. In wrapping arithmetic, `y = -x` is the
unique value such that `x + y == 0`.

```python
def S.neg(a):
    def neg(x):
        return S.Reduce(-x)
    return S.lanewise_unary(neg, a)
```

### Saturating integer arithmetic

Saturating integer arithmetic behaves differently on signed and unsigned lanes.

```python
def S.SignedSaturate(x):
    if x < S.Smin:
        return S.Smin
    if x > S.Smax:
        return S.Smax
    return x

def S.UnsignedSaturate(x):
    if x < 0:
        return 0
    if x > S.Umax:
        return S.Umax
    return x
```

### Saturating integer addition
* `vec.i8.add_sat_s(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.add_sat_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.add_sat_s(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i16.add_sat_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.add_sat_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i32.add_sat_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.add_sat_u(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.i64.add_sat_u(a: vec.i64, b: vec.i64) -> vec.i64`

Lane-wise saturating addition:

```python
def S.add_sat_s(a, b):
    def addsat(x, y):
        return S.SignedSaturate(x + y)
    return S.lanewise_binary(addsat, S.AsSigned(a), S.AsSigned(b))

def S.add_sat_u(a, b):
    def addsat(x, y):
        return S.UnsignedSaturate(x + y)
    return S.lanewise_binary(addsat, S.AsUnsigned(a), S.AsUnsigned(b))
```

### Saturating integer subtraction
* `vec.i8.sub_sat_s(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.sub_sat_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.sub_sat_s(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i16.sub_sat_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.sub_sat_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i32.sub_sat_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.sub_sat_u(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.i64.sub_sat_u(a: vec.i64, b: vec.i64) -> vec.i64`

Lane-wise saturating subtraction:

```python
def S.sub_sat_s(a, b):
    def subsat(x, y):
        return S.SignedSaturate(x - y)
    return S.lanewise_binary(subsat, S.AsSigned(a), S.AsSigned(b))

def S.sub_sat_u(a, b):
    def subsat(x, y):
        return S.UnsignedSaturate(x - y)
    return S.lanewise_binary(subsat, S.AsUnsigned(a), S.AsUnsigned(b))
```

### Lane-wise integer minimum
* `vec.i8.min_s(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.min_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.min_s(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i16.min_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.min_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i32.min_u(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.min_s(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.i64.min_u(a: vec.i64, b: vec.i64) -> vec.i64`

Compares lane-wise signed/unsigned integers, and returns the minimum of
each pair.

```python
def S.min(a, b):
    return S.lanewise_binary(min, a, b)
```

### Lane-wise integer maximum
* `vec.i8.max_s(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.max_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.max_s(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i16.max_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.max_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i32.max_u(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.max_s(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.i64.max_u(a: vec.i64, b: vec.i64) -> vec.i64`

Compares lane-wise signed/unsigned integers, and returns the maximum of
each pair.

```python
def S.max(a, b):
    return S.lanewise_binary(max, a, b)
```

### Lane-wise integer rounding average
* `vec.i8.avgr_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.avgr_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.avgr_u(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.avgr_u(a: vec.i64, b: vec.i64) -> vec.i64`

Lane-wise rounding average:

```python
def S.RoundingAverage(x, y):
    return (x + y + 1) // 2

def S.avgr_u(a, b):
    return S.lanewise_binary(S.RoundingAverage, S.AsUnsigned(a), S.AsUnsigned(b))
```

### Lane-wise integer absolute value
* `vec.i8.abs(a: vec.i8) -> vec.i8`
* `vec.i16.abs(a: vec.i16) -> vec.i16`
* `vec.i32.abs(a: vec.i32) -> vec.i32`
* `vec.i64.abs(a: vec.i64) -> vec.i64`

Lane-wise wrapping absolute value.

```python
def S.abs(a):
    return S.lanewise_unary(abs, S.AsSigned(a))
```


### Bit shifts

### Left shift by scalar
* `vec.i8.shl(a: vec.i8, y: i32) -> vec.i8`
* `vec.i16.shl(a: vec.i16, y: i32) -> vec.i16`
* `vec.i32.shl(a: vec.i32, y: i32) -> vec.i32`
* `vec.i64.shl(a: vec.i64, y: i32) -> vec.i64`

Shift the bits in each lane to the left by the same amount. The shift count is
taken modulo lane width:

```python
def S.shl(a, y):
    # Number of bits to shift: 0 .. S.LaneBits - 1.
    amount = y mod S.LaneBits
    def shift(x):
        return S.Reduce(x << amount)
    return S.lanewise_unary(shift, a)
```

### Right shift by scalar
* `vec.i8.shr_s(a: vec.i8, y: i32) -> vec.i8`
* `vec.i8.shr_u(a: vec.i8, y: i32) -> vec.i8`
* `vec.i16.shr_s(a: vec.i16, y: i32) -> vec.i16`
* `vec.i16.shr_u(a: vec.i16, y: i32) -> vec.i16`
* `vec.i32.shr_s(a: vec.i32, y: i32) -> vec.i32`
* `vec.i32.shr_u(a: vec.i32, y: i32) -> vec.i32`
* `vec.i64.shr_s(a: vec.i64, y: i32) -> vec.i64`
* `vec.i64.shr_u(a: vec.i64, y: i32) -> vec.i64`

Shift the bits in each lane to the right by the same amount. The shift count is
taken modulo lane width.  This is an arithmetic right shift for the `_s`
variants and a logical right shift for the `_u` variants.

```python
def S.shr_s(a, y):
    # Number of bits to shift: 0 .. S.LaneBits - 1.
    amount = y mod S.LaneBits
    def shift(x):
        return x >> amount
    return S.lanewise_unary(shift, S.AsSigned(a))

def S.shr_u(a, y):
    # Number of bits to shift: 0 .. S.LaneBits - 1.
    amount = y mod S.LaneBits
    def shift(x):
        return x >> amount
    return S.lanewise_unary(shift, S.AsUnsigned(a))
```


## Bitwise operations

_TBD_

### Boolean horizontal reductions

_TBD_

### Comparisons

_TBD_


### Load and store

- `vec.v8.load(memarg) -> vec.v8`
- `vec.v16.load(memarg) -> vec.v16`
- `vec.v32.load(memarg) -> vec.v32`
- `vec.v64.load(memarg) -> vec.v64`

Load a vector from the given heap address.

- `vec.v8.store(memarg, data:vec.v8)`
- `vec.v16.store(memarg, data:vec.v16)`
- `vec.v32.store(memarg, data:vec.v32)`
- `vec.v64.store(memarg, data:vec.v64)`

Store a vector to the given heap address.

### Floating-point sign bit operations

_TBD_

### Floating-point min and max

_TBD_

### Floating-point arithmetic

The floating-point arithmetic operations are all lane-wise versions of the
existing scalar WebAssembly operations.

#### Addition

- `vec.f32.add(a: vec.f32, b: vec.f32) -> vec.f32`
- `vec.f64.add(a: vec.f64, b: vec.f64) -> vec.f64`

Lane-wise IEEE `addition`.

#### Subtraction

- `vec.f32.sub(a: vec.f32, b: vec.f32) -> vec.f32`
- `vec.f64.sub(a: vec.f64, b: vec.f64) -> vec.f64`

Lane-wise IEEE `subtraction`.

#### Division

- `vec.f32.div(a: vec.f32, b: vec.f32) -> vec.f32`
- `vec.f64.div(a: vec.f64, b: vec.f64) -> vec.f64`

Lane-wise IEEE `division`.

#### Multiplication

- `vec.f32.mul(a: vec.f32, b: vec.f32) -> vec.f32`
- `vec.f64.mul(a: vec.f64, b: vec.f64) -> vec.f64`

Lane-wise IEEE `multiplication`.

#### Square root

- `vec.f32.sqrt(a: vec.f32, b: vec.f32) -> vec.f32`
- `vec.f64.sqrt(a: vec.f64, b: vec.f64) -> vec.f64`

Lane-wise IEEE `squareRoot`.


### Conversions

_TBD_

### Setting vector length

_TBD whether this should be included_

- 8-bit lanes
  - `vec.i8.set_length(len: i32) -> i32`
  - `vec.i8.set_length_imm(imm: ImmLaneIdx8) -> i32`
- 16-bit lanes
  - `vec.i16.set_length(len: i32) -> i32`
  - `vec.i16.set_length_imm(imm: ImmLaneIdx16) -> i32`
- 32-bit lanes
  - `vec.i32.set_length(len: i32) -> i32`
  - `vec.i32.set_length_imm(imm: ImmLaneIdx32) -> i32`
  - `vec.f32.set_length(len: i32) -> i32`
  - `vec.f32.set_length_imm(imm: ImmLaneIdx32) -> i32`
- 64-bit lanes
  - `vec.i64.set_length(len: i32) -> i32`
  - `vec.i64.set_length_imm(imm: ImmLaneIdx64) -> i32`
  - `vec.f64.set_length(len: i32) -> i32`
  - `vec.f64.set_length_imm(imm: ImmLaneIdx64) -> i32`

The above operations set the number of lanes for corresponding vector type to
the minimum of supported vector length and the requested length. The length is
then returned on the stack.

This sets number of lanes for vector operations working on corresponding vector
types. Setting vector length to zero turns corresponding vector operations
(aside of set length) into NOPs.

The following sections describe operations that work on flexible vector types.


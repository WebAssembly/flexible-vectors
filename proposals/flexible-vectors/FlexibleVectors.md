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

- `vec.v8` : 8-bit lanes
- `vec.v16`: 16-bit lanes
- `vec.v32`: 32-bit lanes
- `vec.v64`: 64-bit lanes

### Lane division interpretation

In semantic pseudocode `S` is the particular vector type, `S.LaneBits` is the
size of the lane in bits, `S.Lanes` is the number of lanes, which is dynamic.

|    S      | S.LaneBits |
|-----------|-----------:|
| `vec.v8`  |          8 |
| `vec.v16` |         16 |
| `vec.v32` |         32 |
| `vec.v64` |         64 |

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

- `vec.v8.length -> i32`
- `vec.v16.length -> i32`
- `vec.v32.length -> i32`
- `vec.v64.length -> i32`

### Constructing vector values

Create vector with identical lanes:

- `vec.v8.splat(x:i32) -> vec.v8`
- `vec.v16.splat(x:i32) -> vec.v16`
- `vec.v32.splat(x:i32) -> vec.v32`
- `vec.v64.splat(x:i64) -> vec.v64`

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

- `vec.i8.extract_lane_s(a: vec.v8, imm: ImmLaneIdxV8) -> i32`
- `vec.i8.extract_lane_u(a: vec.v8, imm: ImmLaneIdxV8) -> i32`
- `vec.i16.extract_lane_s(a: vec.v16, imm: ImmLaneIdxV16) -> i32`
- `vec.i16.extract_lane_u(a: vec.v16, imm: ImmLaneIdxV16) -> i32`
- `vec.i32.extract_lane(a: vec.v32, imm: ImmLaneIdxV32) -> i32`
- `vec.i64.extract_lane(a: vec.v64, imm: ImmLaneIdxV64) -> i64`
- `vec.f32.extract_lane(a: vec.v32, imm: ImmLaneIdxV32) -> f32`
- `vec.f64.extract_lane(a: vec.v64, imm: ImmLaneIdxV64) -> f64`

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

- `vec.i8.replace_lane(a: vec.v8, imm: ImmLaneIdxV8, x: i32) -> vec.v8`
- `vec.i16.replace_lane(a: vec.v16, imm: ImmLaneIdxV16, x: i32) -> vec.v16`
- `vec.i32.replace_lane(a: vec.v32, imm: ImmLaneIdxV32, x: i32) -> vec.v32`
- `vec.i64.replace_lane(a: vec.v64, imm: ImmLaneIdxV64, x: i64) -> vec.v64`
- `vec.f32.replace_lane(a: vec.v32, imm: ImmLaneIdxV32, x: f32) -> vec.v32`
- `vec.f64.replace_lane(a: vec.v64, imm: ImmLaneIdxV64, x: f64) -> vec.v64`

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

- `vec.i8.add(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.i16.add(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.i32.add(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.i64.add(a: vec.v64, b: vec.v64) -> vec.v64`

Lane-wise wrapping integer addition:

```python
def S.add(a, b):
    def add(x, y):
        return S.Reduce(x + y)
    return S.lanewise_binary(add, a, b)
```

#### Integer subtraction

- `vec.i8.sub(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.i16.sub(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.i32.sub(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.i64.sub(a: vec.v64, b: vec.v64) -> vec.v64`

Lane-wise wrapping integer subtraction:

```python
def S.sub(a, b):
    def sub(x, y):
        return S.Reduce(x - y)
    return S.lanewise_binary(sub, a, b)
```

#### Integer multiplication

- `vec.i8.mul(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.i16.mul(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.i32.mul(a: vec.v32, b: vec.v32) -> vec.v32`

Lane-wise wrapping integer multiplication:

```python
def S.mul(a, b):
    def mul(x, y):
        return S.Reduce(x * y)
    return S.lanewise_binary(mul, a, b)
```

#### Integer negation

- `vec.i8.neg(a: vec.v8, b: vec.v8) -> vec.v8`
- `vec.i16.neg(a: vec.v16, b: vec.v16) -> vec.v16`
- `vec.i32.neg(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.i64.neg(a: vec.v64, b: vec.v64) -> vec.v64`

Lane-wise wrapping integer negation. In wrapping arithmetic, `y = -x` is the
unique value such that `x + y == 0`.

```python
def S.neg(a):
    def neg(x):
        return S.Reduce(-x)
    return S.lanewise_unary(neg, a)
```

### Saturating integer arithmetic

_TBD_

### Bit shifts

_TBD_

### Bit wise operations

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

- `vec.f32.add(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.f64.add(a: vec.v64, b: vec.v64) -> vec.v64`

Lane-wise IEEE `addition`.

#### Subtraction

- `vec.f32.sub(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.f64.sub(a: vec.v64, b: vec.v64) -> vec.v64`

Lane-wise IEEE `subtraction`.

#### Division

- `vec.f32.div(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.f64.div(a: vec.v64, b: vec.v64) -> vec.v64`

Lane-wise IEEE `division`.

#### Multiplication

- `vec.f32.mul(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.f64.mul(a: vec.v64, b: vec.v64) -> vec.v64`

Lane-wise IEEE `multiplication`.

#### Square root

- `vec.f32.sqrt(a: vec.v32, b: vec.v32) -> vec.v32`
- `vec.f64.sqrt(a: vec.v64, b: vec.v64) -> vec.v64`

Lane-wise IEEE `squareRoot`.


### Conversions

_TBD_

### Setting vector length

_TBD whether this should be included_

- 8-bit lanes
  - `vec.v8.set_length(len: i32) -> i32`
  - `vec.v8.set_length_imm(imm: ImmLaneIdx8) -> i32`
- 16-bit lanes
  - `vec.v16.set_length(len: i32) -> i32`
  - `vec.v16.set_length_imm(imm: ImmLaneIdx16) -> i32`
- 32-bit lanes
  - `vec.v32.set_length(len: i32) -> i32`
  - `vec.v32.set_length_imm(imm: ImmLaneIdx32) -> i32`
- 64-bit lanes
  - `vec.v64.set_length(len: i32) -> i32`
  - `vec.v64.set_length_imm(imm: ImmLaneIdx64) -> i32`

The above operations set the number of lanes for corresponding vector type to
the minimum of supported vector length and the requested length. The length is
then returned on the stack.

This sets number of lanes for vector operations working on corresponding vector
types. Setting vector length to zero turns corresponding vector operations
(aside of set length) into NOPs.

The following sections describe operations that work on flexible vector types.


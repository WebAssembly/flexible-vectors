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

The rest of this document contains instructions that have uncontroversial
lowering on all platforms. There are two more tiers of instructions: second
tier containing instructions with more complex lowering without effect on other
instructions, and third containing instructions affecting execution semantics
or lowering of other instructions.

See [FlexibleVectorsSecondTier.md](FlexibleVectorsSecondTier.md) for the second
tier and [FlexibleVectorsThirdTier.md](FlexibleVectorsThirdTier.md) for the
third tier.

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

## Lane index operands compatible with SIMD

Some operations acess lanes within part of the vector compatible to existing
SIMD standard. Those operands have a limited valid range, and it is a
validation error if the immediate operands are out of range.

* `ImmLaneIdx2`: A byte with values in the range 0–1 identifying a lane.
* `ImmLaneIdx4`: A byte with values in the range 0–3 identifying a lane.
* `ImmLaneIdx8`: A byte with values in the range 0–7 identifying a lane.
* `ImmLaneIdx16`: A byte with values in the range 0–15 identifying a lane.
* `ImmLaneIdx32`: A byte with values in the range 0–31 identifying a lane.

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

#### Extract lane as a scalar using limited immediate index

- `vec.i8.extract_lane_imm_s(a: vec.i8, imm: ImmLaneIdx16) -> i32`
- `vec.i8.extract_lane_imm_u(a: vec.i8, imm: ImmLaneIdx16) -> i32`
- `vec.i16.extract_lane_imm_s(a: vec.i16, imm: ImmLaneIdx8) -> i32`
- `vec.i16.extract_lane_imm_u(a: vec.i16, imm: ImmLaneIdx8) -> i32`
- `vec.i32.extract_lane_imm(a: vec.i32, imm: ImmLaneIdx4) -> i32`
- `vec.i64.extract_lane_imm(a: vec.i64, imm: ImmLaneIdx2) -> i64`
- `vec.f32.extract_lane_imm(a: vec.f32, imm: ImmLaneIdx4) -> f32`
- `vec.f64.extract_lane_imm(a: vec.f64, imm: ImmLaneIdx2) -> f64`

Extract the scalar value of lane specified in the immediate mode operand `imm`
in `a`. The `{interpretation}.extract_lane{_s}{_u}` instructions are encoded
with one immediate byte providing the index of the lane to extract. Index
values outside lower 128 bits of the vector result in a validation error.

```python
def S.extract_lane(a, i):
    return a[i]
```

The `_s` and `_u` variants will sign-extend or zero-extend the lane value to
`i32` respectively.

#### Replace lane using limited immediate index

- `vec.i8.replace_lane_imm(a: vec.i8, imm: ImmLaneIdx16, x: i32) -> vec.i8`
- `vec.i16.replace_lane_imm(a: vec.i16, imm: ImmLaneIdx8, x: i32) -> vec.i16`
- `vec.i32.replace_lane_imm(a: vec.i32, imm: ImmLaneIdx4, x: i32) -> vec.i32`
- `vec.i64.replace_lane_imm(a: vec.i64, imm: ImmLaneIdx2, x: i64) -> vec.i64`
- `vec.f32.replace_lane_imm(a: vec.f32, imm: ImmLaneIdx4, x: f32) -> vec.f32`
- `vec.f64.replace_lane_imm(a: vec.f64, imm: ImmLaneIdx2, x: f64) -> vec.f64`

Return a new vector with lanes identical to `a`, except for the lane specified
in the immediate mode operand `imm` which has the value `x`. The
`{interpretation}.replace_lane` instructions are encoded with an immediate byte 
providing the index of the lane the value of which is to be replaced. Index
values outside lower 128 bits of the vector result in a validation error.

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

### Shuffles

#### Left lane-wise shift by scalar

* `vec.i8.lshl(a: vec.i8, x: i32) -> vec.i8`
* `vec.i16.lshl(a: vec.i16, x: i32) -> vec.i16`
* `vec.i32.lshl(a: vec.i32, x: i32) -> vec.i32`
* `vec.i64.lshl(a: vec.i64, x: i32) -> vec.i64`
* `vec.f32.lshl(a: vec.f32, x: i32) -> vec.f32`
* `vec.f64.lshl(a: vec.f64, x: i32) -> vec.f64`

Returns a new vector with lanes selected from the lanes of input vector `a` by
shifting lanes of the original to the left by the amount specified in the
integer argument and shifting zero values in.

```python
def S.lshl(a, x):
    result = S.New()
    for i in range(S.Lanes):
        if i < x:
            result[i] = 0
        else:
            result[i] = a[i - x]
    return result
```

#### Right lane-wise shift by scalar

* `vec.i8.lshr(a: vec.i8, x: i32) -> vec.i8`
* `vec.i16.lshr(a: vec.i16, x: i32) -> vec.i16`
* `vec.i32.lshr(a: vec.i32, x: i32) -> vec.i32`
* `vec.i64.lshr(a: vec.i64, x: i32) -> vec.i64`
* `vec.f32.lshr(a: vec.f32, x: i32) -> vec.f32`
* `vec.f64.lshr(a: vec.f64, x: i32) -> vec.f64`

Returns a new vector with lanes selected from the lanes of input vector `a` by
shifting lanes of the original to the right by the amount specified in the
integer argument and shifting zero values in.

```python
def S.lshr(a, x):
    result = S.New()
    for i in range(S.Lanes):
        if i < S.Lanes - x:
            result[i] = a[i + x]
        else:
            result[i] = 0
    return result
```

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

#### Lane-wise integer minimum

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

#### Lane-wise integer maximum

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

#### Lane-wise integer rounding average

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

#### Lane-wise integer absolute value

* `vec.i8.abs(a: vec.i8) -> vec.i8`
* `vec.i16.abs(a: vec.i16) -> vec.i16`
* `vec.i32.abs(a: vec.i32) -> vec.i32`
* `vec.i64.abs(a: vec.i64) -> vec.i64`

Lane-wise wrapping absolute value.

```python
def S.abs(a):
    return S.lanewise_unary(abs, S.AsSigned(a))
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

#### Saturating integer addition

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

#### Saturating integer subtraction

* `vec.i8.sub_sat_s(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.sub_sat_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.sub_sat_s(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i16.sub_sat_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.sub_sat_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i32.sub_sat_u(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.sub_sat_s(a: vec.i64, b: vec.i64) -> vec.i64`
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



### Bit shifts

#### Left shift by scalar

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

#### Right shift by scalar

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


### Bitwise operations

#### Bitwise logic

* `vec.i8.and(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.or(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.xor(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.not(a: vec.i8) -> vec.i8`

The logical operations defined on the scalar integer types are also available
on the `v128` type where they operate bitwise the same way C's `&`, `|`, `^`,
and `~` operators work on an `unsigned` type.

#### Bitwise AND-NOT

* `vec.i8.andnot(a: vec.i8, b: vec.i8) -> vec.i8`

Bitwise AND of bits of `a` and the logical inverse of bits of `b`. This operation is equivalent to `vec.i8.and(a, vec.i8.not(b))`.

#### Bitwise select

* `vec.i8.bitselect(v1: vec.i8, v2: vec.i8, c: vec.i8) -> vec.i8`

Use the bits in the control mask `c` to select the corresponding bit from `v1`
when 1 and `v2` when 0.
This is the same as `vec.i8.or(vec.i8.and(v1, c), vec.i8.and(v2, vec.i8.not(c)))`.

Note that the normal WebAssembly `select` instruction also works with vector
types. It selects between two whole vectors controlled by a single scalar value,
rather than selecting bits controlled by a control mask vector.

### Boolean horizontal reductions

These operations reduce all the lanes of an integer vector to a single scalar
0 or 1 value. A lane is considered "true" if it is non-zero.

#### Any lane true

* `vec.i8.any_true(a: vec.i8) -> i32`
* `vec.i16.any_true(a: vec.i16) -> i32`
* `vec.i32.any_true(a: vec.i32) -> i32`

These functions return 1 if any lane in `a` is non-zero, 0 otherwise.

```python
def S.any_true(a):
    for i in range(S.Lanes):
        if a[i] != 0:
            return 1
    return 0
```

#### All lanes true

* `vec.i8.all_true(a: vec.i8) -> i32`
* `vec.i16.all_true(a: vec.i16) -> i32`
* `vec.i32.all_true(a: vec.i32) -> i32`

These functions return 1 if all lanes in `a` are non-zero, 0 otherwise.

```python
def S.all_true(a):
    for i in range(S.Lanes):
        if a[i] == 0:
            return 0
    return 1
```

### Comparisons

The comparison operations all compare two vectors lane-wise, and produce a mask
vector with the same number of lanes as the input interpretation where the bits
in each lane are `0` for `false` and all ones for `true`.

<details>
  <summary>Implementation notes</summary>

  Some classes of comparison operations (for example in AVX512 and SVE) return
  a mask while others return a vector containing results in its lanes. This
  section mightn need to be tuned.

</details>

#### Equality

* `vec.i8.eq(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.eq(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.eq(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.eq(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.f32.eq(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.eq(a: vec.f64, b: vec.f64) -> vec.f64`

Integer equality is independent of the signed/unsigned interpretation. Floating
point equality follows IEEE semantics, so a NaN lane compares not equal with
anything, including itself, and +0.0 is equal to -0.0:

```python
def S.eq(a, b):
    def eq(x, y):
        return x == y
    return S.lanewise_comparison(eq, a, b)
```

#### Non-equality

* `vec.i8.ne(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.ne(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.ne(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.ne(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.f32.ne(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.ne(a: vec.f64, b: vec.f64) -> vec.f64`

The `ne` operations produce the inverse of their `eq` counterparts:

```python
def S.ne(a, b):
    def ne(x, y):
        return x != y
    return S.lanewise_comparison(ne, a, b)
```

#### Less than

* `vec.i8.lt_s(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.lt_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.lt_s(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i16.lt_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.lt_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i32.lt_u(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.lt_s(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.i64.lt_u(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.f32.lt(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.lt(a: vec.f64, b: vec.f64) -> vec.f64`

#### Less than or equal

* `vec.i8.le_s(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.le_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.le_s(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i16.le_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.le_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i32.le_u(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.le_s(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.i64.le_u(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.f32.le(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.le(a: vec.f64, b: vec.f64) -> vec.f64`

#### Greater than

* `vec.i8.gt_s(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.gt_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.gt_s(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i16.gt_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.gt_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i32.gt_u(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.gt_s(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.f32.gt(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.gt(a: vec.f64, b: vec.f64) -> vec.f64`

#### Greater than or equal

* `vec.i8.ge_s(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i8.ge_u(a: vec.i8, b: vec.i8) -> vec.i8`
* `vec.i16.ge_s(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i16.ge_u(a: vec.i16, b: vec.i16) -> vec.i16`
* `vec.i32.ge_s(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i32.ge_u(a: vec.i32, b: vec.i32) -> vec.i32`
* `vec.i64.ge_s(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.i64.ge_u(a: vec.i64, b: vec.i64) -> vec.i64`
* `vec.f32.ge(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.ge(a: vec.f64, b: vec.f64) -> vec.f64`

#### Load and store

- `vec.i8.load(memarg) -> vec.i8`
- `vec.i16.load(memarg) -> vec.i16`
- `vec.i32.load(memarg) -> vec.i32`
- `vec.i64.load(memarg) -> vec.i64`
- `vec.f32.load(memarg) -> vec.f32`
- `vec.f64.load(memarg) -> vec.f64`

Load a vector from the given heap address.

- `vec.i8.store(memarg, data:vec.i8)`
- `vec.i16.store(memarg, data:vec.i16)`
- `vec.i32.store(memarg, data:vec.i32)`
- `vec.i64.store(memarg, data:vec.i64)`
- `vec.f32.store(memarg, data:vec.f32)`
- `vec.f64.store(memarg, data:vec.f64)`

Store a vector to the given heap address.

### Floating-point sign bit operations

These floating point operations are simple manipulations of the sign bit. No
changes are made to the exponent or trailing significand bits, even for NaN
inputs.

#### Negation

* `vec.f32.neg(a: vec.f32) -> vec.f32`
* `vec.f64.neg(a: vec.f64) -> vec.f64`

Apply the IEEE `negate(x)` function to each lane. This simply inverts the sign
bit, preserving all other bits.

```python
def S.neg(a):
    return S.lanewise_unary(ieee.negate, a)
```

#### Floating-point absolute value

* `vec.f32.abs(a: vec.f32) -> vec.f32`
* `vec.f64.abs(a: vec.f64) -> vec.f64`

Apply the IEEE `abs(x)` function to each lane. This simply clears the sign bit,
preserving all other bits.

```python
def S.abs(a):
    return S.lanewise_unary(ieee.abs, a)
```

### Floating-point min and max

#### Pseudo-minimum

* `vec.f32.pmin(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.pmin(a: vec.f64, b: vec.f64) -> vec.f64`

Lane-wise minimum value, defined as `b < a ? b : a`.

#### Pseudo-maximum

* `vec.f32.pmax(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.pmax(a: vec.f64, b: vec.f64) -> vec.f64`

Lane-wise maximum value, defined as `a < b ? b : a`.

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

#### Integer to floating point

* `vec.f32.convert_s(a: vec.i32) -> vec.f32`
* `vec.f64.convert_s(a: vec.i64) -> vec.f64`

Lane-wise conversion from integer to floating point. Some integer values will be
rounded.

#### Integer to integer narrowing

* `vec.i16.narrow_s(a: vec.i16, b: vec.i16) -> vec.i8`
* `vec.i16.narrow_u(a: vec.i16, b: vec.i16) -> vec.i8`
* `vec.i32.narrow_s(a: vec.i32, b: vec.i32) -> vec.i16`
* `vec.i32.narrow_u(a: vec.i32, b: vec.i32) -> vec.i16`
* `vec.i64.narrow_s(a: vec.i64, b: vec.i64) -> vec.i32`
* `vec.i64.narrow_u(a: vec.i64, b: vec.i64) -> vec.i32`

Converts two input vectors into a smaller lane vector by narrowing each lane,
signed or unsigned. The signed narrowing operation will use signed saturation
to handle overflow, 0x7f or 0x80 for i8x16, the unsigned narrowing operation
will use unsigned saturation to handle overflow, 0x00 or 0xff for i8x16.
Regardless of the whether the operation is signed or unsigned, the input lanes
are interpreted as signed integers.

```python
def S.narrow_T_s(a, b):
    result = S.New()
    for i in range(T.Lanes):
        result[i] = S.SignedSaturate(a[i])
    for i in range(T.Lanes):
        result[T.Lanes + i] = S.SignedSaturate(b[i])
    return result

def S.narrow_T_u(a, b):
    result = S.New()
    for i in range(T.Lanes):
        result[i] = S.UnsignedSaturate(a[i])
    for i in range(T.Lanes):
        result[T.Lanes + i] = S.UnsignedSaturate(b[i])
    return result
```

#### Integer to integer widening

* `vec.i8.widen_low_s(a: vec.i8) -> vec.i16`
* `vec.i8.widen_high_s(a: vec.i8) -> vec.i16`
* `vec.i8.widen_low_u(a: vec.i8) -> vec.i16`
* `vec.i8.widen_high_u(a: vec.i8) -> vec.i16`
* `vec.i16.widen_low_s(a: vec.i16) -> vec.i32`
* `vec.i16.widen_high_s(a: vec.i16) -> vec.i32`
* `vec.i16.widen_low_u(a: vec.i16) -> vec.i32`
* `vec.i16.widen_high_u(a: vec.i16) -> vec.i32`
* `vec.i32.widen_low_s(a: vec.i32) -> vec.i64`
* `vec.i32.widen_high_s(a: vec.i32) -> vec.i64`
* `vec.i32.widen_low_u(a: vec.i32) -> vec.i64`
* `vec.i32.widen_high_u(a: vec.i32) -> vec.i64`

Converts low or high half of the smaller lane vector to a larger lane vector,
sign extended or zero (unsigned) extended.

```python
def S.widen_low_T(ext, a):
    result = S.New()
    for i in range(S.Lanes):
        result[i] = ext(a[i])

def S.widen_high_T(ext, a):
    result = S.New()
    for i in range(S.Lanes):
        result[i] = ext(a[S.Lanes + i])

def S.widen_low_T_s(a):
    return S.widen_low_T(Sext, a)

def S.widen_high_T_s(a):
    return S.widen_high_T(Sext, a)

def S.widen_low_T_u(a):
    return S.widen_low_T(Zext, a)

def S.widen_high_T_u(a):
    return S.widen_high_T(Zext, a)
```


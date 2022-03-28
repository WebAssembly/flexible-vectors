Instructions considered conditionally
=====================================

This document describes instructions considered conditionally pending
performance data. The reasons are listed in "implementation notes" sections.

## Operations

### Accessing lanes

#### Extract lane as a scalar

- `vec.i8.extract_lane_s(a: vec.i8, idx: i32) -> i32`
- `vec.i8.extract_lane_u(a: vec.i8, idx: i32) -> i32`
- `vec.i16.extract_lane_s(a: vec.i16, idx: i32) -> i32`
- `vec.i16.extract_lane_u(a: vec.i16, idx: i32) -> i32`
- `vec.i32.extract_lane(a: vec.i32, idx: i32) -> i32`
- `vec.i64.extract_lane(a: vec.i64, idx: i32) -> i64`
- `vec.f32.extract_lane(a: vec.f32, idx: i32) -> f32`
- `vec.f64.extract_lane(a: vec.f64, idx: i32) -> f64`

Extract the scalar value of lane specified in `idx` operand in `a`. Trap if the
index is out of bounds for the vector type.

```python
def S.extract_lane(a, i):
    assert(i < S.Lanes)
    return a[i]
```

The `_s` and `_u` variants will sign-extend or zero-extend the lane value to
`i32` respectively.

<details>
  <summary>Implementation notes</summary>

  Ensuring that lane index is within bounds requires a runtime check. Such
  check can be elided, but would require at least some dataflow analysis.

</details>

#### Replace lane value

- `vec.i8.replace_lane(a: vec.i8, idx: i32, x: i32) -> vec.i8`
- `vec.i16.replace_lane(a: vec.i16, idx: i32, x: i32) -> vec.i16`
- `vec.i32.replace_lane(a: vec.i32, idx: i32, x: i32) -> vec.i32`
- `vec.i64.replace_lane(a: vec.i64, idx: i32, x: i64) -> vec.i64`
- `vec.f32.replace_lane(a: vec.f32, idx: i32, x: f32) -> vec.f32`
- `vec.f64.replace_lane(a: vec.f64, idx: i32, x: f64) -> vec.f64`

Return a new vector with lanes identical to `a`, except for the lane specified
by `idx` operand which gets the value `x`. Trap if the index is out of bounds
for the vector type.

```python
def S.replace_lane(a, i, x):
    assert(i < S.Lanes)
    result = S.New()
    for j in range(S.Lanes):
        result[j] = a[j]
    result[i] = x
    return result
```

The input lane value, `x`, is interpreted the same way as for the splat
instructions. For the `i8` and `i16` lanes, the high bits of `x` are ignored.

<details>
  <summary>Implementation notes</summary>

  Ensuring that lane index is within bounds requires a runtime check. Such
  check can be elided, but would require at least some dataflow analysis.

</details>

#### Extract lane as a scalar, modulo lane count

- `vec.i8.extract_lane_mod_s(a: vec.i8, idx: i32) -> i32`
- `vec.i8.extract_lane_mod_u(a: vec.i8, idx: i32) -> i32`
- `vec.i16.extract_lane_mod_s(a: vec.i16, idx: i32) -> i32`
- `vec.i16.extract_lane_mod_u(a: vec.i16, idx: i32) -> i32`
- `vec.i32.extract_lane_mod(a: vec.i32, idx: i32) -> i32`
- `vec.i64.extract_lane_mod(a: vec.i64, idx: i32) -> i64`
- `vec.f32.extract_lane_mod(a: vec.f32, idx: i32) -> f32`
- `vec.f64.extract_lane_mod(a: vec.f64, idx: i32) -> f64`

Extract the scalar value of lane specified in `idx` operand in `a`, modulo the
number of lanes.

```python
def S.extract_lane_mod(a, i):
    return a[i % S.Lanes]
```

The `_s` and `_u` variants will sign-extend or zero-extend the lane value to
`i32` respectively.

<details>
  <summary>Implementation notes</summary>

  Taking index modulo the lane count would add an extra operation, which may or
  may not be elided. Additional concern is that modulo semantics are
  potentially more confusing than checked semantics.

</details>

#### Replace lane value, modulo lane count

- `vec.i8.replace_lane_mod(a: vec.i8, idx: i32, x: i32) -> vec.i8`
- `vec.i16.replace_lane_mod(a: vec.i16, idx: i32, x: i32) -> vec.i16`
- `vec.i32.replace_lane_mod(a: vec.i32, idx: i32, x: i32) -> vec.i32`
- `vec.i64.replace_lane_mod(a: vec.i64, idx: i32, x: i64) -> vec.i64`
- `vec.f32.replace_lane_mod(a: vec.f32, idx: i32, x: f32) -> vec.f32`
- `vec.f64.replace_lane_mod(a: vec.f64, idx: i32, x: f64) -> vec.f64`

Return a new vector with lanes identical to `a`, except for the lane specified
by `idx` operand which gets the value `x`, taken modulo the number of lanes.

```python
def S.replace_lane_mod(a, i, x):
    result = S.New()
    for j in range(S.Lanes):
        result[j] = a[j]
    result[i % S.Lanes] = x
    return result
```

The input lane value, `x`, is interpreted the same way as for the splat
instructions. For the `i8` and `i16` lanes, the high bits of `x` are ignored.

<details>
  <summary>Implementation notes</summary>

  Taking index modulo the lane count would add an extra operation, which may or
  may not be elided. Additional concern is that modulo semantics are
  potentially more confusing than checked semantics.

</details>

### Floating-point min and max

These operations are not part of the IEEE 754-2008 standard. They are lane-wise
versions of the existing scalar WebAssembly operations.

<details>
  <summary>Implementation notes</summary>

  NaN queting required for these operation is expensive on x86-based platforms,
  at least without AVX512. See [WebAssembly/simd#186](https://github.com/WebAssembly/simd/issues/186).

</details>

#### NaN-propagating minimum

* `vec.f32.min(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.min(a: vec.f64, b: vec.f64) -> vec.f64`

Lane-wise minimum value, propagating NaNs.

#### NaN-propagating maximum

* `vec.f32.max(a: vec.f32, b: vec.f32) -> vec.f32`
* `vec.f64.max(a: vec.f64, b: vec.f64) -> vec.f64`

Lane-wise maximum value, propagating NaNs.

### Conversions

#### Integer to floating point

* `vec.f32.convert_u(a: vec.i32) -> vec.f32`
* `vec.f64.convert_u(a: vec.i64) -> vec.f64`

Lane-wise conversion from integer to floating point. Some integer values will be
rounded.

#### Floating point to integer with saturation

* `vec.i32.trunc_sat_s(a: vec.f32) -> vec.i32`
* `vec.i32.trunc_sat_u(a: vec.f32) -> vec.i32`
* `vec.i64.trunc_sat_s(a: vec.f64) -> vec.i64`
* `vec.i64.trunc_sat_u(a: vec.f64) -> vec.i64`

Lane-wise saturating conversion from floating point to integer using the IEEE
`convertToIntegerTowardZero` function. If any input lane is a NaN, the
resulting lane is 0. If the rounded integer value of a lane is outside the
range of the destination type, the result is saturated to the nearest
representable integer value.


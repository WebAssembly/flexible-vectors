Instructions considered conditionally
=====================================

This document describes instructions considered conditionally pending
performance data. The reasons are listed in "implementation notes" sections.

## Operations

### Floating-point min and max

These operations are not part of the IEEE 754-2008 standard. They are lane-wise
versions of the existing scalar WebAssembly operations.

<details>
  <summary>Implementation notes</summary>

  NaN queting required for these operation is expensive on x86-based platforms.
  See [WebAssembly/simd#186](https://github.com/WebAssembly/simd/issues/186).

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


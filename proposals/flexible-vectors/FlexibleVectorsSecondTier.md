Instructions considered conditionally
=====================================

This document describes instructions considered conditionally pending
performance data. The reasons are listed in "implementation notes" sections.

## Operations

### Floating-point min and max

These operations are not part of the IEEE 754-2008 standard. They are lane-wise
versions of the existing scalar WebAssembly operations.

<details>
  <summary>Implementation notes</summary

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


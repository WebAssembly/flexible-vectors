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

### Setting vector length

<details>
  <summary>Implementation notes</summary

  Dynamic vector length can be implemented using masks, but would introduce
  additional overhead, and would be extremely expensive for instruction sets
  without masks.

</details>


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


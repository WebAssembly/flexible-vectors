Instructions considered conditionally
=====================================

This document describes instructions considered conditionally pending
performance data with implications for other instructions in the proposal. The
reasons are listed in "implementation notes" sections.

## Operations

### Setting vector length

<details>
  <summary>Implementation notes</summary

  Dynamic vector length can be implemented using masks, but would introduce
  additional overhead, and would be extremely expensive for instruction sets
  without masks. Additionaly it introduces global state that would affect other
  vector instructions.

</details>


- 8-bit lanes
  - `vec.i8.set_length(len: i32) -> i32`
  - `vec.i8.set_length_imm(idx: i32) -> i32`
- 16-bit lanes
  - `vec.i16.set_length(len: i32) -> i32`
  - `vec.i16.set_length_imm(idx: i32) -> i32`
- 32-bit lanes
  - `vec.i32.set_length(len: i32) -> i32`
  - `vec.i32.set_length_imm(idx: i32) -> i32`
  - `vec.f32.set_length(len: i32) -> i32`
  - `vec.f32.set_length_imm(idx: i32) -> i32`
- 64-bit lanes
  - `vec.i64.set_length(len: i32) -> i32`
  - `vec.i64.set_length_imm(idx: i32) -> i32`
  - `vec.f64.set_length(len: i32) -> i32`
  - `vec.f64.set_length_imm(idx: i32) -> i32`

The above operations set the number of lanes for corresponding vector type to
the minimum of supported vector length and the requested length. The difference
is then returned on the stack.

This sets number of lanes for vector operations working on corresponding vector
types. Setting vector length to zero turns corresponding vector operations
(aside of set length) into NOPs.


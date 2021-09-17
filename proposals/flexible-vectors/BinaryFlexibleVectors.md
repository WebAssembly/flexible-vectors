Binary encoding of Flexible Vector operations
=============================================

Vector value types:

```
valtype ::= ...
          | 0x7A => vec.i8
          | 0x79 => vec.i16
          | 0x78 => vec.i32
          | 0x77 => vec.i64
          | 0x76 => vec.f32
          | 0x75 => vec.f64
```

## Instruction encoding

Instructions are prefixed with a byte determining which lane type the
instruction supports, the actual opcode is encoded by variable-length unsigned
integer:


```
instr ::= ...
        | 0x7A vecop:varuint32 ... => vec.i8
        | 0x79 vecop:varuint32 ... => vec.i16
        | 0x78 vecop:varuint32 ... => vec.i32
        | 0x77 vecop:varuint32 ... => vec.i64
        | 0x76 vecop:varuint32 ... => vec.f32
        | 0x75 vecop:varuint32 ... => vec.f64
```

Opcodes are assigned in groups each category of instructions.

Vector length:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.i8.length`          |    `0x7A`|   `0x00`| -                        |
| `vec.i16.length`         |    `0x79`|   `0x00`| -                        |
| `vec.i32.length`         |    `0x78`|   `0x00`| -                        |
| `vec.i64.length`         |    `0x77`|   `0x00`| -                        |
| `vec.f32.length`         |    `0x76`|   `0x00`| -                        |
| `vec.f64.length`         |    `0x75`|   `0x00`| -                        |

Constructing vectors and accessing lanes:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.i8.splat`           |    `0x7A`|   `0x10`| -                        |
| `vec.i16.splat`          |    `0x79`|   `0x10`| -                        |
| `vec.i32.splat`          |    `0x78`|   `0x10`| -                        |
| `vec.i64.splat`          |    `0x77`|   `0x10`| -                        |
| `vec.f32.splat`          |    `0x76`|   `0x10`| -                        |
| `vec.f64.splat`          |    `0x75`|   `0x10`| -                        |
| `vec.i8.extract_lane_u`  |    `0x7A`|   `0x11`| i:ImmLaneIdxV8           |
| `vec.i16.extract_lane_u` |    `0x79`|   `0x11`| i:ImmLaneIdxV16          |
| `vec.i32.extract_lane_u` |    `0x78`|   `0x11`| i:ImmLaneIdxV32          |
| `vec.i64.extract_lane_u` |    `0x77`|   `0x11`| i:ImmLaneIdxV64          |
| `vec.f32.extract_lane_u` |    `0x76`|   `0x11`| i:ImmLaneIdxV32          |
| `vec.f64.extract_lane_u` |    `0x75`|   `0x11`| i:ImmLaneIdxV64          |
| `vec.i8.extract_lane_s`  |    `0x7A`|   `0x12`| i:ImmLaneIdxV8           |
| `vec.i16.extract_lane_s` |    `0x79`|   `0x12`| i:ImmLaneIdxV16          |
| `vec.i32.extract_lane_s` |    `0x78`|   `0x12`| i:ImmLaneIdxV32          |
| `vec.i64.extract_lane_s` |    `0x77`|   `0x12`| i:ImmLaneIdxV64          |
| `vec.f32.extract_lane_s` |    `0x76`|   `0x12`| i:ImmLaneIdxV32          |
| `vec.f64.extract_lane_s` |    `0x75`|   `0x12`| i:ImmLaneIdxV64          |
| `vec.i8.replace_lane`    |    `0x7A`|   `0x13`| i:ImmLaneIdxV8           |
| `vec.i16.replace_lane`   |    `0x79`|   `0x13`| i:ImmLaneIdxV16          |
| `vec.i32.replace_lane`   |    `0x78`|   `0x13`| i:ImmLaneIdxV32          |
| `vec.i64.replace_lane`   |    `0x77`|   `0x13`| i:ImmLaneIdxV64          |
| `vec.f32.replace_lane`   |    `0x76`|   `0x13`| i:ImmLaneIdxV32          |
| `vec.f64.replace_lane`   |    `0x75`|   `0x13`| i:ImmLaneIdxV64          |

Shuffles:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.i8.lshl`            |    `0x7A`|   `0x20`| -                        |
| `vec.i16.lshl`           |    `0x79`|   `0x20`| -                        |
| `vec.i32.lshl`           |    `0x78`|   `0x20`| -                        |
| `vec.i64.lshl`           |    `0x77`|   `0x20`| -                        |
| `vec.f32.lshl`           |    `0x76`|   `0x20`| -                        |
| `vec.f64.lshl`           |    `0x75`|   `0x20`| -                        |
| `vec.i8.lshr`            |    `0x7A`|   `0x21`| -                        |
| `vec.i16.lshr`           |    `0x79`|   `0x21`| -                        |
| `vec.i32.lshr`           |    `0x78`|   `0x21`| -                        |
| `vec.i64.lshr`           |    `0x77`|   `0x21`| -                        |
| `vec.f32.lshr`           |    `0x76`|   `0x21`| -                        |
| `vec.f64.lshr`           |    `0x75`|   `0x21`| -                        |

Non-saturating integer arithmetic:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.i8.add`             |    `0x7A`|   `0x30`| -                        |
| `vec.i16.add`            |    `0x79`|   `0x30`| -                        |
| `vec.i32.add`            |    `0x78`|   `0x30`| -                        |
| `vec.i64.add`            |    `0x77`|   `0x30`| -                        |
| `vec.i8.sub`             |    `0x7A`|   `0x31`| -                        |
| `vec.i16.sub`            |    `0x79`|   `0x31`| -                        |
| `vec.i32.sub`            |    `0x78`|   `0x31`| -                        |
| `vec.i64.sub`            |    `0x77`|   `0x31`| -                        |
| `vec.i8.mul`             |    `0x7A`|   `0x32`| -                        |
| `vec.i16.mul`            |    `0x79`|   `0x32`| -                        |
| `vec.i32.mul`            |    `0x78`|   `0x32`| -                        |
| `vec.i64.mul`            |    `0x77`|   `0x32`| -                        |
| `vec.i8.neg`             |    `0x7A`|   `0x33`| -                        |
| `vec.i16.neg`            |    `0x79`|   `0x33`| -                        |
| `vec.i32.neg`            |    `0x78`|   `0x33`| -                        |
| `vec.i64.neg`            |    `0x77`|   `0x33`| -                        |

Saturating integer arithmetic:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.i8.add_sat_u`       |    `0x7A`|   `0x40`| -                        |
| `vec.i16.add_sat_u`      |    `0x79`|   `0x40`| -                        |
| `vec.i32.add_sat_u`      |    `0x78`|   `0x40`| -                        |
| `vec.i64.add_sat_u`      |    `0x77`|   `0x40`| -                        |
| `vec.i8.add_sat_s`       |    `0x7A`|   `0x41`| -                        |
| `vec.i16.add_sat_s`      |    `0x79`|   `0x41`| -                        |
| `vec.i32.add_sat_s`      |    `0x78`|   `0x41`| -                        |
| `vec.i64.add_sat_s`      |    `0x77`|   `0x41`| -                        |
| `vec.i8.sub_sat_u`       |    `0x7A`|   `0x42`| -                        |
| `vec.i16.sub_sat_u`      |    `0x79`|   `0x42`| -                        |
| `vec.i32.sub_sat_u`      |    `0x78`|   `0x42`| -                        |
| `vec.i64.sub_sat_u`      |    `0x77`|   `0x42`| -                        |
| `vec.i8.sub_sat_s`       |    `0x7A`|   `0x43`| -                        |
| `vec.i16.sub_sat_s`      |    `0x79`|   `0x43`| -                        |
| `vec.i32.sub_sat_s`      |    `0x78`|   `0x43`| -                        |
| `vec.i64.sub_sat_s`      |    `0x77`|   `0x43`| -                        |
| `vec.i8.min_u`           |    `0x7A`|   `0x44`| -                        |
| `vec.i16.min_u`          |    `0x79`|   `0x44`| -                        |
| `vec.i32.min_u`          |    `0x78`|   `0x44`| -                        |
| `vec.i64.min_u`          |    `0x77`|   `0x44`| -                        |
| `vec.i8.min_s`           |    `0x7A`|   `0x45`| -                        |
| `vec.i16.min_s`          |    `0x79`|   `0x45`| -                        |
| `vec.i32.min_s`          |    `0x78`|   `0x45`| -                        |
| `vec.i64.min_s`          |    `0x77`|   `0x45`| -                        |
| `vec.i8.max_u`           |    `0x7A`|   `0x46`| -                        |
| `vec.i16.max_u`          |    `0x79`|   `0x46`| -                        |
| `vec.i32.max_u`          |    `0x78`|   `0x46`| -                        |
| `vec.i64.max_u`          |    `0x77`|   `0x46`| -                        |
| `vec.i8.max_s`           |    `0x7A`|   `0x47`| -                        |
| `vec.i16.max_s`          |    `0x79`|   `0x47`| -                        |
| `vec.i32.max_s`          |    `0x78`|   `0x47`| -                        |
| `vec.i64.max_s`          |    `0x77`|   `0x47`| -                        |
| `vec.i8.avgr_u`          |    `0x7A`|   `0x48`| -                        |
| `vec.i16.avgr_u`         |    `0x79`|   `0x48`| -                        |
| `vec.i32.avgr_u`         |    `0x78`|   `0x48`| -                        |
| `vec.i64.avgr_u`         |    `0x77`|   `0x48`| -                        |
| `vec.i8.abs`             |    `0x7A`|   `0x49`| -                        |
| `vec.i16.abs`            |    `0x79`|   `0x49`| -                        |
| `vec.i32.abs`            |    `0x78`|   `0x49`| -                        |
| `vec.i64.abs`            |    `0x77`|   `0x49`| -                        |

Bitwise operations:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.i8.shl`             |    `0x7A`|   `0x50`| -                        |
| `vec.i16.shl`            |    `0x79`|   `0x50`| -                        |
| `vec.i32.shl`            |    `0x78`|   `0x50`| -                        |
| `vec.i64.shl`            |    `0x77`|   `0x50`| -                        |
| `vec.i8.shr_u`           |    `0x7A`|   `0x51`| -                        |
| `vec.i16.shr_u`          |    `0x79`|   `0x51`| -                        |
| `vec.i32.shr_u`          |    `0x78`|   `0x51`| -                        |
| `vec.i64.shr_u`          |    `0x77`|   `0x51`| -                        |
| `vec.i8.shr_s`           |    `0x7A`|   `0x52`| -                        |
| `vec.i16.shr_s`          |    `0x79`|   `0x52`| -                        |
| `vec.i32.shr_s`          |    `0x78`|   `0x52`| -                        |
| `vec.i64.shr_s`          |    `0x77`|   `0x52`| -                        |
| `vec.i8.and`             |    `0x7A`|   `0x53`| -                        |
| `vec.i8.or`              |    `0x7A`|   `0x54`| -                        |
| `vec.i8.xor`             |    `0x7A`|   `0x55`| -                        |
| `vec.i8.not`             |    `0x7A`|   `0x56`| -                        |
| `vec.i8.andnot`          |    `0x7A`|   `0x57`| -                        |
| `vec.i8.bitselect`       |    `0x7A`|   `0x58`| -                        |

Reductions:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.i8.any_true`        |    `0x7A`|   `0x60`| -                        |
| `vec.i16.any_true`       |    `0x79`|   `0x60`| -                        |
| `vec.i32.any_true`       |    `0x78`|   `0x60`| -                        |
| `vec.i8.all_true`        |    `0x7A`|   `0x61`| -                        |
| `vec.i16.all_true`       |    `0x79`|   `0x61`| -                        |
| `vec.i32.all_true`       |    `0x78`|   `0x61`| -                        |

Comparisons:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.i8.eq`              |    `0x7A`|   `0x70`| -                        |
| `vec.i16.eq`             |    `0x79`|   `0x70`| -                        |
| `vec.i32.eq`             |    `0x78`|   `0x70`| -                        |
| `vec.i64.eq`             |    `0x77`|   `0x70`| -                        |
| `vec.f32.eq`             |    `0x76`|   `0x70`| -                        |
| `vec.f64.eq`             |    `0x75`|   `0x70`| -                        |
| `vec.i8.ne`              |    `0x7A`|   `0x71`| -                        |
| `vec.i16.ne`             |    `0x79`|   `0x71`| -                        |
| `vec.i32.ne`             |    `0x78`|   `0x71`| -                        |
| `vec.i64.ne`             |    `0x77`|   `0x71`| -                        |
| `vec.f32.ne`             |    `0x76`|   `0x71`| -                        |
| `vec.f64.ne`             |    `0x75`|   `0x71`| -                        |
| `vec.i8.lt_u`            |    `0x7A`|   `0x72`| -                        |
| `vec.i16.lt_u`           |    `0x79`|   `0x72`| -                        |
| `vec.i32.lt_u`           |    `0x78`|   `0x72`| -                        |
| `vec.i64.lt_u`           |    `0x77`|   `0x72`| -                        |
| `vec.i8.lt_s`            |    `0x7A`|   `0x73`| -                        |
| `vec.i16.lt_s`           |    `0x79`|   `0x73`| -                        |
| `vec.i32.lt_s`           |    `0x78`|   `0x73`| -                        |
| `vec.i64.lt_s`           |    `0x77`|   `0x73`| -                        |
| `vec.f32.lt`             |    `0x76`|   `0x74`| -                        |
| `vec.f64.lt`             |    `0x75`|   `0x74`| -                        |
| `vec.i8.le_u`            |    `0x7A`|   `0x75`| -                        |
| `vec.i16.le_u`           |    `0x79`|   `0x75`| -                        |
| `vec.i32.le_u`           |    `0x78`|   `0x75`| -                        |
| `vec.i64.le_u`           |    `0x77`|   `0x75`| -                        |
| `vec.i8.le_s`            |    `0x7A`|   `0x76`| -                        |
| `vec.i16.le_s`           |    `0x79`|   `0x76`| -                        |
| `vec.i32.le_s`           |    `0x78`|   `0x76`| -                        |
| `vec.i64.le_s`           |    `0x77`|   `0x76`| -                        |
| `vec.f32.le`             |    `0x76`|   `0x77`| -                        |
| `vec.f64.le`             |    `0x75`|   `0x77`| -                        |
| `vec.i8.gt_s`            |    `0x7A`|   `0x78`| -                        |
| `vec.i16.gt_s`           |    `0x79`|   `0x78`| -                        |
| `vec.i32.gt_s`           |    `0x78`|   `0x78`| -                        |
| `vec.i64.gt_s`           |    `0x77`|   `0x78`| -                        |
| `vec.i8.gt_u`            |    `0x7A`|   `0x79`| -                        |
| `vec.i16.gt_u`           |    `0x79`|   `0x79`| -                        |
| `vec.i32.gt_u`           |    `0x78`|   `0x79`| -                        |
| `vec.i64.gt_u`           |    `0x77`|   `0x79`| -                        |
| `vec.f32.gt`             |    `0x76`|   `0x7A`| -                        |
| `vec.f64.gt`             |    `0x75`|   `0x7A`| -                        |
| `vec.i8.ge_u`            |    `0x7A`|   `0x7B`| -                        |
| `vec.i16.ge_u`           |    `0x79`|   `0x7B`| -                        |
| `vec.i32.ge_u`           |    `0x78`|   `0x7B`| -                        |
| `vec.i64.ge_u`           |    `0x77`|   `0x7B`| -                        |
| `vec.i8.ge_s`            |    `0x7A`|   `0x7C`| -                        |
| `vec.i16.ge_s`           |    `0x79`|   `0x7C`| -                        |
| `vec.i32.ge_s`           |    `0x78`|   `0x7C`| -                        |
| `vec.i64.ge_s`           |    `0x77`|   `0x7C`| -                        |
| `vec.f32.ge`             |    `0x76`|   `0x7D`| -                        |
| `vec.f64.ge`             |    `0x75`|   `0x7D`| -                        |

Memory operations:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.i8.load`            |    `0x7A`|   `0x80`| -                        |
| `vec.i16.load`           |    `0x79`|   `0x80`| -                        |
| `vec.i32.load`           |    `0x78`|   `0x80`| -                        |
| `vec.i64.load`           |    `0x77`|   `0x80`| -                        |
| `vec.f32.load`           |    `0x76`|   `0x80`| -                        |
| `vec.f64.load`           |    `0x75`|   `0x80`| -                        |
| `vec.i8.store`           |    `0x7A`|   `0x87`| -                        |
| `vec.i16.store`          |    `0x79`|   `0x87`| -                        |
| `vec.i32.store`          |    `0x78`|   `0x87`| -                        |
| `vec.i64.store`          |    `0x77`|   `0x87`| -                        |
| `vec.f32.store`          |    `0x76`|   `0x87`| -                        |
| `vec.f64.store`          |    `0x75`|   `0x87`| -                        |

Floating-point:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.f32.neg`            |    `0x76`|   `0x90`| -                        |
| `vec.f64.neg`            |    `0x75`|   `0x90`| -                        |
| `vec.f32.abs`            |    `0x76`|   `0x91`| -                        |
| `vec.f64.abs`            |    `0x75`|   `0x91`| -                        |
| `vec.f32.pmin`           |    `0x76`|   `0x92`| -                        |
| `vec.f64.pmin`           |    `0x75`|   `0x92`| -                        |
| `vec.f32.pmax`           |    `0x76`|   `0x93`| -                        |
| `vec.f64.pmax`           |    `0x75`|   `0x93`| -                        |
| `vec.f32.add`            |    `0x76`|   `0x94`| -                        |
| `vec.f64.add`            |    `0x75`|   `0x94`| -                        |
| `vec.f32.sub`            |    `0x76`|   `0x95`| -                        |
| `vec.f64.sub`            |    `0x75`|   `0x95`| -                        |
| `vec.f32.div`            |    `0x76`|   `0x95`| -                        |
| `vec.f64.div`            |    `0x75`|   `0x95`| -                        |
| `vec.f32.mul`            |    `0x76`|   `0x96`| -                        |
| `vec.f64.mul`            |    `0x75`|   `0x96`| -                        |
| `vec.f32.sqrt`           |    `0x76`|   `0x97`| -                        |
| `vec.f64.sqrt`           |    `0x75`|   `0x97`| -                        |

Conversions:

| Instruction              | `prefix` | `vecop` | Immediate operands       |
| -------------------------|---------:|--------:|--------------------------|
| `vec.f32.convert_s`      |    `0x76`|   `0xA0`| -                        |
| `vec.f64.convert_s`      |    `0x75`|   `0xA0`| -                        |
| `vec.i16.narrow_s`       |    `0x79`|   `0xA1`| -                        |
| `vec.i32.narrow_s`       |    `0x78`|   `0xA1`| -                        |
| `vec.i64.narrow_s`       |    `0x77`|   `0xA1`| -                        |
| `vec.i16.narrow_u`       |    `0x79`|   `0xA2`| -                        |
| `vec.i32.narrow_u`       |    `0x78`|   `0xA2`| -                        |
| `vec.i64.narrow_u`       |    `0x77`|   `0xA2`| -                        |
| `vec.i8.widen_low_u`     |    `0x7A`|   `0xA3`| -                        |
| `vec.i16.widen_low_u`    |    `0x79`|   `0xA3`| -                        |
| `vec.i32.widen_low_u`    |    `0x78`|   `0xA3`| -                        |
| `vec.i8.widen_low_s`     |    `0x7A`|   `0xA4`| -                        |
| `vec.i16.widen_low_s`    |    `0x79`|   `0xA4`| -                        |
| `vec.i32.widen_low_s`    |    `0x78`|   `0xA4`| -                        |
| `vec.i8.widen_high_u`    |    `0x7A`|   `0xA5`| -                        |
| `vec.i16.widen_high_u`   |    `0x79`|   `0xA5`| -                        |
| `vec.i32.widen_high_u`   |    `0x78`|   `0xA5`| -                        |
| `vec.i8.widen_high_s`    |    `0x7A`|   `0xA6`| -                        |
| `vec.i16.widen_high_s`   |    `0x79`|   `0xA6`| -                        |
| `vec.i32.widen_high_s`   |    `0x78`|   `0xA6`| -                        |


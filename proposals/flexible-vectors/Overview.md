# Flexible vectors

This proposal extends WebAssembly to support vector operations with over 128-bit length. This document describes high level overview of the proposal, while there are other documents describing specific changes to WebAssembly this is hoping to make.

This is an early stage proposal and it is expected to evolve.

## Objectives

* Support widely available hardware with SIMD length beyond 128 bits.
* Provide WebAssembly operations to cover various SIMD instruction sets beyond what Wasm SIMD already supports.

## Overview

At its core this proposal is implementing length-agnostic versions of instructions in Wasm SIMD. Other instructions will be added as proposal progresses.

Making WebAssembly vector operations length-agnostic would allow them to be executed on top of hardware with very different SIMD implementations.

In the base variant of the instruction set length of the operation would be unknown at compile time, but fixed at runtime. There is a number of possible extensions to this - allowing adjustments to vector length by the program, making extensive use of masks.

Question to be answered: 

- Setting length at runtime and mask support
- Performance portability between various hardware platforms
- Exposing hardware vector length to Wasm (and ultimately JavaScript) code

The plan is to iteratively implement and evaluate editions of the proposal.


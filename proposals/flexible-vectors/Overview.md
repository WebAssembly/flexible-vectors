# Flexible vectors proposal

## Summary

This proposal adds vector operations with variable length.

## Motivation

* Bridging the gap between various popular SIMD instruction sets.
* Supporting SIMD instruction sets beyond 128 bits.

## Overview

This proposal strives to add new Wasm instructions and types for vector
operations with runtime-defined length.

Goals:

* Same Wasm binary to run all platforms.
* Unambiguous instruction selection.
* Easy transition from Wasm SIMD instruction set.


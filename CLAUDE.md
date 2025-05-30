# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

M30 is a new programming language that combines:
- Python's readable syntax with static typing
- ML-style type inference (Hindley-Milner where possible)
- Lisp's live-coding features (REPL-driven development, image-based persistence)
- Modern structured concurrency
- Multiple dispatch for elegant APIs
- Restartable error conditions

The language aims to be the best tool for exploratory programming while maintaining production-ready performance and safety.

## Key Design Documents

- `docs/core-features.md` - Complete feature overview
- `docs/design-v3-examples.md` - Concrete syntax examples
- `docs/grammar-design.md` - Formal grammar specification
- `docs/type-system-integration.md` - How types work with parser
- `docs/m30-vs-rust.md` - Comparison with similar languages
- `examples/showcase.m30` - Example code demonstrating features

## Development Commands

Since this is a language design project, there are no build commands yet. The focus is on:
1. Iterating on language design in the `docs/` directory
2. Creating example programs in `examples/`
3. Planning implementation strategy

Future commands will include:
- `go build` - Build the M30 compiler/interpreter
- `m30 run file.m30` - Run an M30 program
- `m30 repl` - Start interactive REPL
- `m30 fmt file.m30` - Format code (like gofmt)

## Architecture

### Language Architecture
- **Syntax**: Python-like with significant whitespace
- **Type System**: Static with islands of inference (annotations required at module boundaries)
- **Memory**: Garbage collected for simplicity
- **Concurrency**: Structured concurrency with parallel/race blocks
- **Error Handling**: Result types with ? operator + restartable conditions

### Implementation Plan
1. Hand-written recursive descent parser (like Go)
2. Type checker as separate pass after parsing
3. Initial tree-walking interpreter
4. Later: Compile to bytecode or LLVM

## Unique Features

1. **Image-based development**: Save/restore entire REPL sessions
2. **Multiple dispatch**: Methods selected on all argument types
3. **Restartable conditions**: Better error recovery than exceptions
4. **Live code reloading**: Change functions without losing state
5. **Where clauses**: Define helpers after main logic
6. **Comptime evaluation**: Run code at compile time
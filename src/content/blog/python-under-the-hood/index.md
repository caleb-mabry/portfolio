---
title: Python Under the Hood
author: Caleb Mabry
pubDatetime: 2025-12-25T20:14:28Z
slug: python-under-the-hood
featured: true
tags:
  - Python
description:
  "A deep dive into Python's execution model; from source code to bytecode, how the CPython VM works, and why C extensions aren't always faster."
---

# Python Execution Model: From Source Code to Machine Instructions

## Table of Contents

- [Executive Summary](#executive-summary)
- [Introduction](#introduction)
- [The Python Compilation Pipeline](#the-python-compilation-pipeline)
  - [Overview](#overview)
  - [Abstract Syntax Tree Generation](#abstract-syntax-tree-generation)
  - [Bytecode Generation and Caching](#bytecode-generation-and-caching)
  - [Bytecode Inspection and Analysis](#bytecode-inspection-and-analysis)
- [Comparison with Java's Execution Model](#comparison-with-javas-execution-model)
- [C Extension Integration](#c-extension-integration)
- [Conclusion](#conclusion)

## Executive Summary

This document provides a technical overview of Python's execution model, examining the compilation from source code to bytecode, the role of the CPython virtual machine, and integration with native C extensions for performance optimization.

## Introduction

Python's execution model is often mischaracterized as purely interpreted. In reality, CPython (the reference implementation) employs a hybrid approach: source code is compiled to an intermediate bytecode representation, which is then interpreted by a virtual machine. This architecture provides platform independence while maintaining the flexibility of dynamic interpretation.

## The Python Compilation Pipeline

### Overview

When executing a Python script via `python script.py`, the following stages occur:

1. **Lexical Analysis**: Source code tokenization
2. **Parsing**: Construction of an Abstract Syntax Tree (AST)
3. **Compilation**: AST transformation to bytecode
4. **Interpretation**: Bytecode execution by the CPython VM

### Abstract Syntax Tree Generation

You can inspect the AST representation of Python code using the `ast` module:

```python
import ast

source_code = """
def calculate(x, y):
    result = x * 2 + y
    return result
"""

tree = ast.parse(source_code)
print(ast.dump(tree, indent=2))
```

Output:

```python
Module(
  body=[
    FunctionDef(
      name='calculate',
      args=arguments(
        args=[
          arg(arg='x'),
          arg(arg='y')]),
      body=[
        Assign(
          targets=[
            Name(id='result', ctx=Store())],
          value=BinOp(
            left=BinOp(
              left=Name(id='x', ctx=Load()),
              op=Mult(),
              right=Constant(value=2)),
            op=Add(),
            right=Name(id='y', ctx=Load()))),
        Return(
          value=Name(id='result', ctx=Load()))])])
```

### Bytecode Generation and Caching

#### Automatic Bytecode Compilation

CPython automatically generates bytecode files with a `.pyc` extension, stored in the `__pycache__` directory. The naming convention includes the Python version and platform information:

```
__pycache__/
    module_name.cpython-312.pyc
```

#### Manual Bytecode Compilation

You can explicitly compile Python modules using the `compileall` module:

```bash
python -m compileall -b script.py
```

**Parameters:**
- `-b`: use legacy (pre-PEP3147) compiled file locations
- `-f`: force rebuild even if timestamps are up to date
- `-q`: output only error messages; `-qq` will suppress the error messages as well

### Bytecode Inspection and Analysis

#### Using the dis Module

The `dis` module provides the ability to disassemble which is used to inspect bytecode:

```python
import dis

def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

dis.dis(fibonacci)
```

Output:

```
  3           RESUME                   0

  4           LOAD_FAST                0 (n)
              LOAD_CONST               1 (1)
              COMPARE_OP              58 (bool(<=))
              POP_JUMP_IF_FALSE        2 (to L1)

  5           LOAD_FAST                0 (n)
              RETURN_VALUE

  6   L1:     LOAD_GLOBAL              1 (fibonacci + NULL)
              LOAD_FAST                0 (n)
              LOAD_CONST               1 (1)
              BINARY_OP               10 (-)
              CALL                     1
              LOAD_GLOBAL              1 (fibonacci + NULL)
              LOAD_FAST                0 (n)
              LOAD_CONST               2 (2)
              BINARY_OP               10 (-)
              CALL                     1
              BINARY_OP                0 (+)
              RETURN_VALUE
```

#### Bytecode Operation Categories

| Category | Operations | Description |
|----------|-----------|-------------|
| Stack manipulation | LOAD_FAST, STORE_FAST, POP_TOP | Value stack operations |
| Arithmetic | BINARY_ADD, BINARY_MULTIPLY, INPLACE_ADD | Mathematical operations |
| Control flow | POP_JUMP_IF_FALSE, JUMP_FORWARD | Conditional and unconditional jumps |
| Function calls | CALL_FUNCTION, RETURN_VALUE | Function invocation and return |
| Object operations | LOAD_ATTR, STORE_ATTR, BUILD_LIST | Attribute and container operations |

## Comparison with Java's Execution Model

### Architectural Similarities

Both Python and Java employ bytecode-based execution models:

| Aspect | Python | Java |
|--------|--------|------|
| Source format | .py | .java |
| Compiled format | .pyc (bytecode) | .class (bytecode) |
| Virtual machine | CPython VM | Java Virtual Machine (JVM) |
| Compilation timing | Import-time or runtime | Explicit compile step |
| JIT compilation | Limited | HotSpot JIT compiler |

### Execution Flow Comparison

**Python (CPython):**
```
source.py → Parser → AST → Compiler → bytecode (.pyc) → CPython VM → Machine code
```

**Java:**
```
Source.java → javac → bytecode (.class) → JVM → JIT Compiler → Machine code
```

## C Extension Integration

### Performance Rationale

Python's interpreted bytecode makes it flexible, but libraries like NumPy and Pandas speed things up by using C code that runs as native machine instructions.

### Creating a C Extension Module

#### Step 1: Implement C Code

Create `mathops.c`:

```c
#include <Python.h>
#include <math.h>

static PyObject *fast_sum(PyObject *self, PyObject *args)
{
   double a, b;
   if (!PyArg_ParseTuple(args, "dd", &a, &b))
   {
       return NULL;
   }
   return PyFloat_FromDouble(a + b);
}

static PyObject *factorial(PyObject *self, PyObject *args)
{
   long n, result = 1;
   if (!PyArg_ParseTuple(args, "l", &n))
   {
       return NULL;
   }

   if (n < 0)
   {
       PyErr_SetString(PyExc_ValueError, "Factorial not defined for negative numbers");
       return NULL;
   }

   for (long i = 2; i <= n; i++)
   {
       result *= i;
   }

   return PyLong_FromLong(result);
}

static PyObject *fast_power(PyObject *self, PyObject *args)
{
   double base, exponent, result;
   if (!PyArg_ParseTuple(args, "dd", &base, &exponent))
   {
       return NULL;
   }

   result = pow(base, exponent);

   if (errno == EDOM || errno == ERANGE)
   {
       PyErr_SetString(PyExc_ValueError, "Math domain or range error");
       return NULL;
   }

   return PyFloat_FromDouble(result);
}

static PyMethodDef MathOpsMethods[] = {
   {"fast_sum", fast_sum, METH_VARARGS, "Add two floating-point numbers"},
   {"factorial", factorial, METH_VARARGS, "Calculate factorial of a number"},
   {"fast_power", fast_power, METH_VARARGS, "Raise base to exponent power"},
   {NULL, NULL, 0, NULL}};

static struct PyModuleDef mathopsmodule = {
   PyModuleDef_HEAD_INIT,
   "mathops",                          /* Module name */
   "High-performance math operations", /* Module documentation */
   -1,
   MathOpsMethods};

PyMODINIT_FUNC PyInit_mathops(void)
{
   return PyModule_Create(&mathopsmodule);
}
```

#### Step 2: Build Configuration

Create `setup.py`:

```python
from setuptools import setup, Extension

mathops_module = Extension(
    'mathops',
    sources=['mathops.c'],
    extra_compile_args=['-O3'],  # Optimization level 3
)

setup(
    name='mathops',
    version='1.0',
    description='High-performance mathematical operations',
    ext_modules=[mathops_module],
)
```

#### Step 3: Compilation

```bash
python setup.py build_ext --inplace
```

This generates a shared library `mathops.cpython-313-darwin.so`.

#### Step 4: Usage in Python

```python
import mathops
import time

# Test C extension
print(mathops.fast_sum(15.7, 22.3))  # Output: 38.0
print(mathops.factorial(10))          # Output: 3628800
print(mathops.fast_power(2, 10))      # Output: 1024.0

# Performance comparison
def python_sum(a, b):
    return a + b

# Benchmark
iterations = 10_000_000

start = time.perf_counter()
for i in range(iterations):
    python_sum(3.14, 2.71)
python_time = time.perf_counter() - start

start = time.perf_counter()
for i in range(iterations):
    mathops.fast_sum(3.14, 2.71)
c_time = time.perf_counter() - start

print(f"Python implementation: {python_time:.4f}s")
print(f"C extension: {c_time:.4f}s")
print(f"Speedup: {python_time / c_time:.2f}x")
```

### Wait, why is the Native Python Code Faster?

You might be surprised by the benchmark results, which often show the pure Python `python_sum` is actually faster than the `mathops.fast_sum` C extension. This is not an error; it's a critical concept in Python optimization: **function call overhead**.

Every time you call `mathops.fast_sum`, Python must transition between the interpreter and your compiled C code. This transition involves significant overhead. Here's a breakdown of the costs:

**1. Calling mathops.fast_sum from Python:**
- Python must take its two float objects
- It must package them into a new PyTuple object
- It then makes a generic call to the C-API

**2. Executing fast_sum in C:**
- The C function receives the PyTuple
- It must call `PyArg_ParseTuple` to unpack the tuple and convert the Python float objects into C doubles
- It performs the C-level addition
- It must then call `PyFloat_FromDouble` to repackage the C double result into a new Python float object
- This new Python object is returned back to the Python interpreter

#### Why is python_sum faster?

When you run `return a + b` in pure Python, the interpreter's bytecode executor is already running optimized C code internally. The `BINARY_OP` bytecode instruction calls a highly optimized C function within the Python VM to add two Python objects. There is no packaging, unpacking, or transition overhead. The entire operation stays within the interpreter's internal execution flow.

C extensions are only faster when the computational work done inside the C function is substantial enough to make the function call overhead irrelevant. The `fast_sum` function does almost no work (a single addition), so the benchmark is dominated by the call overhead. If you were to benchmark `mathops.factorial(50)` against a pure Python version, the C extension would be dramatically faster because the cost of the C loop is much, much smaller than the cost of a Python loop.

### Alternative Optimization Strategies

- **PyPy**: JIT-compiled Python implementation
- **Numba**: JIT compilation decorator for numerical Python
- **Cython**: Python-to-C transpiler with gradual typing
- **ctypes/cffi**: Dynamic foreign function interfaces

## Conclusion

Python finds a nice middle ground between developer comfort and raw speed. Its bytecode layer makes it run almost anywhere and lets you look inside the runtime in ways that many languages don't. When you need more power, you can hook into C extensions and run compiled code directly, while still keeping Python's familiar feel.

Once you understand how it all fits together, it's easier to decide where to focus your optimizations and get the most out of Python's mix of interpreted and compiled behavior.

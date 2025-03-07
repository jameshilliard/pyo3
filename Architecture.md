<!-- This file contains a rough overview of the PyO3 codebase. -->
<!-- Please do not make descriptions too specific, so that we can easily -->
<!-- keep this file in sync with the codebase. -->

# PyO3: Architecture

This document roughly describes the high-level architecture of PyO3.
If you want to become familiar with the codebase you are in the right place!

## Overview

PyO3 provides a bridge between Rust and Python, based on the [Python/C API].
Thus, PyO3 has low-level bindings of these API as its core.
On top of that, we have higher-level bindings to operate Python objects safely.
Also, to define Python classes and functions in Rust code, we have `trait PyClass` and a set of
protocol traits (e.g., `PyIterProtocol`) for supporting object protocols (i.e., `__dunder__` methods).
Since implementing `PyClass` requires lots of boilerplate, we have a proc-macro `#[pyclass]`.

To summarize, there are six main parts to the PyO3 codebase.
1. [Low-level bindings of Python/C API.](#1-low-level-bindings-of-python-capi)
    - [`src/ffi`]
2. [Bindings to Python objects.](#2-bindings-to-python-objects)
    - [`src/instance.rs`] and [`src/types`]
3. [`PyClass` and related functionalities.](#3-pyclass-and-related-functionalities)
    - [`src/pycell.rs`], [`src/pyclass.rs`], and more
4. [Protocol methods like `__getitem__`.](#4-protocol-methods)
    - [`src/class`]
5. [Procedural macros to simplify usage for users.](#5-procedural-macros-to-simplify-usage-for-users)
    - [`src/derive_utils.rs`], [`pyo3-macros`] and [`pyo3-macros-backend`]
6. [`build.rs`](#6-buildrs)
    - [`build.rs`](https://github.com/PyO3/pyo3/tree/master/build.rs)

## 1. Low-level bindings of Python/C API
[`src/ffi`] contains wrappers of [Python/C API].

We aim to provide straight-forward Rust wrappers resembling the file structure of
[`cpython/Include`](https://github.com/python/cpython/tree/v3.9.2/Include).

However, we still lack some APIs and are continuously updating the the module to match
the file contents upstream in CPython.
The tracking issue is [#1289](https://github.com/PyO3/pyo3/issues/1289), and contribution is welcome.

In the [`src/ffi`] module, there is lots of conditional compilation such as `#[cfg(Py_LIMITED_API)]`,
`#[cfg(Py_37)]`, and `#[cfg(PyPy)]`.
`Py_LIMITED_API` corresponds to `#define Py_LIMITED_API` macro in Python/C API.
With `Py_LIMITED_API`, we can build a Python-version-agnostic binary called an
[abi3 wheel](https://pyo3.rs/v0.13.2/building_and_distribution.html#py_limited_apiabi3).
`Py_37` means that the API is available from Python >= 3.7.
There are also `Py_38`, `Py_39`, and so on.
`PyPy` means that the API definition is for PyPy.
Those flags are set in [`build.rs`](#6-buildrs).

## 2. Bindings to Python objects
[`src/types`] contains bindings to [built-in types](https://docs.python.org/3/library/stdtypes.html)
of Python, such as `dict` and `list`.
For historical reasons, Python's `object` is called `PyAny` in PyO3 and located in [`src/types/any.rs`].
Currently, `PyAny` is a straightforward wrapper of `ffi::PyObject`, defined as:
```rust
#[repr(transparent)]
pub struct PyAny(UnsafeCell<ffi::PyObject>);
```

All built-in types are defined as a C struct.
For example, `dict` is defined as:
```c
typedef struct {
    /* Base object */
    PyObject ob_base;
    /* Number of items in the dictionary */
    Py_ssize_t ma_used;
    /* Dictionary version */
    uint64_t ma_version_tag;
    PyDictKeysObject *ma_keys;
    PyObject **ma_values;
} PyDictObject;
```

However, we cannot access such a specific data structure with `#[cfg(Py_LIMITED_API)]` set.
Thus, all builtin objects are implemented as opaque types by wrapping `PyAny`, e.g.,:
```rust
#[repr(transparent)]
pub struct PyDict(PyAny);
```

Note that `PyAny` is not a pointer, and it is usually used as a pointer to the object in the
Python heap, as `&PyAny`.
This design choice can be changed
(see the discussion in [#1056](https://github.com/PyO3/pyo3/issues/1056)).

Since we need lots of boilerplate for implementing common traits for these types
(e.g., `AsPyPointer`, `AsRef<PyAny>`, and `Debug`), we have some macros in
[`src/types/mod.rs`].

## 3. `PyClass` and related functionalities
[`src/pycell.rs`], [`src/pyclass.rs`], and [`src/type_object.rs`] contain types and
traits to make `#[pyclass]` work.
Also, [`src/pyclass_init.rs`] and [`src/pyclass_slots.rs`] have related functionalities.

To realize object-oriented programming in C, all Python objects must have the following two fields
at the beginning.
```rust
#[repr(C)]
pub struct PyObject {
    pub ob_refcnt: usize,
    pub ob_type: *mut PyTypeObject,
    ...
}
```
Thanks to this guarantee, casting `*mut A` to `*mut PyObject` is valid if `A` is a Python object.

To ensure this guarantee, we have a wrapper struct `PyCell<T>` in [`src/pycell.rs`] which is roughly:
```rust
#[repr(C)]
pub struct PyCell<T: PyClass> {
    object: crate::ffi::PyObject,
    inner: T,
}
```
Thus, when copying a Rust struct to a Python object, we first allocate `PyCell` on the Python heap and then
move `T` into it.
Also, `PyCell` provides [RefCell](https://doc.rust-lang.org/std/cell/struct.RefCell.html)-like methods
to ensure Rust's borrow rules.
See [the documentation](https://docs.rs/pyo3/latest/pyo3/pycell/struct.PyCell.html) for more.

`PyCell<T>` requires that `T` implements `PyClass`.
This trait is somewhat complex and derives many traits, but the most important one is `PyTypeObject`
in [`src/type_object.rs`].
`PyTypeObject` is also implemented for built-in types.
In Python, all objects have their types, and types are also objects of `type`.
For example, you can see `type({})` shows `dict` and `type(type({}))` shows `type` in Python REPL.
`T: PyTypeObject` implies that `T` has a corresponding type object.

## 4. Protocol methods
Python has some built-in special methods called dunder, such as `__iter__`.
They are called [abstract objects layer](https://docs.python.org/3/c-api/abstract.html) in
Python/C API.
We provide a way to implement those protocols by using `#[pyproto]` and specific traits, such
as `PyIterProtocol`.
[`src/class`] defines these traits.
Each protocol method has a corresponding FFI function.
For example, `PyIterProtocol::__iter__` has
`pub unsafe extern "C" fn iter<T>(slf: *mut PyObject) -> *mut PyObject`.
When `#[pyproto]` finds that `T` implements `PyIterProtocol::__iter__`, it automatically
sets `iter<T>` on the type object of `T`.

Also, [`src/class/methods.rs`] has utilities for `#[pyfunction]` and [`src/class/impl_.rs`] has
some internal tricks for making `#[pyproto]` flexible.

## 5. Procedural macros to simplify usage for users.
[`pyo3-macros`] provides six proc-macro APIs: `pymodule`, `pyproto`, `pyfunction`, `pyclass`,
`pymethods`, and `#[derive(FromPyObject)]`.
[`pyo3-macros-backend`] has the actual implementations of these APIs.
[`src/derive_utils.rs`] contains some utilities used in code generated by these proc-macros,
such as parsing function arguments.

## 6. `build.rs`
PyO3's [`build.rs`](https://github.com/PyO3/pyo3/tree/master/build.rs) is relatively long
(about 900 lines) to support multiple architectures, interpreters, and usages.
Below is a non-exhaustive list of its functionality:
- Cross-compiling support.
  - If `TARGET` architecture and `HOST` architecture differ, we find cross compile information
  from environment variables (`PYO3_CROSS_INCLUDE_DIR` and `PYO3_CROSS_PYTHON`) or system files.
- Find the interpreter for build and detect the Python version.
  - We have to set some version flags like `Py_37`.
  - If the interpreter is PyPy, we set `PyPy`.
- Check if we are building a Python extension.
  - If we are building an extension (e.g., Python library installable by `pip`),
  we don't link `libpython`.
  Currently we use the `extension-module` feature for this purpose. This may change in the future.
  See [#1123](https://github.com/PyO3/pyo3/pull/1123).

<!-- External Links -->
[Python/C API]: https://docs.python.org/3/c-api/
<!-- Crates -->
[`pyo3-macros`]: https://github.com/PyO3/pyo3/tree/master/pyo3-macros
[`pyo3-macros-backend`]: https://github.com/PyO3/pyo3/tree/master/pyo3-macros-backend
<!-- Directories -->
[`src/class`]: https://github.com/PyO3/pyo3/tree/master/src/class
[`src/ffi`]: https://github.com/PyO3/pyo3/tree/master/src/ffi
[`src/types`]: https://github.com/PyO3/pyo3/tree/master/src/types
<!-- Files -->
[`src/derive_utils.rs`]: https://github.com/PyO3/pyo3/tree/master/src/derive_utils.rs
[`src/instance.rs`]: https://github.com/PyO3/pyo3/tree/master/src/instance.rs
[`src/pycell.rs`]: https://github.com/PyO3/pyo3/tree/master/src/pycell.rs
[`src/pyclass.rs`]: https://github.com/PyO3/pyo3/tree/master/src/pyclass.rs
[`src/pyclass_init.rs`]: https://github.com/PyO3/pyo3/tree/master/src/pyclass_init.rs
[`src/pyclass_slot.rs`]: https://github.com/PyO3/pyo3/tree/master/src/pyclass_slot.rs
[`src/type_object.rs`]: https://github.com/PyO3/pyo3/tree/master/src/type_object.rs
[`src/class/methods.rs`]: https://github.com/PyO3/pyo3/tree/master/src/class/methods.rs
[`src/class/impl_.rs`]: https://github.com/PyO3/pyo3/tree/master/src/class/impl_.rs
[`src/types/any.rs`]: https://github.com/PyO3/pyo3/tree/master/src/types/any.rs
[`src/types/mod.rs`]: https://github.com/PyO3/pyo3/tree/master/src/types/mod.rs

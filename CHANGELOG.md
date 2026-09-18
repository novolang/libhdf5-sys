# Changelog

All notable changes to libhdf5-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-16

The first release: forty-six entry points of the HDF5 C library, one
`@ffi` declaration each, and no logic.

### Added

- `libhdf5` — the whole surface, in eight groups.
  - The library and its identifiers: `H5open`, `H5close`,
    `H5get_libversion` and `H5Iis_valid`.
  - The error stack: `H5Eclear2` and `H5Eprint2`.
  - The file: `H5Fcreate`, `H5Fopen`, `H5Fclose`, `H5Fflush`,
    `H5Fis_hdf5` and `H5Fget_filesize`.
  - The group and its links: `H5Gcreate2`, `H5Gopen2`, `H5Gclose`,
    `H5Lexists` and `H5Ldelete`.
  - The dataspace: `H5Screate`, `H5Screate_simple`, `H5Sclose`,
    `H5Sget_simple_extent_ndims`, `H5Sget_simple_extent_dims`,
    `H5Sget_simple_extent_npoints` and `H5Sselect_hyperslab`.
  - The dataset: `H5Dcreate2`, `H5Dopen2`, `H5Dclose`, `H5Dread`,
    `H5Dwrite`, `H5Dget_space`, `H5Dget_type` and
    `H5Dget_storage_size`.
  - The attribute: `H5Acreate2`, `H5Aopen`, `H5Aclose`, `H5Aread`,
    `H5Awrite`, `H5Aexists` and `H5Aget_type`.
  - The datatype: `H5Tcreate`, `H5Tcopy`, `H5Tclose`, `H5Tget_size`,
    `H5Tset_size`, `H5Tget_class` and `H5Tequal`.
- `tests/libhdf5_tests.nv` — ten tests over the signatures. Every file
  the suite writes is made under a fresh directory from `fs.temp_dir`
  and deleted again.

### The package reads a file and cannot create a numeric dataset

HDF5 names its predefined datatypes with macros: `H5T_NATIVE_INT` is
`(H5OPEN H5T_NATIVE_INT_g)`, and `H5T_NATIVE_INT_g` is a C global
variable the library fills in when it starts. The same is true of
`H5T_IEEE_F64LE` and of every property list class, where
`H5P_DATASET_CREATE` is `H5P_CLS_DATASET_CREATE_ID_g`. A binding
declares functions; it cannot read a global variable, so none of those
names can be reached.

What remains is a package that reads. A dataset already in a file
answers its own datatype through `H5Dget_type` and its own shape
through `H5Dget_space`, and those two identifiers are what `H5Dread`
wants, so every dataset in every file can be read. An attribute does
the same through `H5Aget_type`.

What is gone is creating a dataset of numbers. `H5Dcreate2` needs a
datatype identifier, and the only datatypes this package can make are
the four classes `H5Tcreate` builds: a fixed-length string, an opaque
block, a compound type and an enumeration. A program that needs to
write an array of native integers supplies the identifier from a C
helper of its own — one line returning `H5T_NATIVE_INT` — exactly as a
program that wants tree-sitter's by-value node API supplies a shim.

Every property list argument in this package is therefore passed as
`H5P_DEFAULT`, which is 0. Chunking and deflate compression are set
through a property list, so they wait on the same helper, and they are
named as missing below.

### The numbered names are the symbols

`H5Gcreate`, `H5Gopen`, `H5Dcreate`, `H5Dopen`, `H5Acreate`, `H5Eclear`
and `H5Eprint` are C macros that expand to the form ending in a digit.
A macro is not a symbol a binding can resolve, so the declarations name
`H5Gcreate2`, `H5Dopen2` and the rest.

### The types

`hid_t` is a signed 64-bit integer from HDF5 1.10 onwards and crosses
as an `Int` unchanged; a negative one is a failure. `hsize_t` is an
unsigned 64-bit count, and a list of them is a run of eight-byte words
the caller lays out with `ptr.write_word`. `herr_t` and `htri_t` are
both a C `int`, which is 32 bits wide, so an answer is compared with a
negative number only after `as i32`.

### Unverified

HDF5 is not installed on the machine where this package was written,
so the suite has never linked: `novo test` stops at
`/usr/bin/ld: cannot find -lhdf5`. The
declarations were checked against the HDF5 C reference manual, and
`novo pkg build` type-checks them, which is the whole of what has been
measured. Treat the package as unmeasured until someone runs the suite
against a real library.

### Named as missing

**The predefined datatypes and the property list classes.** They are
global variables behind macros, and the reason is above.

**The property list setters.** `H5Pcreate`, `H5Pset_chunk` and
`H5Pset_deflate` are left out, because `H5Pcreate` takes a class that
is one of those globals and there is no way to name one. Chunking and
compression come with them.

**Everything that iterates.** `H5Literate2`, `H5Ovisit3`, `H5Aiterate2`
and `H5Tconvert` take a C function pointer, which the novo-lang foreign
function interface cannot pass. Listing the members of a group is the
loss a reader will notice first; `H5Lexists` answers for a name a
program already knows.

**`H5Eset_auto2` and `H5Ewalk2`.** Both take a C function pointer. The
first is also the only way to turn off HDF5's own printing, so a
failure prints an error stack to the standard error stream and a
program cannot stop it.

**The references, the variable-length types and the compound members.**
`H5Rcreate`, `H5Tvlen_create`, `H5Tinsert` and `H5Tget_member_offset`
describe a value whose layout the caller would have to lay out by hand
against a header that is free to change it. They are left out of the
first release.

**The parallel interface.** `H5Pset_fapl_mpio` takes an `MPI_Comm` and
an `MPI_Info`, which are structures passed by value in most MPI
implementations.

**The high-level interface.** `H5LTmake_dataset` and its neighbours are
in `libhdf5_hl`, a second shared object. A binding package declares one
library, so the high-level interface belongs in a package of its own —
and every one of its calls takes a predefined datatype anyway.

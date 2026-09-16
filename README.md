# libhdf5-sys

HDF5 is a file format for large scientific datasets and the C library
that reads and writes it. One file holds a tree of named arrays, each
with its own shape and element type, and a program reads a rectangular
piece of an array without reading the rest. The format and the library
are documented in
[the HDF5 reference manual](https://support.hdfgroup.org/documentation/hdf5/latest/_r_m.html).
This package declares forty-six of that library's entry points to
novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libhdf5. The package contains no logic of
its own, and it does nothing without the C library installed. The
forty-six entry points read any HDF5 file and write strings, groups and
attributes into one; the section "What is not included" says what a
program still cannot do with them alone, and the first item there is
the one that matters.

**Unverified.** HDF5 is not installed on the machine where this package
was written, so the test suite has never been linked. See the "Tests"
section.

## What it is

A **file** holds a tree. The nodes of the tree are **groups**, which
work like directories, and the leaves are **datasets**, which are
arrays. The root group is `/`, and a dataset has a path such as
`/run3/temperature`. What connects a group to what it holds is a
**link**, and a name is a property of the link rather than of the
object: two links can name the same dataset.

A **dataset** has three things. Its **datatype** says what one element
is — an eight-byte float, a four-byte integer, a fixed-length string.
Its **dataspace** says the shape: how many dimensions and how long each
one is. Its data is the elements themselves.

A **dataspace** also describes a **selection**, which is the part of an
array a read or a write moves. The common selection is a **hyperslab**,
a rectangular block given as a starting corner, a stride, a count and a
block size in each dimension. A read names two dataspaces: one for the
part of the file it wants and one for the part of memory it goes into.

An **attribute** is a small named value carried beside a group or a
dataset. It has a datatype and a dataspace like a dataset does, but no
selection: an attribute is read and written whole. Attributes are where
units, timestamps and instrument settings go.

Everything the library hands out is an **identifier**, a signed 64-bit
integer. A file, a group, a dataset, an attribute, a dataspace and a
datatype are all identifiers, and each is closed by the call that
matches the one that made it. A negative identifier is a failure.

The library **converts as it reads**. A dataset of four-byte integers
read with an eight-byte integer as the memory type arrives as
eight-byte integers. This is why the memory type is an argument to
`H5Dread` rather than something the dataset decides.

## Install

```
novo pkg add libhdf5-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its header come from the system package `libhdf5-dev`:

```
sudo apt install libhdf5-dev
```

On macOS the Homebrew formula is `hdf5`. On other systems the library
builds from the HDF Group's source with CMake.

Debian and Ubuntu ship a serial build and two parallel ones, and name
their pkg-config files `hdf5-serial`, `hdf5-openmpi` and `hdf5-mpich`.
This package names `hdf5`, which is what the upstream build installs. A
program that wants one of the others sets its own link flags.

## Example

The datatype and the shape of a dataset, read out of the file that
holds it:

```novo ignore
use libhdf5

fn main() [io, ffi]
    // 0 is H5F_ACC_RDONLY and 0 is H5P_DEFAULT.
    let file = libhdf5.h5f_open("run3.h5", 0, 0)
    if file < 0
        println("not an HDF5 file")
        return

    let dset = libhdf5.h5d_open2(file, "/temperature", 0)
    let file_type = libhdf5.h5d_get_type(dset)
    let space = libhdf5.h5d_get_space(dset)
    let count = libhdf5.h5s_get_simple_extent_npoints(space)
    let width = libhdf5.h5t_get_size(file_type)
    println("${count} elements of ${width} bytes")

    // The datatype the file reports is the one to read with.
    let buf = ptr.alloc(count * width)
    let rc = libhdf5.h5d_read(dset, file_type, 0, 0, 0, buf) as i32
    if rc < 0
        let _printed = libhdf5.h5e_print2(0, 0)

    ptr.free(buf)
    let _a = libhdf5.h5s_close(space)
    let _b = libhdf5.h5t_close(file_type)
    let _c = libhdf5.h5d_close(dset)
    let _d = libhdf5.h5f_close(file)
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and
not the ones in this file. The same calls are in
`tests/libhdf5_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `libhdf5` | Every entry point, in eight groups: the library, the error stack, the file, the group and its links, the dataspace, the dataset, the attribute and the datatype. |

The eight groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Library and identifiers | 4 | Starts and stops the library, reports its version, and says whether an identifier is still open. |
| Error stack | 2 | Prints the stack of failures to the standard error stream and empties it. |
| File | 6 | Creates, opens, flushes, closes and measures a file, and recognises one by its signature. |
| Group and links | 5 | Creates, opens and closes a group, and tests and removes a link by name. |
| Dataspace | 7 | Makes a scalar or a simple shape, reports the rank, the dimensions and the element count, and selects a rectangular block. |
| Dataset | 8 | Creates, opens and closes a dataset, reads and writes it, and reports its datatype, its shape and its size on disk. |
| Attribute | 7 | Creates, opens and closes an attribute, reads and writes it whole, tests one by name, and reports its datatype. |
| Datatype | 7 | Builds a string, opaque, compound or enumeration type, copies one, sets and reads its size and class, and compares two. |

## How to choose an entry point

`H5Dread` and `H5Dwrite` move the whole of a dataset when both
dataspace arguments are 0, which is `H5S_ALL`. Pass real dataspaces
when only part of it is wanted.

`H5Dget_type` and `H5Dget_space` are how a program that did not create
a dataset learns what is in it. Read with the datatype the file
reports, and the bytes arrive in the file's own representation.

`H5Tcreate` is the one way to make a datatype from nothing, and it
makes four classes: a fixed-length string, an opaque block, a compound
type and an enumeration. Every other datatype in this package comes out
of a file.

`H5Aread` and `H5Awrite` have no selection arguments, because an
attribute has no partial access. An attribute is for a value small
enough to move whole.

`H5Eprint2` with both arguments 0 prints the failures of the last call
to the standard error stream, innermost last. It is the only diagnosis
this package offers.

## The rules a user needs

1. **An identifier is an `Int`, and a negative one is a failure.**
   `H5Fopen`, `H5Dopen2` and every other opener answer an identifier or
   a negative number, and the failure carries no code worth reading:
   the detail is on the error stack.
2. **A `herr_t` and an `htri_t` are a C `int`.** Write `as i32` before
   comparing an answer with a negative number. An `htri_t` is 1 for
   yes, 0 for no and negative for a failure, and a comparison against
   an un-narrowed value misses the third case.
3. **Every identifier has a matching close.** A file needs `H5Fclose`,
   a group `H5Gclose`, a dataset `H5Dclose`, an attribute `H5Aclose`, a
   dataspace `H5Sclose` and a datatype `H5Tclose`. A file is not
   complete on disk until everything inside it has been closed.
4. **`H5P_DEFAULT`, `H5S_ALL` and `H5E_DEFAULT` are all 0.** Every
   property list argument in this package is passed as 0.
5. **The flag and class numbers are numbers**, because the C header
   spells them as macros and enumerations.

   | Name | Number | What it means |
   | --- | --- | --- |
   | `H5F_ACC_RDONLY` | 0 | open for reading |
   | `H5F_ACC_RDWR` | 1 | open for reading and writing |
   | `H5F_ACC_TRUNC` | 2 | create, overwriting any existing file |
   | `H5F_ACC_EXCL` | 4 | create, failing if one exists |
   | `H5F_SCOPE_LOCAL` | 0 | flush this file only |
   | `H5S_SCALAR` | 0 | a dataspace of exactly one element |
   | `H5S_SIMPLE` | 1 | a dataspace with dimensions |
   | `H5S_NULL` | 2 | a dataspace of no elements |
   | `H5S_SELECT_SET` | 0 | replace the selection |
   | `H5S_SELECT_OR` | 1 | add to the selection |
   | `H5T_INTEGER` | 0 | the class of an integer type |
   | `H5T_FLOAT` | 1 | the class of a floating-point type |
   | `H5T_STRING` | 3 | the class of a string type |
   | `H5T_OPAQUE` | 5 | the class of an opaque block |
   | `H5T_COMPOUND` | 6 | the class of a record type |
   | `H5T_ENUM` | 8 | the class of an enumeration |

6. **A dimension list is a run of eight-byte counts.**
   `ptr.alloc(8 * rank)` reserves it and `ptr.write_word` fills each
   one. `H5Screate_simple`, `H5Sget_simple_extent_dims` and
   `H5Sselect_hyperslab` all take lists of this shape, and
   `H5Sselect_hyperslab` takes four of them.
7. **The memory type and the file type are separate arguments**, and
   the library converts between them. Reading with the file's own type
   is the one case where nothing is converted.
8. **A buffer is the caller's, and its size is not checked.** `H5Dread`
   writes as many bytes as the selection and the memory type imply. Ask
   the dataspace for the element count and the datatype for the element
   size, and multiply.
9. **Only `H5Tcreate`'s four classes can be built from nothing.** A
   string, an opaque block, a compound type and an enumeration.
   `H5T_NATIVE_INT` and its neighbours are global variables, so a
   program that writes numbers gets the identifier from a C helper of
   its own. Rule 10 of "What is not included" says why.
10. **A failure prints itself.** HDF5 writes its error stack to the
    standard error stream by default, and the call that turns that off
    takes a function pointer. A program cannot stop it through this
    package.
11. **A deleted link does not shrink the file.** `H5Ldelete` removes the
    name, and the bytes stay where they were. The tool that compacts a
    file is `h5repack`, a separate program.
12. **`H5close` closes everything.** It flushes and closes every open
    object, and the next call to any entry point starts the library
    again.

## What is not included

- **The predefined datatypes.** `H5T_NATIVE_INT`, `H5T_IEEE_F64LE` and
  the rest are C macros over GLOBAL VARIABLES — `H5T_NATIVE_INT` is
  `(H5OPEN H5T_NATIVE_INT_g)` — and a binding declares functions. A
  program that must create a dataset of numbers supplies the identifier
  from a one-line C helper of its own. Reading is unaffected:
  `H5Dget_type` answers the datatype the file already holds.
- **The property lists.** `H5Pcreate` takes a class that is one of
  those same globals, so it is left out, and `H5Pset_chunk` and
  `H5Pset_deflate` with it. Chunking and deflate compression wait on
  the same helper. Every property list argument in this package is
  `H5P_DEFAULT`.
- **Everything that iterates.** `H5Literate2`, `H5Ovisit3` and
  `H5Aiterate2` take a C function pointer, which the novo-lang foreign
  function interface cannot pass. Listing the members of a group is
  what this removes; `H5Lexists` answers for a name already known.
- **`H5Eset_auto2` and `H5Ewalk2`.** Both take a C function pointer.
  The first is the only way to stop HDF5 printing its own failures.
- **References, variable-length types and compound members.**
  `H5Rcreate`, `H5Tvlen_create`, `H5Tinsert` and `H5Tget_member_offset`
  describe values whose memory layout a caller would lay out by hand
  against a header that may change it. They are left out of the first
  release.
- **The parallel interface.** `H5Pset_fapl_mpio` takes an `MPI_Comm`
  and an `MPI_Info`, which most MPI implementations pass by value.
- **The high-level interface.** `H5LTmake_dataset` and its neighbours
  are in `libhdf5_hl`, a second shared object, and a binding package
  declares one library.

## Related packages

There is no novo-lang replacement for this package and none is planned.
HDF5 is a format defined by its implementation: the specification is a
description of what the C library writes, the library is several
hundred thousand lines, and a second implementation that agreed with it
in every case would be a project of its own. A program that needs to
read an HDF5 file reads it with libhdf5.

`libnetcdf-sys` is the same arrangement for NetCDF, whose version 4
format is HDF5 underneath. A program that only needs the NetCDF subset
— arrays with named dimensions and attributes — has a smaller interface
there, and one that does not run into the missing datatypes: NetCDF
names its element types with plain integers.

`parquet-nv` and `arrow-nv` are columnar formats for the same kind of
data, ported to novo-lang. They are the answer when the file format is
the program's own choice.

## Tests

`tests/libhdf5_tests.nv` holds ten tests written against the
signatures. They call the C library, so `novo test` needs HDF5
installed and linkable:

```
novo test tests/libhdf5_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

Every file the suite writes is made under a fresh directory from
`fs.temp_dir` and deleted again, and nothing reaches the network. The
dataspace tests assert that a scalar space holds one element and that a
simple one reports the dimensions it was given. The datatype tests
build a fixed-length string type, resize a copy of it and assert that
the copy is then a different type. The file tests create a file,
recognise it by its signature, reopen it and assert that its size is
more than nothing. The group test creates a group, finds it by name,
opens it, removes the link and asserts that it is gone. The dataset
test writes three four-byte strings, reopens the file, reads the
datatype and the shape out of it, and asserts the bytes come back
unchanged. The hyperslab test selects the second of those three and
asserts that it alone arrives. The attribute test round-trips a
four-byte string on the root group. The last test opens a file that was
never written, asserts the failure, prints the error stack and shuts
the library down.

**Unverified.** HDF5 is not installed on the staging machine, so the
suite has never linked: `novo test` stops at `/usr/bin/ld: cannot find
-lhdf5`. Every assertion above is written from the reference manual and
none of them has been observed to pass.

## Implementation status

| Group | State |
| --- | --- |
| Library and identifiers | Complete. |
| Error stack | Complete for printing and clearing. |
| File | Complete for the serial driver. |
| Group and links | Complete for creating, opening and removing by name. |
| Dataspace | Complete for the scalar, simple and hyperslab cases. |
| Dataset | Complete for reading, and for writing a type `H5Tcreate` can build. |
| Attribute | Complete. |
| Datatype | Complete for the four classes that can be built from nothing. |
| Predefined datatypes | Absent. They are global variables, not functions. |
| Property lists | Absent, and chunking and compression with them. |
| Iteration | Absent. Every iterator takes a C function pointer. |
| References and variable-length types | Absent. Left out of the first release. |
| Parallel interface | Absent. It takes an MPI communicator by value. |
| High-level interface | Absent. It is a second shared object. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

HDF5 itself is distributed under a BSD-style licence from The HDF
Group, and installing it is the reader's own step.

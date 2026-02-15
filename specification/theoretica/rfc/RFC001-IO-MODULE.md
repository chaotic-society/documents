# RFC 001: IO Module

## Abstract
An implementation is proposed for an IO module that would enable saving and loading data structures to and from file.
Supported file formats shall include CSV and HDF5, with other potential extensions.

## Motivation
Theoretica currently lacks support for standard file formats, missing a key necessity of scientific computing.
Supporting at least CSV and HDF5 files will greatly increase the usability of the library.

## Design
The IO module should include read and write functions for each format, and all utility functions such as `th::println` in `utility.h` should be moved to this new module.
The specific ways in which data structures are written and loaded is two-fold:
- Functions for each data structure, specialized through overloading
- Functions for data tables, which can then be interpreted as a certain data structure

For example, the API for reading and writing a matrix in CSV format should be:

```cpp
mat<real> A = { ... };

// Pass by const reference, write to file
io::write_csv("filename.csv", A);

// Pass by reference, overwrite A with result
io::read_csv("filename.csv", A);

// Declare and load in a single line
mat<real> B = io::read_csv<mat<real>>("filename.csv");
```

This same structure should be followed for all major data structures of the library which could benefit from saving and loading to files, such as `vec<real>`, `polynomial<real>`, `histogram` and so on. Smaller data structures such as `complex` or `dual` may not be particularly suited for IO. The `io::read_csv` and `io::write_csv` functions should be implemented inside a `io/format_csv.h` header, for each data type. Dependencies between modules should be considered in this step, as supporting reading and writing of data structures requires depending on them and can potentially bloat the IO module. For instance, an intermediate representation of the class could be supplied by a dedicated `serialize()` method which shall return a string matrix or a type similar to `data_table` which represent uniquely the starting data. Adding an intermediate step could slow execution and increase memory footprint for large structures. For this reason, an intermediate representation could be evaluated for smaller data structures.

As part of the IO module, a new data structure `data_table` should be implemented (`io/data_table.h`), with support for generic tables of mixed type (string and numeric) with headers. This data table is analogous to "data frames" available in major data analysis frameworks. As this feature is added, new methods should be implemented for common data structures to be read and written through data tables, such as `pack()` and `unpack()` methods that convert matrices to vector form. The API for reading, writing and using data tables should be:


```cpp
data_table table;

// Read table from CSV format, supporting both strings and numerical entries
io::read_csv("filename.csv", table);

// Create a new column which is a transformed version of another
table["newColumn"] = parallel::atan(table["oldColumn"]);

// Load a table column into a 512 x 512 matrix
mat<real> A (table["myMatrix"], 512, 512);

// Modify the matrix A ...

// Unpack the modified matrix back into the table
table["myMatrix"] = A.unpack();

// Write the transformed table to file as CSV
io::write_csv("filename.csv", table);
```

Specialized tables for only real values could be implemented, improving performance in loading entries and avoiding using `std::variant` or similar solutions for supporting both numerical and string values.

## Drawbacks
The main drawback of IO support with a module depending on other modules to read and write data structures is the increased compilation time and potential bloat of the library. Careful consideration should be given to which data structures are supported for IO, and whether intermediate representations should be used to avoid direct dependencies. For advanced modules, which are not part of the core library, IO support could be implemented in the module itself rather than in the IO module, to avoid unnecessary dependencies.

Data tables should also be optimized for performance, as they could potentially be used for large datasets. In particular, lazy evaluation could be considered as a future enhancement.

HDF5 support requires linking against the HDF5 library, which is an additional dependency for Theoretica. However, HDF5 is a widely used standard in scientific computing, and its inclusion would greatly enhance the library's capabilities. This dependency should be optional, with the IO module falling back to only CSV support if HDF5 is not available.

## Alternatives
An alternative to implementing a dedicated IO module would be to implement read and write functions for each data structure in their respective modules. However, this would lead to code duplication and a lack of consistency in the API. A dedicated IO module provides a centralized location for all file operations, making it easier to maintain and extend in the future.

## Unresolved
Unresolved questions include future support for additional file formats such as NetCDF, JSON, XML, or binary formats. The design of the `data_table` structure also requires further discussion, particularly regarding its internal representation and performance optimizations.

## Implementation Plan
- [x] Create `io` module and `io/io.h` with features from `utility.h`
- [x] Implement CSV support for `mat<real>` and `vec<real>` in `io/format_csv.h`
- [ ] Implement HDF5 support for `mat<real>` and `vec<real>` in `io/format_hdf5.h`
- [x] Implement base `data_table` structure in `io/data_table.h` and support CSV read/write
- [x] Add `pack()` and `unpack()` methods for `mat<real>` and `vec<real>` to support data tables (`pack()` implemented instead as a constructor)
- [ ] Consider extensions of the `data_table` structure and additional file formats

## Testing Strategy
The IO module is fundamentally different from other modules, as it primarily deals with file operations rather than numerical computations. Therefore, the testing strategy should focus on ensuring the correctness and robustness of file read and write operations. This could be achieved by using back-and-forth tests, where data structures are written to a file and then read back, verifying that the original and loaded data structures are identical.

## Documentation
The usual documentation standards of Theoretica should be followed, with detailed explanations of the IO module's API, including examples of reading and writing data structures and using data tables. Additionally, a dedicated tutorial should be created.

## Backwards Compatibility
This module is compatible with existing code, but features from `utility.h` will be ported to `io/io.h`, moving functions to the `io` namespace (e.g. `th::println()` will move to `th::io::println()`).

## References
- [HDF5 API Documentation](https://support.hdfgroup.org/documentation/hdf5/latest/)
- [CSV File Format Specification](https://tools.ietf.org/html/rfc4180)

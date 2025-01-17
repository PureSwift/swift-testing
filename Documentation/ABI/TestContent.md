# Runtime-discoverable test content

<!--
This source file is part of the Swift.org open source project

Copyright (c) 2024 Apple Inc. and the Swift project authors
Licensed under Apache License v2.0 with Runtime Library Exception

See https://swift.org/LICENSE.txt for license information
See https://swift.org/CONTRIBUTORS.txt for Swift project authors
-->

This document describes the format and location of test content that the testing
library emits at compile time and can discover at runtime.

> [!WARNING]
> The content of this document is subject to change pending efforts to define a
> Swift-wide standard mechanism for runtime metadata emission and discovery.
> Treat the information in this document as experimental.

## Basic format

Swift Testing stores test content records in a dedicated platform-specific
section in built test products:

| Platform | Binary Format | Section Name |
|-|:-:|-|
| macOS, iOS, watchOS, tvOS, visionOS | Mach-O | `__DATA_CONST,__swift5_tests` |
| Linux, FreeBSD, OpenBSD, Android | ELF | `swift5_tests` |
| WASI | WebAssembly | `swift5_tests` |
| Windows | PE/COFF | `.sw5test$B`[^windowsPadding] |

[^windowsPadding]: On Windows, the Swift compiler [emits](https://github.com/swiftlang/swift/blob/main/stdlib/public/runtime/SwiftRT-COFF.cpp)
  leading and trailing padding into this section, both zeroed and of size
  `MemoryLayout<UInt>.stride`. Code that walks this section must skip over this
  padding.

### Record layout

Regardless of platform, all test content records created and discoverable by the
testing library have the following layout:

```swift
typealias TestContentRecord = (
  kind: UInt32,
  reserved1: UInt32,
  accessor: (@convention(c) (_ outValue: UnsafeMutableRawPointer, _ hint: UnsafeRawPointer?) -> CBool)?,
  context: UInt,
  reserved2: UInt
)
```

This type has natural size, stride, and alignment. Its fields are native-endian.
If needed, this type can be represented in C as a structure:

```c
struct SWTTestContentRecord {
  uint32_t kind;
  uint32_t reserved1;
  bool (* _Nullable accessor)(void *outValue, const void *_Null_unspecified hint);
  uintptr_t context;
  uintptr_t reserved2;
};
```

### Record content

#### The kind field

Each record's _kind_ determines how the record will be interpreted at runtime. A
record's kind is a 32-bit unsigned value. The following kinds are defined:

| As Hexadecimal | As [FourCC](https://en.wikipedia.org/wiki/FourCC) | Interpretation |
|-:|:-:|-|
| `0x00000000` | &ndash; | Reserved (**do not use**) |
| `0x74657374` | `'test'` | Test or suite declaration |
| `0x65786974` | `'exit'` | Exit test |

<!-- When adding cases to this enumeration, be sure to also update the
corresponding enumeration in TestContentGeneration.swift. -->

#### The accessor field

The function `accessor` is a C function. When called, it initializes the memory
at its argument `outValue` to an instance of some Swift type and returns `true`,
or returns `false` if it could not generate the relevant content. On successful
return, the caller is responsible for deinitializing the memory at `outValue`
when done with it.

If `accessor` is `nil`, the test content record is ignored. The testing library
may, in the future, define record kinds that do not provide an accessor function
(that is, they represent pure compile-time information only.)

The second argument to this function, `hint`, is an optional input that can be
passed to help the accessor function determine if its corresponding test content
record matches what the caller is looking for. If the caller passes `nil` as the
`hint` argument, the accessor behaves as if it matched (that is, no additional
filtering is performed.)

The concrete Swift type of the value written to `outValue` and the value pointed
to by `hint` depend on the kind of record:

- For test or suite declarations (kind `0x74657374`), the accessor produces an
  asynchronous Swift function that returns an instance of `Test`:

  ```swift
  @Sendable () async -> Test
  ```

  This signature is not the signature of `accessor`, but of the Swift function
  reference it writes to `outValue`. This level of indirection is necessary
  because loading a test or suite declaration is an asynchronous operation, but
  C functions cannot be `async`.

  Test content records of this kind do not specify a type for `hint`. Always
  pass `nil`.

- For exit test declarations (kind `0x65786974`), the accessor produces a
  structure describing the exit test (of type `__ExitTest`.)

  Test content records of this kind accept a `hint` of type `SourceLocation`.
  They only produce a result if they represent an exit test declared at the same
  source location (or if the hint is `nil`.)

#### The context field

This field can be used by test content to store additional context for a test
content record that needs to be made available before the accessor is called:

- For test or suite declarations (kind `0x74657374`), this field contains a bit
  mask with the following flags currently defined:

  | Bit | Value | Description |
  |-:|-:|-|
  | `1 << 0` | `1` | This record contains a suite declaration |
  | `1 << 1` | `2` | This record contains a parameterized test function declaration |

  Other bits are reserved for future use and must be set to `0`.

- For exit test declarations (kind `0x65786974`), this field is reserved for
  future use and must be set to `0`.

#### The reserved1 and reserved2 fields

These fields are reserved for future use. Always set them to `0`.

## Third-party test content

Testing tools may make use of the same storage and discovery mechanisms by
emitting their own test content records into the test record content section.

Third-party test content should set the `kind` field to a unique value only used
by that tool, or used by that tool in collaboration with other compatible tools.
At runtime, Swift Testing ignores test content records with unrecognized `kind`
values. To reserve a new unique `kind` value, open a [GitHub issue](https://github.com/swiftlang/swift-testing/issues/new/choose)
against Swift Testing.

The layout of third-party test content records must be compatible with that of
`TestContentRecord` as specified above. Third-party tools are ultimately
responsible for ensuring the values they emit into the test content section are
correctly aligned and have sufficient padding; failure to do so may render
downstream test code unusable.

<!--
TODO: elaborate further, give examples
TODO: standardize a mechanism for third parties to produce `Test` instances
      since we don't have a public initializer for the `Test` type.
-->

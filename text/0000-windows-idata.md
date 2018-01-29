- Feature Name: windows_idata
- Start Date: 2018-00-00
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add a new attribute, such as `kind="dll"` that, when used on Windows, produces
an appropriate `idata` LLVM section.

# Motivation
[motivation]: #motivation

This would allow compiling and linking Rust code on Windows without requiring
additional runtime or libraries, and dramatically improve the situation with
cross-compiling on Windows.

SlugFiller: "First, removing a dependency on a specific toolchain. For instance, a file with the appropriate idata section could be compiled with LLVM and linked with LLD without needing an additional runtime or libraries."

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When trying to link a Windows library, use the `kind="dll"` attribute:

```rust
#[cfg(windows)]
#[link(name = "kernel32", kind = "dll")]
#[allow(non_snake_case)]
extern "system" {
    fn GetStdHandle(nStdHandle: u32) -> *mut u8;
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Import libraries are a mapping from symbols such as `_foo@4` to corresponding
symbols in a `dll` such as `foo`.

SlugFiller: "Using the right markers for the globals, it's possible to make LLVM handle that gracefully, by concatenating the necessary elements. However, a better solution would be for rust to do overlap checking prior to generating the LLVM code, and generate a single import block for the entire program, thus avoiding any duplicate imports."

Example of link:

```rust
#[cfg(windows)]
#[link(name = "kernel32", kind = "dll")]
#[allow(non_snake_case)]
extern "system" {
    fn GetStdHandle(nStdHandle: u32) -> *mut u8;
    fn WriteConsoleA(hConsoleOutput: *mut u8, lpBuffer: *const u8, nNumberOfCharsToWrite: u32, lpNumberOfCharsWritten: *mut u32, lpReserved: *const u8) -> i32;
    
    #[link_name]
    ??? decorated example ???
    
    #[link_name]
    ??? \x01undecorated example ???
    
    #[link_ordinal]
    ???? ordinal example ???
}
```

Example of x32 idata (for 64-bit target, a bunch of i32s should replaced with i64s):

```llvm
@__ImageBase = external global i8

define internal i8* @GetStdHandle(i32 %nStdHandle) unnamed_addr alwaysinline nounwind {
    %lpaddr = bitcast i8** getelementptr ([3 x i8*]* @.import.kernel32.ptr, i32 0, i32 0) to i8*(i32 )**
    %addr = load i8*(i32)** %lpaddr
    %ret = call cc 64 i8* %addr(i32 %nStdHandle)
    ret i8* %ret
}

define internal i32 @WriteConsoleA(i8* %hConsoleOutput, i8* %lpBuffer, i32 %nNumberOfCharsToWrite, i32* %lpNumberOfCharsWritten, i8* %lpReserved) unnamed_addr alwaysinline nounwind {
    %lpaddr = bitcast i8** getelementptr ([3 x i8*]* @.import.kernel32.ptr, i32 0, i32 1) to i32(i8*, i8*, i32, i32*, i8*)**
    %addr = load i32(i8*, i8*, i32, i32*, i8*)** %lpaddr
    %ret = call cc 64 i32 %addr(i8* %hConsoleOutput, i8* %lpBuffer, i32 %nNumberOfCharsToWrite, i32* %lpNumberOfCharsWritten, i8* %lpReserved)
    ret i32 %ret
}

%.win32.image_import_descriptor = type {
    i8*, ; Characteristics
    i32, ; TimeDateStamp
    i32, ; ForwarderChain
    i8*, ; DLL Name
    i8* ; FirstThunk
}

@.import.kernel32.dllname = private constant [16 x i8] c"KERNEL32.DLL\00\00\00\00", section ".idata$7"

@.import.kernel32.func.GetStdHandle = private constant [18 x i8] c"\00\00GetStdHandle\00\00\00\00", section ".idata$6"
@.import.kernel32.func.WriteConsoleA = private constant [19 x i8] c"\00\00WriteConsoleA\00\00\00\00", section ".idata$6"

@.import.kernel32.desc = private global [3 x i8*] [
    i8* inttoptr (i32 sub(i32 ptrtoint([18 x i8]* @.import.kernel32.func.GetStdHandle to i32), i32 ptrtoint(i8* @__ImageBase to i32)) to i8*),
    i8* inttoptr (i32 sub(i32 ptrtoint([19 x i8]* @.import.kernel32.func.WriteConsoleA to i32), i32 ptrtoint(i8* @__ImageBase to i32)) to i8*),
    i8* null
], section ".idata$4"
@.import.kernel32.ptr = private global [3 x i8*] [
    i8* inttoptr (i32 sub(i32 ptrtoint([18 x i8]* @.import.kernel32.func.GetStdHandle to i32), i32 ptrtoint(i8* @__ImageBase to i32)) to i8*),
    i8* inttoptr (i32 sub(i32 ptrtoint([19 x i8]* @.import.kernel32.func.WriteConsoleA to i32), i32 ptrtoint(i8* @__ImageBase to i32)) to i8*),
    i8* null
], section ".idata$3"

@.dllimport = appending constant [2 x %.win32.image_import_descriptor] [
    %.win32.image_import_descriptor {
        i8* inttoptr (i32 sub(i32 ptrtoint([3 x i8*]* @.import.kernel32.desc to i32), i32 ptrtoint(i8* @__ImageBase to i32)) to i8*),
        i32 0,
        i32 0,
        i8* inttoptr (i32 sub(i32 ptrtoint([16 x i8]* @.import.kernel32.dllname to i32), i32 ptrtoint(i8* @__ImageBase to i32)) to i8*),
        i8* inttoptr (i32 sub(i32 ptrtoint([3 x i8*]* @.import.kernel32.ptr to i32), i32 ptrtoint(i8* @__ImageBase to i32)) to i8*)
    },
    %.win32.image_import_descriptor {
        i8* null,
        i32 0,
        i32 0,
        i8* null,
        i8* null
    }
], section ".idata$2"
```

# Drawbacks
[drawbacks]: #drawbacks

None expected, aside from the work needed to implement and document.

# Rationale and alternatives
[alternatives]: #alternatives

The alternative is the current situation, where Rust code on Windows is dependent on import libraries.

# Unresolved questions
[unresolved]: #unresolved-questions

Are `link="dll"` and `#[link_ordinal]` the best names for those attributes?

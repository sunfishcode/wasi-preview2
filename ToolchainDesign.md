# Whare are the choices a toolchain needs to make?

 - Dimension: state sharability (memory/indirect-table/stack-pointer-global/related-metadata)

   - Option: *defines-state*
      - core module
      - defines its own state
      - exports its memory/table (but only for use in host calls)
   - Option: *imports-state*
      - core module
      - imports its state
      - re-exports its memory/table (but only for use in host calls)
   - Option: *provides-state*
      - like defines-state, but exports all of the state
   - Option: *shared-nothing*
      - component
      - For now, we'll plan to have wasm-ld continue to emit core wasm, and wit-component can be used to build components out of them. This story may evolve, of course.

 - Dimension: instance lifetime
   - Option: *command*
      - Called exactly once; instance lifetime is scoped by the call.
      - This works by using a component which imports the "arguments" as value
        imports, and exports a result as a value export, and which runs code via
        the wasm `start` function mechanism.
   - Option: *library*
      - Lifetime is determined dynamically by something external
      - wasm `start` function is called before any other export is called
      - No guaranteed "finalization" callback (but higher layers may have best-effort conventional ones)

# What kind of wasm modules can a toolchain produce?

| state sharability  | + | lifetime  | = | artifact type
| ------------------ | - | --------- | - | -------------
| defines-state      | + | command   | = | *statically-linked command*, aka *static executable*
| defines-state      | + | library   | = | *statically-linked library*
| imports-state      | + | command   | = | *dynamically-linked command*, aka *dynamic executable*
| imports-state      | + | library   | = | *dynamically linked library*, aka *dll*
| provides-state     | + | command   | = | (useless)
| provides-state     | + | library   | = | *allocator*

In the future, toolchains may produce components directly, however for the
preview2 milestone, we'll plan to have a separate tool handle that.

# How do the Rust crate-types map to this?

 - crate-type:
    - "bin" => static executable (for now, may be dynamic executable in the future)
    - "dylib" => dll
    - "staticlib" => ye-olde unix archive of .o files
 - TODO: Should we have a way to produce dynamic executables?
 - TODO: Should we have a way to produce "allocator" libraries in Rust?

# What is the C toolchain view on this:
 - cc argument
    - (default) => static executable (again, for now)
    - "-shared" => "dll" ("shared" is an unfortunate name, but keep it for compatibility?)
 - There is no need for -fpic/-fPIC/-fpie/-fPIE
 - TODO: Should we have a way to produce dynamic executables?

# What standard library roles should we define?
 - liballoc
   - an *allocator*
   - provides a malloc API (not necessarily fully C-ABI-compatible; libc could wrap it)
   - provides a `__cxa_atexit` API (ditto),
      - with a `__cxa_finalize`-like export to be called by command-rt when
        exiting normally (and not on trap)
   - Traditionally this would all be part of libc, but libc is big and has overhead,
     so we're splitting it out so that people who don't need libc can
     use it. Yes, We Care About Code Size (tm).
   - liballoc may be composed of static libraries, to allow swapping in
     different malloc implementations.
   - In fully static executables, liballoc would be statically linked in.
 - libc
   - a "dll" in dynamic linking use cases, could also be statically linked in.
   - POSIX compatibility API
 - command-rt
   - statically linked into commands and executables
   - some or all of this may be synthesized by wasm-ld or other tools
   - calls static initializers
   - provides an `exit` facility
      - eventually, this will work by having a try block outside the user `main` and having `exit` unwind to it
      - as a temporary measure, this can be a magic host call like `proc_exit` in wasi, because Preview2
        doesn't support multiple components.

# How do dynamic libraries work?
 - [Component-based Dynamic Linking]
  - Two-level namespacing, first level is resolved at exe/dll-link time.
     - symbol visibilities, weak symbols, comdat-like functionality, and
       preemption (if we want it) are handled at this time as well, so
       that startup time is simple.
     - If we ever want to do these kinds of things at startup time or runtime,
       we'll need core-wasm support and a dynamic linker.
  - Libraries that need static memory (hi libc) can call malloc in
    their start functions and store the pointer in a wasm global.
  - `dlopen` is out of scope until we get core wasm support
  - Functions don't need PIC, but global variables do.
     - A symbol in an executable can be resolved to a static address.
     - Symbols can also be resolved go globals carrying their addresess,
       initialized at startup time.
  - Symbol visibilities/linkage:
     - "public" (clang/llvm call this "default", but it isn't the default)
        - This does not imply an export from a wasm component
        - Spurious exports are disallowed in command-lifetime programs
     - "hidden" (the actual default): visible to other TU's, but not exported from the dll/exe
     - "local": not visible to other TUs
     - WASM_EXPORT: same as "public", but you can pick the export name
     - extern: in symbol-table, linker error if not resolved at exe/dll-link time
        - may resolve to either something else in the same linker-unit, or to a name defined in
          a library, in which case the linker remembers the library and handles it like `WASM_IMPORT`.
     - WASM_IMPORT: imported from exe/dll with given module name and import
        - Does not imply an import from a wasm component
        - TODO: What is the module-name namespace?

[Component-based Dynamic Linking]: https://github.com/WebAssembly/component-model/blob/main/design/mvp/examples/SharedEverythingDynamicLinking.md

# How do scoped-lifetime modules work?

It's [Typed Main](TypedMain.md) time.

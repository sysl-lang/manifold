# Survey of `manifoldc.h` (Manifold 3.5.4)

Every declaration of the C API, classified before the binding was written: **expressible** (declares
directly), **shim-able** (needs C of its own), or **needs the language to move**.

`nm -gU libmanifoldc.dylib` exports all 294 functions the header declares — none is a macro or a
`static inline` — and `c/c.sysl` declares all 294.

## Result

| bucket | count | what |
|---|---|---|
| expressible | 294 functions, 13 opaque handles, 6 records, 4 enumerations, 5 callback types | everything |
| shim-able | 0 | — |
| needs the language to move | 0 | — |

No shim is needed and no sysl gap was found.

## The shapes, and how each declares

| shape | where | how |
|---|---|---|
| opaque handle (`ManifoldManifold`, `…Vec`, `CrossSection`, `SimplePolygon`, `Polygons`, `MeshGL`, `MeshGL64`, `Box`, `Rect`, `Triangulation`, `ExecutionContext`, `RayHitVec`) | 13 typedefs | `opaque struct`, reached through `*T` |
| caller storage `void *mem` | every constructor | `manifold_alloc_*()` (malloc of `manifold_*_size()`), released by `manifold_delete_*` (destruct + free) — the pleasant layer's `Drop` |
| record by value: `ManifoldVec2`, `ManifoldVec3`, `ManifoldRayHit`, `ManifoldManifoldPair` (two handles) | 13 functions return one; 3 callback signatures carry one | `struct`, by value both ways |
| record by pointer: `ManifoldMeshGLOptions`, `ManifoldMeshGL64Options` | 2 | `struct` of pointers and lengths, built on the stack |
| unused records: `ManifoldIVec3`, `ManifoldVec4`, `ManifoldProperties` | 0 functions | not declared — nothing takes or answers one |
| enumeration (`OpType`, `Error`, `FillRule`, `JoinType`) | 8 functions | `i32`; values written as `val`s (see below) |
| array copied **in** (`float *vert_props`, `uint32_t *tri_verts`, `ManifoldVec2 *`, `ManifoldVec3 *`, `size_t *half_edges`, `double *smoothness`) | 13 | `as_ptr` of a `[]const T` |
| array copied **out** into caller storage (`manifold_meshgl_vert_properties(void *mem, …)` and 20 more) | 21 | a `Buf[T]` reserved to the reported length, its `spare()` handed over, `set_len` after |
| `char *` in (`manifold_read_obj`) | 2 | `sysl.text.cstring` |
| callback, scalars only: `ManifoldSdf` `double (*)(double, double, double, void *)`; `void (*)(char *, void *)` (OBJ writer) | 6 functions | a plain `private` function's address; the `void *` carries a pointer to a frame-local struct holding the closure |
| callback with a record by value: `ManifoldVec3 (*)(double, double, double, void *)` (warp), `ManifoldVec2 (*)(double, double, void *)` (2D warp), `void (*)(double *, ManifoldVec3, const double *, void *)` (set_properties) | 3 functions | `@export` on the trampoline, as the reference requires for an aggregate in a callback signature; one trampoline per signature serves every closure |
| callback run on Manifold's thread pool (warp, level set, set_properties) | — | the closure is a `&sync Fn`, so the compiler checks its captures |
| global state (`manifold_set_circular_segments` and three more) | 4 | plain functions |

Every callback is synchronous — it runs inside the call that registered it — so the closure's box
lives on the calling frame and nothing has to be kept alive afterwards.

## What forced a decision, though not a gap

- **Homebrew's `manifold.pc` is a C++ file.** Its `Cflags` are `-I… -std=c++17`; sysl passes a `.pc`'s
  `Cflags` to every C compile of the package, `c const` probes included, and clang refuses
  `-std=c++17` for C (`error: invalid argument '-std=c++17' not allowed with 'C'`). It also names only
  `-lmanifold`, not the C wrapper. So the package uses `pkg_config` for the library directory only,
  names `manifoldc` with `@link`, and compiles no C: no `@include`, no `c const`, no `@assert` against
  the header. The record layouts (all 8-byte fields) are pinned in `tests.sysl` to what clang measured,
  and the enumeration values (each a C `enum` counting from 0) are written out and checked by
  behaviour — a wrong `OpType`, `FillRule` or `JoinType` code changes an area or volume a test asserts,
  and a wrong `Error` code changes a status a test asserts.
- **`sysl import-c --follow manifold/types.h`** brought `types.h`'s records across but not its
  `enum`s ("an enum declared by another header"), so the eight functions taking one were written by
  hand. Reported as a finding against `import-c`.
- **`Rect`/`Box` `transform` carry only the two corners** (Manifold's own definition), so they are
  right for axis-aligned maps only; said in the docs and tested with a scale.

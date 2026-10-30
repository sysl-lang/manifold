# manifold

Robust boolean operations on solids — union, difference, intersection, hulls, Minkowski sums,
extrusion, level sets — bound to [Manifold](https://github.com/elalish/manifold) through its C API.

*An independent sysl package for Manifold. It is not affiliated with or endorsed by Manifold's authors or trademark holders.*

```sysl
import sh.sysl.manifold.{cube, cylinder, square}

val block = cube(20.0, 20.0, 10.0, center = true)
val drilled = block.difference(cylinder(12.0, 4.0, 4.0, 64, center = true))

print(drilled.volume(), drilled.genus())            // 3498.15…  1: the hole is a handle

val bracket = square(30.0, 10.0).difference(square(10.0, 6.0).translate(10.0, 4.0)).extrude(5.0)

print(bracket.volume())                              // 1200
```

```
dependencies {
  manifold { git = "github.com/sysl-lang/manifold", version = "0.1.0" }
}
```

The coordinate names the package, `manifold`; the module a program imports is `sh.sysl.manifold`.

## Installing Manifold

```
brew install manifold
```

That brings `libmanifold` (the C++ library), `libmanifoldc` (its C API, which this binds) and their
dependencies, TBB and Clipper2. `sysl test .` needs no flags.

**How the library is found.** The manifest's `pkg_config { manifold }` asks `pkg-config` where the
library is, and `@link("manifoldc")` names the C wrapper beside it. No header is compiled: Homebrew's
`manifold.pc` is written for C++ — its `Cflags` carry `-std=c++17`, which clang refuses for C, and sysl
hands a `.pc`'s `Cflags` to every C compile of a package. So this package declares the C API by hand
(`sh/sysl/manifold/c/c.sysl`, all 294 functions) and compiles no C at all; the tests pin every record
layout to what clang measured from `types.h`.

## What it is

Two layers. `sh.sysl.manifold.c` is the C API one declaration per C declaration, caller storage and
all. `sh.sysl.manifold` is what a program imports, and needs no C interop:

| | |
|---|---|
| **`Manifold`** | a solid, held as `&Manifold`; its `Drop` deletes the C object. Immutable: every operation answers a new one. `status()`, `volume()`, `surface_area()`, `genus()`, `num_vert/edge/tri/prop()`, `bounding_box()`, `min_gap`, `ray_cast` |
| primitives | `cube`, `sphere`, `cylinder` (and cones), `tetrahedron`, `empty`, `hull_points` |
| booleans | `union`, `difference`, `intersection`, `boolean(other, Op)`; `batch(solids, Op)` in one evaluation; `split`, `split_by_plane`, `trim_by_plane`; `minkowski_sum`/`_difference`; `decompose`/`compose` |
| hulls | `hull()`, `batch_hull`, `hull_points` |
| transforms | `translate`, `rotate`, `scale`, `mirror`, `transform` (a 3x4 matrix as four column vectors), `warp(f)` with a closure |
| refinement | `refine`, `refine_to_length`, `refine_to_tolerance`, `smooth_out`, `smooth_by_normals`, `smooth(mesh, …)`, `simplify`, `set_tolerance` |
| properties | `set_properties(n, f)` with a closure, `calculate_normals`, `calculate_curvature`, `as_original`, `original_id`, `reserve_ids` |
| **`CrossSection`** | a 2D region: `square`, `circle`, `cross_section(contours, FillRule)`, `hull_of_points`; booleans and `batch_sections`; `offset(delta, JoinType)`; `translate`/`rotate`/`scale`/`mirror`/`transform`/`warp`; `simplify`, `decompose`, `area`, `bounds`; **`extrude`** (with twist and taper) and **`revolve`**; `Manifold.slice(z)` and `project()` answer one |
| **`MeshGL`**, **`MeshGL64`** | meshes in and out. `mesh(num_prop, vert_properties, tri_verts)` builds one from sysl slices (copied in); `of_mesh` turns it into a solid or answers `Err(Status)`; `Manifold.to_mesh()` and every array accessor copy out into a `Buf`. Merge vectors, runs, face ids, tangents through `mesh_with_options` |
| level sets | `level_set(sdf, bounds, edge_length)` with a closure |
| OBJ | `Manifold.to_obj()`, `read_obj(text)`, `read_obj_mesh(text)` |
| **`Context`** | progress and cancellation: `with_context`, and an optional context on `of_mesh`, `smooth`, `level_set` |
| `Box`, `Rect` | bounds: containment, overlap, union, `include` |
| `triangulate` | contours to triangle indices |
| quality | `set_circular_segments`, `set_min_circular_angle`, `set_min_circular_edge_length`, `reset_to_circular_defaults`, `circular_segments(r)` |

**A failed construction does not trap.** Manifold answers an empty solid whose `status()` says why,
and carries the first failure through every later operation; `of_mesh`, `smooth` and `read_obj`
answer that `Status` as an `Err`. `mesh()` refuses arrays that cannot be a mesh at all (fewer than
three properties, a property array that is not a whole number of vertices, a partial triangle) before
C sees them.

**Closures that Manifold may run on its thread pool are `&sync Fn`** — `warp`, `level_set`,
`set_properties` and `CrossSection.warp` — so the compiler checks what they capture.

## What is deliberately absent

- **`manifold_level_set_seq`** (and its context twin): the sequential level set exists for languages
  whose closures cannot run in parallel. A sysl `&sync Fn` can, so `level_set` always takes the
  parallel one. Both are in the raw layer.
- **The `…_vec` and `SimplePolygon`/`Polygons` handles** are not exposed: a set of solids is a
  `[]&Manifold` going in and a `Buf[&Manifold]` coming out, and contours are `[][]Vec2` in and
  `Buf[Buf[Vec2]]` out, so nothing a program holds points into C storage.
- **`manifold_copy`, `manifold_meshgl_copy`, `manifold_cross_section_copy`**: every object is immutable
  and held by `&T`, so a second reference is the copy.
- **`manifold_meshgl_w_tangents`**: `mesh_with_options` takes the tangents.
- **The `*_size` and `alloc`/`destruct` functions** are the pleasant layer's own plumbing.

## Testing

```
sysl test .
SYSL_EXTRA_CFLAGS="-fsanitize=address -g" env DYLD_INSERT_LIBRARIES=/opt/homebrew/opt/tbb/lib/libtbbmalloc.2.dylib sysl test .
```

**The `DYLD_INSERT_LIBRARIES` is for Homebrew's TBB (2023.1), not for this package.** Under ASan,
TBB unloads `libtbbmalloc` (which it `dlopen`ed itself) before its own static destructor
`__TBB_InitOnce::~__TBB_InitOnce` calls into it, so every test binary dies *at exit*, after its test
has passed, with `SEGV on unknown address` under `governor::release_resources`. A test binary that
only links `libmanifoldc` and calls nothing does the same. An inserted library is never unloaded,
which keeps TBB's teardown valid; with it the ASan suite is clean. It is set through `env` because
macOS strips `DYLD_*` variables crossing a system binary such as `caffeinate` or `nice`.

The oracle is closed-form geometry: a box's volume and area, a sphere and a cylinder converging to
their formulae as segments grow, Pappus's theorem for a revolution, the overlap of offset cubes,
a mitred offset's 3 x 3 square, each fill rule's area over two overlapping contours.

Apache-2.0 is Manifold's licence; this binding is ISC.

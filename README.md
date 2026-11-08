# freetype

Ferron bindings to the system FreeType 2 (`libfreetype`, 2.10 or newer),
enough for a game to rasterize text: font faces from a file or from bytes,
pixel sizes, glyphs rendered to 8-bit grayscale bitmaps, vertical metrics,
and kerning. Part of [Ferrebar](https://github.com/ferrebar), the Ferron
game framework. The generated `extern "C"` modules in `src/sys/` stay
inside the package; the safe layer over them is what other packages see.

## Use

Add it to the `anvil.toml` of a package as a git dependency pinned by tag:

```toml
[dependencies]
freetype = { git = "https://github.com/ferrebar/freetype", tag = "v0.1.0" }
```

and import it as `freetype`. Programs using the package link the system
`libfreetype` (`[native] link` in this package's `anvil.toml`), so
`cc -lfreetype` must find it (`libfreetype-dev` on Debian and Ubuntu,
`freetype2` on Arch, `freetype-devel` on Fedora). Versions stay `v0.x` until
the API settles: the minor version marks changes that may break callers,
the patch version fixes; `CHANGELOG.md` lists them.

```ferron
use std.env.Env
use std.fs.Path
use freetype.error.FreeTypeError
use freetype.library.Library

fn main(env: Env) -> Result[Unit, FreeTypeError] {
  with library = Library.new()? {
    with face = library.open_face(env.fs, Path.from("LiberationSans-Regular.ttf"))? {
      face.set_pixel_size(32)?
      let metrics = face.metrics()
      var pen = 0
      var previous = 0
      for code_point in "AVATAR".code_points() {
        let index = face.char_index(code_point)
        if previous != 0 {
          pen += face.kerning(previous, index)?
        }
        let glyph = face.glyph(index)?
        // Draw `glyph.pixels` (`glyph.rows` rows of `glyph.pitch` bytes) with
        // its top-left at (pen + glyph.left, baseline - glyph.top).
        env.console.print("{index}: {glyph.width}x{glyph.rows} at x {pen + glyph.left}")
        pen += glyph.advance
        previous = index
      }
      env.console.print("line height {metrics.line_height}, width {pen}")
    }
  }
  return Ok(Unit)
}
```

`examples/text` renders a line of text into a PNG (`anvil run -- "text"
out.png` in that directory).

- `freetype.library.Library`: the initialized library, a resource:
  `Library.new()` runs `FT_Init_FreeType`. `open_face(fs, path)` opens the
  first face of a font file (FreeType reads it as glyphs load);
  `face_from_bytes(bytes)` opens one from memory, such as an `#[embed]`ded
  font, and the face keeps its own copy of the bytes for its life
  (`FT_New_Memory_Face` reads them until `FT_Done_Face`).
- `freetype.face.Face`: a face, a resource closed with `FT_Done_Face`.
  `set_pixel_size(pixels)` (pixels per em; call it before the rest),
  `char_index(code_point)` (a `u32` from `String.code_points()`; 0, the
  missing-glyph box, when the face has none), `glyph(index)`,
  `kerning(left, right)` in pixels (0 for a pair without kerning),
  `has_kerning()`, `metrics()`, `family_name()`, and `style_name()`.
- `freetype.face.Glyph`: one glyph rendered with `FT_LOAD_RENDER` to 8-bit
  coverage (0 empty, 255 full): `index`, `width`, `rows`, `pitch` (bytes
  from row to row, at least `width`), `left` and `top` (the bearings from
  the pen position to the bitmap's left edge and up from the baseline to
  its top row), `advance`, and `pixels`, `rows * pitch` bytes (empty for a
  space). Values are whole pixels at the face's size.
- `freetype.face.FaceMetrics`: `ascender`, `descender` (negative), and
  `line_height`, in whole pixels at the face's size.
- `freetype.error.FreeTypeError`: `kind` (`CannotOpen`, `UnknownFormat`,
  `InvalidFormat`, `InvalidGlyph`, `InvalidPixelSize`, `InvalidArgument`,
  `OutOfMemory`, `Unsupported`, `Other`), FreeType's `FT_Error` `code`, and
  its message (`FT_Error_String`); it displays as `<kind>: <message>`, such
  as `CannotOpen: cannot open resource`.

### Closing order

FreeType destroys every face of a library in `FT_Done_FreeType`, so a face
must close before its library. Ferron has no lifetimes to tie a `Face` to
the `Library` it came from, so the package counts instead: each face holds
the library, and `FT_Done_FreeType` runs when the `Library` and the last of
its faces have closed, whatever order their scopes end in. A face returned
from the function that made its library, or kept in a font cache longer
than the library's `with` block, stays valid.

### Threads: not `send`

FreeType's rules: an `FT_Library` and its faces are not thread-safe; one
face may be used by one thread at a time, and creating or destroying faces
of one library needs a lock; FreeType suggests one `FT_Library` per thread.
D86's `send` would assert that any thread may use and destroy a handle on
its own, and a face shares its library with the library's other faces, so
neither `Library` nor `Face` is `send`: they stay in the actor that made
them, which keeps FreeType's rules by construction with no lock. A worker
actor that rasterizes glyphs (an atlas builder) makes its own `Library` and
opens its faces from the font's `Bytes`, which can be sent.

## Develop

Needs the FreeType headers (`/usr/include/freetype2`) and `clang` (for
`anvil ffi gen`).

```sh
anvil ffi gen --check
anvil check
anvil test          # every module's tests; `anvil test pkg.tests` filters
anvil fmt
```

The tests use the bundled `tests/fonts/LiberationSans-Regular.ttf`
(Liberation Fonts 2.1.5, SIL Open Font License 1.1, see
`tests/fonts/LiberationSans-LICENSE.txt`), chosen because it has a `kern`
table, which `FT_Get_Kerning` reads (`GPOS` kerning needs a shaper such as
HarfBuzz).

## C bindings

`anvil ffi gen` writes `src/sys/` from `freetype/freetype.h`,
`freetype/ftimage.h`, and `freetype/fterrors.h` (and the `fttypes.h` and
`ftsystem.h` records they hold). `ffi.toml` pins the binding to the
declarations the safe layer uses (`ignore = "."` plus a `require` list),
all present since FreeType 2.10, so the output is the same for the 2.13
headers Ubuntu 24.04 ships and the 2.15 ones of newer distributions; using
another function means adding it to `require`. Nothing in that list is
skipped. The records the package reads (`FT_FaceRec`, the glyph slot, its
bitmap, the size metrics) are `#[c_struct]`s with clang's layout checked by
the compiler. `FT_New_Memory_Face` is hand-written in `src/sys/hand.fer`
with a `Ptr[u8]`: FreeType keeps reading the buffer after the call, and the
generated `CArray[u8]` would pass a copy freed when the call returns. Load
flags, pixel modes, and error codes are `#define`s, which the generator
does not read; `src/face.fer` and `src/error.fer` spell the ones the package
uses.

## CI

`.github/workflows/ci.yml` builds the compiler from `ferron-lang/ferron` at
`FERRON_REF`, a release tag or a commit, installs `libfreetype-dev` and
`clang`, and runs `anvil check`, `anvil test` (checking that the runtime
reports `live_handles=0` afterwards), `anvil fmt --check`, the example, and
`anvil ffi gen --check`. `ferron-lang/ferron` is private, so the workflow
checks it out with the repository secret `FERRON_REPO_TOKEN`: a fine-grained
personal access token with read access to the contents of
`ferron-lang/ferron`.

## License

MIT; see `LICENSE`. The test font is under the SIL Open Font License 1.1.

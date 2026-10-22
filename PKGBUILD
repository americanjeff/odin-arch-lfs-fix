# Local rebuild of the Arch extra/odin PKGBUILD with the Git LFS vendor fix.
#
# Upstream tracks its prebuilt vendor libraries (raylib, SDL3, wgpu, box3d,
# ~24 files) with Git LFS. makepkg checks out git+ sources with
# GIT_LFS_SKIP_SMUDGE=1 (same as the Arch build daemons), so the packaged
# vendor binaries end up as ~130-byte LFS pointer text files and anything
# that links a packaged vendor lib fails. prepare() fetches the LFS objects
# explicitly, which makepkg's smudge-disabled checkout never does.
#
# Full diagnosis: ~/dev/upstream/odin-arch/odin-lfs-bug.md
#
# pkgrel: the repo package for this pkgver is extra/odin dev_2026_10-1, and
# the installed one (CachyOS rebuild) is -1.1, so pkgrel must exceed 1.1 or
# a plain `pacman -Syu` re-upgrades back to the broken repo package. It is
# 3 (2 had only the LFS fix; 3 adds the box3d wasm binding patch below). The
# next upstream release (new pkgver) still upgrades over it, since pacman
# compares pkgver before pkgrel.
#
# Deltas vs the upstream PKGBUILD (kept minimal so re-diffs stay easy):
#   - prepare(): the Git LFS pull (the fix itself)
#   - prepare(): strip the libc-shim require from vendor/box3d/box3d_wasm.odin
#     (the wasm binding otherwise force-embeds libc-shim symbols that collide
#     with emscripten's libc in an emcc build)
#   - package(): explicit install -d for the doc/license dirs (install -D
#     with a multi-file + directory destination does not create them)
#   - no check() (skips the core test suite to keep build times down)
#
# Build:  GIT_LFS_SKIP_SMUDGE=1 makepkg -s && GIT_LFS_SKIP_SMUDGE=1 makepkg
#         (skip-smudge is required, not just faster: as of dev-2026-10 at
#         least one LFS object is missing from the upstream remote — the
#         windows box3d.dll — so any smudge of that tag fails; the Arch
#         build daemons skip smudge for the same reason. prepare() pulls
#         the LFS objects itself and verifies the linux prebuilts.)
# Install: sudo pacman -U odin-${pkgver}-${pkgrel}-*.pkg.tar.zst
#
# Rebuild when extra/odin bumps pkgver/pkgrel and re-diff this file against
# https://gitlab.archlinux.org/archlinux/packaging/packages/odin/-/raw/main/PKGBUILD
# before bumping here.

# Maintainer: Alexander F. Rødseth <xyproto@archlinux.org>
# Contributor: Ivan Souza <ivanttt54@gmail.com>
# Contributor: Jesus Alvarez <jeezusjr@gmail.com>
# Contributor: Kaizhao Zhang <zhangkaizhao@gmail.com>
# Contributor: Payn <ivanttt54@gmail.com>
# Contributor: Johannes Müller <mail@jotrorox.com>

pkgname=odin
pkgver=dev_2026_10 # monthly release tags
pkgrel=3
_raygui=5.0
pkgdesc='Data-oriented programming language'
arch=(x86_64)
url='https://odin-lang.org/'
license=(BSD-3-Clause Zlib)
depends=(clang libx11 llvm21-libs miniaudio python raylib)
makedepends=(git git-lfs llvm21 setconf)
options=(staticlibs)
source=("git+https://github.com/odin-lang/Odin#tag=${pkgver//_/-}"
        "raygui-$_raygui.tar.gz::https://github.com/raysan5/raygui/archive/refs/tags/$_raygui.tar.gz")
b2sums=('b5cd4f27c04a4375aafa5b7090745fcd41b2ffafcceeb86b7568eb1f57ba0532612d4334ff66b213672e401dcd5bdb71e8f822b5e9a43d5e554f2c153485aee3'
        '6f97a2174de0f8b140b8f534386ac92d9425f69ce93ae4dc45c131afd39d812aa5ec8c640144fb36ac7ff818c8ce373f0cfc2ff7c7ed23c22218cea4323766e6')

prepare() {
  cd Odin

  # The LFS fix: makepkg's GIT_LFS_SKIP_SMUDGE=1 checkout leaves the ~24
  # prebuilt vendor libraries (box3d, wgpu, SDL3, ...) as pointer stubs.
  # makepkg builds with HOME=/dev/null, so `git lfs install` (which writes
  # the global gitconfig) fails with "could not lock config file /dev/null";
  # --local writes this repo's .git/config instead. The working copy's
  # origin is makepkg's local bare repo, which holds no LFS objects —
  # they live on GitHub's LFS server — so pull from the real remote.
  unset GIT_LFS_SKIP_SMUDGE
  git lfs install --local
  git remote add upstream "https://github.com/odin-lang/Odin" 2>/dev/null \
    || git remote set-url upstream "https://github.com/odin-lang/Odin"
  # Some LFS objects are missing from the upstream remote (the windows
  # box3d.dll as of dev-2026-10), so the pull can fail even after fetching
  # everything that exists. Tolerate the exit code, then fail loudly if a
  # linux prebuilt is still a pointer stub.
  git lfs pull upstream || true
  missing=""
  for f in $(git lfs ls-files -n); do
    if [ -f "$f" ] && head -1 "$f" | grep -q '^version https://git-lfs'; then
      case "$f" in *linux*) missing="$missing $f" ;; esac
    fi
  done
  if [ -n "$missing" ]; then
    echo "error: linux prebuilts still LFS pointer stubs:$missing" >&2
    exit 1
  fi

  # The vendor:box3d wasm binding force-imports the libc-shim (the
  # @(require) in box3d_wasm.odin, a wasm-only file). That embeds strong
  # references to the libc-shim's libc symbols in every wasm object that
  # imports vendor:box3d; an app that compiles the toolchain's Box3D C with
  # emcc and links one wasm module then collides with emscripten's own
  # libc. Strip the require (the binding works without it; the native build
  # never compiles this file).
  sed -i '/vendor:libc-shim/d' vendor/box3d/box3d_wasm.odin

  # Trickery for using the shared system library for Raylib
  rm vendor/raylib/linux/{libraylib.a,libraylib.so.600,libraygui.a,libraygui.so}
  ln -s /usr/lib/libraylib.so.600 vendor/raylib/linux/libraylib.so.600
  setconf vendor/raylib/raylib.odin RAYLIB_SHARED '#config(RAYLIB_SHARED, true)'
  setconf vendor/raylib/rlgl/rlgl.odin RAYLIB_SHARED '#config(RAYLIB_SHARED, true)'

  # Use miniaudio from the system too
  cd vendor/miniaudio
  setconf common.odin LIB '"system:miniaudio"'
  sed -i '/^when !#exists(LIB) {/,/^}/d;s/f v != BINDINGS_VERSION {/f v.xy != BINDINGS_VERSION.xy {/' common.odin
}

build() {
  cd Odin

  export CXX=clang++
  export LLVM_CONFIG=/usr/lib/llvm21/bin/llvm-config
  make release

  vendor/cgltf/src/build_cgltf.sh
  vendor/stb/src/build_stb.sh

  cd ../raygui-$_raygui/src
  cc $CFLAGS -fno-lto -fPIC -DRAYGUI_IMPLEMENTATION -c -x c raygui.h
  ar rcs ../../Odin/vendor/raylib/linux/libraygui.a raygui.o
  cc $LDFLAGS -shared -Wl,-soname,libraygui.so -o ../../Odin/vendor/raylib/linux/libraygui.so raygui.o -lraylib -lm
}

package() {
  cd Odin

  install -d "$pkgdir/usr/"{bin,lib/$pkgname}
  cp odin "$pkgdir/usr/lib/$pkgname/odin"
  cp -r base core shared vendor "$pkgdir/usr/lib/$pkgname/"
  ln -s "/usr/lib/$pkgname/odin" "$pkgdir/usr/bin/odin"

  # install -D only creates parent dirs for a single-file destination, so
  # the doc/license dirs must be made explicitly.
  install -d "$pkgdir/usr/share/doc/$pkgname" "$pkgdir/usr/share/licenses/$pkgname"
  install -m644 PROPOSAL-PROCESS.md README.md "$pkgdir/usr/share/doc/$pkgname/"
  install -m644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
  install -m644 ../raygui-$_raygui/LICENSE "$pkgdir/usr/share/licenses/$pkgname/raygui-LICENSE"
}

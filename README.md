# odin (local PKGBUILD: Git LFS vendor fix)

Local rebuild of the Arch `extra/odin` package. The upstream package ships
the prebuilt vendor libraries (raylib, SDL3, wgpu, box3d — ~24 files) as
Git LFS pointer stubs, because makepkg checks out `git+` sources with
`GIT_LFS_SKIP_SMUDGE=1`. This PKGBUILD adds `git lfs pull` in `prepare()`
so the real archives land in `/usr/lib/odin/vendor/`.

## What differs from extra/odin

- `makedepends`: adds `git-lfs`
- `prepare()`: fetches the LFS objects. Three quirks are worked around:
  `HOME=/dev/null` during the build breaks global `git lfs install`
  (use `--local`, which writes the repo's own `.git/config`), the working
  copy's `origin` is makepkg's local bare repo, which holds no LFS objects
  (pull from an added `upstream` remote pointing at GitHub), and at least
  one object is missing from the upstream LFS server as of `dev-2026-10`
  (the windows `box3d.dll`) — so the pull tolerates failures and then
  verifies that every *linux* prebuilt is a real archive.
- no `check()`: the core test suite is skipped to keep build times down.
- `pkgrel` exceeds the installed repo package's (CachyOS rebuilds bump
  pkgrel to `-1.1`), so a plain `pacman -Syu` will not clobber this fixed
  build with the still-broken repo package; a fixed upstream release (new
  pkgver) still upgrades over it automatically, since pacman compares
  pkgver before pkgrel.

Everything else mirrors the extra PKGBUILD as of `dev_2026_10-1` (rebased
on the upstream GitLab source, including its raygui-from-source build and
the `llvm21` pin — this machine has llvm21, and the installed binary
links `libLLVM.so.21.1`).

## Build and install

```sh
GIT_LFS_SKIP_SMUDGE=1 makepkg             # builds in this dir
sudo pacman -U odin-dev_2026_10-4-x86_64.pkg.tar.zst
```

(skip-smudge is required, not just faster: with a missing upstream LFS
object, any smudge of the tag fails.)

Verify the fix before installing:

```sh
bsdtar -xOf odin-dev_2026_10-4-x86_64.pkg.tar.zst \
  usr/lib/odin/vendor/box3d/lib/linux-amd64/libbox3d.a \
  | head -c 20 | od -c     # expect an ar archive, not "version https://git-lfs…"
```

Smoke test after installing:

```sh
printf 'package smoke\n\nimport "core:fmt"\n\nmain :: proc() {\n\tfmt.println("toolchain ok")\n}\n' > /tmp/smoke.odin
odin run /tmp/smoke.odin -file
```

## Keeping up with upstream

When `extra/odin` bumps:

1. Diff this PKGBUILD against the new extra PKGBUILD
   (`https://gitlab.archlinux.org/archlinux/packaging/packages/odin/-/raw/main/PKGBUILD`).
2. Bump `pkgver`; set `pkgrel` above the installed repo package's (check
   `pacman -Q odin`), re-run `makepkg -g` for the new checksums, rebuild,
   reinstall.
3. If the LFS bug is fixed upstream (and no LFS objects are missing),
   drop the `prepare()` block, the `git-lfs` makedep, and the
   `GIT_LFS_SKIP_SMUDGE` env and this project can be retired — just use
   the repo package.

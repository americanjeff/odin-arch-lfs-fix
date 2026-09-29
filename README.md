# odin (local PKGBUILD with the Git LFS vendor fix)

Local rebuild of the Arch `extra/odin` package. The upstream package ships
the prebuilt vendor libraries (raylib, SDL3, wgpu, box3d — ~24 files) as
Git LFS pointer stubs, because makepkg checks out `git+` sources with
`GIT_LFS_SKIP_SMUDGE=1`. This PKGBUILD adds `git lfs pull` in `prepare()`
so the real archives land in `/usr/lib/odin/vendor/`.

Full diagnosis of the upstream bug:
`~/dev/upstream/odin-arch/odin-lfs-bug.md`.

## What differs from extra/odin

- `makedepends`: adds `git-lfs`
- `prepare()`: fetches the LFS objects. Two makepkg quirks are worked
  around: `HOME=/dev/null` during the build breaks global
  `git lfs install` (use `--local`, which writes the repo's own
  `.git/config`), and the working copy's `origin` is makepkg's local bare
  repo, which holds no LFS objects (pull from an added `upstream` remote
  pointing at GitHub).
- `pkgrel` matches the current extra pkgrel for this `pkgver` on purpose:
  `pacman -Syu` will not clobber this fixed build with the still-broken
  repo package, and a fixed upstream release (higher pkgrel or new pkgver)
  upgrades over it automatically.

Everything else mirrors the extra PKGBUILD as of `dev_2026_09-3`
(reconstructed from the installed package manifest + upstream tree; the
ABS/GitLab PKGBUILD source was not reachable unauthenticated). The
generated `raygui-LICENSE` matches the installed file except for a few
trailing spaces.

## Build and install

```sh
makepkg -s                                  # builds in this dir
sudo pacman -U odin-${pkgver}-*.pkg.tar.zst
```

Verify the fix before installing:

```sh
bsdtar -xOf odin-*.pkg.tar.zst usr/lib/odin/vendor/raylib/linux/libraylib.a \
  | head -c 20 | od -c     # expect an ar archive, not "version https://git-lfs…"
```

## Keeping up with upstream

When `extra/odin` bumps:

1. Diff this PKGBUILD against the new extra PKGBUILD
   (ABS/GitLab, or `pacman -Si odin` for the metadata).
2. Bump `pkgver`/`pkgrel` to match, re-run `makepkg -g`, rebuild, reinstall.
3. If the LFS bug is fixed upstream, drop the `prepare()` block and the
   `git-lfs` makedep and this project can be retired — just use the repo
   package.

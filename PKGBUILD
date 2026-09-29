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
# pkgrel deliberately matches the current extra/odin pkgrel for this pkgver:
# `pacman -Syu` then treats this fixed build as the current version (it will
# not clobber it with the still-broken repo package), and a fixed upstream
# release (higher pkgrel or new pkgver) upgrades over it automatically.
#
# Build:  makepkg -s
# Install: sudo pacman -U odin-${pkgver}-${pkgrel}-*.pkg.tar.zst
#
# Rebuild when extra/odin bumps pkgver/pkgrel and re-diff this file against
# https://abs.archlinux.org/extra/odin/trunk/PKGBUILD before bumping here.

pkgname=odin
pkgver=dev_2026_09
pkgrel=3
pkgdesc="Data-oriented programming language"
url="https://odin-lang.org/"
license=(BSD-3-Clause Zlib)
arch=(x86_64)

# extra pins llvm21; this build uses the system llvm (22 here), so track the
# unversioned packages and adjust if the machine's default llvm changes.
depends=(clang libx11 llvm-libs miniaudio python raylib)
makedepends=(git git-lfs llvm stb)

_repository="https://github.com/odin-lang/Odin"
source=("git+${_repository}#tag=${pkgver//_/-}")

sha256sums=('330e1ef09bcc30a062187bcfb9c9a0061ed16afcc41b3b3bb3102efccdeb82ee')

prepare() {
	cd "$srcdir/Odin"
	# makepkg builds with HOME=/dev/null, so `git lfs install` (which writes
	# the global gitconfig) fails with "could not lock config file /dev/null";
	# --local writes this repo's .git/config instead. The working copy's
	# origin is makepkg's local bare repo, which holds no LFS objects —
	# they live on GitHub's LFS server — so pull from the real remote.
	git lfs install --local
	git remote add upstream "$_repository" 2>/dev/null \
		|| git remote set-url upstream "$_repository"
	git lfs pull upstream
}

build() {
	cd "$srcdir/Odin"
	make release
	cd vendor/cgltf/src
	./build_cgltf.sh
}

package() {
	cd "$srcdir/Odin"

	install -Dm755 odin "$pkgdir/usr/lib/odin/odin"
	mkdir -p "$pkgdir/usr/bin"
	ln -s ../lib/odin/odin "$pkgdir/usr/bin/odin"

	cp -r base core shared vendor "$pkgdir/usr/lib/odin/"

	# install -D only creates parent dirs for a single-file destination, so
	# the doc/license dirs must be made explicitly.
	mkdir -p "$pkgdir/usr/share/doc/odin" "$pkgdir/usr/share/licenses/odin"
	install -m644 README.md PROPOSAL-PROCESS.md "$pkgdir/usr/share/doc/odin/"
	install -m644 LICENSE "$pkgdir/usr/share/licenses/odin/LICENSE"

	# raygui zlib license, extracted from the rlgl.odin header comment
	# (mirrors the raygui-LICENSE shipped by the extra package)
	{
		echo "zlib License"
		printf '\t\n'
		sed -n '/Copyright (c) 2014-2026 Ramon/,/source distribution/p' \
			vendor/raylib/rlgl/rlgl.odin \
			| sed -e 's/^\*   //' -e 's/^\*$//'
	} > "$pkgdir/usr/share/licenses/odin/raygui-LICENSE"
}

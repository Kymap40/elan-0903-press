# Maintainer: (add your name/handle)
# Patched libfprint for the ELAN 04f3:0903 96x96 press-and-hold fingerprint
# sensor (ASUS VivoBook / Zenbook family). Adds press-mode capture and SIGFM
# matching on top of upstream libfprint v1.94.100, adapted from the AUR package
# libfprint-elan-0c28-patch (LGPL-2.1-or-later) — see CREDITS.md.

pkgname=libfprint-elan-0903-patch
pkgver=1.94.100
pkgrel=1
pkgdesc="libfprint patched for the ELAN 04f3:0903 96x96 press-and-hold sensor"
arch=(x86_64)
url="https://gitlab.freedesktop.org/libfprint/libfprint"
license=(LGPL-2.1-or-later)
depends=(glib2 glibc libgusb libgudev nss pixman)
makedepends=(meson ninja glib2-devel)
provides=("libfprint=$pkgver" "libfprint-2.so=2-64")
conflicts=(libfprint)
groups=(fprint)
source=("https://gitlab.freedesktop.org/libfprint/libfprint/-/archive/v$pkgver/libfprint-v$pkgver.tar.gz"
        "elan-0903-press-sigfm.patch")
sha256sums=('edc90e02f330a7595ceaf37f2c6ec32ed43541347fe936d3273b0bb2524fd19c'
            'b361b82383c166cd76964487c00ecd93538d3c0e2079ae383c5d18345358d18f')

prepare() {
  cd "libfprint-v$pkgver"
  patch -Np1 -i "$srcdir/elan-0903-press-sigfm.patch"
}

build() {
  meson setup "libfprint-v$pkgver/build" "libfprint-v$pkgver" \
    --prefix=/usr --libdir=lib --buildtype=plain \
    -Ddrivers=all -Ddoc=false -Dintrospection=false -Dinstalled-tests=false
  meson compile -C "libfprint-v$pkgver/build"
}

package() {
  meson install -C "libfprint-v$pkgver/build" --destdir "$pkgdir"
}

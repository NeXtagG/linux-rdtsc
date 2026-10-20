# Linux 7.2.5 + rdtsc.patch (KVM hides the VM-exit spike of RDTSC from the guest).
# Сборка:  makepkg -si
pkgname=linux-rdtsc
pkgver=7.2.5
pkgrel=2
pkgdesc='Linux 7.2.5 (custom rdtsc build)'
url='https://github.com/NeXtagG/linux-rdtsc'
arch=(x86_64)
license=(GPL-2.0-only)
makedepends=(bc cpio gettext libelf pahole perl python rust rust-bindgen rust-src tar xz zstd)
depends=(coreutils kmod initramfs)
optdepends=(linux-firmware)
provides=(KSMBD-MODULE NTSYNC-MODULE VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE)
options=('!strip' '!debug')
source=("https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/linux-$pkgver.tar.xz"
        rdtsc.patch
        config)
sha256sums=('55ddf0df8325d9dad96fcff7bd93977d22e3f50af06527572af59b77c7632b78'
            '5718135588388ac1266f2a195b9d8d4434351a1ab4c67f8b5cd24ddac5034360'
            '853ff082f8a08d4c5e833df1d06e429d168156c793a5b4464b6b16d33d72c137')

# Сколько тактов срезать с каждого шага TSC (1500-4000), например: VMEXIT_COST=3000 makepkg -si
: "${VMEXIT_COST:=2500}"

prepare() {
  cd "linux-$pkgver"
  patch -Np1 < ../rdtsc.patch
  cp ../config .config
  make olddefconfig
}

build() {
  cd "linux-$pkgver"
  make KCFLAGS="-DVMEXIT_COST=${VMEXIT_COST}ULL" all
}

package() {
  cd "linux-$pkgver"
  local kernver="$(<include/config/kernel.release)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"
  echo "$pkgname" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  ZSTD_CLEVEL=19 make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install

  rm -f "$modulesdir/build"
}

# Maintainer: graysky <graysky@archlinux.us>
# Maintainer: Kevin Mihelich <kevin@archlinuxarm.org>
# Maintainer: Oleg Rakhmanov <oleg@archlinuxarm.org>
# Maintainer: Dave Higham <pepedog@archlinuxarm.org>
# Contributer: Jan Alexander Steffens (heftig) <heftig@archlinux.org>

pkgbase=linux-rpi-sane
_commit=a90998a3e549911234f9f707050858b98b71360f
_srcname=linux-${_commit}
_kernelname=${pkgbase#linux}
pkgver=5.15.56
pkgrel=2
pkgdesc='Linux'
url="http://www.kernel.org/"
#arch=(armv7h aarch64)
arch=(aarch64)
license=(GPL2)
makedepends=(
  bc kmod
)
options=('!strip')
#source_armv7h=('config')
source_aarch64=('config8')
source=("linux-$pkgver-${_commit:0:10}.tar.gz::https://github.com/raspberrypi/linux/archive/${_commit}.tar.gz"
        0001-Make-proc-cpuinfo-consistent-on-arm64-and-arm.patch
        linux.preset
        60-linux.hook
        90-linux.hook
)
md5sums=('1dcd7b6ed031b4b54ee95bf6bc9a4eed'
         'f66a7ea3feb708d398ef57e4da4815e9'
         '73007c2069a22d142e5772b54572d58c'
         '0a5f16bfec6ad982a2f6782724cca8ba'
         '441ec084c47cddc53e592fb0cbce4edf')
#md5sums_armv7h=('5d58e497340f67bb54e0b86855457904')
md5sums_aarch64=('919ed2e5819c3200f7983dda93889d8a')

# setup vars
if [[ $CARCH == "armv7h" ]]; then
  _kernel="kernel7.img" KARCH=arm _image=zImage _config=config
elif [[ $CARCH == "aarch64" ]]; then
  _kernel="kernel8.img" KARCH=arm64 _image=Image _config=config8
fi

prepare() {
  cd "${srcdir}/${_srcname}"

  # consistent behavior of lscpu on arm/arm64
  patch -Np1 -i ../0001-Make-proc-cpuinfo-consistent-on-arm64-and-arm.patch

  echo "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  echo "Setting config..."
  cp ../"$_config" .config
  make olddefconfig

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd "${srcdir}/${_srcname}"

  make "$_image" modules dtbs
}

_package() {
  pkgdesc="RPi Foundation patched Linux kernel and modules"
  depends=('coreutils' 'linux-firmware' 'kmod' 'mkinitcpio>=0.7' 'firmware-raspberrypi')
  optdepends=('wireless-regdb: to set the correct wireless channels of your country')
  provides=("linux=${pkgver}" 'WIREGUARD-MODULE')
  conflicts=('linux' 'uboot-raspberrypi')
  #install=${pkgname}.install
  #backup=('boot/config.txt' 'boot/cmdline.txt')
  replaces=('linux-raspberrypi-latest' 'linux-raspberrypi4')

  cd "${srcdir}/${_srcname}"

  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install  # Suppress depmod

  # remove build and source links
  rm "$modulesdir"/{source,build}

  echo "Installing Arch ARM specific stuff..."
  local dtbdir="${pkgdir}/boot/${pkgbase}/"
  mkdir -p "${dtbdir}"
  make INSTALL_DTBS_PATH="${dtbdir}" dtbs_install

  if [[ $CARCH == "aarch64" ]]; then
    # drop hard-coded devicetree=foo.dtb in /boot/config.txt for
    # autodetected load of supported of models at boot
    find "${dtbdir}/broadcom" -type f -print0 | xargs -0 mv -t "${dtbdir}"
    rmdir "${dtbdir}/broadcom"
  fi

  cp arch/$KARCH/boot/$_image "${pkgdir}/boot/${pkgbase}/$_kernel"
  cp arch/$KARCH/boot/dts/overlays/README "${pkgdir}/boot/${pkgbase}/overlays/"

  # sed expression for following substitutions
  local _subst="
    s|%PKGBASE%|${pkgbase}|g
    s|%KERNVER%|${kernver}|g
  "

  # install mkinitcpio preset file
  sed "${_subst}" ../linux.preset |
    install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"

  # install pacman hooks
  sed "${_subst}" ../60-linux.hook |
    install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/60-${pkgbase}.hook"
  sed "${_subst}" ../90-linux.hook |
    install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/90-${pkgbase}.hook"
}

_package-headers() {
  pkgdesc="RPi Foundation header and scripts for building modules for Linux kernel"
  provides=("linux-headers=${pkgver}")
  #conflicts=('linux-headers')
  #replaces=('linux-raspberrypi-latest-headers' 'linux-raspberrypi4-headers')

  cd ${_srcname}
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/$KARCH" -m644 "arch/$KARCH/Makefile"
  cp -t "$builddir" -a scripts

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/$KARCH" -a "arch/$KARCH/include"
  install -Dt "$builddir/arch/$KARCH/kernel" -m644 "arch/$KARCH/kernel/asm-offsets.s"

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local _arch
  for _arch in "$builddir"/arch/*/; do
    if [[ $CARCH == "aarch64" ]]; then
      [[ $_arch = */"$KARCH"/ || $_arch == */arm/ ]] && continue
    else
      [[ $_arch = */"$KARCH"/ ]] && continue
    fi
    echo "Removing $(basename "$_arch")"
    rm -r "$_arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=("${pkgbase}" "${pkgbase}-headers")
for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    _package${_p#${pkgbase}}
  }"
done

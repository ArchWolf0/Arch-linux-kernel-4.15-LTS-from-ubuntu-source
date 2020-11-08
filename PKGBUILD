set -u
pkgbase="linux-ubuntu"
pkgver="4.15.122"
_srcname="linux-${pkgver%.*}"
pkgrel='124'
arch=('x86_64')
url="https://git.launchpad.net/"
license=('GPL2')
makedepends=('xmlto' 'kmod' 'inetutils' 'bc' 'libelf' 'git')
options=('!strip')
source=(
  'config'         # the main kernel config file
  '60-linux.hook'  # pacman hook for depmod
  '90-linux.hook'  # pacman hook for initramfs regeneration
  'linux-ubuntu.preset'   # standard config files for mkinitcpio ramdisk
  'git://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/bionic'

)
if ! :; then
  source+=("${source[0]/.xz/.sign}")
  source+=("${source[1]/.xz/.sign}") # patch>=4.14.59 are not signed
fi


md5sums=('7f54a16de09349d4dfbae122d25f75d5'
         'ce6c81ad1ad1f8b333fd6077d47abdaf'
         'a85bfae59eb537b973c388ffadb281ff'
         'a329f9581060d555dc7358483de9760a'
         'SKIP')

sha256sums=('SKIP'
            'SKIP'
            'SKIP'
            'SKIP'
            'SKIP')

_kernelname=${pkgbase#linux}

# CFLAGS CXXFLAGS are not used at all.
# LDFLAGS only works on the make command line. It is not used from the environment.
_makeopts=(
  #CC='gcc-5' CXX='g++-5' HOSTCC='gcc-5' HOSTCXX='g++-5' # LD='ld.gold'
  #CC='gcc-6' CXX='g++-6' HOSTCC='gcc-6' HOSTCXX='g++-6'
  #LDFLAGS='-z max-page-size=0x200000' # http://lists.gnu.org/archive/html/bug-binutils/2018-03/msg00193.html
)
#makedepends+=('gcc5')


prepare() {
  set -u
  mv "${srcdir}"/bionic "${srcdir}/linux-4.15"
  cd "${_srcname}" && 
  cp ./debian/scripts/retpoline-extract-one ./scripts/ubuntu-retpoline-extract-one
  chmod +x tools/objtool/sync-check.sh  # GNU patch doesn't support git-style file mode

  local _f
 
 shopt -u nullglob

  declare -A _config=([x86_64]='config')
  cat "${srcdir}/${_config[${CARCH}]}" > './.config'
  if [ "${_kernelname}" != "" ]; then
    sed -i "s|CONFIG_LOCALVERSION=.*|CONFIG_LOCALVERSION=\"${_kernelname}\"|g" ./.config
    sed -i "s|CONFIG_LOCALVERSION_AUTO=.*|CONFIG_LOCALVERSION_AUTO=n|" ./.config
  fi
  cp './.config' "${srcdir}/config.cmp"
  rm -f "${startdir}/${_config[${CARCH}]}.new"

  # set extraversion to pkgrel
  sed -ri "s|^(EXTRAVERSION =).*|\1 -${pkgrel}|" Makefile

  # don't run depmod on 'make install'. We'll do this ourselves in packaging
  sed -i '2iexit 0' scripts/depmod.sh

  set +u; msg2 'get kernel version'; set +u
  set -x
  make -s -j1 prepare "${_makeopts[@]}"
  set +x

  if ! diff -pNau5 "${srcdir}/config.cmp" './.config'; then
    ln -s "${PWD}/.config" "${startdir}/${_config[${CARCH}]}.new"
    rm "${srcdir}/config.cmp"
    set +u
    echo 'Some changes were made. Please merge for automation.'
    false
  else
    rm "${srcdir}/config.cmp"
  fi

  # load configuration
  # Configure the kernel. Replace the line below with one of your choice.
  make menuconfig # CLI menu for configuration
  #make nconfig # new CLI menu for configuration
  #make xconfig # X-based configuration
  #make oldconfig # using old config from previous kernel version
  # ... or manually edit .config

  set +u; msg2 'rewrite configuration'; set +u
  set -x
  yes "" | make -j1 config "${_makeopts[@]}" >/dev/null
  set +x
  set +u
}

build() {
  set -u
  cd "${srcdir}/${_srcname}"

  local _mflags=()
  local _nproc="$(nproc)"; _nproc=$((_nproc>8?8:_nproc))
  if [ -z "${MAKEFLAGS:=}" ] || [ "${MAKEFLAGS//-j/}" = "${MAKEFLAGS}" ]; then
    _mflags+=('-j' "${_nproc}")
  fi

  rm -f vmlinux vmlinux.o # force relink, for debugging binutils 2.31.1

  set -x
  nice make  "${_makeopts[@]}" "${_mflags[@]}" ${MAKEFLAGS} LOCALVERSION= bzImage modules
  set +x
  set +u
}

_package() {
  set -u
  pkgdesc="The ${pkgbase/linux/Linux} kernel and modules"
  #[ "${pkgbase}" = "linux" ] && groups=('base')
  depends=('coreutils' 'linux-firmware' 'kmod' 'mkinitcpio>=0.7')
  optdepends=('crda: to set the correct wireless channels of your country')
  backup=("etc/mkinitcpio.d/${pkgbase}.preset")
  install=linux-ubuntu.install
  provides=("linux=${pkgver}")

  cd "${_srcname}"

  # get kernel version
  _kernver="$(make -j1 "${_makeopts[@]}" LOCALVERSION= kernelrelease)"
  _basekernel=${_kernver%%-*}
  _basekernel=${_basekernel%.*}

  mkdir -p "${pkgdir}"/{boot,usr/lib/modules}
  make -j1 "${_makeopts[@]}" LOCALVERSION= INSTALL_MOD_PATH="${pkgdir}/usr" modules_install
  cp arch/x86/boot/bzImage "${pkgdir}/boot/vmlinuz-${pkgbase}"

  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  ln -sr "${pkgdir}/boot/vmlinuz-${pkgbase}" "${pkgdir}/usr/lib/modules/${_kernver}/vmlinuz"

  # make room for external modules
  local _extramodules="extramodules-${_basekernel}${_kernelname:--ubuntu}"
  ln -s "../${_extramodules}" "${pkgdir}/usr/lib/modules/${_kernver}/extramodules"

  # add real version for building modules and running depmod from hook
  echo "${_kernver}" |
    install -Dm644 /dev/stdin "${pkgdir}/usr/lib/modules/${_extramodules}/version"

  # remove build and source links
  rm "${pkgdir}"/usr/lib/modules/${_kernver}/{source,build}

  # now we call depmod...
  depmod -b "${pkgdir}/usr" -F System.map "${_kernver}"

  # add vmlinux
  install -Dt "${pkgdir}/usr/lib/modules/${_kernver}/build" -m644 vmlinux

  # sed expression for following substitutions
  local _subst="
    s|%PKGBASE%|${pkgbase}|g
    s|%KERNVER%|${_kernver}|g
    s|%EXTRAMODULES%|${_extramodules}|g
  "

  # hack to allow specifying an initially nonexisting install file
  sed "${_subst}" "${startdir}/${install}" > "${startdir}/${install}.pkg"
  true && install=${install}.pkg

  # install mkinitcpio preset file
  sed "${_subst}" ../linux-ubuntu.preset |
    install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"

  # install pacman hooks
  sed "${_subst}" ../60-linux.hook |
    install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/60-${pkgbase}.hook"
  sed "${_subst}" ../90-linux.hook |
    install -Dm644 /dev/stdin "${pkgdir}/usr/share/libalpm/hooks/90-${pkgbase}.hook"
  set +u
}

_package-headers() {
  set -u
  pkgdesc="Header files and scripts for building modules for ${pkgbase/linux/Linux} kernel"
  provides=("linux-headers=${pkgver}")

  cd ${_srcname}
  local _builddir="${pkgdir}/usr/lib/modules/${_kernver}/build"

  install -Dt "${_builddir}" -m644 Makefile .config Module.symvers
  install -Dt "${_builddir}/kernel" -m644 kernel/Makefile

  mkdir "${_builddir}/.tmp_versions"

  cp -t "${_builddir}" -a include scripts

  install -Dt "${_builddir}/arch/x86" -m644 arch/x86/Makefile
  install -Dt "${_builddir}/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  cp -t "${_builddir}/arch/x86" -a arch/x86/include

  install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
  install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

  # http://bugs.archlinux.org/task/9912
  install -Dt "${_builddir}/drivers/media/dvb-core" -m644 drivers/media/dvb-core/*.h

  # http://bugs.archlinux.org/task/13146
  install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # http://bugs.archlinux.org/task/20402
  install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # add xfs and shmem for aufs building
  mkdir -p "${_builddir}"/{fs/xfs,mm}

  # copy in Kconfig files
  find . -name Kconfig\* -exec install -Dm644 {} "${_builddir}/{}" \;

  # add objtool for external module building and enabled VALIDATION_STACK option
  install -Dt "${_builddir}/tools/objtool" tools/objtool/objtool

  # remove unneeded architectures
  local _arch
  for _arch in "${_builddir}"/arch/*/; do
    [[ ${_arch} == */x86/ ]] && continue
    rm -r "${_arch}"
  done

  # remove files already in linux-docs package
  rm -r "${_builddir}/Documentation"

  # remove now broken symlinks
  find -L "${_builddir}" -type l -printf 'Removing %P\n' -delete

  # Fix permissions
  chmod -R u=rwX,go=rX "${_builddir}"

  # strip scripts directory
  local _binary _strip
  while read -rd '' _binary; do
    case "$(file -bi "${_binary}")" in
      *application/x-sharedlib*)  _strip="${STRIP_SHARED}"   ;; # Libraries (.so)
      *application/x-archive*)    _strip="${STRIP_STATIC}"   ;; # Libraries (.a)
      *application/x-executable*) _strip="${STRIP_BINARIES}" ;; # Binaries
      *) continue ;;
    esac
    /usr/bin/strip ${_strip} "${_binary}"
  done < <(find "${_builddir}/scripts" -type f -perm -u+w -print0 2>/dev/null)
  set +u
}

_package-docs() {
  set -u
  pkgdesc="Kernel hackers manual - HTML documentation that comes with the ${pkgbase/linux/Linux} kernel"

  cd "${_srcname}"
  local _builddir="${pkgdir}/usr/lib/modules/${_kernver}/build"

  mkdir -p "${_builddir}"
  cp -t "${_builddir}" -a Documentation

  # Fix permissions
  chmod -R u=rwX,go=rX "${_builddir}"
  set +u
}

pkgname=("${pkgbase}" "${pkgbase}-headers" "${pkgbase}-docs")
for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    $(declare -f "_package${_p#${pkgbase}}")
    _package${_p#${pkgbase}}
  }"
done

set +u

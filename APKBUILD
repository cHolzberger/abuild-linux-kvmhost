# Maintainer: Natanael Copa <ncopa@alpinelinux.org>

_flavor=kvm
pkgname=linux-${_flavor}
pkgver=4.9.63
case $pkgver in
	*.*.*)	_kernver=${pkgver%.*};;
	*.*) _kernver=$pkgver;;
esac
pkgrel=5
pkgdesc="Linux kvm kernel"
url="http://kernel.org"
depends="mkinitfs linux-firmware"
makedepends="perl sed installkernel bash gmp-dev bc linux-headers elfutils-dev"
options="!strip"
_config=${config:-config-kvm.${CARCH}}
install=
source="https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/linux-$_kernver.tar.xz
	0001-HID-apple-fix-Fn-key-Magic-Keyboard-on-bluetooth.patch
	config-kvm.x86_64
	
	"
if [ "${pkgver%.0}" = "$pkgver" ]; then
	source="$source
	https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/patch-$pkgver.xz"
fi
subpackages="$pkgname-dev::$CBUILD_ARCH"
arch="all"
license="GPL2"

_abi_release=${pkgver}-${pkgrel}-${_flavor}
_carch=${CARCH}
case "$_carch" in
aarch64*) _carch="arm64" ;;
arm*) _carch="arm" ;;
ppc*) _carch="powerpc" ;;
s390*) _carch="s390" ;;
esac

HOSTCC="${CC:-gcc}"
HOSTCC="${HOSTCC#${CROSS_COMPILE}}"

prepare() {
	local _patch_failed=
	cd "$srcdir"/linux-$_kernver
	if [ "$_kernver" != "$pkgver" ]; then
		msg "Applying patch-$pkgver.xz"
		unxz -c < "$srcdir"/patch-$pkgver.xz | patch -p1 -N || return 1
	fi

	# first apply patches in specified order
	for i in $source; do
		case $i in
		*.patch)
			msg "Applying $i..."
			if ! patch -s -p1 -N -i "$srcdir"/$i; then
				echo $i >>failed
				_patch_failed=1
			fi
			;;
		esac
	done

	if ! [ -z "$_patch_failed" ]; then
		error "The following patches failed:"
		cat failed
		return 1
	fi

	mkdir -p "$srcdir"/build
	cp "$srcdir"/$_config "$srcdir"/build/.config || return 1
	make -C "$srcdir"/linux-$_kernver O="$srcdir"/build ARCH="$_carch" HOSTCC="$HOSTCC" \
		silentoldconfig
}

# this is so we can do: 'abuild menuconfig' to reconfigure kernel
menuconfig() {
	cd "$srcdir"/build || return 1
	make ARCH="$_carch" menuconfig
	cp .config "$startdir"/$_config
}

build() {
	cd "$srcdir"/build
	unset LDFLAGS
	make ARCH="$_carch" CC="${CC:-gcc}" \
		KBUILD_BUILD_VERSION="$((pkgrel + 1 ))-Alpine" \
		|| return 1
}

package() {
	cd "$srcdir"/build

	mkdir -p "$pkgdir"/boot "$pkgdir"/lib/modules

	local _install
	case "$CARCH" in
	aarch64*|arm*)	_install="zinstall dtbs_install" ;;
	*)		_install="install" ;;
	esac

	make -j1 modules_install firmware_install $_install \
		ARCH="$_carch" \
		INSTALL_MOD_PATH="$pkgdir" \
		INSTALL_PATH="$pkgdir"/boot \
		INSTALL_DTBS_PATH="$pkgdir"/usr/lib/linux-${_abi_release} \
		|| return 1

	rm -f "$pkgdir"/lib/modules/${_abi_release}/build \
		"$pkgdir"/lib/modules/${_abi_release}/source
	rm -rf "$pkgdir"/lib/firmware

	install -D include/config/kernel.release \
		"$pkgdir"/usr/share/kernel/$_flavor/kernel.release
}

dev() {
	# copy the only the parts that we really need for build 3rd party
	# kernel modules and install those as /usr/src/linux-headers,
	# simlar to what ubuntu does
	#
	# this way you dont need to install the 300-400 kernel sources to
	# build a tiny kernel module
	#
	pkgdesc="Headers and script for third party modules for grsec kernel"
	depends="gmp-dev bash perl"
	local dir="$subpkgdir"/usr/src/linux-headers-${_abi_release}

	# first we import config, run prepare to set up for building
	# external modules, and create the scripts
	mkdir -p "$dir"
	cp "$srcdir"/$_config "$dir"/.config
	make -j1 -C "$srcdir"/linux-$_kernver O="$dir" ARCH="$_carch" HOSTCC="$HOSTCC" \
		silentoldconfig prepare modules_prepare scripts

	# needed for 3rd party modules
	# https://bugzilla.kernel.org/show_bug.cgi?id=11143
	case "$CARCH" in
	ppc*) (cd "$dir" && make arch/powerpc/lib/crtsavres.o);;
	esac

	# remove the stuff that points to real sources. we want 3rd party
	# modules to believe this is the soruces
	rm "$dir"/Makefile "$dir"/source

	# copy the needed stuff from real sources
	#
	# this is taken from ubuntu kernel build script
	# http://kernel.ubuntu.com/git/ubuntu/ubuntu-zesty.git/tree/debian/rules.d/3-binary-indep.mk

	cd "$srcdir"/linux-$_kernver
	find . -path './include/*' -prune \
		-o -path './scripts/*' -prune -o -type f \
		\( -name 'Makefile*' -o -name 'Kconfig*' -o -name 'Kbuild*' -o \
		   -name '*.sh' -o -name '*.pl' -o -name '*.lds' \) \
		-print | cpio -pdm "$dir" || return 1
	cp -a scripts include "$dir"
	find $(find arch -name include -type d -print) -type f \
		| cpio -pdm "$dir"

	install -Dm644 "$srcdir"/build/Module.symvers \
		"$dir"/Module.symvers

	mkdir -p "$subpkgdir"/lib/modules/${_abi_release}
	ln -sf /usr/src/linux-headers-${_abi_release} \
		"$subpkgdir"/lib/modules/${_abi_release}/build
}

sha512sums="bf67ff812cc3cb7e5059e82cc5db0d9a7c5637f7ed9a42e4730c715bf7047c81ed3a571225f92a33ef0b6d65f35595bc32d773356646df2627da55e9bc7f1f1a  linux-4.9.tar.xz
5373728be2b507c3db5e042e1d768740df7965078868afdc46418b1adc4cae3d8f9f1aedb59975a0f2acf8754340499354fcf97c503397a5d9886ccc9689b782  0001-HID-apple-fix-Fn-key-Magic-Keyboard-on-bluetooth.patch
3ade120405fed735ba857e075e2fdd7637fe2c547f7073c35f092521e5b228f27ecb0a5e6666c2eea4601ef5b5dcc24615aacca6dc485b25a9ecbbdb5eaa1c8b  config-kvm.x86_64
51fa0a5c19989be5326130421afe29aeb22bc7be65adbe882fc32b0b22c414e8631e3e7cefb6fecfc23545851a19211f5feddd7196934f5c84ec507b4f2984c0  patch-4.9.63.xz"

# Maintainer: Natanael Copa <ncopa@alpinelinux.org>

_flavor=ck
pkgname=linux-${_flavor}
pkgver=4.14.8
case $pkgver in
	*.*.*)	_kernver=${pkgver%.*};;
	*.*) _kernver=$pkgver;;
esac
pkgrel=1
pkgdesc="Linux kvm kernel"
url="http://kernel.org"
depends="mkinitfs linux-firmware"
makedepends="perl sed installkernel bash gmp-dev bc linux-headers elfutils-dev"
options="!strip"
_config=${config:-config-kvm.${CARCH}}
install=
source="https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/linux-$_kernver.tar.xz
	0001-HID-apple-fix-Fn-key-Magic-Keyboard-on-bluetooth.patch
	0001-MuQSS-version-0.162-CPU-scheduler.patch
	0002-Make-preemptible-kernel-default.patch
	0003-Expose-vmsplit-for-our-poor-32-bit-users.patch
	0004-Create-highres-timeout-variants-of-schedule_timeout-.patch
	0005-Special-case-calls-of-schedule_timeout-1-to-use-the-.patch
	0006-Convert-msleep-to-use-hrtimers-when-active.patch
	0007-Replace-all-schedule-timeout-1-with-schedule_min_hrt.patch
	0008-Replace-all-calls-to-schedule_timeout_interruptible-.patch
	0009-Replace-all-calls-to-schedule_timeout_uninterruptibl.patch
	0010-Don-t-use-hrtimer-overlay-when-pm_freezing-since-som.patch
	0011-Make-hrtimer-granularity-and-minimum-hrtimeout-confi.patch
	0012-Reinstate-default-Hz-of-100-in-combination-with-MuQS.patch
	0013-Make-threaded-IRQs-optionally-the-default-which-can-.patch
	0014-Swap-sucks.patch
	0015-Enable-BFQ-io-scheduler-by-default.patch
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

oldconfig() {
	cd "$srcdir"/build || return 1
	make ARCH="$_carch" oldconfig
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

	make -j1 modules_install $_install \
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

sha512sums="77e43a02d766c3d73b7e25c4aafb2e931d6b16e870510c22cef0cdb05c3acb7952b8908ebad12b10ef982c6efbe286364b1544586e715cf38390e483927904d8  linux-4.14.tar.xz
5373728be2b507c3db5e042e1d768740df7965078868afdc46418b1adc4cae3d8f9f1aedb59975a0f2acf8754340499354fcf97c503397a5d9886ccc9689b782  0001-HID-apple-fix-Fn-key-Magic-Keyboard-on-bluetooth.patch
9773f2dd65270c5e0e1bdf915be01d987a5148ee329d0ef3e4d4fa0abc89c90150805a911d6036765ea8c30645fb9a80f8f183337aa5a17360dbe8ed596000df  0001-MuQSS-version-0.162-CPU-scheduler.patch
c9a160209167ad1816fafeddd2f1e6b53cd3e801ccd158bcd348c5c5735cf7a0b6a13d7c019731eef1a4ce6b2ea10fe99bc78702378dfeb672fe9d214d486276  0002-Make-preemptible-kernel-default.patch
c98f0f77cd06a2ca021215a20c922acfa7878f8f9bf5b50d91fadaac7b47a1171440d6c1dcc09cd82be34e88161b1faabcb92bff64f5c2030d13d67218263421  0003-Expose-vmsplit-for-our-poor-32-bit-users.patch
aa41f0e7096478969724f73624d5481bdf6a01cc4295ad3d8ddf575539b2b6f4521a3df33a729107763468f9e6e5cffea57517e23d301fcb58ab4e77ca3343bf  0004-Create-highres-timeout-variants-of-schedule_timeout-.patch
55c349688784880291f91e11800d7f3331a45e3ccaf68c0ab351f9a980f710d5109334d62054cfda30fa1181c33d8199d073726a52524e478fa79b835872aea0  0005-Special-case-calls-of-schedule_timeout-1-to-use-the-.patch
298e40652ebc0050cbfd7c31b77471fa19a18723f91d2de47129a4164b3f1426fad7ca629f76e676e38d486c33db9e67c4d405214d7ce4a2ca41df6ba0233f33  0006-Convert-msleep-to-use-hrtimers-when-active.patch
aba7b866c291e4716ed791d5d563c8c8fd5479b94bcca5d3129e365ae4dac77fbf0c7f6ac9cf6a845b8e2ba05f0cbd4ad27cdd58faff3286592ab6a1d6c3aa20  0007-Replace-all-schedule-timeout-1-with-schedule_min_hrt.patch
88046e381d9d1ac6fc1b6d0cdcdf28c7d0b2ffddec680dba7c084c943ff89a296b8556fcee6e9b823b27dbfcab50b278ca81324f1bf68160a49c9f2122f95b84  0008-Replace-all-calls-to-schedule_timeout_interruptible-.patch
9273876d7d20500b84c32e72076bde63373b9bf7c22ed988434a8f664274d175d8fa68da7e153838b18e29929d9319f90572a957fbc8a2243fce36af14a86e50  0009-Replace-all-calls-to-schedule_timeout_uninterruptibl.patch
dd3d60ab435bf67b65dc9a8a68443fc2a0c1e8e9d822ea82367429d00f46dfd23119a64e2fae5d2681b31a5e7a4827e9d17d21c4a8e04f6a067f2a4a8f359a76  0010-Don-t-use-hrtimer-overlay-when-pm_freezing-since-som.patch
e93c2152b7c3225f20cb4a5986a64a302eb536cf0a8973d782498617d466b3e2d635fa8dd253d2791da7d818216c17ce31f6ef9fa42ca3fd293ae6f37cd4642d  0011-Make-hrtimer-granularity-and-minimum-hrtimeout-confi.patch
77c4e875935bff08f196099194d849eedd2f38a3c22adb792da52a3c7f3fb5060f06cfc32bdcb21fa63437ff5c6466d16bef8d3d3d2c32da240aafc7b08c7926  0012-Reinstate-default-Hz-of-100-in-combination-with-MuQS.patch
c7ffe643b54010362015300ac87cb08da2474c8f11a14771bb3a9bdf0f393b048f5f5a41234eb411490d3acb0fcaf2981440a3191db4e9b2fab60a9ee8e3aad3  0013-Make-threaded-IRQs-optionally-the-default-which-can-.patch
fd55a7dc69506ff2fecbd918c3f6ff4311eeaafe7911580a502f524adb02e7b576e1d698a565cdcb8bb5553a4f8c5042401f42548bc72e01c8e903df0a10d590  0014-Swap-sucks.patch
c0efabe09795f73ccb4c3be43e52d7d05780018c37f1a3bf487c6aa0fa9e02102ee870df1b8af1dbed63ba9995f4b525004536d762dfb9b5e17c50e07c58a983  0015-Enable-BFQ-io-scheduler-by-default.patch
dfc4b85c8697f1be4fd180e3178668067cb2c60d9f4000f480a5e6f94cb87c88858e517bc8562fa08ddd6398b4f217382e26f8326acf4ead6f0f6a2f229bec51  config-kvm.x86_64
62aa92e671cfc9265cf1690e0d64058dfa779400074cd909161d4e49f5313f58c1303ad301cecdbb64ee85d653f0bcb42fa609f25827289aad3bc94561d94390  patch-4.14.8.xz"

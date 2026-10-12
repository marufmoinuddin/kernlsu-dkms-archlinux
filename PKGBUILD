# shellcheck shell=bash
# Based on the AUR kernelsu-dkms recipe by Shadichy <shadichy@blisslabs.org>

_pkg=kernelsu
pkgname=${_pkg}-dkms
pkgver=3.3.0+77+gd8de2e37
pkgrel=4
_commit=d8de2e37a85d2c4c99bc854fb0596bb02a13eb6e
# Bundled so this package has no AUR-only dependency: `makepkg -s` resolves
# deps with pacman, which cannot see the AUR, and the AUR modloader package is
# not in any binary repo. See build_modloader().
_modloader_pkg=modloader
_modloader_url=https://github.com/shadichy/modloader
_modloader_commit=27d86fbeee20f895fda1fe1b8a864f869d37c4e8
pkgdesc="KernelSU as a DKMS module for container-based Android such as Waydroid, shipping a certificate-matched Manager"
arch=('x86_64')
url="https://github.com/supechicken/KernelSU"
license=('GPL-2.0-only' 'GPL-3.0-or-later')
depends=('dkms' 'waydroid')
makedepends=(
	'git'
	'rust'
	'cargo'
	'rustup'
	'clang'
	'llvm'
	'lld'
	'pkgconf'
	'openssl'
	'python'
	'ninja'
	'jdk21-openjdk'
	'util-linux'
	'android-sdk-cmdline-tools-latest'
)
options=('!strip' '!emptydirs')

# --- Android build environment -------------------------------------------------
# Override KSU_SDK to point at a different SDK root. Set KSU_SDK_AUTO_PROVISION=1
# to let the build install the pinned SDK components itself (accepts the SDK
# license prompts on your behalf).
_sdk="${KSU_SDK:-$HOME/Android/Sdk}"
_ndk_ver=29.0.14206865
_cmake_ver=3.22.1
_platform=android-37.0
_build_tools=37.0.0
_ksud_triple=x86_64-linux-android
_ksud_min_api=26
_android_cli=/opt/android-sdk/cmdline-tools/latest/bin/android

_manager_pkg=me.weishu.kernelsu
_manager_code=32678

# --- Signing ------------------------------------------------------------------
# Credentials live outside $srcdir and $pkgdir and are never packaged: only the
# signed APK and the public certificate end up in the package. Reuse the same
# key across rebuilds, otherwise the certificate baked into the module stops
# matching the Manager and the Manager has to be reinstalled.
#
# Established by _ksu_ensure_signing_credentials(), which runs first in build().
_ksu_ensure_signing_credentials() {
	export KSU_KEYSTORE="${KSU_KEYSTORE:-$HOME/.config/kernelsu-waydroid/manager.p12}"
	export KSU_PASSWORD_FILE="${KSU_PASSWORD_FILE:-$HOME/.config/kernelsu-waydroid/manager-password}"
	export KSU_KEY_ALIAS="${KSU_KEY_ALIAS:-waydroid-ksu}"

	local keytool=/usr/lib/jvm/java-21-openjdk/bin/keytool
	[[ -x $keytool ]] || keytool=$(command -v keytool) || {
		printf 'error: keytool not found; install jdk21-openjdk\n' >&2
		return 1
	}

	# Credential setup runs in a subshell so its umask, fd and trap do not
	# leak into the rest of the build.
	(
		umask 077

		local temp_password='' temp_key_dir='' supplied_password=''

		mkdir -p -- "$(dirname -- "$KSU_KEYSTORE")" \
			"$(dirname -- "$KSU_PASSWORD_FILE")" || exit 1

		# Refuse to follow a symlink: the target may not be what it looks
		# like, and we would otherwise overwrite it.
		[[ ! -L $KSU_KEYSTORE && ! -L $KSU_PASSWORD_FILE ]] || {
			printf 'error: signing credential paths must not be symlinks\n' >&2
			exit 1
		}

		# Serialise concurrent builds sharing this keystore.
		exec {signing_lock_fd}>"${KSU_KEYSTORE}.lock" || exit 1
		flock "$signing_lock_fd" || exit 1

		trap '
			[[ -z "$temp_password" ]] || rm -f -- "$temp_password"
			[[ -z "$temp_key_dir" ]] || rm -rf -- "$temp_key_dir"
		' EXIT

		validate() {
			"$keytool" -list -keystore "$KSU_KEYSTORE" \
				-alias "$KSU_KEY_ALIAS" -storepass:file "$1" >/dev/null 2>&1
		}

		# An empty existing file means an interrupted earlier run, not a
		# blank password. Never overwrite either one.
		[[ ! -e $KSU_KEYSTORE || -s $KSU_KEYSTORE ]] || {
			printf 'error: %s exists but is empty; refusing to replace it\n' "$KSU_KEYSTORE" >&2
			exit 1
		}
		[[ ! -e $KSU_PASSWORD_FILE || -s $KSU_PASSWORD_FILE ]] || {
			printf 'error: %s exists but is empty\n' "$KSU_PASSWORD_FILE" >&2
			exit 1
		}

		if [[ ! -e $KSU_PASSWORD_FILE ]]; then
			temp_password=$(mktemp "${KSU_PASSWORD_FILE}.tmp.XXXXXX") || exit 1

			if [[ -e $KSU_KEYSTORE ]]; then
				# An existing keystore can only be unlocked by its real
				# password. Generating a random one here would lock the
				# user out of their own key.
				if [[ -n ${KSU_SIGNING_PASSWORD:-} ]]; then
					supplied_password=$KSU_SIGNING_PASSWORD
				elif [[ ${KSU_ALLOW_PASSWORD_PROMPT:-1} == 1 ]] &&
					{ true >/dev/tty; } 2>/dev/null; then
					printf 'Enter the existing password for %s (alias %s): ' \
						"$KSU_KEYSTORE" "$KSU_KEY_ALIAS" >/dev/tty
					IFS= read -r -s supplied_password </dev/tty || {
						printf '\n' >/dev/tty
						exit 1
					}
					printf '\n' >/dev/tty
				else
					{
						printf 'error: %s has no password file and cannot be unlocked automatically.\n' "$KSU_KEYSTORE"
						printf 'Create %s containing its password, or set KSU_PASSWORD_FILE.\n' "$KSU_PASSWORD_FILE"
						printf 'For unattended builds, inject KSU_SIGNING_PASSWORD.\n'
						printf 'The existing keystore will not be replaced.\n'
					} >&2
					exit 1
				fi

				[[ -n $supplied_password ]] || {
					printf 'error: empty password supplied\n' >&2
					exit 1
				}

				printf '%s' "$supplied_password" >"$temp_password" || exit 1
				unset supplied_password

				# Persist only a password that actually opens the keystore.
				if ! validate "$temp_password"; then
					printf 'error: password rejected by %s; nothing was saved\n' "$KSU_KEYSTORE" >&2
					exit 1
				fi
			else
				# Brand new key: generate its password alongside it.
				openssl rand -base64 48 | tr -d '\n' >"$temp_password" || exit 1
				[[ -s $temp_password ]] || {
					printf 'error: failed to generate a signing password\n' >&2
					exit 1
				}
			fi

			chmod 600 "$temp_password" || exit 1
			mv -- "$temp_password" "$KSU_PASSWORD_FILE" || exit 1
			temp_password=''
		fi

		if [[ ! -e $KSU_KEYSTORE ]]; then
			# Generate into a temporary directory; publish only on success so
			# a failed keygen never leaves a half-written keystore behind.
			temp_key_dir=$(mktemp -d "$(dirname -- "$KSU_KEYSTORE")/.ksu-keygen.XXXXXX") || exit 1

			"$keytool" -genkeypair \
				-keystore "$temp_key_dir/manager.p12" \
				-storetype PKCS12 \
				-alias "$KSU_KEY_ALIAS" \
				-keyalg RSA \
				-keysize 2048 \
				-validity 10000 \
				-dname 'CN=Waydroid KernelSU Local' \
				-storepass:file "$KSU_PASSWORD_FILE" \
				-keypass:file "$KSU_PASSWORD_FILE" >/dev/null || {
					printf 'error: keytool -genkeypair failed\n' >&2
					exit 1
				}

			chmod 600 "$temp_key_dir/manager.p12" || exit 1
			mv -- "$temp_key_dir/manager.p12" "$KSU_KEYSTORE" || exit 1
		fi

		chmod 600 "$KSU_KEYSTORE" "$KSU_PASSWORD_FILE" || exit 1

		# Final gate: never build against credentials that do not work.
		if ! validate "$KSU_PASSWORD_FILE"; then
			printf 'error: %s does not open with %s (alias %s).\n' \
				"$KSU_KEYSTORE" "$KSU_PASSWORD_FILE" "$KSU_KEY_ALIAS" >&2
			printf 'Check that the password is correct and the alias exists:\n' >&2
			printf '  keytool -list -keystore %s -storepass:file %s\n' \
				"$KSU_KEYSTORE" "$KSU_PASSWORD_FILE" >&2
			exit 1
		fi

		printf '==> Signing credentials ready: %s\n' "$KSU_KEYSTORE"
	) || return 1

	# Keep an injected password out of any later build subprocess.
	unset KSU_SIGNING_PASSWORD
}

# Resolved at run time; $srcdir is not set while the PKGBUILD is sourced.
_artifacts() { printf '%s/ksu-artifacts' "$srcdir"; }

source=(
	"${_pkg}::git+${url}#commit=${_commit}"
	"${_modloader_pkg}::git+${_modloader_url}#commit=${_modloader_commit}"
	'Makefile'
	'0001-Patch-Kbuild-for-dkms.patch'
	'dkms.conf'
	'00-kernelsu.conf'
	'load-kernelsu.in'
	'apkcert'
	'kernelsu-waydroid-apply'
	'kernelsu-waydroid-load'
	'kernelsu-waydroid-apply.service'
	'kernelsu-waydroid-apply.timer'
	'kernelsu-waydroid-load.service'
	'waydroid-container-kernelsu.conf'
	'99-kernelsu-waydroid.hook'
)

sha256sums=(
	'SKIP'
	'SKIP'
	'a836794d044068ee44f0e60b4f45f4f58e810a23e79cb800b4cba1dd9004e369'
	'a45b6d9b38cc0fa06ca3984999ddd6940f4b4e2a84a4fccbf7801784903ef762'
	'bf325c92678f6e35b669f4a57b62f669b0c30b18c7d110a9df5164548ee7eff1'
	'05feaafbbac794a68c7eeea8c0a4c5616fc9f6ef7e4b7540baf3f5d43fad5fb0'
	'92065b3f1736bfc2dbd0411f4fd0c654582a3c8a71df7738e2e78536dc5a08d0'
	'1135dc25d2f54ec0eb73f63eac8916dd2969343048697cbddba785151659c359'
	'62809db670341a389c3b13229e97cc62b9a754eb316be4a5960dc5e357f5fa3e'
	'de5fb825cf5444fff5b8f975b438c68240ff6f34d30be9fc4d1d6cac7829f58e'
	'19a30682d3165c5c639a018f08624b7e6dedfe49c0bd668bdd59e635a41b18ed'
	'83c8a7e9ad39e7f55a2701279d382bf83845a8acaf08dbb84e15944e779dc4b0'
	'eb80c60f29eb6eac39be1aeb77cb9ccfc46dac0e8bac6162f7aabcab3c631923'
	'93f5a95f88351cdac45ba9ff738d089f60fb5d1f5f1742ccd152b90d311ac5b2'
	'8ef8abd9ecd5b3ac52f7f0457ca41b36d23d4fc91802bc2edf6d8af0a150a499'
)

prepare() {
	cd "$srcdir/$_pkg"
	git apply "$srcdir/0001-Patch-Kbuild-for-dkms.patch"

	# x86_64-only Manager.
	sed -i \
		's/abiFilters += listOf("arm64-v8a", "x86_64", "riscv64")/abiFilters += listOf("x86_64")/' \
		manager/app/build.gradle.kts

	# Pin CMake so Gradle does not try to provision its own.
	sed -i \
		's#\(path = file("src/main/cpp/CMakeLists.txt")\)#\1\n            version = "'"$_cmake_ver"'"#' \
		manager/app/build.gradle.kts
}

sdk_install() {
	local rc
	if yes | "$_android_cli" --sdk="$_sdk" sdk install "$1"; then
		return 0
	else
		rc=${PIPESTATUS[1]}
		[[ $rc == 0 ]] || return "$rc"
	fi
}

check_sdk() {
	local p
	[[ -x $_android_cli ]] || {
		printf 'error: %s not found (install android-sdk-cmdline-tools-latest)\n' \
			"$_android_cli" >&2
		return 1
	}
	for p in \
		"platforms/$_platform" \
		"build-tools/$_build_tools" \
		"ndk/$_ndk_ver" \
		"cmake/$_cmake_ver"; do
		if [[ ! -d "$_sdk/$p" ]]; then
			if [[ ${KSU_SDK_AUTO_PROVISION:-0} == 1 ]]; then
				printf '==> provisioning SDK component %s\n' "$p"
				sdk_install "$p" || return 1
			else
				{
					printf 'error: missing Android SDK component: %s/%s\n\n' "$_sdk" "$p"
					printf 'Provision it with:\n'
					printf '  yes | %s --sdk=%s sdk install %s\n\n' \
						"$_android_cli" "$_sdk" "$p"
					printf 'or rebuild with KSU_SDK_AUTO_PROVISION=1 to do it automatically.\n'
				} >&2
				return 1
			fi
		fi
	done
}

build_ksuinit() {
	cd "$srcdir/$_pkg/userspace/ksuinit"
	cargo build --release --target-dir .
}

build_ksud() {
	cd "$srcdir/$_pkg"
	rustup target add "$_ksud_triple"
	# setup-rust-build.sh derives the NDK paths from ANDROID_NDK_HOME.
	export ANDROID_NDK_HOME="$_sdk/ndk/$_ndk_ver"
	# shellcheck disable=SC1091
	source .github/scripts/setup-rust-build.sh "$_ksud_triple" "$_ksud_min_api"
	# Output lands in $srcdir/$_pkg/target/$_ksud_triple/release/ksud, which is
	# where repack_apk.py looks for it.
	cargo build --locked --release --target "$_ksud_triple" \
		--manifest-path userspace/ksud/Cargo.toml
}

build_manager() {
	check_sdk
	printf 'sdk.dir=%s\n' "$_sdk" >"$srcdir/$_pkg/manager/local.properties"
	cd "$srcdir/$_pkg/manager"
	./gradlew --no-daemon assembleRelease -PIS_PR_BUILD=false

	python - "$srcdir/$_pkg/manager/app/build/outputs/apk/release/output-metadata.json" <<-'EOF'
		import json, sys
		m = json.load(open(sys.argv[1]))
		assert m["applicationId"] == "me.weishu.kernelsu", m["applicationId"]
		assert m["elements"][0]["versionCode"] == 32678, m["elements"][0]["versionCode"]
	EOF
}

build_signed_manager() {
	local keystore password_file alias password repack_config out keytool

	# Resolved by _ksu_ensure_signing_credentials() at the top of build().
	: "${KSU_KEYSTORE:?signing credentials were not initialised}"
	: "${KSU_PASSWORD_FILE:?signing credentials were not initialised}"
	: "${KSU_KEY_ALIAS:?signing credentials were not initialised}"

	keytool=/usr/lib/jvm/java-21-openjdk/bin/keytool
	[[ -x $keytool ]] || keytool=$(command -v keytool)

	keystore=$(realpath -e "$KSU_KEYSTORE") || {
		printf 'error: keystore not found: %s\n' "$KSU_KEYSTORE" >&2
		return 1
	}
	password_file=$(realpath -e "$KSU_PASSWORD_FILE") || {
		printf 'error: keystore password file not found: %s\n' "$KSU_PASSWORD_FILE" >&2
		return 1
	}
	alias=$KSU_KEY_ALIAS

	# Strip a trailing newline: keytool -storepass:file tolerates one, but the
	# password handed to apksigner must match byte for byte.
	password=$(<"$password_file")
	password=${password%$'\n'}
	[[ -n $password ]] || {
		printf 'error: %s is empty\n' "$password_file" >&2
		return 1
	}

	# Already validated in _ksu_ensure_signing_credentials(); this guards
	# against the files changing underneath us mid-build.
	"$keytool" -list -keystore "$keystore" -alias "$alias" \
		-storepass:file "$password_file" >/dev/null

	mkdir -p "$(_artifacts)"

	# repack_apk.py takes the password on the command line, so hand it a
	# throwaway config in the build dir instead of leaking it into the process
	# table or the build log.
	repack_config="$(_artifacts)/repack.json"
	(
		umask 077
		python - "$repack_config" "$keystore" "$alias" "$password" <<-'EOF'
			import json, sys
			path, keystore, alias, password = sys.argv[1:]
			json.dump(
			    {
			        "signing": {
			            "keystore_path": keystore,
			            "key_alias": alias,
			            "keystore_pass": password,
			            "key_pass": password,
			        }
			    },
			    open(path, "w"),
			)
		EOF
	)

	local rc=0
	(
		cd "$srcdir/$_pkg"
		ANDROID_SDK_ROOT="$_sdk" \
		ANDROID_HOME="$_sdk" \
		ANDROID_NDK_HOME="$_sdk/ndk/$_ndk_ver" \
			python repack_apk.py repack -c "$repack_config" -b release -t release \
			-a x86_64 --strip -n manager -o "$(_artifacts)"
	) || rc=$?
	shred -u "$repack_config" 2>/dev/null || rm -f "$repack_config"
	((rc == 0)) || {
		printf 'error: APK repack/signing failed (exit %d)\n' "$rc" >&2
		return "$rc"
	}

	out="$(_artifacts)/manager.apk"
	[[ -f $out ]] || {
		printf 'error: repack produced no APK at %s\n' "$out" >&2
		return 1
	}

	# Public certificate only; this is what gets baked into dkms.conf.
	"$keytool" -exportcert -keystore "$keystore" -alias "$alias" \
		-storepass:file "$password_file" -file "$(_artifacts)/manager-cert.der"

	# Fail loudly rather than shipping a package whose module will not accept
	# its own Manager.
	"$_sdk/build-tools/$_build_tools/apksigner" verify --verbose --print-certs "$out" \
		>"$(_artifacts)/apksigner.log"
	local cert_hash
	cert_hash=$(sha256sum "$(_artifacts)/manager-cert.der" | awk '{print $1}')
	grep -Fq "$cert_hash" "$(_artifacts)/apksigner.log" || {
		printf 'error: APK signature does not match the exported certificate\n' >&2
		return 1
	}

	python - "$out" <<-'EOF'
		import sys
		from zipfile import ZipFile
		with ZipFile(sys.argv[1]) as z:
		    assert z.getinfo("lib/x86_64/libksud.so").file_size > 0
		    assert not any(
		        n.startswith(("lib/arm64-v8a/", "lib/riscv64/")) for n in z.namelist()
		    )
	EOF

	local cert_size
	cert_size=$(printf '0x%04x' "$(stat -c %s "$(_artifacts)/manager-cert.der")")
	printf 'source_commit=%s\nmanager_package=%s\nmanager_code=%s\ncertificate_size=%s\ncertificate_sha256=%s\n' \
		"$_commit" "$_manager_pkg" "$_manager_code" "$cert_size" "$cert_hash" \
		>"$(_artifacts)/manifest.txt"

	shred -u "$(_artifacts)/apksigner.log" 2>/dev/null || rm -f "$(_artifacts)/apksigner.log"
}

build_modloader() {
	# Built from source and shipped in usr/bin/modloader. The AUR package of
	# the same name is not in any binary repository, so depending on it would
	# make `makepkg -s` fail for anyone without an AUR helper.
	cd "$srcdir/$_modloader_pkg"
	cargo build --release --locked
}

build() {
	# Credentials first: fail here rather than after a 20-minute Gradle build,
	# and generate or capture the signing key before anything needs it.
	_ksu_ensure_signing_credentials || return 1

	build_ksuinit
	build_modloader
	build_ksud
	build_manager
	build_signed_manager
}

package() {
	local dest="$pkgdir/usr/src/kernelsu-${pkgver}"
	local cert cert_size cert_hash manager_args
	mkdir -p "$dest"

	cd "$srcdir"
	cp -rpRLt "$dest" "${_pkg}/kernel/."

	cd "$_pkg"

	local _major _count
	_major=${pkgver%%.*}
	_count=$(git rev-list --count HEAD 2>/dev/null)

	echo "ksu git ver: $_count" >&2

	# Bake the Manager's identity into the module so every kernel built from
	# this package accepts exactly the APK we ship.
	cert="$(_artifacts)/manager-cert.der"
	[[ -f $cert ]] || {
		printf 'error: %s missing; build_signed_manager did not run\n' "$cert" >&2
		return 1
	}
	cert_size=$(printf '0x%04x' "$(stat -c %s "$cert")")
	cert_hash=$(sha256sum "$cert" | awk '{print $1}')
	manager_args="KSU_PACKAGE_NAME=$_manager_pkg KSU_IS_PR_BUILD=0 KSU_EXPECTED_SIZE=$cert_size KSU_EXPECTED_HASH=$cert_hash"

	sed "s|@PKGVER@|${pkgver}|g;\
    s|@KSU_GIT_VERSION@|${_count}|g;\
    s|@KSU_MANAGER_ARGS@|${manager_args}|g;" \
		"$srcdir/dkms.conf" >"$dest/dkms.conf"

	install -Dm644 "$srcdir/Makefile" "$dest/Makefile"

	# Install module config
	mkdir -p "$pkgdir/etc/modprobe.d"
	install -Dm644 "$srcdir/00-kernelsu.conf" "$pkgdir/etc/modprobe.d/"

	# Install loader
	cd "$srcdir/$_pkg/userspace/ksuinit"
	install -Dm755 release/ksuinit "$pkgdir/usr/bin/kernelsu-loader"

	# Install modloader, the kernel module loader load-kernelsu pipes into
	cd "$srcdir/$_modloader_pkg"
	install -Dm755 target/release/modloader "$pkgdir/usr/bin/modloader"

	# Install load script (for modloader)
	install -Dm755 "$srcdir/load-kernelsu.in" "$pkgdir/usr/bin/load-kernelsu"

	# Manager APK, its public certificate and the build manifest
	install -Dm644 "$(_artifacts)/manager.apk" \
		"$pkgdir/usr/share/kernelsu-waydroid/manager.apk"
	install -Dm644 "$(_artifacts)/manager-cert.der" \
		"$pkgdir/usr/share/kernelsu-waydroid/manager-cert.der"
	install -Dm644 "$(_artifacts)/manifest.txt" \
		"$pkgdir/usr/share/kernelsu-waydroid/manifest.txt"

	# Runtime integration helper and load helper
	install -Dm755 "$srcdir/kernelsu-waydroid-apply" \
		"$pkgdir/usr/lib/kernelsu-waydroid/apply"
	install -Dm755 "$srcdir/kernelsu-waydroid-load" \
		"$pkgdir/usr/lib/kernelsu-waydroid/load"
	install -Dm755 "$srcdir/apkcert" \
		"$pkgdir/usr/lib/kernelsu-waydroid/apkcert"

	# systemd units
	install -Dm644 "$srcdir/kernelsu-waydroid-apply.service" \
		"$pkgdir/usr/lib/systemd/system/kernelsu-waydroid-apply.service"
	install -Dm644 "$srcdir/kernelsu-waydroid-apply.timer" \
		"$pkgdir/usr/lib/systemd/system/kernelsu-waydroid-apply.timer"
	install -Dm644 "$srcdir/kernelsu-waydroid-load.service" \
		"$pkgdir/usr/lib/systemd/system/kernelsu-waydroid-load.service"
	install -Dm644 "$srcdir/waydroid-container-kernelsu.conf" \
		"$pkgdir/usr/lib/systemd/system/waydroid-container.service.d/50-kernelsu.conf"

	# Trigger the integration once the DKMS hooks have finished
	install -Dm644 "$srcdir/99-kernelsu-waydroid.hook" \
		"$pkgdir/usr/share/libalpm/hooks/99-kernelsu-waydroid.hook"
}

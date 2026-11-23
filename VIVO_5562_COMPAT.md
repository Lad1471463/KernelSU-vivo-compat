# vivo compatibility branch based on KernelSU CI 5562

This branch is based on upstream `2a60b2893b8d0d885d1ceff2287d8ad854f0e0f4`.
The tested target is iPA2575 / MT8799, Android 16, kernel
`6.6.127-android15-8-g24559e55e8d4-abogki519894238-4k`.

## Changes

- `KSU_VIVO_COMPAT=1` opts into the `init_module` / `finit_module` filter
  from [vivo-196, commit 0760b7c5](https://github.com/morannlx/testksu/commit/0760b7c52ae883209787e5e2177934cd70f573c2).
  The two filter files are copied unchanged. The filter checks ELF `.modinfo`
  for the exact module name `vr` and returns success without loading that
  module, preserving the previous vivo fork's boot behavior. Other module
  requests continue through the original handler. No vendor partition files
  are removed or replaced. Omitting this build option leaves the upstream
  initialization path in place.
- The Manager initializes its normal and global-mount shells on demand,
  using one shared initialization lock. It does not eagerly create an
  application-owned duplicate app-list ViewModel.
- The home state starts with a loading indicator and is computed on the
  existing IO refresh path. Activation state follows the ViewModel lifetime.
  Home, superuser, module, settings, and installation routes remain present.
- The Manager's ARM64 ksud is rebuilt from the same upstream Rust sources
  with the matching custom 6.6 module embedded. No Rust implementation,
  SELinux parser, policy, IOCTL, or app-profile layout is changed.

## Evidence and limits

Prior device pstore showed `libksud.so` exit paths entering `vr`, followed
by scheduling-while-atomic reports, RCU stalls, and watchdog resets. The
official build also had successful launches. This is evidence of an
intermittent compatibility problem, not a proof of every internal cause.

The candidate passed the Android15-6.6 DDK symbol checker. Its 243 undefined
symbol names match the upstream module. The modules have empty `__versions`
tables, so this check does not establish per-symbol CRC equality. Full
unstripped module symbols are retained separately from the installed module.

On the target device, two candidate boots loaded KernelSU without `vr`.
The full Manager opened, loaded the original superuser and module lists,
and opened settings. On the second boot it was launched as soon as boot
completion was observed, at approximately 27 seconds of uptime. The existing
ZygiskNext reported all four configured ARM64 extensions loaded and zero
module issues. This is bounded device validation, not a general compatibility
guarantee for other vivo models or firmware.

Detector observations are recorded separately from compatibility checks.
An ADB `exec-out` UI capture on this device creates a root-owned PTY and can
itself contaminate a concurrent devpts scan. Receipt collection must use
`adb shell -T`, an explicit dump file, and non-PTY screenshot transport.
Keep the contaminated report as well as subsequent uncontaminated reports.

## Matching signatures and rebuilding

The Manager and module must be a matching pair. Generate and retain a private
signing key, sign the release APK using APK v2, and pass its certificate size
and SHA-256 digest through `KSU_EXPECTED_SIZE` and `KSU_EXPECTED_HASH` when
building the module. A different-signed APK cannot directly update the
upstream-signed package. Preserve app data, app grants, module configuration,
and the original boot image before replacing the pair.

For the kernel build, use the upstream Android15-6.6 DDK and add
`KSU_VIVO_COMPAT=1`, `KSU_PACKAGE_NAME=me.weishu.kernelsu`, and the matching
certificate values to the usual make arguments. Copy the resulting module
to `userspace/ksud/bin/aarch64/android15-6.6_kernelsu.ko`, retain the upstream
matching `ksuinit` and auxiliary binaries, then rebuild ARM64 Android ksud.
Package that helper as `lib/arm64-v8a/libksud.so` in the release Manager,
perform 16 KiB native-library alignment, and sign the final APK.

The test label is `KernelSU vivo #5562`. This is a locally signed derivative,
not an official-signed upstream CI artifact. Do not distribute the private
keystore, passwords, personal app data, device configuration, or raw diagnostic
reports containing personal state. A source patch and public binary hashes
are sufficient for reviewing this change.

# UKI signing key

This kernel snap ships a unified kernel image (`kernel.efi` / `kernel.img`) and
signs it before packing it. This directory, which sits next to `snap/` in the
project, is where you put the key pair to sign it with.

Nothing is generated automatically here, and nothing in this directory is ever
committed -- see `.gitignore`. If you do not put a key here, the build still
works: it generates a throwaway key, signs with that, and prints a warning. The
image is then signed by a key that exists for the duration of that one build and
that no firmware can be told to trust, which is fine for development and useless
for secure boot.

## Generating a key

Two files are expected, with exactly these names:

| file      | what it is                                        |
|-----------|---------------------------------------------------|
| `uki.key` | private key, PEM, RSA -- keep it secret           |
| `uki.pem` | matching self-signed certificate, PEM             |

```sh
openssl req -x509 -newkey rsa:2048 -sha256 -nodes -days 7300 \
  -keyout uki.key -out uki.pem \
  -subj "/CN=My Company UKI signing key/"
chmod 600 uki.key
```

`-days 7300` is 20 years. Pick a lifetime longer than the devices will live: an
expired certificate is a brick if the firmware checks validity dates. RSA-2048
with SHA-256 is what `sbsign` and the UEFI implementations in use here handle
without argument; RSA-4096 also works.

Back the private key up somewhere that is not this directory, and not this
repository. Losing it means you can no longer produce an image the devices you
have already shipped will accept.

## How the build uses it

The `signing` part takes this directory as its source and copies the pair into
`${CRAFT_STAGE}/signing/`, the `initrd` part passes it to the snapcraft initrd
plugin via `initrd-efi-image-key` and `initrd-efi-image-cert`, and the plugin
signs the UKI with it inside its build chroot. The private key is staged but
deliberately not primed, so it does not end up inside the snap.

Because the directory is a part source, dropping a key in here after a build is
enough: craft-parts sees the change, re-runs the `signing` part and re-signs the
image. Nothing extra to clean.

The certificate does: it is shipped at the root of the snap as `uefi.crt`, the
same as Canonical's own kernel snaps do. Nothing in snapd reads that file. It is
there so that whoever prepares the board firmware can enroll it, and so you can
tell after the fact which key signed a given snap:

```sh
openssl x509 -in uefi.crt -noout -subject -dates -fingerprint
```

## What still has to happen on the device

Signing the image does not make the board check the signature. On MediaTek Genio,
the earlier boot stages (BL2, BL31, OP-TEE, U-Boot) are verified by MediaTek's own
mechanism, keyed by fuses in the SoC. What loads `kernel.efi` is U-Boot, so U-Boot
is what has to verify it: it must be built with EFI secure boot support and have
this certificate enrolled in its variable store (`db`). Until that is done, the
signature is carried along and never checked.

Once the firmware does report secure boot as enabled, the kernel notices and locks
itself down (`CONFIG_LOCK_DOWN_IN_SECURE_BOOT=y`). From then on it only loads
modules signed by a key it trusts. The modules inside the snap are fine -- the
kernel build signs them with a key it generates and embeds -- but out-of-tree or
DKMS modules built later will be refused unless you sign them with a key the
kernel trusts.

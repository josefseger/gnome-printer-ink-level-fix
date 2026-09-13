# Build and install on Debian 13 (Trixie)

These are the steps used to build and test the fix against Debian package `gnome-control-center` version `1:48.4-1~deb13u1`.

## 1. Enable Debian source repositories

Create `/etc/apt/sources.list.d/debian-src.list` with:

```text
deb-src https://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb-src https://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb-src https://deb.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
```

Then refresh package metadata.

## 2. Download the exact source package

```bash
mkdir -p ~/src/gnome-control-center-inklevel
cd ~/src/gnome-control-center-inklevel
apt-get source gnome-control-center=1:48.4-1~deb13u1
cd gnome-control-center-48.4
```

Confirm the relevant source block:

```bash
grep -n -B 3 -A 8 'tonerCartridge' panels/printers/pp-printer-entry.c
```

## 3. Install build dependencies

Install the standard Debian build tools and the package build dependencies:

```bash
sudo apt install build-essential devscripts fakeroot quilt
sudo apt build-dep gnome-control-center
```

## 4. Apply the patch

Copy this repository's patch into the Debian source tree:

```text
debian/patches/gnome-control-center-48.4-hyphenated-marker-types.patch
```

and append that filename to:

```text
debian/patches/series
```

Alternatively, use Quilt to create an equivalent patch.

Verify that the source now accepts all of these values:

```text
ink
toner
inkCartridge
tonerCartridge
ink-cartridge
toner-cartridge
```

## 5. Give the local build a distinct version

For the original tested machine we used the suffix `+q8b1`. For a generic local build, a suffix such as `+inklevel1` is clearer.

Example changelog version:

```text
1:48.4-1~deb13u1+inklevel1
```

The changelog entry used for the tested build was:

```text
Printers: accept IPP marker types ink-cartridge and toner-cartridge.
```

## 6. Build

```bash
dpkg-buildpackage -us -uc -b -j"$(nproc)"
```

The two packages needed for the fix are:

```text
gnome-control-center_48.4-1~deb13u1+inklevel1_arm64.deb
gnome-control-center-data_48.4-1~deb13u1+inklevel1_all.deb
```

The Debian epoch (`1:`) appears in package metadata but not in the `.deb` filename.

## 7. Verify the built binary before installing

Extract the package into a temporary directory and inspect the binary strings:

```bash
rm -rf /tmp/gnome-control-center-inklevel-verify
mkdir -p /tmp/gnome-control-center-inklevel-verify

dpkg-deb -x ../gnome-control-center_48.4-1~deb13u1+inklevel1_arm64.deb \
  /tmp/gnome-control-center-inklevel-verify

strings /tmp/gnome-control-center-inklevel-verify/usr/bin/gnome-control-center | \
  grep -E 'inkCartridge|tonerCartridge|ink-cartridge|toner-cartridge'
```

Expected output:

```text
inkCartridge
tonerCartridge
ink-cartridge
toner-cartridge
```

## 8. Simulate installation first

Use APT's simulation mode with both locally built packages. In the tested case, the result was:

```text
Upgrading: 2, Installing: 0, Removing: 0
```

Only `gnome-control-center` and `gnome-control-center-data` were replaced.

## 9. Install the two packages

Install the matching local `gnome-control-center` and `gnome-control-center-data` packages together.

After installation, verify again that `/usr/bin/gnome-control-center` contains:

```text
inkCartridge
tonerCartridge
ink-cartridge
toner-cartridge
```

## 10. Restart GNOME Control Center

Find the current process:

```bash
pgrep -af gnome-control-center
```

Stop that process, then start the printer panel again:

```bash
gnome-control-center printers
```

Note: `pkill -x gnome-control-center` may fail because Linux process `comm` names are limited to 15 characters while `gnome-control-center` is longer.

## Result

On the tested HP Color LaserJet Pro MFP M479fdn using IPP Everywhere, GNOME Settings then displayed the same toner levels that CUPS had already been reporting:

```text
Black    60%
Cyan     10%
Magenta  30%
Yellow   60%
```

No HPLIP queue was required.

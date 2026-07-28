# Unbound .deb Builder for Ubuntu

Automatically downloads the latest stable [Unbound](https://nlnetlabs.nl/projects/unbound/about/) DNS resolver release, compiles it from source, and packages it as `.deb` files for Ubuntu.

## What this does

- **Scans daily** for new Unbound releases at [nlnetlabs.nl](https://nlnetlabs.nl/downloads/unbound/)
- **Skips pre-releases** — only builds stable versions
- **Builds in clean Docker containers** for both Ubuntu 22.04 and 26.04 (amd64)
- **Creates a GitHub Release** automatically with the `.deb` files and `SHA256SUMS` attached
- **No manual tracking file** — compares against existing GitHub Releases to avoid duplicate builds
- **Build verification** — confirms linked libraries, loaded modules, and config syntax before packaging

## Download the latest package

1. Go to the **Releases** page of this repository.
2. Download the `.deb` for your Ubuntu version.

| File | Target |
|------|--------|
| `unbound_X.Y.Z-ubuntu22.04_amd64.deb` | Ubuntu 22.04 (Jammy) |
| `unbound_X.Y.Z-ubuntu26.04_amd64.deb` | Ubuntu 26.04 |

Verify integrity after download:

```bash
sha256sum -c SHA256SUMS
```

## Installing on Ubuntu

```bash
# Download the .deb from the Releases page, then:
sudo dpkg -i unbound_1.25.2-ubuntu22.04_amd64.deb

# If you see dependency errors, fix them with:
sudo apt-get install -f

# Or install via apt (handles deps automatically):
sudo apt install ./unbound_1.25.2-ubuntu22.04_amd64.deb
```

**Post-install:** Ensure the `unbound` user exists before starting the service:

```bash
sudo useradd -r -s /bin/false unbound 2>/dev/null || true
sudo mkdir -p /var/run/unbound
sudo chown unbound:unbound /var/run/unbound
sudo systemctl enable --now unbound
```

---

## ⚠️ FIREWALLA WARNING — READ THIS

> **DO NOT install this package on Firewalla unless you really know what you're doing.**

Firewalla devices run Ubuntu under the hood, but **Unbound is deeply intertwined with the Firewalla system**. It is not just a standalone DNS resolver on those boxes — it is likely integrated into:

- **DNS filtering and blocking** (ad block, family protect, etc.)
- **VPN and tunnel routing decisions**
- **The Firewalla mobile app's live monitoring and configuration**
- **Internal DNSSEC validation and caching logic**
- **Custom network segmentation and policy enforcement**

Replacing the system Unbound with this generic upstream build **will likely break core Firewalla functionality**, and you may not notice immediately. Your network could appear to work while security features silently fail.

**If you are a Firewalla user:**
- This `.deb` is **not tested** on Firewalla hardware or software.
- There is **no uninstall/rollback path** provided here.
- Firewalla may overwrite your changes on the next system update.
- You assume **all risk** if you proceed.

This package is intended for **standard Ubuntu servers and workstations**, not embedded security appliances.

---

## Build Features

### Full module support

Unbound is compiled with an extensive feature set:

```text
--prefix=/usr
--sysconfdir=/etc
--localstatedir=/var
--with-run-dir=/var/run/unbound
--with-username=unbound
--with-libevent
--with-libnghttp2
--with-ssl
--with-libhiredis
--enable-cachedb
--enable-dnscrypt
--enable-dnstap
--enable-subnet
--enable-ipsecmod
--enable-tfo-client
--enable-tfo-server
--enable-systemd
--enable-pie
--with-pythonmodule
--with-pyunbound
--disable-rpath
```

| Flag | What it enables |
|------|-----------------|
| `--with-libhiredis` `--enable-cachedb` | Redis-backed cache module |
| `--with-libngtcp2` `--with-libnghttp3` | DNS-over-QUIC (DoQ) — **conditional**, see below |
| `--enable-dnscrypt` | DNSCrypt protocol support |
| `--enable-dnstap` | Structured query logging |
| `--enable-subnet` | EDNS Client Subnet |
| `--enable-ipsecmod` | IPsec-triggered resolution hook |
| `--enable-tfo-client/server` | TCP Fast Open |
| `--with-pythonmodule` `--with-pyunbound` | Python scripting + client library bindings |
| `--enable-pie` + hardening flags | Position-independent executable with stack protector, RELRO, and `_FORTIFY_SOURCE` |

### DNS-over-QUIC (DoQ) — conditional

`libngtcp2-dev` and `libnghttp3-dev` are recent libraries. Ubuntu 26.04's repositories include them; **Ubuntu 22.04's default repos may not**. The build detects their presence automatically:

- **If found** → DoQ support is compiled in.
- **If missing** → DoQ is skipped; the build continues successfully with all other features.

No manual intervention required — the same workflow adapts to both Ubuntu versions.

### Build verification

Before packaging, the workflow validates:

1. **`unbound -V`** — prints the version and lists all compiled-in modules
2. **`ldd unbound`** — confirms all shared libraries resolve correctly at runtime
3. **`unbound-checkconf`** — validates a minimal config file for syntax errors

If any of these fail, the build stops before a broken `.deb` is published.

### Release checksums

Every release includes a `SHA256SUMS` file covering all `.deb` packages. Verify after download:

```bash
sha256sum -c SHA256SUMS
```

---

## Manual trigger

You can force a build at any time:

1. Go to **Actions** → **Build Unbound .deb Packages**
2. Click **Run workflow**
3. (Optional) Enter a specific version like `1.25.2` to force-build that version
4. Click **Run workflow**

## How it works

| Step | Details |
|------|---------|
| **Check** | Queries the NLnet Labs download page for the latest stable version |
| **Compare** | Queries existing GitHub Releases to see if it's already been built |
| **Build** | Spins up `ubuntu:22.04` and `ubuntu:26.04` containers, installs build deps, compiles from the upstream tarball |
| **Verify** | Runs `unbound -V`, `ldd`, and `unbound-checkconf` to confirm a healthy binary |
| **Package** | Uses `dpkg-deb` to create proper `.deb` packages with a systemd service file |
| **Checksum** | Generates `sha256sum` for each package |
| **Release** | Creates a GitHub Release (e.g., `unbound-1.25.2`) and attaches both `.deb` files plus `SHA256SUMS` |

## Local build (without GitHub Actions)

If you prefer to build locally instead of using GitHub Actions:

```bash
chmod +x build-unbound.sh
./build-unbound.sh [version] [ubuntu_version]

# Examples:
./build-unbound.sh                    # defaults to 1.25.2 on 22.04
./build-unbound.sh 1.25.2 26.04      # specific version and OS
```

This runs the same build inside a clean Docker container and drops the `.deb` and `SHA256SUMS` into `./output/`.

## Build dependencies

The following packages are installed in the build container:

```text
build-essential wget ca-certificates fakeroot dpkg-dev
libssl-dev libexpat1-dev libevent-dev libnghttp2-dev
libsystemd-dev libprotobuf-c-dev protobuf-c-compiler
libhiredis-dev libfstrm-dev libsodium-dev
python3-dev swig doxygen
bison flex pkg-config
libngtcp2-dev libnghttp3-dev   # (optional, 26.04 only)
```

## Troubleshooting

### Ubuntu 26.04 build fails

The `ubuntu:26.04` Docker image may not be available yet. If the 26.04 job fails:

1. Edit `.github/workflows/build-unbound.yml`
2. Find the `matrix` section
3. Change `ubuntu_version: "26.04"` to `ubuntu_version: "24.04"` (or `"devel"`)
4. Commit the change

### "Release already exists"

If you want to rebuild an existing version, use the **Run workflow** button and enter the version in the `force_version` field.

### wget exit code 8 during download

If the build fails with a download error, the upstream tarball URL may have changed. Unbound releases are hosted at `https://nlnetlabs.nl/downloads/unbound/`, not as GitHub release assets. The workflow is configured to use the official NLnet Labs download URL.

### Missing DoQ support on 22.04

This is expected. If you need DNS-over-QUIC on 22.04, you must build `libngtcp2` and `libnghttp3` from source before compiling Unbound. The workflow intentionally skips them on 22.04 rather than failing.

## License

The workflow and packaging scripts in this repository are provided as-is. Unbound itself is licensed under the BSD-3-Clause license.

* * *

_Built with GitHub Actions. Not affiliated with NLnet Labs or Firewalla._

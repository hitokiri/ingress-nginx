# Security scans

🇪🇸 [Versión en español](README.es.md)

Scan results for the published image, produced with [Docker Scout](https://docs.docker.com/scout/).
Full context (patches, dependency bumps and tests) is in
[SECURITY-PATCH-NGINX2ROOT.md](../../SECURITY-PATCH-NGINX2ROOT.md).

## `hikohitokiri/ingress-nginx-controller:v1.15.2` (2026-11-25)

- Digest: `sha256:94aae840697d408f0932cb30e9dd9a1ed4cd6b554b0e42d2bfd7ca96eebe3936`
- Commit: `c3c540f0b` · Alpine 3.23.8 · NGINX 1.27.1 with patches 00–52 · Go 1.26.11

| Critical | High | Medium | Low | Fix available |
|:---:|:---:|:---:|:---:|:---:|
| 1 | 2 | 9 | 5 | **0** |

All remaining CVEs are in Alpine packages **with no fixed version published yet**:

| Package | Version | CVE | Used by default? |
|---|---|---|---|
| `grpc` | 1.76.0-r2 | 🔴 CVE-2026-33186 (critical) | **Likely a false positive** (see below). Also, only the OpenTelemetry module loads it |
| `protobuf` | 31.1-r1 | 🟠 CVE-2026-0994 (high) | No. Only the OpenTelemetry module loads it |
| `libxml2` | 2.13.9-r1 | 🟠 CVE-2026-86140 (high), 5 medium, 5 low | No. Only ModSecurity (`enable-modsecurity`) and njs use it |
| `patch` | 2.8-r0 | 2 medium | No. It is a command-line tool that nginx never runs |
| `busybox` | 1.37.0-r30 | 1 medium | It is the image shell; nginx does not run it to serve traffic |
| `nghttp2` | 1.69.0-r0 | 1 medium | No. NGINX has its own HTTP/2 implementation. Only `curl` and the ModSecurity and OpenTelemetry modules link it |

**About CVE-2026-33186:** according to [SUSE's CVE page](https://www.suse.com/security/cve/CVE-2026-33186/),
it is a bug in the **gRPC-Go server** (HTTP/2 `:path` validation), fixed in gRPC-Go 1.79.3. It is not in
the C++ library Alpine ships, and in our image that library only acts as the OpenTelemetry client.
The controller's Go code already uses `google.golang.org/grpc v1.79.3`.

Scout policies: non-root user ✓, base image up to date ✓, no fixable critical or high CVEs ✓.
It flags missing supply chain attestations (expected for a local build) and 33 copyleft packages.

> **Note:** Scout cannot see the NGINX built from source, because it is not an apk package. Scout
> reporting no NGINX CVEs proves nothing; NGINX fixes were verified by applying the patches and testing.

## Comparison with the SUSE image (`v1.15.2-prime7`)

Image: `registry.rancher.com/rancher/nginx-ingress-controller:v1.15.2-prime7`
(`sha256:908547c54f6a93daa0f3a6f2f72e1e88229b51b5f1db72be74d8a79fd7dad9b7`), based on SUSE Linux Enterprise Server 16.0.

| | Ours `v1.15.2` | SUSE `v1.15.2-prime7` |
|---|---|---|
| Base OS | Alpine 3.23.8 | SLES 16.0 |
| Scout result | 1 critical, 2 high, 9 medium, 5 low | 0 critical, 0 high, 2 medium |
| CVEs with a fix available | **0** | **2** (OpenTelemetry Go 1.45.0: CVE-2026-128085 and 128086) |
| `grpc` (C++) | 1.76.0 | 1.60.0 |
| `protobuf` | 31.1 | 28.3 |
| `libxml2` | 2.13.9 | 2.13.8 |

**The SUSE image is not actually cleaner; Scout has less vulnerability data for SLES 16.**
According to SUSE's CVE database, for SLES 16.0:

- **CVE-2026-86140 (libxml2):** "In progress", no fix released. Their image is vulnerable too, but Scout does not report it.
- **CVE-2026-0994 (protobuf):** SUSE has already released the fix.
- **CVE-2026-33186:** it is a gRPC-Go bug, so it does not affect either image's C++ library.

SUSE is also still on OpenTelemetry Go 1.45.0, with two CVEs we have already fixed (1.47.0).

## Files

| File | Contents |
|---|---|
| [controller-v1.15.2-docker-scout-cves.md](controller-v1.15.2-docker-scout-cves.md) | Our image: full report by package and CVE |
| [controller-v1.15.2-docker-scout-quickview.txt](controller-v1.15.2-docker-scout-quickview.txt) | Our image: `docker scout quickview` summary and policies |
| [controller-v1.15.2-docker-scout.sarif.json](controller-v1.15.2-docker-scout.sarif.json) | Our image: SARIF format, importable into GitHub Code Scanning |
| [suse-v1.15.2-prime7-docker-scout-cves.md](suse-v1.15.2-prime7-docker-scout-cves.md) | SUSE: full report |
| [suse-v1.15.2-prime7-docker-scout-quickview.txt](suse-v1.15.2-prime7-docker-scout-quickview.txt) | SUSE: quickview and policies |

## Re-scan

```bash
I=hikohitokiri/ingress-nginx-controller:v1.15.2
O=security/scans/controller-v1.15.2-docker-scout
docker scout cves --format markdown $I > $O-cves.md
docker scout cves --format sarif --output $O.sarif.json $I
docker scout quickview $I | sed 's/\x1b\[[0-9;]*m//g' > $O-quickview.txt
```

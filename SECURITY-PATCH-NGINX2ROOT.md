# NGINX2Root security patch — ingress-nginx 1.15.1 (custom build)

🇪🇸 [Versión en español](PARCHE-SEGURIDAD-NGINX2ROOT.md)

Date: 2026-11-18

## Background

- The community project `kubernetes/ingress-nginx` is retired. Its last release is
  `controller-v1.15.1`, and there will be no more releases or fixes
  ([kubernetes#143852](https://github.com/kubernetes/kubernetes/issues/143852)).
- On 2026-11-18, nginx released versions 1.30.7 and 1.31.9, which fix a batch of vulnerabilities
  known as **"NGINX2Root"**
  ([nginx.org advisories](https://nginx.org/en/security_advisories.html), [CHANGES-1.30](https://nginx.org/en/CHANGES-1.30)).
- The controller base image (`nginx:v2.2.8`) uses **NGINX 1.27.1**, which is vulnerable.
  On top of that, ingress-nginx enables **HTTP/2 by default**.
- nginx.org did not publish standalone patches for this batch. They had to be ported by hand from the
  `github.com/nginx/nginx` commits.

### CVEs in the batch (fixed in 1.30.7)

| CVE | Description | Severity (nginx) |
|---|---|---|
| CVE-2026-123007 | Memory corruption in HTTP/2 flow control (RCE, public PoC available) | major |
| CVE-2026-123633 | Use-after-free in HTTP/2 | medium |
| CVE-2026-121810 | Use-after-free in HTTP/3 | medium |
| CVE-2026-123634 | Use-after-free in the resolver | medium |
| CVE-2026-113425 | Use-after-free in the mail proxy | major |
| CVE-2026-118899 | Infinite loop reading the request body over TLS 1.3 | major |
| CVE-2026-125283 | Buffer overflow with `alias` + `try_files` | minor |
| CVE-2026-120967 | Stream: client certificate not validated against the server selected by SNI | medium |
| CVE-2026-123640 | `ssl_preread`: malformed ClientHello parsing | minor |

## SUSE / Rancher fork

- Repo: <https://github.com/rancher/ingress-nginx>. Image: `rancher/nginx-ingress-controller`.
- Last tag reviewed: `v1.15.2-prime5` (commit `eae3c331`, 2026-11-18, *"Backport nginx HTTP/2 memory
  safety fixes"*).
- Their patches 00–34 are **identical** to ours.
- They also carry patches our repo was missing:
  - **35–49**: 15 nginx CVEs published earlier in 2026: 40460, 40701, 42934, 42945, 42946, 9256, 42055,
    48142, 56434, 42533, 15176, 93479, 95101, 97629 and 97205.
  - **50–51**: of NGINX2Root, they only cover CVE-2026-123007 and CVE-2026-123633.
- SUSE also uses Go **1.26.11**, while we were on 1.26.1. Their `build.sh`, `Dockerfile`
  and `go.mod` differ from ours. Those changes were **not** copied.

## What was done

All changes are in `images/nginx/rootfs/patches/`:

1. Patches `35_` to `51_` were copied **unchanged** from `rancher/ingress-nginx` `v1.15.2-prime5`.
2. `52_nginx-1.27.1-CVE-2026-nginx-1.30.7-security.patch` was created with the rest of the 1.30.7 fixes
   that SUSE did not have. They are upstream commits `63d74d54 1e668b7e 77f90f1c 1b723277 d1640655 338f6231
   7426c29f 9e811a43 df9d8d94 e2d23d45 5deda780 4c5ace9f c58d2a22 b99b6b52`.
   - These two commits were **not** included because SUSE already covers them:
     - `b89c4100` is in patch 50.
     - `0c87dd1d` is in patch 51. Including it again duplicated the line.
   - `b99b6b52` (try_files) was adapted by hand: it is one line, and the context differs in 1.27.1.

The NGINX version is still 1.27.1. `nginx -V` will show `1.27.1`, which is expected.

3. **Go was bumped from 1.26.1 to 1.26.11**, same as SUSE and following the pattern of upstream bump #14622. It changes in:
   `GOLANG_VERSION`, `go.mod` and `images/{custom-error-pages,ext-auth-example-authsvc,fastcgi-helloserver,go-grpc-greeter-server,kube-webhook-certgen}/rootfs/go.mod`.
   `go mod tidy` did not change any dependencies.
4. Git remotes: `origin` points to `https://github.com/hitokiri/ingress-nginx` (the fork) and
   `upstream` to the original repo `kubernetes/ingress-nginx`.
5. The work was done on the `security/nginx2root` branch and merged into the fork's `main`.

## Verification

- On the `release-1.27.1` source, patches 00–52 apply in order with `patch -p1`
  (same as `build.sh`) with no failures.
- NGINX builds with the same modules ingress uses (`http_v2`, `http_v3`, `stream`,
  `ssl_preread`, `threads`, …) and `nginx -t` passes with a test config.
- The mail module fixes (113425) do not affect the image, because ingress does not build
  `--with-mail`. That module also no longer compiled because of patch 34. They were kept for completeness.

### Local build (2026-11-18, linux/amd64, rootless Docker)

| Image | Result |
|---|---|
| `local/ingress-nginx/nginx:v2.2.9-cikume` | Built OK. Alpine 3.23.8; patches 00–52 applied during the build. |
| `local/ingress-nginx/controller:v1.15.2-cikume` | Built OK. Binary built with `go1.26.11`; `nginx -V` → 1.27.1; `nginx -t` OK. |

### Tests

- **Unit tests** (`test/test.sh` inside the official `e2e-test-runner` image, with Go 1.26.11):
  **67 packages OK, 0 failures**.
  - With a plain `make test`, `pkg/flags` fails with *"port 10246 is already in use"*. This happens because
    packages run in parallel and compete for the same port; it is not a regression. It passes when run
    serially (`GOFLAGS='-buildvcs=false -p=1'`) and in isolation.
- **Functional smoke test of the patched NGINX** (`h2test.sh` script, inside the controller image):
  - GET over HTTP/2 + TLS: 200.
  - 200 KB POST over HTTP/2 + TLS 1.3 with the body buffered to disk (the CVE-2026-118899 fix path): 200.
  - 50 parallel HTTP/2 connections: OK, 0 crit/alert/signal errors in error.log.
- Note: the ingress NGINX is built **without** the `root`/`alias` directives (patch `00_drop-alias-root`),
  so CVE-2026-125283 (alias + try_files) is not exploitable in this image. The fix is included anyway.

**Not run**: e2e tests (`make kind-e2e-test`), because they need `kind` and a local cluster (not
installed) and take hours. There was also no load testing and no image scan with Trivy/Grype.

## How to build

```bash
# 1. nginx base image (with a plain docker build; the "make build" target uses multi-arch buildx)
docker build --platform linux/amd64 -t <registry>/nginx:v2.2.9-cikume images/nginx/rootfs

# 2. Controller on top of that base
make build image ARCH=amd64 REGISTRY=<registry> TAG=v1.15.2-cikume \
  BASE_IMAGE=<registry>/nginx:v2.2.9-cikume

# 3. Unit tests
ARCH=amd64 build/run-in-docker.sh PKG=k8s.io/ingress-nginx ARCH=amd64 COMMIT_SHA=x REPO_INFO=x TAG=x \
  "GOFLAGS='-buildvcs=false -p=1'" test/test.sh
```

## Local cluster test (next step)

The `local/...` images only exist in the local Docker. They have to be loaded into the cluster:

```bash
kind create cluster --name ingress-test
kind load docker-image local/ingress-nginx/controller:v1.15.2-cikume --name ingress-test

helm install ingress-nginx charts/ingress-nginx -n ingress-nginx --create-namespace \
  --set controller.image.registry=local \
  --set controller.image.image=ingress-nginx/controller \
  --set controller.image.tag=v1.15.2-cikume \
  --set controller.image.digest="" \
  --set controller.image.pullPolicy=Never

# Full e2e suite (takes hours)
make kind-e2e-test
```

What to validate:
- The pod starts and `nginx -V` inside the pod shows 1.27.1.
- HTTP/2 and HTTP/1.1 traffic reaches a test Ingress.
- `use-http2: "true"` (the default) does not produce errors in the logs.

## Mitigation while rolling out

Disable HTTP/2 in the controller ConfigMap. This closes CVE-2026-123007 and CVE-2026-123633:

```yaml
data:
  use-http2: "false"
```

## Update 2026-11-25: SUSE review and OpenTelemetry bump

### SUSE fork review (`v1.15.2-prime6` and `prime7`, 2026-11-19)

- **#248**: SUSE added patches 52–57, one per CVE for the rest of NGINX2Root (123634, 121810,
  118899, 125283, 120967, 123640). We cover them in our combined `52_` patch.
  - Both patch sets were applied on nginx 1.27.1 and the resulting trees compared. The code is
    **identical**, except `src/mail/*`. We include CVE-2026-113425 (mail) and SUSE does not;
    ingress does not build mail, so there is no practical difference. **Nothing was copied.**
- **#247**: adds the OpenSSL FIPS provider to their SLE image (`packages.txt`). **Not applicable**:
  our image is Alpine.
- **#249**: CI workflow to monitor nginx CVEs. Not copied.

### OpenTelemetry Go bump (CVE-2026-128085)

- `go.opentelemetry.io/otel` and `otel/trace` were bumped from v1.42.0 to **v1.47.0**, the fixed version
  per [GHSA-q7gw-24fr-5p69](https://github.com/open-telemetry/opentelemetry-go/security/advisories/GHSA-q7gw-24fr-5p69).
  It is the same change as SUSE's PR #250, which was still open at that date.
- Because of Go's version resolution, testify (1.12.1), logr (1.4.4) and yaml/v3 (3.0.5) are also
  bumped: they are the minimum versions otel 1.47 requires. Only `go.mod` and `go.sum` change.
- `go mod why` shows otel only comes in through `test/e2e` (via grpc), not through the controller binary.
  The bump is preventive and avoids scanner alerts.
- Verification: `go build ./...` OK with Go 1.26.11. Unit tests (`test/test.sh`, serial):
  **67 packages OK, 0 failures**.
- Commit `35416dd19`, from the `security/otel-v1.47.0` branch, merged into the fork's `main` by fast-forward.

### Docker Scout scan and `golang.org/x/*` bump

- The controller was rebuilt with the otel bump, reusing the unchanged `nginx:v2.2.9-cikume` base.
  `docker scout` found 2 critical and 9 high CVEs with fixes, all in Go libraries:
  `golang.org/x/net` 0.52.0 (CVE-2026-39821, 78663, 97032, 78669, 78661, 78660, 78659, 46600, 33814)
  and `golang.org/x/text` 0.35.0 (CVE-2026-56852, 56851).
- `x/net` was bumped to **v0.62.0**. Its minimum requirements also bump `x/text` 0.43.0, `x/crypto`
  0.58.0, `x/sys` 0.49.0, `x/term` 0.47.0, `x/sync` 0.24.0, `x/mod` 0.41.0 and `x/tools` 0.51.0.
- Verification: `go build ./...` OK; unit tests **67 packages OK, 0 failures**; image rebuilt,
  `nginx -t` OK. Scout no longer reports **any critical or high CVE with a fix**.
- **Still unfixed** (Alpine 3.23 packages, no fixed version published):
  `grpc` 1.76.0-r2 (CVE-2026-33186, critical), `protobuf` 31.1-r1 (CVE-2026-0994, high) and
  `libxml2` 2.13.9-r1 (CVE-2026-86140, high). Re-check them when Alpine publishes fixes.
- Note: Scout cannot see the nginx built from source (it is not an apk package). Scout reporting no
  nginx CVEs proves nothing; NGINX2Root coverage was verified with the patches and the tests.
- The three unfixed packages **are not used with the default configuration** (verified with `ldd`):
  - `grpc` and `protobuf` are only linked by `otel_ngx_module.so`, which is loaded only with
    `enable-opentelemetry: "true"` or the annotation of the same name (`nginx.tmpl:32-34`). The default
    is `false`.
  - `libxml2` is only used by ModSecurity (loaded only with `enable-modsecurity`) and njs (the template
    never loads it).
  - Neither the `nginx` binary nor the controller binary links any of the three.
- Alpine alternatives reviewed: 3.24 only fixes grpc, and adds a high CVE in `musl`.
  Edge (3.25 alpha) fixes all three, but is not fit for production. We stay on 3.23 for now.
- **CVE-2026-33186 is likely a false positive.** According to
  [SUSE's CVE page](https://www.suse.com/security/cve/CVE-2026-33186/), it is a bug in the **gRPC-Go
  server** (HTTP/2 `:path` validation), fixed in gRPC-Go 1.79.3. It is not in Alpine's C++ library,
  which here only acts as the OpenTelemetry client. The controller already uses
  `google.golang.org/grpc v1.79.3`.

### Comparison with the SUSE image (`v1.15.2-prime7`)

- Image `registry.rancher.com/rancher/nginx-ingress-controller:v1.15.2-prime7`, based on SLES 16.0.
  Scout reports 0 critical, 0 high and 2 medium. The 2 medium are OpenTelemetry Go 1.45.0
  (CVE-2026-128085 and 128086): SUSE has not merged its PR #250, and we are already on 1.47.0.
- **It is not actually cleaner.** It ships older versions (grpc 1.60.0, protobuf 28.3, libxml2
  2.13.8), but Scout has less data for SLES 16. According to SUSE's CVE database for SLES 16.0:
  - CVE-2026-86140 (libxml2) is "In progress": their image is vulnerable too, but Scout does not report it.
  - CVE-2026-0994 (protobuf) already has a fix released by SUSE.
- Scout reports for both images and a comparison table: [security/scans/](security/scans/README.md).

### Published image

- **`hikohitokiri/ingress-nginx-controller:v1.15.2`** on Docker Hub (public repo), digest
  `sha256:94aae840697d408f0932cb30e9dd9a1ed4cd6b554b0e42d2bfd7ca96eebe3936`.
- Built from `main` at `c3c540f0b` with `TAG=v1.15.2`; `--version` shows `Release: v1.15.2`.
- The `nginx:v2.2.9-cikume` base image is still local only; it is only needed to rebuild.
- Docker Scout reports for this image (Markdown, SARIF and quickview): [security/scans/](security/scans/README.md).

### Local cluster test (2026-11-25, k3d `k3d-mi-laboratorio`, Kubernetes v1.27.4)

- Installed with the repo chart and the Docker Hub image, in the `ingress-nginx-test` namespace.
  A dedicated `nginx-test` IngressClass and a `ClusterIP` Service with port-forward were used, so it
  could coexist with the cluster's Traefik without touching it.
- Backend: `agnhost netexec`, with an Ingress and TLS (self-signed certificate for `test.local`).

| Test | Result |
|---|---|
| Controller pod | `Running`, 0 restarts |
| `nginx -V` in the pod | 1.27.1 |
| HTTP/1.1 on port 80 | 308, redirects to HTTPS |
| HTTPS with HTTP/1.1 and HTTP/2 | 200 / 200 |
| TLS version | TLSv1.3 (`TLS_AES_256_GCM_SHA384`) |
| 200 KB POST over HTTP/2 + TLS 1.3 (CVE-2026-118899 case) | 200 |
| 50 parallel HTTP/2 requests | 50 × 200 |
| `crit`, `alert`, `emerg` or `signal` errors in logs | 0 |

- The controller logs only had normal warnings: startup ones (backend without endpoints yet and the
  TLS secret not yet synced) and that the one-day test certificate expires soon.
  There were no errors caused by the Kubernetes version, even though v1.15 is only officially supported
  from 1.31.
- **Not covered:** the e2e suite, real load testing, or OpenTelemetry/ModSecurity enabled.

## Documentation

- **Official (frozen):** <https://kubernetes.github.io/ingress-nginx/>. Still online, but the
  `kubernetes/ingress-nginx` repo is archived and the last deploy is from 2026-03-19. It is still the
  complete reference for annotations and the ConfigMap.
- **Local copy:** `docs/` in this repo (MkDocs, with `requirements.txt`), in case the site goes away.
- **SUSE:** `rancher/ingress-nginx` does not publish its own docs; it has no GitHub Pages. The closest
  is the RKE2 documentation: <https://docs.rke2.io/networking/networking_services>. It explains how RKE2
  packages ingress-nginx; in RKE2 v1.36 it is deprecated and in v1.37 it is removed for new clusters.
  Extended support is only available with Rancher Prime.

## Pending / recommendations

- ~~Test on a local cluster~~: done on 2026-11-25 on k3d. ~~Scan the image~~: done with Docker Scout.
  The e2e suite with kind still has to be run before production.
- ~~Publish the image~~: the controller is on Docker Hub (`hikohitokiri/ingress-nginx-controller:v1.15.2`).
- Re-check `grpc`, `protobuf` and `libxml2` when Alpine 3.23 or 3.24 publishes fixes, and confirm on the
  clusters that OpenTelemetry and ModSecurity are not enabled.
- Review CVE-2026-116472 (`mirror-target` annotation, low severity). Mitigation: use
  `annotations-risk-level` Medium/Low or block the annotation.
- Long-term plan: migrate to a maintained controller (Gateway API, Traefik, F5 NGINX Ingress, etc.).

## Sources

- https://nginx.org/en/security_advisories.html
- https://nginx.org/en/CHANGES-1.30
- https://github.com/rancher/ingress-nginx (tags `v1.15.2-prime5` to `prime7`, PR #250)
- https://github.com/open-telemetry/opentelemetry-go/security/advisories/GHSA-q7gw-24fr-5p69
- https://www.suse.com/security/cve/CVE-2026-33186/ · https://www.suse.com/security/cve/CVE-2026-0994/ · https://www.suse.com/security/cve/CVE-2026-86140/
- https://github.com/kubernetes/kubernetes/issues/143852
- https://www.upwind.io/blog/nginx-cve-2026-123007-and-cve-2026-123633-four-web-facing-rces-one-patch-and-where-kubernetes-is-exposed
- https://blog.qualys.com/product-tech/patch-management/2026/11/18/nginx2root-five-nginx-flaws-led-by-cve-2026-123007-how-to-find-mitigate-and-patch-now
- https://discuss.kubernetes.io/t/security-advisory-cve-2026-116472-ingress-nginx-mirror-target-self-dos/35554

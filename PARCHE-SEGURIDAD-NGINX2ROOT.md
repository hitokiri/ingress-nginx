# Parche de seguridad NGINX2Root — ingress-nginx 1.15.1 (build propio)

Fecha: 2026-11-18

## Contexto

- El proyecto comunitario `kubernetes/ingress-nginx` está retirado. Su última versión es
  `controller-v1.15.1` y no habrá más releases ni fixes
  ([kubernetes#143852](https://github.com/kubernetes/kubernetes/issues/143852)).
- El 2026-11-18, nginx publicó las versiones 1.30.7 y 1.31.9, que corrigen un lote de vulnerabilidades
  conocido como **"NGINX2Root"**
  ([avisos nginx.org](https://nginx.org/en/security_advisories.html), [CHANGES-1.30](https://nginx.org/en/CHANGES-1.30)).
- La imagen base del controlador (`nginx:v2.2.8`) usa **NGINX 1.27.1**, que es vulnerable.
  Además, ingress-nginx activa **HTTP/2 por defecto**.
- nginx.org no publicó parches sueltos para este lote. Hay que portarlos a mano desde los commits de
  `github.com/nginx/nginx`.

### CVE del lote (corregidas en 1.30.7)

| CVE | Descripción | Severidad (nginx) |
|---|---|---|
| CVE-2026-123007 | Corrupción de memoria en el control de flujo HTTP/2 (RCE, hay PoC pública) | major |
| CVE-2026-123633 | Use-after-free en HTTP/2 | medium |
| CVE-2026-121810 | Use-after-free en HTTP/3 | medium |
| CVE-2026-123634 | Use-after-free en el resolver | medium |
| CVE-2026-113425 | Use-after-free en el proxy de mail | major |
| CVE-2026-118899 | Bucle infinito al leer el request body por TLS 1.3 | major |
| CVE-2026-125283 | Buffer overflow con `alias` + `try_files` | minor |
| CVE-2026-120967 | Stream: el certificado de cliente no se valida contra el servidor elegido por SNI | medium |
| CVE-2026-123640 | `ssl_preread`: parseo de ClientHello malformado | minor |

## Fork de SUSE / Rancher

- Repo: <https://github.com/rancher/ingress-nginx>. Imagen: `rancher/nginx-ingress-controller`.
- Último tag revisado: `v1.15.2-prime5` (commit `eae3c331`, 2026-11-18, *"Backport nginx HTTP/2 memory
  safety fixes"*).
- Sus parches 00–34 son **idénticos** a los nuestros.
- Además trae parches que a nuestro repo le faltaban:
  - **35–49**: 15 CVE de nginx publicadas antes en 2026: 40460, 40701, 42934, 42945, 42946, 9256, 42055,
    48142, 56434, 42533, 15176, 93479, 95101, 97629 y 97205.
  - **50–51**: de NGINX2Root solo cubre CVE-2026-123007 y CVE-2026-123633.
- SUSE también usa Go **1.26.11**, mientras que nosotros usamos 1.26.1. Sus `build.sh`, `Dockerfile`
  y `go.mod` difieren de los nuestros. Esos cambios **no** se copiaron.

## Qué se hizo

Todo el cambio está en `images/nginx/rootfs/patches/`:

1. Se copiaron **sin cambios** los parches `35_` a `51_` de `rancher/ingress-nginx` `v1.15.2-prime5`.
2. Se creó `52_nginx-1.27.1-CVE-2026-nginx-1.30.7-security.patch` con el resto de los fixes de 1.30.7
   que SUSE no tiene. Son los commits upstream `63d74d54 1e668b7e 77f90f1c 1b723277 d1640655 338f6231
   7426c29f 9e811a43 df9d8d94 e2d23d45 5deda780 4c5ace9f c58d2a22 b99b6b52`.
   - Estos dos commits **no** se incluyeron porque SUSE ya los cubre:
     - `b89c4100` está en el parche 50.
     - `0c87dd1d` está en el parche 51. Incluirlo de nuevo duplicaba la línea.
   - `b99b6b52` (try_files) se adaptó a mano: es una línea, y el contexto difiere en 1.27.1.

La versión de NGINX sigue siendo 1.27.1. `nginx -V` mostrará `1.27.1` y es lo esperado.

3. Se subió **Go de 1.26.1 a 1.26.11**, igual que SUSE y siguiendo el patrón del bump upstream #14622. Cambia en:
   `GOLANG_VERSION`, `go.mod` y `images/{custom-error-pages,ext-auth-example-authsvc,fastcgi-helloserver,go-grpc-greeter-server,kube-webhook-certgen}/rootfs/go.mod`.
   `go mod tidy` no cambió dependencias.
4. Remotes de git: `origin` apunta a `https://github.com/hitokiri/ingress-nginx` (el fork) y
   `upstream` al repo original `kubernetes/ingress-nginx`.

## Verificación realizada

- Sobre el código fuente de `release-1.27.1`, los parches 00–52 se aplican en orden con `patch -p1`
  (igual que `build.sh`) sin fallos.
- NGINX compila con los mismos módulos que usa ingress (`http_v2`, `http_v3`, `stream`,
  `ssl_preread`, `threads`, …) y `nginx -t` pasa con una config de prueba.
- Los fixes del módulo mail (113425) no afectan a la imagen, porque ingress no compila
  `--with-mail`. Ese módulo además ya no compilaba por el parche 34. Se dejaron por completitud.

### Build local (2026-11-18, linux/amd64, Docker rootless)

| Imagen | Resultado |
|---|---|
| `local/ingress-nginx/nginx:v2.2.9-cikume` | Construida OK. Alpine 3.23.8; los parches 00–52 se aplicaron en el build. |
| `local/ingress-nginx/controller:v1.15.2-cikume` | Construida OK. Binario con `go1.26.11`; `nginx -V` → 1.27.1; `nginx -t` OK. |

### Tests

- **Unit tests** (`test/test.sh` dentro de la imagen oficial `e2e-test-runner`, con Go 1.26.11):
  **67 paquetes OK, 0 fallos**.
  - Con `make test` normal, `pkg/flags` falla con *"port 10246 is already in use"*. Pasa porque los
    paquetes corren en paralelo y compiten por el mismo puerto; no es una regresión. En serie
    (`GOFLAGS='-buildvcs=false -p=1'`) y en aislamiento pasa.
- **Smoke test funcional del NGINX parcheado** (script `h2test.sh`, dentro de la imagen del controlador):
  - GET por HTTP/2 + TLS: 200.
  - POST de 200 KB por HTTP/2 + TLS 1.3 con body bufferizado a disco (camino del fix CVE-2026-118899): 200.
  - 50 conexiones HTTP/2 en paralelo: OK, 0 errores crit/alert/signal en el error.log.
- Nota: el NGINX de ingress se compila **sin** las directivas `root`/`alias` (parche `00_drop-alias-root`),
  así que CVE-2026-125283 (alias + try_files) no es explotable en esta imagen. El fix se incluye igual.

**No ejecutado**: tests e2e (`make kind-e2e-test`), porque requieren `kind` y un cluster local (que no
está instalado) y tardan horas. Tampoco hubo pruebas de carga ni escaneo de la imagen con Trivy/Grype.

## Cómo construir

```bash
# 1. Imagen base nginx (con docker build directo; el target "make build" usa buildx multi-arch)
docker build --platform linux/amd64 -t <registry>/nginx:v2.2.9-cikume images/nginx/rootfs

# 2. Controlador sobre esa base
make build image ARCH=amd64 REGISTRY=<registry> TAG=v1.15.2-cikume \
  BASE_IMAGE=<registry>/nginx:v2.2.9-cikume

# 3. Unit tests
ARCH=amd64 build/run-in-docker.sh PKG=k8s.io/ingress-nginx ARCH=amd64 COMMIT_SHA=x REPO_INFO=x TAG=x \
  "GOFLAGS='-buildvcs=false -p=1'" test/test.sh
```

## Mitigación mientras se despliega

Desactivar HTTP/2 en el ConfigMap del controlador. Esto cierra CVE-2026-123007 y CVE-2026-123633:

```yaml
data:
  use-http2: "false"
```

## Pendiente / recomendaciones

- Correr los e2e con kind antes de producción y escanear la imagen con Trivy.
- Revisar la CVE-2026-116472 (anotación `mirror-target`, severidad baja). Mitigación: usar
  `annotations-risk-level` Medium/Low o bloquear la anotación.
- Plan de fondo: migrar a un controlador mantenido (Gateway API, Traefik, F5 NGINX Ingress, etc.).

## Fuentes

- https://nginx.org/en/security_advisories.html
- https://nginx.org/en/CHANGES-1.30
- https://github.com/rancher/ingress-nginx (tag `v1.15.2-prime5`)
- https://github.com/kubernetes/kubernetes/issues/143852
- https://www.upwind.io/blog/nginx-cve-2026-123007-and-cve-2026-123633-four-web-facing-rces-one-patch-and-where-kubernetes-is-exposed
- https://blog.qualys.com/product-tech/patch-management/2026/11/18/nginx2root-five-nginx-flaws-led-by-cve-2026-123007-how-to-find-mitigate-and-patch-now
- https://discuss.kubernetes.io/t/security-advisory-cve-2026-116472-ingress-nginx-mirror-target-self-dos/35554

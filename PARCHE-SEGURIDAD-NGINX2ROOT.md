# Parche de seguridad NGINX2Root — ingress-nginx 1.15.1 (build propio)

🇬🇧 [English version](SECURITY-PATCH-NGINX2ROOT.md)

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
5. El trabajo se hizo en la rama `security/nginx2root` y se mergeó a `main` del fork.

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

## Prueba en cluster local (siguiente paso)

Las imágenes `local/...` solo existen en el Docker local. Hay que cargarlas al cluster:

```bash
kind create cluster --name ingress-test
kind load docker-image local/ingress-nginx/controller:v1.15.2-cikume --name ingress-test

helm install ingress-nginx charts/ingress-nginx -n ingress-nginx --create-namespace \
  --set controller.image.registry=local \
  --set controller.image.image=ingress-nginx/controller \
  --set controller.image.tag=v1.15.2-cikume \
  --set controller.image.digest="" \
  --set controller.image.pullPolicy=Never

# Suite e2e completa (tarda horas)
make kind-e2e-test
```

Qué validar:
- El pod arranca y `nginx -V` dentro del pod muestra 1.27.1.
- Que haya tráfico HTTP/2 y HTTP/1.1 a un Ingress de prueba.
- Que `use-http2: "true"` (el default) no genere errores en los logs.

## Mitigación mientras se despliega

Desactivar HTTP/2 en el ConfigMap del controlador. Esto cierra CVE-2026-123007 y CVE-2026-123633:

```yaml
data:
  use-http2: "false"
```

## Actualización 2026-11-25: revisión de SUSE y bump de OpenTelemetry

### Revisión del fork de SUSE (`v1.15.2-prime6` y `prime7`, 2026-11-19)

- **#248**: SUSE añadió los parches 52–57, uno por CVE del resto de NGINX2Root (123634, 121810,
  118899, 125283, 120967, 123640). Nosotros los cubrimos en nuestro parche `52_` combinado.
  - Se aplicaron los dos juegos de parches sobre nginx 1.27.1 y se compararon los árboles. El código
    queda **idéntico**, salvo `src/mail/*`. Nosotros incluimos CVE-2026-113425 (mail) y SUSE no;
    ingress no compila mail, así que no hay diferencia práctica. **No se copió nada.**
- **#247**: añade el proveedor FIPS de OpenSSL a su imagen SLE (`packages.txt`). **No aplica**:
  nuestra imagen es Alpine.
- **#249**: workflow de CI para vigilar CVE de nginx. No se copió.

### Bump de OpenTelemetry Go (CVE-2026-128085)

- `go.opentelemetry.io/otel` y `otel/trace` se subieron de v1.42.0 a **v1.47.0**, la versión corregida
  según [GHSA-q7gw-24fr-5p69](https://github.com/open-telemetry/opentelemetry-go/security/advisories/GHSA-q7gw-24fr-5p69).
  Es el mismo cambio que el PR #250 de SUSE, que seguía abierto en esa fecha.
- Por la resolución de versiones de Go también suben testify (1.12.1), logr (1.4.4) y yaml/v3
  (3.0.5): son las versiones mínimas que exige otel 1.47. Solo cambian `go.mod` y `go.sum`.
- `go mod why` muestra que otel solo entra por `test/e2e` (vía grpc), no por el binario del controlador.
  El bump es preventivo y evita alertas de los escáneres.
- Verificación: `go build ./...` OK con Go 1.26.11. Unit tests (`test/test.sh`, en serie):
  **67 paquetes OK, 0 fallos**.
- Commit `35416dd19`, de la rama `security/otel-v1.47.0`, integrado en `main` del fork por fast-forward.

### Escaneo con Docker Scout y bump de `golang.org/x/*`

- Se reconstruyó el controlador con el bump de otel, reutilizando la base `nginx:v2.2.9-cikume` sin cambios.
  `docker scout` encontró 2 críticas y 9 altas con fix, todas en librerías Go:
  `golang.org/x/net` 0.52.0 (CVE-2026-39821, 78663, 97032, 78669, 78661, 78660, 78659, 46600, 33814)
  y `golang.org/x/text` 0.35.0 (CVE-2026-56852, 56851).
- Se subió `x/net` a **v0.62.0**. Por sus requisitos mínimos también suben `x/text` 0.43.0, `x/crypto`
  0.58.0, `x/sys` 0.49.0, `x/term` 0.47.0, `x/sync` 0.24.0, `x/mod` 0.41.0 y `x/tools` 0.51.0.
- Verificación: `go build ./...` OK; unit tests **67 paquetes OK, 0 fallos**; imagen reconstruida,
  `nginx -t` OK. Scout ya no reporta **ninguna crítica ni alta con fix**.
- **Quedan sin fix** (paquetes de Alpine 3.23, sin versión corregida publicada):
  `grpc` 1.76.0-r2 (CVE-2026-33186, crítica), `protobuf` 31.1-r1 (CVE-2026-0994, alta) y
  `libxml2` 2.13.9-r1 (CVE-2026-86140, alta). Hay que revisarlos cuando Alpine publique fixes.
- Ojo: Scout no ve el nginx compilado desde código (no es un paquete apk). Que no reporte CVE de
  nginx no prueba nada; la cobertura de NGINX2Root se verificó con los parches y los tests.
- Los tres paquetes sin fix **no se usan con la configuración por defecto** (verificado con `ldd`):
  - `grpc` y `protobuf` solo los enlaza `otel_ngx_module.so`, que se carga únicamente con
    `enable-opentelemetry: "true"` o con la anotación del mismo nombre (`nginx.tmpl:32-34`). Por defecto
    está en `false`.
  - `libxml2` solo lo usan ModSecurity (se carga solo con `enable-modsecurity`) y njs (el template no
    lo carga nunca).
  - Ni el binario de `nginx` ni el del controlador enlazan ninguna de las tres.
- Alternativas de Alpine revisadas: en 3.24 solo se corrige grpc, y además aparece una alta en `musl`.
  Edge (3.25 alpha) corrige las tres, pero no sirve para producción. Por ahora se mantiene 3.23.
- **CVE-2026-33186 es probablemente un falso positivo.** Según la
  [ficha de SUSE](https://www.suse.com/security/cve/CVE-2026-33186/), es un fallo del **servidor de
  gRPC-Go** (validación de `:path` en HTTP/2), corregido en gRPC-Go 1.79.3. No es de la librería C++ de
  Alpine, que aquí solo actúa como cliente de OpenTelemetry. El controlador ya usa
  `google.golang.org/grpc v1.79.3`.

### Comparación con la imagen de SUSE (`v1.15.2-prime7`)

- Imagen `registry.rancher.com/rancher/nginx-ingress-controller:v1.15.2-prime7`, basada en SLES 16.0.
  Scout le da 0 críticas, 0 altas y 2 medias. Las 2 medias son OpenTelemetry Go 1.45.0
  (CVE-2026-128085 y 128086): SUSE no ha mergeado su PR #250, y nosotros ya estamos en 1.47.0.
- **No está realmente más limpia.** Trae versiones más viejas (grpc 1.60.0, protobuf 28.3, libxml2
  2.13.8), pero Scout tiene menos datos de SLES 16. Según la base de CVE de SUSE para SLES 16.0:
  - CVE-2026-86140 (libxml2) está "In progress": su imagen también es vulnerable, pero Scout no la reporta.
  - CVE-2026-0994 (protobuf) ya tiene fix publicado por SUSE.
- Reportes de Scout de ambas imágenes y tabla comparativa: [security/scans/](security/scans/README.es.md).

### Imagen publicada

- **`hikohitokiri/ingress-nginx-controller:v1.15.2`** en Docker Hub (repo público), con digest
  `sha256:94aae840697d408f0932cb30e9dd9a1ed4cd6b554b0e42d2bfd7ca96eebe3936`.
- Construida desde `main` en `c3c540f0b` con `TAG=v1.15.2`; `--version` muestra `Release: v1.15.2`.
- La imagen base `nginx:v2.2.9-cikume` sigue solo en local; solo hace falta para volver a construir.
- Reportes de Docker Scout de esta imagen (Markdown, SARIF y quickview): [security/scans/](security/scans/README.es.md).

### Prueba en cluster local (2026-11-25, k3d `k3d-mi-laboratorio`, Kubernetes v1.27.4)

- Instalación con el chart del repo y la imagen de Docker Hub, en el namespace `ingress-nginx-test`.
  Se usó la IngressClass propia `nginx-test` y Service `ClusterIP` con port-forward, para convivir
  con el Traefik del cluster sin tocarlo.
- Backend `agnhost netexec`, con Ingress y TLS (certificado autofirmado de `test.local`).

| Prueba | Resultado |
|---|---|
| Pod del controlador | `Running`, 0 reinicios |
| `nginx -V` en el pod | 1.27.1 |
| HTTP/1.1 por el puerto 80 | 308, redirige a HTTPS |
| HTTPS con HTTP/1.1 y con HTTP/2 | 200 / 200 |
| Versión de TLS | TLSv1.3 (`TLS_AES_256_GCM_SHA384`) |
| POST de 200 KB por HTTP/2 + TLS 1.3 (caso de CVE-2026-118899) | 200 |
| 50 peticiones HTTP/2 en paralelo | 50 × 200 |
| Errores `crit`, `alert`, `emerg` o `signal` en logs | 0 |

- En los logs del controlador solo hubo avisos normales: los del arranque (el backend aún sin endpoints
  y el secret TLS sin sincronizar) y los de que el certificado de prueba, de un día, vence pronto.
  No hubo errores por la versión de Kubernetes, aunque la v1.15 solo está soportada oficialmente
  desde la 1.31.
- **No cubierto:** la suite e2e, pruebas de carga reales, ni OpenTelemetry o ModSecurity activados.

## Documentación

- **Oficial (congelada):** <https://kubernetes.github.io/ingress-nginx/>. Sigue en línea, pero el repo
  `kubernetes/ingress-nginx` está archivado y el último deploy es del 2026-03-19. Sigue siendo la
  referencia completa de anotaciones y ConfigMap.
- **Copia local:** `docs/` de este repo (MkDocs, con `requirements.txt`), por si el sitio desaparece.
- **SUSE:** `rancher/ingress-nginx` no publica documentación propia; no tiene GitHub Pages. Lo más
  cercano es la doc de RKE2: <https://docs.rke2.io/networking/networking_services>. Explica cómo RKE2
  empaqueta ingress-nginx; en RKE2 v1.36 queda obsoleto y en v1.37 se elimina para clusters nuevos.
  El soporte extendido es solo con Rancher Prime.

## Pendiente / recomendaciones

- ~~Probar en un cluster local~~: hecho el 2026-11-25 en k3d. ~~Escanear la imagen~~: hecho con Docker Scout.
  Falta correr los e2e con kind antes de producción.
- ~~Publicar la imagen~~: el controlador está en Docker Hub (`hikohitokiri/ingress-nginx-controller:v1.15.2`).
- Revisar `grpc`, `protobuf` y `libxml2` cuando Alpine 3.23 o 3.24 publique fixes, y en los clusters
  confirmar que OpenTelemetry y ModSecurity no estén activados.
- Revisar la CVE-2026-116472 (anotación `mirror-target`, severidad baja). Mitigación: usar
  `annotations-risk-level` Medium/Low o bloquear la anotación.
- Plan de fondo: migrar a un controlador mantenido (Gateway API, Traefik, F5 NGINX Ingress, etc.).

## Fuentes

- https://nginx.org/en/security_advisories.html
- https://nginx.org/en/CHANGES-1.30
- https://github.com/rancher/ingress-nginx (tags `v1.15.2-prime5` a `prime7`, PR #250)
- https://github.com/open-telemetry/opentelemetry-go/security/advisories/GHSA-q7gw-24fr-5p69
- https://www.suse.com/security/cve/CVE-2026-33186/ · https://www.suse.com/security/cve/CVE-2026-0994/ · https://www.suse.com/security/cve/CVE-2026-86140/
- https://github.com/kubernetes/kubernetes/issues/143852
- https://www.upwind.io/blog/nginx-cve-2026-123007-and-cve-2026-123633-four-web-facing-rces-one-patch-and-where-kubernetes-is-exposed
- https://blog.qualys.com/product-tech/patch-management/2026/11/18/nginx2root-five-nginx-flaws-led-by-cve-2026-123007-how-to-find-mitigate-and-patch-now
- https://discuss.kubernetes.io/t/security-advisory-cve-2026-116472-ingress-nginx-mirror-target-self-dos/35554

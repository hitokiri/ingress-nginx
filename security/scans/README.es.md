# Escaneos de seguridad

🇬🇧 [English version](README.md)

Resultados del escaneo de la imagen publicada con [Docker Scout](https://docs.docker.com/scout/).
El contexto completo (parches, bumps y pruebas) está en
[PARCHE-SEGURIDAD-NGINX2ROOT.md](../../PARCHE-SEGURIDAD-NGINX2ROOT.md).

## `hikohitokiri/ingress-nginx-controller:v1.15.2` (2026-11-25)

- Digest: `sha256:94aae840697d408f0932cb30e9dd9a1ed4cd6b554b0e42d2bfd7ca96eebe3936`
- Commit: `c3c540f0b` · Alpine 3.23.8 · NGINX 1.27.1 con los parches 00–52 · Go 1.26.11

| Crítica | Alta | Media | Baja | Con fix disponible |
|:---:|:---:|:---:|:---:|:---:|
| 1 | 2 | 9 | 5 | **0** |

Todas las CVE restantes están en paquetes de Alpine **sin versión corregida publicada**:

| Paquete | Versión | CVE | ¿Se usa por defecto? |
|---|---|---|---|
| `grpc` | 1.76.0-r2 | 🔴 CVE-2026-33186 (crítica) | **Probable falso positivo** (ver abajo). Además, solo lo carga el módulo de OpenTelemetry |
| `protobuf` | 31.1-r1 | 🟠 CVE-2026-0994 (alta) | No. Solo lo carga el módulo de OpenTelemetry |
| `libxml2` | 2.13.9-r1 | 🟠 CVE-2026-86140 (alta), 5 medias, 5 bajas | No. Solo lo usan ModSecurity (`enable-modsecurity`) y njs |
| `patch` | 2.8-r0 | 2 medias | No. Es una herramienta de línea de comandos que nginx no ejecuta |
| `busybox` | 1.37.0-r30 | 1 media | Es el shell de la imagen; nginx no lo ejecuta para servir tráfico |
| `nghttp2` | 1.69.0-r0 | 1 media | No. NGINX implementa HTTP/2 por su cuenta. Solo lo enlazan `curl` y los módulos de ModSecurity y OpenTelemetry |

**Sobre CVE-2026-33186:** según la [ficha de SUSE](https://www.suse.com/security/cve/CVE-2026-33186/), es
un fallo del **servidor de gRPC-Go** (validación de `:path` en HTTP/2), corregido en gRPC-Go 1.79.3. No es
de la librería C++ que trae Alpine, y en nuestra imagen esa librería solo actúa como cliente de OpenTelemetry.
La parte en Go del controlador ya usa `google.golang.org/grpc v1.79.3`.

Políticas de Scout: usuario no root ✓, imagen base al día ✓, sin CVE críticas ni altas con fix ✓.
Marca la falta de attestations de supply chain (normal en un build local) y 33 paquetes copyleft.

> **Ojo:** Scout no ve el NGINX compilado desde código, porque no es un paquete apk. Que no reporte
> CVE de NGINX no prueba que estén corregidas; eso se verificó aplicando los parches y con los tests.

## Comparación con la imagen de SUSE (`v1.15.2-prime7`)

Imagen: `registry.rancher.com/rancher/nginx-ingress-controller:v1.15.2-prime7`
(`sha256:908547c54f6a93daa0f3a6f2f72e1e88229b51b5f1db72be74d8a79fd7dad9b7`), basada en SUSE Linux Enterprise Server 16.0.

| | Nuestra `v1.15.2` | SUSE `v1.15.2-prime7` |
|---|---|---|
| Sistema base | Alpine 3.23.8 | SLES 16.0 |
| Resultado de Scout | 1 crítica, 2 altas, 9 medias, 5 bajas | 0 críticas, 0 altas, 2 medias |
| CVE con fix disponible | **0** | **2** (OpenTelemetry Go 1.45.0: CVE-2026-128085 y 128086) |
| `grpc` (C++) | 1.76.0 | 1.60.0 |
| `protobuf` | 31.1 | 28.3 |
| `libxml2` | 2.13.9 | 2.13.8 |

**La imagen de SUSE no está realmente más limpia; Scout tiene menos datos de vulnerabilidades de SLES 16.**
Según la base de CVE de SUSE, para SLES 16.0:

- **CVE-2026-86140 (libxml2):** "In progress", sin fix publicado. Su imagen también es vulnerable, pero Scout no la reporta.
- **CVE-2026-0994 (protobuf):** SUSE ya publicó el fix.
- **CVE-2026-33186:** es de gRPC-Go, así que no afecta a ninguna de las dos librerías C++.

Además, SUSE sigue en OpenTelemetry Go 1.45.0, con dos CVE que nosotros ya corregimos (1.47.0).

## Archivos

| Archivo | Contenido |
|---|---|
| [controller-v1.15.2-docker-scout-cves.md](controller-v1.15.2-docker-scout-cves.md) | Nuestra imagen: reporte completo por paquete y CVE |
| [controller-v1.15.2-docker-scout-quickview.txt](controller-v1.15.2-docker-scout-quickview.txt) | Nuestra imagen: resumen `docker scout quickview` y políticas |
| [controller-v1.15.2-docker-scout.sarif.json](controller-v1.15.2-docker-scout.sarif.json) | Nuestra imagen: formato SARIF, importable en GitHub Code Scanning |
| [suse-v1.15.2-prime7-docker-scout-cves.md](suse-v1.15.2-prime7-docker-scout-cves.md) | SUSE: reporte completo |
| [suse-v1.15.2-prime7-docker-scout-quickview.txt](suse-v1.15.2-prime7-docker-scout-quickview.txt) | SUSE: quickview y políticas |

## Volver a escanear

```bash
I=hikohitokiri/ingress-nginx-controller:v1.15.2
O=security/scans/controller-v1.15.2-docker-scout
docker scout cves --format markdown $I > $O-cves.md
docker scout cves --format sarif --output $O.sarif.json $I
docker scout quickview $I | sed 's/\x1b\[[0-9;]*m//g' > $O-quickview.txt
```

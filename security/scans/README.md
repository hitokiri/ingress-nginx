# Escaneos de seguridad

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
| `grpc` | 1.76.0-r2 | 🔴 CVE-2026-33186 (crítica) | No. Solo lo carga el módulo de OpenTelemetry (`enable-opentelemetry`) |
| `protobuf` | 31.1-r1 | 🟠 CVE-2026-0994 (alta) | No. Solo lo carga el módulo de OpenTelemetry |
| `libxml2` | 2.13.9-r1 | 🟠 CVE-2026-86140 (alta), 5 medias, 5 bajas | No. Solo lo usan ModSecurity (`enable-modsecurity`) y njs |
| `patch` | 2.8-r0 | 2 medias | No. Es una herramienta de línea de comandos que nginx no ejecuta |
| `busybox` | 1.37.0-r30 | 1 media | Es el shell de la imagen; nginx no lo ejecuta para servir tráfico |
| `nghttp2` | 1.69.0-r0 | 1 media | No. NGINX implementa HTTP/2 por su cuenta. Solo lo enlazan `curl` y los módulos de ModSecurity y OpenTelemetry |

Políticas de Scout: usuario no root ✓, imagen base al día ✓, sin CVE críticas ni altas con fix ✓.
Marca la falta de attestations de supply chain (normal en un build local) y 33 paquetes copyleft.

> **Ojo:** Scout no ve el NGINX compilado desde código, porque no es un paquete apk. Que no reporte
> CVE de NGINX no prueba que estén corregidas; eso se verificó aplicando los parches y con los tests.

## Archivos

| Archivo | Contenido |
|---|---|
| [controller-v1.15.2-docker-scout-cves.md](controller-v1.15.2-docker-scout-cves.md) | Reporte completo, por paquete y CVE, legible en GitHub |
| [controller-v1.15.2-docker-scout-quickview.txt](controller-v1.15.2-docker-scout-quickview.txt) | Resumen `docker scout quickview` y políticas |
| [controller-v1.15.2-docker-scout.sarif.json](controller-v1.15.2-docker-scout.sarif.json) | Formato SARIF, importable en GitHub Code Scanning u otras herramientas |

## Volver a escanear

```bash
I=hikohitokiri/ingress-nginx-controller:v1.15.2
O=security/scans/controller-v1.15.2-docker-scout
docker scout cves --format markdown $I > $O-cves.md
docker scout cves --format sarif --output $O.sarif.json $I
docker scout quickview $I | sed 's/\x1b\[[0-9;]*m//g' > $O-quickview.txt
```

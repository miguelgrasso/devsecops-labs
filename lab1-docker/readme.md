# Lab 1 — Docker Hardening + Escaneo de Vulnerabilidades (Trivy)

## Objetivo
Demostrar el impacto de un Dockerfile bien construido: mismo código, dos imágenes, resultados radicalmente distintos en tamaño y superficie de ataque.

## Qué construí
App Flask mínima ("Hello DevSecOps") empaquetada de dos formas:

### Dockerfile.bad (el anti-patrón)
- Base: `python:3.12` completa (~1GB de sistema operativo innecesario)
- Corre como **root**
- Sin `.dockerignore` (todo el contexto entra a la imagen)
- Single-stage: herramientas de build viajan a producción

### Dockerfile.good (el endurecido)
- **Multistage build**: stage 1 compila dependencias, stage 2 solo copia lo necesario
- Base final: `python:3.12-slim`
- Usuario no privilegiado (`appuser`) — principio de mínimo privilegio
- `.dockerignore` para reducir contexto y evitar filtrar archivos sensibles

## Resultados (escaneo con Trivy)

| Métrica | Dockerfile.bad | Dockerfile.good | Mejora |
|---|---|---|---|
| Tamaño | 1.62 GB | 191 MB | **-88%** |
| CVEs HIGH/CRITICAL | 424 | 23 | **-95%** |
| CVEs críticos | 60 | 4 | **-93%** |

## Comandos clave
```bash
docker build -f Dockerfile.bad -t hello-bad .
docker build -f Dockerfile.good -t hello-good .
trivy image hello-bad
trivy image hello-good
docker run -d -p 8080:8080 hello-good && curl localhost:8080
```

## Lecciones
1. **La mayoría de las vulnerabilidades no están en tu código — están en la base que eliges.** Una imagen slim elimina cientos de CVEs sin tocar una línea de la app.
2. **Multistage separa "lo que necesito para construir" de "lo que necesito para correr"** — compiladores y herramientas de build no pertenecen a producción.
3. **Non-root por defecto**: si comprometen el contenedor, el atacante no es root.
4. **Trivy como gate**: este escaneo pertenece al pipeline (bloqueante ante CRITICAL), no a una auditoría posterior. Shift-left.
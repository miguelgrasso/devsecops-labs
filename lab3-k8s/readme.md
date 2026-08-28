# Lab 3 — Probes, Limits y Autoscaling (romper para aprender)

## Qué construí
- Deployment con livenessProbe + readinessProbe (httpGet /, delays 5s/3s) y resources (requests 64Mi/100m, limits 128Mi/250m)
- HPA (autoscaling/v2): target CPU 50%, min 2 / max 6, sobre metrics-server

## Sabotaje 1 — OOMKilled
- Limits bajados a 8Mi/16Mi → contenedor muere al nacer: **exit 137** (128+9, SIGKILL del kernel vía cgroup)
- Rolling update ATASCADO: pods viejos sanos siguieron sirviendo (protección de K8s)
- Salida del incidente: `kubectl rollout undo` (emergencia) + `kubectl apply` del manifiesto bueno (reconciliación del last-applied) — el undo arregla la realidad, el apply arregla la verdad
- `kubectl diff` para verificar alineación archivo↔cluster

## Sabotaje 2 — CrashLoopBackOff por comando roto
- `command: ["python", "archivo_que_no_existe.py"]` → **exit 2** (Python: file not found — murió solo, vs 137: lo mataron)
- **Backoff exponencial MEDIDO**: 13s → 34s → 77s → 2m43s → 5m30s → tope ~5min (46 min de watch)
- Autopsia: describe (Last State) + logs --previous POR NOMBRE (el selector -l agarra al pod sano testigo)

## HPA bajo carga
- 3 generadores de carga (busybox + wget en loop) → CPU 49% → 107% → 233%
- **REPLICAS 2 → 4 → 6** (fórmula pedía ~9, respetó maxReplicas) — [pegar salida del watch]
- Tolerancia ±10% del target y stabilization window (~5min) para el des-escalado

## Lecciones
1. Exit codes: 128+ = lo mataron (137 OOM, 143 SIGTERM) / bajos = murió solo (1, 2)
2. CrashLoopBackOff es un ESTADO (pausa entre reintentos), no un error — la causa vive en Last State
3. Liveness impaciente mata apps sanas en arranque lento → startupProbe
4. El % del HPA es sobre los REQUESTS — sin requests bien puestos no hay autoscaling sano
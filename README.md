# Practica SA - Repositorio GitOps (P8 + P9 + P10)

Estado deseado del sistema. **ArgoCD** lo lee con el patron **app-of-apps**: una sola aplicacion raiz
(`p10-root-app`, namespace `argocd`, rama **p10**) apunta a la carpeta `apps/` y crea todo lo demas. Terraform (P10)
instala ArgoCD y crea esa aplicacion raiz; a partir de ahi el sistema se levanta solo.

- Repositorio de aplicacion (codigo, charts, politicas Kyverno, instrumentacion OpenTelemetry): https://github.com/BillyDread1531/Practicas-SA-B-201901385
- Rama de trabajo de P10: `p10` (continua desde `p9`)

## Estructura

| Ruta | Contenido |
|---|---|
| `apps/` | Aplicaciones hijas del app-of-apps (todas con `automated: prune + selfHeal` y `retry` con backoff) |
| `apps/sealed-secrets.yaml` | Controlador de Sealed Secrets, **ola 0** |
| `apps/argo-rollouts.yaml` | Argo Rollouts (entrega progresiva), **ola 0** |
| `apps/kyverno.yaml` | Kyverno (politicas de admision), **ola 0** |
| `apps/p8-kyverno-policies.yaml` | Politicas `p8-*` desde el repo de codigo, **ola 1** |
| `apps/prometheus.yaml` | Prometheus + Alertmanager: metricas de aplicacion, SLO (recording rules) y alertas por presupuesto de error, **ola 0** |
| `apps/otel-collector.yaml` | OpenTelemetry Collector: recibe trazas OTLP de los 3 servicios instrumentados y las reenvia a Tempo, **ola 0** |
| `apps/tempo.yaml` | Backend de trazas, **ola 0** |
| `apps/loki.yaml` | Almacen de registros (modo SingleBinary, filesystem), **ola 0** |
| `apps/promtail.yaml` | Agente que envia los logs JSON de todos los pods a Loki, **ola 0** |
| `apps/grafana.yaml` | Tablero de operacion + datasources Prometheus/Loki/Tempo correlacionados, **ola 0** |
| `apps/sa-platform-dev.yaml` | Plataforma de microservicios, PostgreSQL y RabbitMQ (con la instrumentacion y el canary por metricas), **ola 2** |
| `environments/dev/values.yaml` | Valores del entorno: imagenes en ghcr.io, variables de OpenTelemetry, flags de fallo inducido |
| `bootstrap/root.yaml` | Copia de referencia de la aplicacion raiz (la crea Terraform) |
| `security/sealed-secret.yaml` | Copia de referencia del SealedSecret (el que se aplica es el del chart) |

## Orden de sincronizacion (sync waves)

```
ola 0: sealed-secrets | argo-rollouts | kyverno | prometheus | otel-collector | tempo | loki | promtail | grafana
ola 1: p8-kyverno-policies
ola 2: sa-platform-dev   (microservicios + datos + Rollout del gateway con canary por metricas)
```

Todo el stack de observabilidad vive en el namespace **`observability`** (excluido de las politicas Kyverno
`require-non-root` y `require-resources`, igual que `argocd`/`kyverno`/`velero`/`argo-rollouts`: es infraestructura de
terceros via Helm, no la aplicacion bajo la cadena de suministro de P8).

## Observabilidad (P10)

- **Instrumentacion:** `gateway`, `auth-service` y `cursos-service` (codigo en `P5/*/src/telemetry/` del repo de
  aplicacion) con OpenTelemetry (trazas, propagacion de contexto W3C automatica) + `prom-client` (metricas) + `pino`
  (logs JSON con `trace_id`/`span_id`).
- **Canary por metricas:** el `AnalysisTemplate` `gateway-metrics` (en el chart `gateway`) consulta Prometheus con
  tasa de error y p95 de latencia filtrados por `rollouts_pod_template_hash` del canary. Sustituye al antiguo
  `gateway-health` (basado en un `GET /health` sintetico de P8).
- **Fallo inducido:** `CANARY_FAULT_INJECTION` (gateway, gobierna la demo del canary) y `FAULT_INJECTION`
  (auth-service, gobierna la demo de sintoma-a-causa). Ver `P10/runbooks/` y `P10/scripts/`.

## Reglas

Los cambios de despliegue se hacen por Git y ArgoCD los sincroniza. No se usa `kubectl apply`, `kubectl set image` ni
`helm upgrade` desde los workflows de CI/CD, salvo el procedimiento documentado de restauracion de datos de P9.

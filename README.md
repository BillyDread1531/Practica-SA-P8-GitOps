# Practica SA - Repositorio GitOps (P8 + P9)

Estado deseado del sistema. **ArgoCD** lo lee con el patron **app-of-apps**: una sola aplicacion raiz
(`p9-root-app`, namespace `argocd`) apunta a la carpeta `apps/` y crea todo lo demas. Terraform (P9) instala ArgoCD y
crea esa aplicacion raiz; a partir de ahi el sistema se levanta solo.

- Repositorio de aplicacion (codigo, charts, politicas Kyverno): https://github.com/BillyDread1531/Practicas-SA-B-201901385
- Rama de trabajo de P9: `p9`

## Estructura

| Ruta | Contenido |
|---|---|
| `apps/` | Aplicaciones hijas del app-of-apps (todas con `automated: prune + selfHeal` y `retry` con backoff) |
| `apps/sealed-secrets.yaml` | Controlador de Sealed Secrets (chart `bitnami.github.io/sealed-secrets`), **ola 0**. Sin rotacion automatica de llaves |
| `apps/argo-rollouts.yaml` | Argo Rollouts (entrega progresiva), **ola 0** |
| `apps/kyverno.yaml` | Kyverno (politicas de admision), **ola 0** |
| `apps/p8-kyverno-policies.yaml` | Politicas `p8-*` desde el repo de codigo, **ola 1** (necesitan los CRD de Kyverno) |
| `apps/sa-platform-dev.yaml` | Plataforma de microservicios, PostgreSQL y RabbitMQ, **ola 2** |
| `environments/dev/values.yaml` | Valores del entorno: imagenes por SHA inmutable, 2 replicas, requests ajustados al uso real, hook de Velero para PostgreSQL |
| `bootstrap/root.yaml` | Copia de referencia de la aplicacion raiz (la crea Terraform) |
| `security/sealed-secret.yaml` | Copia de referencia del SealedSecret (el que se aplica es el del chart) |

## Orden de sincronizacion (sync waves)

```
ola 0: sealed-secrets | argo-rollouts | kyverno      (controladores y CRD)
ola 1: p8-kyverno-policies                            (ClusterPolicy p8-*)
ola 2: sa-platform-dev                                (microservicios + datos + Rollout del gateway)
```

## Recursos de resiliencia (P9)

Definidos en los charts del repo de aplicacion y activados desde `environments/dev/values.yaml`:

- 2 replicas por microservicio (gateway como `Rollout` canary), 1 PostgreSQL y 1 RabbitMQ con PVC.
- `PodDisruptionBudget`: `minAvailable: 1` en los 5 servicios, `maxUnavailable: 1` en PostgreSQL/RabbitMQ.
- `podAntiAffinity` preferida por `kubernetes.io/hostname`; probes de startup, readiness y liveness.
- Requests de CPU ajustados al uso real para que un solo nodo pueda alojar todo el sistema tras perder el otro.
- Hook de Velero (`pre.hook.backup.velero.io`) que ejecuta `CHECKPOINT` en PostgreSQL antes de copiar el volumen.

## Reglas

Los cambios de despliegue se hacen por Git y ArgoCD los sincroniza. No se usa `kubectl apply`, `kubectl set image` ni
`helm upgrade` desde los workflows de CI/CD. La unica excepcion documentada es el procedimiento de restauracion de datos
(`P9/scripts/restore-datos.ps1`), que pausa la sincronizacion de forma temporal y la reactiva al terminar.

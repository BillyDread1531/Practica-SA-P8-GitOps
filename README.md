# Practica SA P8 - GitOps

Repositorio independiente para el estado deseado de despliegue de la Practica 8.

## Repositorio de aplicación

https://github.com/BillyDread1531/Practicas-SA-B-201901385

## Aplicación ArgoCD

- Nombre: `sa-platform-dev`
- Namespace: `argocd`
- Namespace destino: `sa-p8`
- Fuente: `P8/charts/sa-platform`
- Valores: `values-dev.yaml`

## GitOps

Los cambios de despliegue deben gestionarse mediante Git y sincronizarse mediante ArgoCD.

No se utilizará `kubectl apply`, `kubectl set image` ni `helm upgrade` desde los workflows de CI/CD.

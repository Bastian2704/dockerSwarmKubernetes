# Grafana con Argo CD y Kubernetes (GitOps)

Este repositorio es la **fuente de verdad**: Argo CD observa la carpeta `argocd/` de la rama `argocd` y sincroniza automáticamente el clúster con cada cambio.

| Archivo | Recurso |
|---|---|
| `argocd/deployment.yml` | Deployment de Grafana: 2 réplicas, requests/limits de CPU y memoria, puerto 3000 |
| `argocd/service.yml` | Service NodePort `30300` → `targetPort 3000` |
| `argocd/hpa.yml` | HPA: 2–5 réplicas, objetivo 50 % de CPU |

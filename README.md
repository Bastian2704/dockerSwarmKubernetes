# Grafana en Kubernetes (Minikube): Pod, Deployment y ReplicaSet

Tarea individual: desplegar una plataforma distinta a la vista en clase aplicando **Deployment** y **ReplicaSet**.
Plataforma elegida: **Grafana** (visualización de métricas y dashboards).

## Archivos

| Archivo | Contenido |
|---|---|
| `pod.yaml` | Un Pod de Grafana suelto (sin controlador) |
| `grafana.yaml` | Deployment (3 réplicas, rolling update) + Service NodePort `30300` |

## Conceptos

- **Pod**: unidad mínima de Kubernetes; uno o más contenedores que comparten red y almacenamiento. Si un Pod suelto muere, **nadie lo vuelve a crear**.
- **ReplicaSet**: controlador que garantiza que siempre haya **N Pods** iguales corriendo (los identifica por *labels*). Si uno muere, crea otro.
- **Deployment**: capa sobre el ReplicaSet. Lo crea y lo administra, y agrega **actualizaciones progresivas (rolling update)**, **historial de versiones** y **rollback**. En la práctica nunca se crea un ReplicaSet a mano: se crea un Deployment.

```
Deployment (grafana)
   └── ReplicaSet (grafana-<hash>)   ← uno nuevo por cada versión del template
         ├── Pod grafana-<hash>-xxxxx
         ├── Pod grafana-<hash>-yyyyy
         └── Pod grafana-<hash>-zzzzz
Service NodePort grafana-svc :30300 → reparte el tráfico entre los Pods con label app=grafana
```

## Paso a paso

### 1. Levantar el clúster
```bash
minikube start
kubectl get nodes
```
📸 *Captura 1: nodo `minikube` en estado Ready.*

### 2. Primer Pod
```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl delete pod grafana-pod      # se borra y NO vuelve
kubectl get pods
```
📸 *Captura 2: el Pod corriendo y luego desaparecido.*

### 3. Deployment + ReplicaSet + Service
```bash
kubectl apply -f grafana.yaml
kubectl get deploy,rs,pods,svc -o wide
```
Resultado: `deployment/grafana 3/3`, un `replicaset/grafana-<hash>` con DESIRED=3 y 3 Pods `Running`.

📸 *Captura 3: salida de `kubectl get deploy,rs,pods,svc`.*

### 4. Autorreparación (ReplicaSet)
```bash
kubectl delete pod <nombre-de-un-pod-grafana>
kubectl get pods        # aparece un Pod nuevo con AGE de pocos segundos
```
📸 *Captura 4: el ReplicaSet recreó el Pod borrado.*

### 5. Escalado
```bash
kubectl scale deploy/grafana --replicas=5
kubectl get rs          # DESIRED 5 / READY 5
```
📸 *Captura 5.*

### 6. Rolling update y rollback (Deployment)
```bash
kubectl set image deploy/grafana grafana=grafana/grafana:11.3.0
kubectl rollout status deploy/grafana
kubectl get rs              # RS nuevo con 5 Pods, RS viejo con 0
kubectl rollout history deploy/grafana
kubectl rollout undo deploy/grafana   # vuelve a 11.2.0
```
Se ven **dos ReplicaSets**: el Deployment va subiendo el nuevo y bajando el viejo de a uno (`maxSurge: 1`, `maxUnavailable: 1`), así el servicio nunca se cae. El RS viejo queda con 0 réplicas para poder hacer rollback.

📸 *Captura 6: dos ReplicaSets y el historial.*

### 7. Exposición de la plataforma
```bash
kubectl get svc grafana-svc
minikube service grafana-svc --url     # o: http://$(minikube ip):30300
curl http://$(minikube ip):30300/api/health
```
Abrir la URL en el navegador → login `admin` / `admin123`.

📸 *Captura 7: pantalla de login / home de Grafana en el navegador.*

### Limpieza
```bash
kubectl delete -f grafana.yaml -f pod.yaml
minikube stop
```

## Configuración destacada (`grafana.yaml`)

| Campo | Para qué |
|---|---|
| `replicas: 3` | El ReplicaSet mantiene 3 Pods vivos |
| `selector.matchLabels: app=grafana` | Cómo el RS y el Service encuentran sus Pods |
| `strategy: RollingUpdate` | Actualizar sin downtime |
| `env GF_SECURITY_ADMIN_*` | Usuario/clave del admin |
| `readinessProbe /api/health` | El Service solo manda tráfico a Pods listos |
| `resources` | Requests/limits de CPU y memoria |
| `Service NodePort 30300` | Expone Grafana fuera del clúster |

> Nota: cada réplica usa su propia base SQLite interna, así que los dashboards no se comparten entre réplicas. En producción se configuraría una base externa (PostgreSQL/MySQL) para que Grafana funcione bien con varias réplicas.

## Usos de Grafana

- **Monitoreo de infraestructura**: CPU, RAM, disco y red de servidores y clústers Kubernetes (junto con Prometheus).
- **Observabilidad de aplicaciones**: latencia, errores, peticiones por segundo; logs con Loki y trazas con Tempo.
- **Alertas**: avisos por correo, Slack, Teams, etc. cuando una métrica pasa un umbral.
- **Dashboards de negocio**: ventas, usuarios activos, KPIs leyendo bases de datos SQL.
- **IoT**: visualizar datos de sensores en tiempo real (InfluxDB).

## Usos de Deployment / ReplicaSet

- **Alta disponibilidad**: si un Pod o nodo falla, el ReplicaSet lo reemplaza solo.
- **Escalado horizontal**: subir/bajar réplicas según la carga (manual o con HPA).
- **Despliegues sin caída**: rolling updates de versión.
- **Rollback**: volver a la versión anterior con un comando si algo sale mal.

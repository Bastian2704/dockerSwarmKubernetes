# Bitácora: lo que se hizo, paso por paso

Este archivo cuenta **en orden** todo lo que se ejecutó para la tarea, con el comando exacto, qué hace, por qué se hizo y qué salió.
(El informe de entrega es `README.md`. Este archivo es para que entiendas el proceso.)

---

## Paso 0 — Revisar el entorno

**Comando:**
```bash
ls -la
which minikube kubectl docker gh git
minikube status
kubectl get pods -A
```

**Qué hace cada uno:**
- `ls -la`: muestra qué hay en la carpeta. Estaba **vacía**.
- `which ...`: dice si cada programa está instalado y dónde. Estaban todos:
  - `minikube` → crea el clúster Kubernetes local.
  - `kubectl` → la herramienta para darle órdenes al clúster.
  - `docker` → minikube lo usa como "driver" (el clúster corre dentro de un contenedor Docker).
  - `git` y `gh` → para crear el repositorio y subirlo a GitHub.
- `minikube status`: el clúster existía pero estaba **Stopped** (apagado).
- `kubectl get pods -A`: falló con `no route to host` porque el clúster estaba apagado (kubectl no tenía a quién hablarle).

**Conclusión:** no faltaba instalar nada, solo había que encender minikube.

---

## Paso 1 — Encender el clúster

**Comando:**
```bash
minikube start
kubectl get nodes
```

**Qué hace:**
- `minikube start` levanta el clúster (un nodo que es a la vez *control plane* y *worker*). También configura `kubectl` para que apunte a ese clúster.
- `kubectl get nodes` lista los nodos del clúster.

**Resultado:**
```
NAME       STATUS   ROLES           AGE     VERSION
minikube   Ready    control-plane   6d15h   v1.37.0
```
`Ready` = el clúster está listo para recibir trabajo.

---

## Paso 2 — Elegir la plataforma

La tarea pedía algo distinto a lo visto en clase (ElasticSearch, Redis, MongoDB, Grafana…).
Se eligió **Grafana** porque:
- Tiene **interfaz web** → es fácil sacar capturas que demuestren la "exposición de la plataforma".
- Tiene un endpoint de salud (`/api/health`) → sirve para comprobar que responde y para la `readinessProbe`.
- Es liviana y arranca rápido.

---

## Paso 3 — Crear `pod.yaml` (el primer Pod)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: grafana-pod
  labels:
    app: grafana-pod
spec:
  containers:
    - name: grafana
      image: grafana/grafana:11.2.0
      ports:
        - containerPort: 3000
```

**Línea por línea:**
| Línea | Significado |
|---|---|
| `apiVersion: v1` | Los Pods pertenecen a la API base `v1` de Kubernetes |
| `kind: Pod` | Tipo de objeto: un Pod |
| `metadata.name` | Nombre del Pod: `grafana-pod` |
| `metadata.labels` | Etiqueta `app: grafana-pod`. Es distinta a la del Deployment para que el Service **no** le mande tráfico a este Pod suelto |
| `containers` | Lista de contenedores dentro del Pod (aquí solo uno) |
| `image: grafana/grafana:11.2.0` | Imagen de Docker Hub, con versión fija (no `latest`, para que sea reproducible) |
| `containerPort: 3000` | Puerto donde escucha Grafana dentro del contenedor |

**Por qué existe:** la tarea dice "tenemos nuestro primer POD, ahora vamos con Deployment y ReplicaSet". Este archivo es ese primer Pod, y sirve para mostrar la diferencia: **un Pod suelto, si se borra o muere, no vuelve**.

---

## Paso 4 — Crear `grafana.yaml` (Deployment + Service)

El archivo tiene **dos objetos** separados por `---`.

### 4.1 El Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  replicas: 3
  selector:
    matchLabels:
      app: grafana
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:11.2.0
          ports:
            - containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_USER
              value: admin
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: admin123
          readinessProbe:
            httpGet:
              path: /api/health
              port: 3000
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 256Mi }
```

**Línea por línea:**
| Campo | Significado |
|---|---|
| `apiVersion: apps/v1` | Deployments y ReplicaSets están en la API `apps/v1` |
| `kind: Deployment` | Tipo de objeto |
| `replicas: 3` | Quiero **3 copias** del Pod siempre vivas. El Deployment se lo pasa al ReplicaSet, que es quien realmente las mantiene |
| `selector.matchLabels: app=grafana` | Cómo el ReplicaSet reconoce "sus" Pods: todos los que tengan la etiqueta `app=grafana` |
| `strategy: RollingUpdate` | Al cambiar de versión, reemplaza los Pods **de a poco**, no todos de golpe |
| `maxUnavailable: 1` | Durante la actualización, como máximo 1 Pod puede estar caído |
| `maxSurge: 1` | Durante la actualización, puede haber como máximo 1 Pod extra por encima de `replicas` |
| `template` | El "molde" de cada Pod. Es igual a un `pod.yaml`, pero sin `apiVersion` ni `kind` |
| `template.metadata.labels: app=grafana` | **Tiene que coincidir** con el `selector`; si no, Kubernetes rechaza el Deployment |
| `env GF_SECURITY_ADMIN_USER / PASSWORD` | Variables de entorno que Grafana lee para definir el usuario admin (`admin` / `admin123`) |
| `readinessProbe` | Kubernetes hace `GET /api/health` al puerto 3000; solo cuando responde OK el Pod se marca `READY 1/1` y empieza a recibir tráfico |
| `resources.requests` | Lo mínimo que se le reserva al Pod: 0.1 CPU y 128 MiB de RAM |
| `resources.limits` | El máximo que puede usar: 0.5 CPU y 256 MiB. Evita que un Pod se coma toda la máquina |

**Importante:** en el YAML **no se escribe ningún ReplicaSet**. El Deployment lo crea solo. Así se trabaja en la práctica.

### 4.2 El Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-svc
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30300
```

| Campo | Significado |
|---|---|
| `kind: Service` | Punto de acceso estable a un grupo de Pods (los Pods cambian de IP cuando se recrean; el Service no) |
| `type: NodePort` | Abre un puerto en el nodo para poder entrar **desde fuera del clúster** |
| `selector: app=grafana` | Manda el tráfico a los Pods con esa etiqueta (los del Deployment; el `grafana-pod` suelto queda fuera) |
| `port: 3000` | Puerto del Service dentro del clúster |
| `targetPort: 3000` | Puerto del contenedor al que reenvía |
| `nodePort: 30300` | Puerto público en la IP del nodo minikube (el rango permitido es 30000–32767) |

---

## Paso 5 — Aplicar todo y esperar a que esté listo

**Comando:**
```bash
kubectl apply -f pod.yaml -f grafana.yaml
kubectl rollout status deploy/grafana --timeout=300s
kubectl wait --for=condition=Ready pod/grafana-pod --timeout=120s
kubectl get deploy,rs,pods,svc -o wide
```

**Qué hace:**
- `kubectl apply -f` → envía los YAML al clúster; Kubernetes crea lo que describen.
- `kubectl rollout status` → espera hasta que las 3 réplicas del Deployment estén disponibles.
- `kubectl wait` → espera a que el Pod suelto esté `Ready`.
- `kubectl get deploy,rs,pods,svc -o wide` → muestra todo junto.

**Resultado (resumido):**
```
pod/grafana-pod created
deployment.apps/grafana created
service/grafana-svc created

deployment.apps/grafana   3/3   ...
replicaset.apps/grafana-599c5d6dbc   DESIRED 3   CURRENT 3   READY 3

pod/grafana-599c5d6dbc-6tz88   1/1   Running
pod/grafana-599c5d6dbc-ltslc   1/1   Running
pod/grafana-599c5d6dbc-txf8v   1/1   Running
pod/grafana-pod                1/1   Running

service/grafana-svc   NodePort   10.104.120.27   3000:30300/TCP
```

**Cómo leerlo:**
- El Deployment `grafana` creó **automáticamente** el ReplicaSet `grafana-599c5d6dbc`.
  - `599c5d6dbc` es un *hash* del template del Pod. Si cambias el template (por ejemplo la imagen), cambia el hash y se crea otro ReplicaSet.
- Los Pods se llaman `<replicaset>-<sufijo aleatorio>`: así se ve que **pertenecen** a ese ReplicaSet.
- `grafana-pod` no tiene hash: no pertenece a nadie.

---

## Paso 6 — Probar la autorreparación (ReplicaSet)

**Comando:**
```bash
P=$(kubectl get pod -l app=grafana -o name | head -1)
kubectl delete $P --wait=false
sleep 3
kubectl get pods -l app=grafana
```

**Qué hace:**
- `kubectl get pod -l app=grafana -o name | head -1` → toma el nombre del primer Pod del Deployment (`-l` filtra por etiqueta).
- `kubectl delete` → lo borra (simula que el Pod "se murió").
- Espera 3 segundos y vuelve a listar.

**Resultado:**
```
pod "grafana-599c5d6dbc-6tz88" deleted
grafana-599c5d6dbc-ltslc   1/1   Running   39s
grafana-599c5d6dbc-p259n   0/1   Running   4s    ← NUEVO
grafana-599c5d6dbc-txf8v   1/1   Running   39s
```

**Explicación:** el ReplicaSet vio que había 2 Pods en vez de 3 y **creó uno nuevo** (`p259n`, de 4 segundos). Sale `0/1` porque la `readinessProbe` todavía no da OK; en unos segundos pasa a `1/1`.
Con el `grafana-pod` suelto esto **no** pasa: si lo borras, desaparece para siempre.

---

## Paso 7 — Escalar

**Comando:**
```bash
kubectl scale deploy/grafana --replicas=5
kubectl rollout status deploy/grafana --timeout=180s
kubectl get rs
```

**Qué hace:** cambia `replicas` de 3 a 5 sin editar el YAML. El Deployment le pasa el nuevo número al ReplicaSet, y este crea 2 Pods más.

**Resultado:**
```
NAME                 DESIRED   CURRENT   READY
grafana-599c5d6dbc   5         5         5
```

---

## Paso 8 — Rolling update (actualizar la versión sin cortar el servicio)

**Comando:**
```bash
kubectl set image deploy/grafana grafana=grafana/grafana:11.3.0
kubectl rollout status deploy/grafana --timeout=400s
kubectl get rs
kubectl rollout history deploy/grafana
```

**Qué hace:**
- `kubectl set image` → cambia la imagen del contenedor `grafana` de la 11.2.0 a la 11.3.0.
- Como el template cambió, el Deployment **crea un ReplicaSet nuevo** y va moviendo los Pods de a uno (respetando `maxSurge: 1` y `maxUnavailable: 1`).
- `rollout history` → muestra las versiones (revisiones) que tuvo el Deployment.

**Resultado:**
```
Waiting ... 2 out of 5 new replicas have been updated...
Waiting ... 3 out of 5 ...
Waiting ... 4 out of 5 ...
Waiting ... 1 old replicas are pending termination...
deployment "grafana" successfully rolled out

NAME                 DESIRED   CURRENT   READY
grafana-547d968fc8   5         5         5     ← ReplicaSet NUEVO (v11.3.0)
grafana-599c5d6dbc   0         0         0     ← ReplicaSet VIEJO (v11.2.0)

REVISION
1
2
```

**Explicación:**
- Esta es **la diferencia clave entre Deployment y ReplicaSet**: el ReplicaSet solo mantiene N copias de **una** versión; el Deployment maneja **varios** ReplicaSets para pasar de una versión a otra.
- El ReplicaSet viejo no se borra: queda en 0 para poder volver atrás con `kubectl rollout undo deploy/grafana`. (El rollback **no** se ejecutó; está documentado en el README para que lo hagas tú y lo captures.)

---

## Paso 9 — Comprobar la exposición

**Comando:**
```bash
curl -s http://$(minikube ip):30300/api/health
```

**Qué hace:**
- `minikube ip` → da la IP del nodo (aquí `192.168.49.2`).
- `curl` → hace una petición HTTP al NodePort 30300, como lo haría un navegador.

**Resultado:**
```json
{
  "database": "ok",
  "version": "11.3.0",
  "commit": "d9455ff7db73b694db7d412e49a68bec767f2b5a"
}
```
Grafana responde desde **fuera del clúster** y ya está en la **versión nueva** (11.3.0), lo que confirma que el rolling update funcionó.

---

## Paso 10 — Escribir el informe (`README.md`)

Contiene:
- Teoría de Pod / ReplicaSet / Deployment con un diagrama.
- Los comandos de los pasos 1 al 7 para que **tú** los repitas y saques las capturas (marcadas con 📸).
- Una tabla con la configuración destacada del YAML.
- Una nota técnica: cada réplica de Grafana usa su propia base SQLite, así que en producción haría falta una base externa.
- **Usos de Grafana**: monitoreo de infraestructura, observabilidad de aplicaciones, alertas, dashboards de negocio, IoT.
- **Usos de Deployment/ReplicaSet**: alta disponibilidad, escalado, despliegues sin caída, rollback.

---

## Paso 11 — Repositorio git (solo local)

**Comando:**
```bash
git init -q -b main
git add .
git commit -m "Grafana en Kubernetes: Pod, Deployment, ReplicaSet y Service"
gh auth status
```

**Qué hace:**
- `git init -b main` → convierte la carpeta en un repositorio con la rama `main`.
- `git add .` + `git commit` → guarda `pod.yaml`, `grafana.yaml` y `README.md` en el primer commit (`8f50e59`).
- `gh auth status` → confirma que la CLI de GitHub tiene sesión iniciada (cuenta `Bastian2704`).

**Todavía NO se subió a GitHub.** Falta tu confirmación para crearlo como repo **público**.

---

## Estado actual del clúster

| Objeto | Estado |
|---|---|
| `grafana-pod` | Corriendo (Pod suelto) |
| `deployment/grafana` | 5 réplicas, imagen **11.3.0** (revisión 2) |
| `replicaset grafana-547d968fc8` | 5 Pods (actual) |
| `replicaset grafana-599c5d6dbc` | 0 Pods (versión anterior, guardada para rollback) |
| `service/grafana-svc` | NodePort 30300 → `http://192.168.49.2:30300` |

## Pendiente

1. Sacar las capturas (conviene reiniciar desde cero con `kubectl delete -f grafana.yaml -f pod.yaml`).
2. Abrir Grafana en el navegador y capturar el login/home.
3. Publicar el repositorio en GitHub como público.
4. Agregar el link del repo y las capturas al informe.

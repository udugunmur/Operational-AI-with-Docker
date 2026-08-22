# Parte 2: Operacionalización de Modelos de IA

## Capítulo 5: Ejecución de Contenedores de Modelos ML en Kubernetes

En el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781807301095/5), aprendió a ejecutar modelos de **IA** localmente con **Docker Model Runner**. Construyó un chatbot compuesto por un frontend en React, un backend en Go y un modelo ejecutándose en segundo plano que sirve una API compatible con OpenAI. Todo funciona perfectamente en su máquina.

Pero piense en lo que sucede cuando diez personas intentan usar ese chatbot al mismo tiempo. O cuando necesita implementar una nueva versión del modelo sin interrumpir el servicio. O cuando un compañero clona el repositorio, ejecuta `docker compose up` en su portátil y pasa la siguiente hora depurando por qué el contenedor del modelo se bloquea en su máquina pero no en la suya.

Ahí es donde entra **Kubernetes**.

Kubernetes es una plataforma de código abierto para ejecutar cargas de trabajo contenerizadas en un clúster de máquinas. Mientras que Docker Compose orquesta contenedores en un solo host, Kubernetes los orquesta a través de múltiples hosts, reiniciando automáticamente lo que falla, equilibrando el tráfico entre réplicas saludables, implementando nuevas versiones con cero tiempo de inactividad y gobernando los recursos para que ninguna carga de trabajo deje sin recursos a las demás.

Para las cargas de trabajo de **ML** en particular, Kubernetes resuelve un conjunto de problemas para los que Docker Compose simplemente no fue diseñado. Pero seamos honestos: añade una complejidad real. La buena noticia es que no necesita una cuenta en la nube ni un rack de servidores para aprenderlo. A partir de la versión 4.38 de Docker Desktop, Kubernetes basado en **kind** viene integrado directamente. Sin herramientas separadas ni scripts complejos. Solo un interruptor en la configuración de Docker Desktop.

En este capítulo aprenderá:
- ¿Por qué Kubernetes para ML y cuándo no utilizarlo?
- Configuración de Kubernetes en Docker Desktop
- Primitivas de Kubernetes a través de la perspectiva de ML
- Despliegue de Docker Model Runner en Kubernetes
- Gestión de recursos y pruebas de salud (*health probes*) para contenedores de modelos
- Escalado de cargas de trabajo de ML en Kubernetes
- El ecosistema de ML en Kubernetes

---

### Requisitos técnicos

Para seguir los ejemplos de este capítulo, necesitará lo siguiente:

**Software:**
- **Docker Desktop 4.38 o superior**: Kubernetes basado en *kind* se incluye como función integrada a partir de la versión 4.38. Descárguelo desde [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/). Verifique su versión en el icono de ajustes de Docker Desktop → *About Docker Desktop*.

**Hardware:**
- Mínimo 8 GB de RAM (16 GB recomendados). Ejecutar un clúster de Kubernetes multinodo junto con Docker Model Runner requiere un uso intensivo de memoria.
- 10 GB de espacio libre en disco para imágenes de nodos de Kubernetes, archivos de modelos e imágenes de contenedores.

**Conocimientos previos:**
- Este capítulo se basa directamente en el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781807301095/5). Debe sentirse cómodo ejecutando aplicaciones de Docker Compose, comprendiendo configuraciones multiservicio y trabajando con variables de entorno. No se asumen conocimientos previos de Kubernetes. Cada concepto se introduce desde cero.

Los ejemplos de código están disponibles en el repositorio oficial en `chap-05`.

---

### ¿Por qué Kubernetes para ML y cuándo no utilizarlo?

Kubernetes resuelve problemas específicos y solo compensa su sobrecarga operativa cuando esos problemas son reales para usted. Veamos qué no puede hacer Docker Compose, qué puede hacer Kubernetes y dónde se encuentra el límite entre ambos.

*Figura 5.1: El problema central por el cual Docker Compose encuentra limitaciones a escala, y lo que añade Kubernetes*

#### Lo que Docker Compose no puede hacer

Docker Compose fue diseñado para despliegues en una sola máquina. Dentro de esa restricción, es una herramienta excelente, pero tiene límites estrictos:
- **Sin tolerancia a fallos del host**: Si el contenedor falla a las 3 AM en un solo host, Docker Compose lo reiniciará, pero si la máquina falla por completo, todo se cae. No puede mover cargas de trabajo a otro host.
- **Sin escalado automático ni distribución**: Escalar con `docker compose up --scale backend=3` requiere configurar balanceadores manualmente y no responde a picos de tráfico en tiempo real.
- **Despliegues con interrupción**: Actualizar un modelo implica detener el contenedor antiguo e iniciar el nuevo, generando una ventana de indisponibilidad.

#### Los problemas específicos que Kubernetes resuelve para ML

- **Autorreparación (*Self-healing*)**: Kubernetes monitoriza continuamente cada contenedor. Si un Pod de modelo falla o deja de responder debido a que una solicitud consumió toda la memoria, Kubernetes lo reemplaza automáticamente en cualquier nodo disponible del clúster sin intervención manual.
- **Programación multinodo (*Multi-node scheduling*)**: Distribuye cargas de trabajo en un grupo de máquinas. Puede mantener nodos equipados con GPU y permitir que el planificador (*scheduler*) coloque los contenedores de modelos donde haya capacidad disponible.
- **Escalado automático (*Autoscaling*)**: El **Horizontal Pod Autoscaler (HPA)** monitoriza el servicio de inferencia y añade réplicas cuando aumenta la carga, eliminándolas cuando disminuye.
- **Despliegues sin tiempo de inactividad (*Zero-downtime deployments*)**: Kubernetes inicia los nuevos Pods primero y redirige el tráfico solo después de que superan las pruebas de salud (*health checks*).
- **Gobernanza de recursos**: Establece límites estrictos de CPU y memoria por contenedor, evitando que un proceso descontrolado agote la memoria del nodo.

#### Cuándo Kubernetes es la opción equivocada

> **Cuándo permanecer con Docker Compose:**
> - Si está en fase de experimentación activa, cambiando la arquitectura con frecuencia y explorando modelos.
> - Si opera una herramienta interna de bajo tráfico que una sola máquina maneja cómodamente.
> - Si el equipo es pequeño y la sobrecarga operativa de Kubernetes ralentizaría el desarrollo más de lo que aportarían sus ventajas.

#### De Docker Compose a Kubernetes: El mapa mental

| Concepto en Docker Compose | Equivalente en Kubernetes | Propósito |
| :--- | :--- | :--- |
| `service:` | **Deployment + Service** | Ejecutar una carga de trabajo contenerizada y exponerla |
| `ports:` | **Service (ClusterIP)** | Endpoint de red estable para tráfico interno |
| `environment:` | **ConfigMap / Secret** | Pasar configuración a los contenedores |
| `volumes:` | **PersistentVolumeClaim (PVC)** | Adjuntar almacenamiento duradero a un contenedor |
| `depends_on:` | **Readiness probes / Init containers** | Controlar el orden de inicio |
| `restart: always` | **Deployment self-healing** | Mantener las cargas de trabajo en ejecución automáticamente |
| `deploy.replicas:` | `spec.replicas` en **Deployment** | Ejecutar múltiples instancias |
| `networks:` | **Services + NetworkPolicy** | Controlar cómo se comunican los contenedores |

*Tabla 5.1: Mapa mental comparativo de conceptos entre Docker Compose y Kubernetes*

---

### Configuración de Kubernetes en Docker Desktop

Docker Desktop 4.38 introdujo **kind** (*Kubernetes in Docker*) como aprovisionador integrado, permitiendo ejecutar clústeres multinodo reales como contenedores locales sin requerir instalaciones adicionales.

Requisitos previos:
1. Iniciar sesión en su cuenta de Docker en Docker Desktop.
2. Habilitar el almacén de imágenes `containerd` (*Settings > General > Use containerd for pulling and storing images*).

#### Paso 1: Habilitar Kubernetes basado en kind

1. Abra Docker Desktop y haga clic en el icono de engranaje (*Settings*).
2. En la barra lateral izquierda, seleccione **Kubernetes**.
3. Active la casilla **Enable Kubernetes** y elija **kind** como aprovisionador.
4. Seleccione la versión de Kubernetes deseada (por ejemplo, `1.34.3`).
5. Configure el número de nodos de trabajo (*worker nodes*) en **2** (1 nodo de plano de control y 2 workers).
6. Haga clic en **Apply & Restart** y luego en **Install**.

*Figura 5.2 a 5.6: Configuración del clúster de Kubernetes en Docker Desktop*

#### Paso 2: Verificar que el clúster esté listo

```bash
kubectl get nodes
```

Salida esperada:
```text
NAME STATUS ROLES AGE VERSION
desktop-control-plane Ready control-plane 2m39s v1.34.3
desktop-worker Ready <none> 2m25s v1.34.3
```

Confirmar el contexto activo:

```bash
kubectl config current-context
```

Salida:
```text
docker-desktop
```

#### Paso 3: Crear un espacio de nombres (Namespace)

```bash
kubectl create namespace ai-app
```

Salida:
```text
namespace/ai-app created
```

Verificar los namespaces:

```bash
kubectl get namespaces
```

```text
NAME STATUS AGE
default Active 5m
kube-system Active 5m
kube-public Active 5m
kube-node-lease Active 5m
ai-app Active 3s
```

*Figura 5.7: Panel de control de Kubernetes en Docker Desktop*

---

### Primitivas de Kubernetes a través de la perspectiva de ML

- **Pod**: La unidad mínima de ejecución en Kubernetes. Contiene uno o más contenedores que comparten red y almacenamiento.
- **Deployment**: Declara el estado deseado de su servicio (imagen, número de réplicas y estrategias de actualización).
- **Service**: Proporciona una IP virtual y un nombre DNS estable para enrutar el tráfico hacia los Pods saludables.
- **ConfigMap**: Almacena variables de configuración no confidenciales como pares clave-valor.
- **Secret**: Almacena credenciales sensibles (claves de API, contraseñas) codificadas en base64.
- **PersistentVolumeClaim (PVC)**: Solicita almacenamiento persistente e independiente del ciclo de vida del Pod para los pesos de los modelos.

---

### Despliegue de Docker Model Runner en Kubernetes

*Figura 5.8: El chatbot del Capítulo 3 migrado a Kubernetes*

#### Paso 1: Crear el almacenamiento del modelo (storage.yaml)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage
  namespace: ai-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Aplicar el manifiesto:

```bash
kubectl apply -f storage.yaml
```

```text
persistentvolumeclaim/model-storage created
```

#### Paso 2: Almacenar configuración y credenciales (config.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dmr-config
  namespace: ai-app
data:
  MODEL_NAME: "llama3.2:3B-Q4_K_M"
  DMR_HOST: "0.0.0.0"
  DMR_PORT: "12434"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: ai-app
type: Opaque
stringData:
  ANTHROPIC_API_KEY: "your-api-key-here"
```

Aplicar el manifiesto:

```bash
kubectl apply -f config.yaml
```

```text
configmap/dmr-config created
secret/app-secrets created
```

#### Paso 3: Desplegar Docker Model Runner (dmr-deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-model-runner
  namespace: ai-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: docker-model-runner
  template:
    metadata:
      labels:
        app: docker-model-runner
    spec:
      initContainers:
        - name: changeowner
          image: busybox
          command: ["sh", "-c", "chmod a+rwx /models"]
          volumeMounts:
            - name: model-files
              mountPath: /models
      containers:
        - name: model-runner
          image: docker/model-runner:latest
          ports:
            - containerPort: 12434
          env:
            - name: DMR_ORIGINS
              value: "http://localhost:31245,http://localhost:12434"
          envFrom:
            - configMapRef:
                name: dmr-config
          volumeMounts:
            - name: model-files
              mountPath: /models
          resources:
            requests:
              memory: "4Gi"
              cpu: "1000m"
            limits:
              memory: "8Gi"
              cpu: "4000m"
          readinessProbe:
            httpGet:
              path: /v1/models
              port: 12434
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 6
          livenessProbe:
            httpGet:
              path: /v1/models
              port: 12434
            initialDelaySeconds: 120
            periodSeconds: 30
            failureThreshold: 3
        - name: model-init
          image: curlimages/curl:8.14.1
          envFrom:
            - configMapRef:
                name: dmr-config
          command: ["/bin/sh", "-c"]
          args:
            - |
              set -ex
              MODEL_RUNNER=http://localhost:12434
              echo "Pre-pulling models..."
              if [ -n "$MODEL_NAME" ]; then
                echo "Pulling model: $MODEL_NAME"
                curl -d "{\"from\": \"$MODEL_NAME\"}" "$MODEL_RUNNER"/models/create
              fi
              echo "Model pre-pull complete"
              tail -f /dev/null
          volumeMounts:
            - name: model-files
              mountPath: /models
      volumes:
        - name: model-files
          persistentVolumeClaim:
            claimName: model-storage
```

#### Paso 4: Exponer el Model Runner con un Service

Añada la siguiente sección al final de `dmr-deployment.yaml`:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: docker-model-runner
  namespace: ai-app
spec:
  selector:
    app: docker-model-runner
  ports:
    - port: 12434
      targetPort: 12434
```

#### Paso 5: Desplegar la pila completa del chatbot

Estructura de archivos:

```text
chap-05/
  manifests/
    storage.yaml            # PVC for model files
    config.yaml             # ConfigMap and Secret
    dmr-deployment.yaml     # Model Runner Deployment + Service
    backend-deployment.yaml # Go backend Deployment + Service
    frontend-deployment.yaml# React frontend Deployment + Service
```

Importar imágenes locales en los nodos kind de Docker Desktop:

```bash
docker save go-backend:latest | docker exec -i desktop-worker ctr -n k8s.io images import -
docker save react-frontend:latest | docker exec -i desktop-worker ctr -n k8s.io images import -
```

Aplicar todos los manifiestos:

```bash
kubectl apply -f manifests/
```

```text
persistentvolumeclaim/model-storage created
configmap/dmr-config created
secret/app-secrets created
deployment.apps/docker-model-runner created
service/docker-model-runner created
deployment.apps/go-backend created
service/go-backend created
deployment.apps/react-frontend created
service/react-frontend created
```

Monitorear el despliegue:

```bash
kubectl get deployments -n ai-app --watch
```

```text
NAME READY UP-TO-DATE AVAILABLE AGE
go-backend 1/1 1 1 6s
react-frontend 1/1 1 1 6s
docker-model-runner 0/1 1 0 6s
docker-model-runner 1/1 1 1 78s
```

Acceder a la aplicación mediante reenvío de puertos (*port-forward*):

```bash
kubectl port-forward svc/react-frontend 3000:3000 -n ai-app
```

Abra `http://localhost:3000` en su navegador.

---

### Gestión de recursos y pruebas de salud para contenedores de modelos

#### Solicitudes y límites de recursos (Requests and Limits)

```yaml
resources:
  requests:
    memory: "4Gi" # Mínimo garantizado para cargar el modelo
    cpu: "1000m"  # 1 núcleo de CPU (1000 millicores)
  limits:
    memory: "8Gi" # Límite estricto; si se supera provoca OOMKilled
    cpu: "4000m"  # 4 núcleos de CPU máximos durante ráfagas de inferencia
```

*Figura 5.9: Comparación entre Requests y Limits en Kubernetes*

#### Pruebas de salud: El error más común en ML

Los contenedores de modelos pueden tardar 60-90 segundos en cargar los pesos en memoria.

**Configuración incorrecta (Provoca CrashLoopBackOff):**
```yaml
# NO HACER ESTO: tiempos demasiado cortos que matan el contenedor
livenessProbe:
  httpGet:
    path: /health
    port: 12434
  initialDelaySeconds: 10 # Demasiado corto, el modelo aún está cargando
  periodSeconds: 5
  failureThreshold: 3
```

**Configuración correcta:**
```yaml
# HACER ESTO: tiempos adaptados a la carga real del modelo
readinessProbe:
  httpGet:
    path: /v1/models # Endpoint de DMR que devuelve 200 solo cuando el modelo está cargado
    port: 12434
  initialDelaySeconds: 60 # Esperar antes de iniciar las comprobaciones
  periodSeconds: 10
  failureThreshold: 6    # Tolerancia suficiente
livenessProbe:
  httpGet:
    path: /health
    port: 12434
  initialDelaySeconds: 120
  periodSeconds: 30
  failureThreshold: 3
```

#### Diagnóstico de fallos comunes

```bash
kubectl describe pod -l app=docker-model-runner -n ai-app
```

| Síntoma | Causa raíz | Solución |
| :--- | :--- | :--- |
| **CrashLoopBackOff** inmediato | Las sondas se activan antes de que el modelo termine de cargar | Incrementar `initialDelaySeconds` |
| Pod en estado **Pending** continuo | Ningún nodo tiene recursos suficientes | Reducir `requests` o añadir nodos |
| **OOMKilled** en eventos del Pod | El modelo excede el límite de memoria | Incrementar `limits.memory` o usar modelos más cuantizados |
| Pod listo pero devuelve **503** | La sonda de readiness usa una ruta que responde antes de cargar el modelo | Usar `/v1/models` en lugar de `/health` |
| El modelo se descarga en cada reinicio | No hay PVC configurado | Montar los pesos desde un `PersistentVolumeClaim` |

*Tabla 5.2: Patrones de fallo comunes en Kubernetes y cómo solucionarlos*

---

### Escalado de cargas de trabajo de ML en Kubernetes

#### Escalado manual

```bash
kubectl scale deployment go-backend --replicas=3 -n ai-app
kubectl get pods -n ai-app -l app=go-backend
```

```text
NAME READY STATUS RESTARTS AGE
go-backend-7d9f8b-xk2p1 1/1 Running 0 14m
go-backend-7d9f8b-nwq94 0/1 ContainerCreating 0 3s
go-backend-7d9f8b-mh7j3 0/1 ContainerCreating 0 3s
```

#### Escalado automático con Horizontal Pod Autoscaler (HPA)

*Figura 5.10: Flujo de trabajo del Horizontal Pod Autoscaler (HPA)*

Crear `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: ai-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: go-backend
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Aplicar y verificar HPA:

```bash
kubectl apply -f hpa.yaml
kubectl get hpa -n ai-app
```

```text
NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS
backend-hpa Deployment/go-backend 18%/70% 1 5 1
```

Prueba de carga:

```bash
# Terminal 1: Observar HPA
kubectl get hpa -n ai-app --watch

# Terminal 2: Generar carga
kubectl run load-generator \
  --image=busybox \
  --namespace=ai-app \
  -- /bin/sh -c "while true; do \
    wget -q -O- http://go-backend:8080/api/health; \
  done"
```

Limpieza de recursos del capítulo:

```bash
kubectl delete namespace ai-app
```

---

### El ecosistema de ML en Kubernetes

*Figura 5.11: El ecosistema de herramientas de ML sobre Kubernetes*

#### KubeRay – Entrenamiento e inferencia distribuida

Extiende Kubernetes para gestionar clústeres de Ray:
- `RayCluster`: Aprovisiona grupos de workers.
- `RayJob`: Ejecuta trabajos de entrenamiento distribuidos y libera recursos al finalizar.
- `RayService`: Despliega modelos con Ray Serve para enrutamiento avanzado.

```bash
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo update
helm install kuberay-operator kuberay/kuberay-operator \
  --namespace kuberay-system \
  --create-namespace
```

Ejemplo de `RayJob` (`hello-ray.yaml`):

```yaml
apiVersion: ray.io/v1
kind: RayJob
metadata:
  name: hello-ray
  namespace: ai-app
spec:
  entrypoint: python -c "import ray; ray.init(); print(ray.cluster_resources())"
  rayClusterSpec:
    headGroupSpec:
      rayStartParams:
        dashboard-host: "0.0.0.0"
      template:
        spec:
          containers:
            - name: ray-head
              image: rayproject/ray:2.9.0
              resources:
                requests:
                  cpu: "1"
                  memory: "2Gi"
    workerGroupSpecs:
      - replicas: 2
        groupName: worker-group
        rayStartParams: {}
        template:
          spec:
            containers:
              - name: ray-worker
                image: rayproject/ray:2.9.0
                resources:
                  requests:
                    cpu: "1"
                    memory: "2Gi"
```

```bash
kubectl create ns ai-app
kubectl apply -f hello-ray.yaml
kubectl get rayjob hello-ray -n ai-app --watch
```

#### Kubeflow – Canalizaciones de ML completas

Automatiza el ciclo de vida de los modelos (ingesta, entrenamiento, evaluación y despliegue) mediante **Kubeflow Pipelines**.

```bash
export PIPELINE_VERSION=2.15.0
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"
kubectl wait --for condition=established \
  --timeout=60s crd/applications.app.k8s.io
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/platform-agnostic?ref=$PIPELINE_VERSION"
kubectl wait pods -l app=ml-pipeline-ui \
  -n kubeflow --for=condition=Ready --timeout=180s
kubectl port-forward svc/ml-pipeline-ui 8080:80 -n kubeflow
```

*Figura 5.12: Panel de Kubeflow Pipelines en kind*

Definición de pipeline en Python (`pipeline.py`):

```python
from kfp import dsl

@dsl.component(base_image="python:3.11-slim")
def preprocess(data_path: str, output_path: str):
    # In a real pipeline, this would clean and split your dataset
    print(f"Preprocessing {data_path} -> {output_path}")

@dsl.component(base_image="python:3.11-slim")
def train(data_path: str, model_output: str):
    # In a real pipeline, this would run your training loop
    print(f"Training on {data_path}, saving to {model_output}")

@dsl.pipeline(name="simple-ml-pipeline")
def ml_pipeline(data_path: str = "/data/raw"):
    preprocess_task = preprocess(
        data_path=data_path,
        output_path="/data/processed"
    )
    train_task = train(
        data_path=preprocess_task.output,
        model_output="/models/output"
    )

if __name__ == "__main__":
    from kfp import compiler
    compiler.Compiler().compile(ml_pipeline, "pipeline.yaml")
```

#### KServe – Servicio de modelos en producción

Proporciona escalado a cero (*scale-to-zero*), despliegues canary y servicio multimodelo estandarizado sobre Knative:

```bash
# Instalar cert-manager y KServe
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.0/cert-manager.yaml
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=120s
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.13.0/kserve.yaml
```

Manifiesto de `InferenceService`:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-iris
  namespace: ai-app
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
```

#### MLflow – Registro de modelos y seguimiento de experimentos

Centraliza métricas, hiperparámetros y artefactos de modelos:

```yaml
# Despliegue de servidor de seguimiento de MLflow
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow
  namespace: ai-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mlflow
  template:
    metadata:
      labels:
        app: mlflow
    spec:
      containers:
        - name: mlflow
          image: ghcr.io/mlflow/mlflow:v2.15.0
          command:
            - mlflow
            - server
            - --host=0.0.0.0
            - --port=5000
          ports:
            - containerPort: 5000
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: mlflow
  namespace: ai-app
spec:
  selector:
    app: mlflow
  ports:
    - port: 5000
      targetPort: 5000
EOF
```

*Figura 5.13: Interfaz de usuario de MLflow Tracking*

Registro de métricas en Python:

```python
import mlflow

# Point MLflow at the in-cluster tracking server
mlflow.set_tracking_uri("http://mlflow:5000")
mlflow.set_experiment("chatbot-model-fine-tune")

with mlflow.start_run():
    # Log hyperparameters
    mlflow.log_param("learning_rate", 0.001)
    mlflow.log_param("batch_size", 32)
    mlflow.log_param("epochs", 10)

    # ... your training loop runs here ...

    # Log final metrics
    mlflow.log_metric("eval_loss", 0.42)
    mlflow.log_metric("eval_accuracy", 0.91)

    # Save the model artefact
    mlflow.log_artifact("model.pkl")
```

#### Resumen comparativo de herramientas

| Herramienta | Problema que resuelve / Cuándo utilizarla |
| :--- | :--- |
| **KubeRay** | Entrenamiento e inferencia distribuida con Ray, ajuste fino multigpu y búsqueda de hiperparámetros |
| **Kubeflow Pipelines** | Automatización de flujos de trabajo completos de ML reproducibles de extremo a extremo |
| **KServe** | Servicio de modelos a gran escala con escalado a cero y despliegues canary |
| **MLflow** | Seguimiento de experimentos, registro de métricas y versionado/promoción de modelos |

*Tabla 5.3: Cuándo usar cada herramienta del ecosistema ML en Kubernetes*

---

### Resumen

En este capítulo ha aprendido a desplegar y escalar cargas de trabajo de ML en **Kubernetes**:

- **Kind integrado en Docker Desktop**: Proporciona un clúster multinodo local completo sin dependencias externas.
- **Traducción directa desde Compose**: Cada servicio se convierte en un par Deployment/Service, la configuración se extrae a ConfigMaps/Secrets y el almacenamiento a PVCs.
- **Ajuste de sondas y recursos**: Las sondas de readiness configuradas con `/v1/models` y tiempos adecuados de `initialDelaySeconds` evitan falsos reinicios y fallos en cascada.
- **Escalado horizontal (HPA)**: Permite ajustar réplicas automáticamente ante picos de demanda.
- **Ecosistema avanzado**: Herramientas como KubeRay, Kubeflow, KServe y MLflow amplían las capacidades para escenarios de entrenamiento distribuido y pipelines complejos.

En el próximo capítulo, exploraremos la integración basada en protocolos con **Model Context Protocol (MCP)** para conectar modelos de IA con herramientas externas y APIs de forma segura.

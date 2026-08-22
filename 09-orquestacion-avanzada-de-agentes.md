# Parte 3: Construcción de Sistemas Inteligentes y Autónomos

## Capítulo 9: Orquestación Avanzada de Agentes

En el [Capítulo 7](https://subscription.packtpub.com/book/cloud-and-networking/9781807301095/10), construyó agentes autónomos de forma manual: escribiendo controladores en Python, configurando Redis para la memoria y enlazando archivos de Docker Compose con docenas de líneas de infraestructura. En el [Capítulo 8](https://subscription.packtpub.com/book/cloud-and-networking/9781807301095/11), orquestó múltiples modelos y agentes en sistemas colaborativos. Esos enfoques funcionan y comprenderlos es fundamental. Sin embargo, representaban una gran cantidad de trabajo de infraestructura y fontanería (*plumbing*) para lo que debería ser una tarea directa: poner en marcha agentes, mantenerlos seguros y permitirles colaborar.

*Figura 9.1: Exceso de configuración e infraestructura manual previa*

En este capítulo cubriremos los siguientes temas principales:

- Seguridad en la ejecución de agentes mediante **Docker Sandboxes**
- Equipos declarativos de agentes con **Docker Agent**
- Orquestación nativa de Kubernetes con **kagent**
- Patrones de despliegue en producción y mejores prácticas

Al final de este capítulo, sabrá cómo elegir la herramienta de orquestación adecuada para cada escenario: desde un único agente de programación ejecutándose de forma segura en su portátil, hasta una flota de agentes especializados procesando cargas de trabajo en un clúster de Kubernetes.

---

### Requisitos técnicos

Para seguir los ejemplos de este capítulo, necesitará las siguientes herramientas y configuraciones:

**Software:**
- **Docker Desktop 4.58 o superior**: Requerido para **Docker Sandboxes** con soporte para microVMs. Verifique con `docker version`.
- **Docker Agent CLI**: Instale mediante Homebrew (`brew install docker-agent`) o desde los lanzamientos de GitHub.
- **kubectl y kind**: Para las secciones de Kubernetes y kagent. Instale kind desde [https://kind.sigs.k8s.io](https://kind.sigs.k8s.io/).
- **Helm 3.x**: Para instalar kagent en Kubernetes.
- **Claves de API (al menos una requerida para Docker Agent)**: OpenAI, Anthropic o Google Gemini. También puede utilizar **Docker Model Runner** como alternativa gratuita y local.

**Hardware:**
- 16 GB de RAM recomendados (ejecutar sandboxes, LLMs y Kubernetes simultáneamente consume recursos significativos).
- 20 GB de espacio libre en disco para imágenes, microVMs y modelos.

**Conocimientos previos:**
- Se basa en los Capítulos 7 y 8 (bucle del agente, patrones de aislamiento y comunicación multiagente).

Los ejemplos de código están disponibles en el repositorio oficial en `chap-09`.

---

### Seguridad en la ejecución de agentes mediante Docker Sandboxes

#### Por qué el aislamiento con microVMs es crucial para agentes de código

*Figura 9.2: Docker Sandboxes y aislamiento mediante microVMs*

Los agentes de programación (como Claude Code, Gemini CLI o Codex) no se limitan a invocar APIs: instalan paquetes npm/pip, reescriben archivos del sistema, ejecutan scripts arbitrarios y construyen imágenes de contenedores mediante Docker. Otorgarles acceso directo a su sistema operativo host expondría claves SSH, credenciales y código confidencial. Un comando destructivo como `rm -rf` o código malicioso generado por el modelo podría comprometer su entorno local o cadena de suministro.

*Figura 9.3: Arquitectura de Docker Sandboxes*

Aunque los contenedores estándar ofrecen aislamiento, comparten el kernel del host y ejecutar Docker dentro de Docker (*Docker-in-Docker*) suele exigir modo privilegiado (`--privileged`), lo cual anula el aislamiento de seguridad.

**Docker Sandboxes** resuelve esto ejecutando cada agente dentro de una **microVM ligera dedicada**:
- Dispone de su **propio kernel independiente**, proporcionando una barrera de seguridad de hardware estricta.
- Contiene su **propio demonio de Docker**, permitiendo construir y ejecutar contenedores de forma aislada sin tocar el Docker del host.
- Cuenta con **listas de control de red (allow/deny lists)** para limitar exactamente los dominios a los que puede acceder el agente.

#### Primeros pasos con Docker Sandboxes

Verifique que Docker Sandboxes esté disponible:

```bash
docker sandbox --help
```

Configure su clave de API de forma global en su shell (el demonio de sandbox requiere variables de entorno en el inicio del sistema, por lo que debe reiniciar Docker Desktop después):

```bash
# For Claude Code
echo 'export ANTHROPIC_API_KEY=sk-ant-your-key-here' >> ~/.zshrc
source ~/.zshrc
# Restart Docker Desktop so the daemon picks up the new variable
```

Crear y ejecutar un sandbox en el directorio del proyecto:

```bash
# Create a sandbox for Claude Code in your project directory
docker sandbox run claude ~/my-project
```

Dentro del sandbox, Claude Code se inicia con `--dangerously-skip-permissions` habilitado de forma segura, ya que está completamente confinado en su propia máquina virtual. Su directorio de trabajo se sincroniza bidireccionalmente con la misma ruta absoluta.

#### Gestión de Sandboxes

Los sandboxes no aparecen en `docker ps` porque son microVMs, no contenedores simples. Se gestionan con comandos específicos:

```bash
# List all sandboxes
docker sandbox ls

# Run a command inside a sandbox
docker sandbox exec claude-my-project -- ls -la

# Open an interactive shell for debugging
docker sandbox exec -it claude-my-project -- bash

# Stop a sandbox (preserves state)
docker sandbox stop claude-my-project

# Reconnect to a stopped sandbox
docker sandbox run claude-my-project

# Remove a sandbox permanently
docker sandbox rm claude-my-project
```

#### Aislamiento de red (*Network Isolation*)

Por defecto, el sandbox puede conectarse a cualquier dominio. Puede restringir la conectividad aplicando el principio de mínimo privilegio:

```bash
# Deny all network access by default
docker sandbox network proxy claude-my-project --policy deny

# Allow specific domains the agent needs
docker sandbox network proxy claude-my-project \
  --allow api.anthropic.com \
  --allow github.com \
  --allow registry.npmjs.org \
  --allow pypi.org
```

La configuración se almacena en `~/.docker/sandboxes/vm/<vm-name>/proxy-config.json`.

#### Ejecución de Docker dentro del Sandbox

Dentro de la microVM, el agente puede compilar imágenes y levantar pilas de Compose de forma transparente:

```bash
docker build -t my-app .
docker run -d -p 8080:8080 my-app
docker compose up --build
```

Ninguna de estas operaciones afecta al Docker daemon del host.

#### Agentes soportados y montajes de múltiples espacios de trabajo

```bash
# Claude Code
docker sandbox run claude ~/my-project

# Gemini CLI
docker sandbox run gemini ~/my-project

# Codex CLI
docker sandbox run codex ~/my-project

# Mount multiple workspaces (read-only for reference docs)
docker sandbox run claude ~/my-project ~/docs:ro ~/shared-libs:ro
```

#### Cuándo usar Sandboxes vs. Contenedores

| Característica | Aislamiento por Contenedores (Capítulo 7) | Docker Sandboxes |
| :--- | :--- | :--- |
| **Nivel de aislamiento** | Namespaces / cgroups | MicroVM completa |
| **Kernel** | Compartido con el host | Dedicado e independiente por sandbox |
| **Caso de uso principal** | Agentes de servicios de larga duración | Agentes de programación / desarrollo interactivo |
| **Ciclo de vida** | Servicios persistentes | Entornos desechables |
| **Control de red** | Redes de Docker | Listas Allow / Deny |
| **Sobrecarga (*Overhead*)** | Mínima | ~200 MB por microVM |

---

### Equipos declarativos de agentes con Docker Agent

#### De lo imperativo a lo declarativo

En los capítulos anteriores, construir un sistema multiagente requería ~9 archivos y más de 400 líneas de código (Python, Dockerfiles, Compose, Redis, variables de entorno). **Docker Agent** permite definir todo el equipo de agentes, herramientas y modelos en **un único archivo YAML**.

| Componente manual (Capítulo 7) | Equivalente en Docker Agent |
| :--- | :--- |
| Docker Model Runner + Código de API | Campo `model:` |
| MCP Gateway + Llamadas a herramientas | Sección `toolsets:` |
| Redis + Persistencia de estado | Herramienta integrada `memory` (SQLite) |
| Bucle del agente en Python | Runtime automático de Docker Agent |

#### Instalación y configuración de Docker Agent

```bash
# macOS
brew install docker-agent

# Windows
winget install docker.docker-agent

# Verify installation
docker agent version
```

Configurar al menos un proveedor:

```bash
# Option 1: OpenAI
export OPENAI_API_KEY=sk-your-key-here

# Option 2: Anthropic
export ANTHROPIC_API_KEY=sk-ant-your-key-here

# Option 3: Google Gemini
export GEMINI_API_KEY=your-key-here
```

#### Su primer agente declarativo

Crear `agents.yaml`:

```yaml
agents:
  root:
    model: openai/gpt-4.1-mini
    description: A helpful assistant that talks like a pirate
    instruction: |
      You are a helpful assistant that talks like a pirate. Always respond with "Arrr!" and use pirate slang.
```

Ejecutar el agente:

```bash
docker agent run agents.yaml
```

#### Conjuntos de herramientas integrados (*Built-in Toolsets*)

Docker Agent incluye 6 toolsets nativos:
- `filesystem`: Lectura, escritura y edición de archivos.
- `shell`: Ejecución de comandos del sistema.
- `memory`: Almacenamiento persistente local mediante SQLite (sin necesidad de contenedores Redis adicionales).
- `think`: Bloque estructurado de razonamiento para planificar antes de actuar.
- `todo`: Gestión y seguimiento de tareas pendientes.
- `environment`: Acceso a variables de entorno.

Ejemplo de agente de revisión de código:

```yaml
agents:
  root:
    model: anthropic/claude-sonnet-4-5
    description: Reviews code and suggests improvements
    instruction: |
      You are an expert code reviewer. When given a codebase:
      1. Use filesystem to read the project structure
      2. Create a todo list of files to review
      3. Use think to analyze patterns and issues
      4. Write a review summary using filesystem
      5. Use memory to track which files you've reviewed
    toolsets:
      - type: filesystem
      - type: shell
      - type: memory
        path: ./review-memory.db
      - type: think
      - type: todo
```

Ejecutar:

```bash
docker agent run agents.yaml
```

#### Asignación multimodelo y modelos locales

```yaml
agents:
  root:
    model: openai/gpt-4.1-mini
    description: Fast triage of incoming requests
    instruction: |
      Classify incoming requests as: bug, feature, question, or other.
      Be fast and concise. For complex issues, delegate to deep-analyzer.
    sub_agents: [deep-analyzer]
    
  deep-analyzer:
    model: anthropic/claude-sonnet-4-5
    description: Deep analysis of complex issues
    instruction: |
      Perform thorough analysis of technical issues.
      Consider edge cases, security implications, and performance.
    toolsets:
      - type: think
      - type: filesystem
```

Uso de modelos locales mediante Docker Model Runner:

```yaml
models:
  local-llm:
    provider: dmr
    model: ai/llama3.2:3B-Q8_0

agents:
  root:
    model: local-llm
    description: Runs entirely on local hardware
    instruction: Analyze files in the current directory.
    toolsets:
      - type: filesystem
```

Configuración avanzada de parámetros de inferencia:

```yaml
models:
  creative-writer:
    provider: openai
    model: gpt-4.1-mini
    temperature: 0.9
  precise-coder:
    provider: anthropic
    model: claude-sonnet-4-5
    temperature: 0.1
  deep-thinker:
    provider: anthropic
    model: claude-sonnet-4-5
    thinking_budget: 8192

agents:
  root:
    model: creative-writer
    description: Creative writing assistant
    instruction: You are a creative writer. Be imaginative and expressive.

  coder:
    model: precise-coder
    description: Precise coding assistant
    instruction: You are an expert coder. Be exact, minimal, and correct.
    toolsets:
      - type: filesystem
      - type: shell

  analyst:
    model: deep-thinker
    description: Deep technical analyst
    instruction: Perform thorough analysis. Think through edge cases and implications.
    toolsets:
      - type: think
```

#### Construcción de equipos multiagente

*Figura 9.4: Jerarquía de equipos multiagente y sub-agentes en Docker Agent*

Docker Agent soporta delegación jerárquica mediante `sub_agents` y transferencias completas (*handoffs*).

Ejemplo de equipo de resolución de bugs (`bug-investigator.yaml`):

```yaml
agents:
  root:
    model: anthropic/claude-sonnet-4-5
    description: Lead investigator that coordinates bug fixes
    instruction: |
      You lead bug investigations. When a bug is reported:
      1. Analyze the bug report
      2. Delegate code investigation to the code-analyst
      3. Delegate fix implementation to the fixer
      4. Review the fix and provide final assessment
    toolsets:
      - type: think
      - type: todo
    sub_agents: [code-analyst, fixer]

  code-analyst:
    model: anthropic/claude-sonnet-4-5
    description: Investigates code to find root causes
    instruction: |
      Analyze the codebase to find the root cause of bugs.
      Read relevant files, trace execution paths, and report findings.
    toolsets:
      - type: filesystem
      - type: think

  fixer:
    model: anthropic/claude-sonnet-4-5
    description: Implements bug fixes
    instruction: |
      Implement fixes for bugs identified by the code-analyst.
      Make minimal, targeted changes and explain what you changed.
    toolsets:
      - type: filesystem
      - type: shell
```

Ejecutar:

```bash
docker agent run bug-investigator.yaml
```

#### Integración con herramientas externas mediante MCP

```yaml
agents:
  root:
    model: openai/gpt-4.1-mini
    description: Researches topics using web search and GitHub
    instruction: |
      Research the given topic thoroughly.
      Search the web for recent information.
      Check GitHub for relevant repositories and code.
    toolsets:
      - type: think
      - type: memory
        path: ./research-memory.db
      - type: mcp
        ref: docker:duckduckgo
      - type: mcp
        ref: docker:github-official
```

Restringir permisos en servidores MCP:

```yaml
    toolsets:
      - type: mcp
        ref: docker:github-official
        tools: [search_repositories, get_file_contents, list_issues]
        # Agent can only search, read files, and list issues
        # Cannot create issues, push code, or delete anything
```

Integración con servidores autenticados con OAuth:

```yaml
    toolsets:
      - type: mcp
        ref: docker:slack
      - type: mcp
        ref: docker:google-drive
```

#### Distribución de agentes como artefactos OCI en Docker Hub

```bash
# Push an agent team to Docker Hub
docker agent share push agents.yaml docker.io/yourusername/research-team:v1.0

# Pull and run someone else's agent
docker agent share pull docker.io/someuser/code-reviewer:latest
docker agent run code-reviewer.yaml
```

#### Ejecución de agentes como servidores de API y herramientas MCP

```bash
# Run as an HTTP API server
docker agent serve api agents.yaml

# Specify a custom port
docker agent serve api agents.yaml --listen :8080

# Other services can now call your agent:
# POST http://localhost:8080/v1/chat/completions

# Run as an MCP server (your agent becomes a tool for other agents):
docker agent serve mcp agents.yaml
```

#### Generación de agentes con IA

```bash
docker agent new "Create an agent that reviews pull requests, checks for security issues, and suggests improvements"
```

---

### Orquestación nativa de Kubernetes con kagent

*Figura 9.5: Arquitectura de agentes nativa de Kubernetes con kagent*

Cuando se requiere procesar miles de solicitudes por hora a través de múltiples nodos físicos con autoescalado, autorrecuperación y observabilidad distribuida, **kagent** convierte a los agentes en recursos nativos de Kubernetes.

#### Comparativa: Docker Compose vs. Kubernetes + kagent

| Capacidad | Docker Compose | Kubernetes y kagent |
| :--- | :--- | :--- |
| **Escalado** | Flag manual `--scale` | HPA automático según métricas de uso |
| **Recuperación** | `restart: on-failure` | Reemplazo y recreación automática de Pods |
| **Distribución** | Solo un único nodo/host | Distribuido en clúster multinodo |
| **Descubrimiento** | DNS interno de Compose | DNS de clúster, Services e Ingress |
| **Secretos** | Archivos `.env` en texto plano | Kubernetes Secrets encriptados en reposo |
| **Observabilidad** | Configuración manual | Integración nativa con Prometheus y Jaeger |
| **Despliegues** | Caída breve en reinicios | Despliegues continuos sin tiempo de inactividad |

#### Configuración de un clúster local con kind

```bash
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

Verificar el clúster:

```bash
kubectl get nodes
```

Salida esperada:
```text
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   60s   v1.31.0
kind-worker          Ready    <none>          40s   v1.31.0
kind-worker2         Ready    <none>          40s   v1.31.0
```

#### Instalación de kagent con Helm

```bash
# Install the CRDs chart first (required):
helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --namespace kagent \
  --create-namespace

# Install kagent with your API key:
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --namespace kagent \
  --set providers.default=openAI \
  --set providers.openAI.apiKey=$OPENAI_API_KEY

# Verify the installation:
kubectl get pods -n kagent
```

> **Resolución de problemas (kagent 0.8.1):**
> Si observa estados `ErrImagePull` o `ImagePullBackOff`, aplique los parches a las imágenes oficiales en `ghcr.io` y `postgres:17`:
> ```bash
> for deploy in $(kubectl get deployments -n kagent -o name); do
>   kubectl patch $deploy -n kagent --type='json' \
>     -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"ghcr.io/kagent-dev/kagent/app:0.8.1"}]' \
>     2>/dev/null && echo "Patched $deploy"
> done
> 
> # Patch infrastructure components:
> kubectl set image deployment/kagent-controller controller=ghcr.io/kagent-dev/kagent/controller:0.8.1 -n kagent
> kubectl set image deployment/kagent-ui ui=ghcr.io/kagent-dev/kagent/ui:0.8.1 -n kagent
> kubectl set image deployment/kagent-postgresql postgres=docker.io/library/postgres:17 -n kagent
> ```

Instalación del CLI y acceso a la interfaz web:

```bash
# macOS
brew install kagent

# Verify
kagent version

# Access the dashboard via port-forward:
kubectl -n kagent port-forward service/kagent-ui 8080:8080
```

Abra [http://localhost:8080](http://localhost:8080) en su navegador.

#### Definición de Agentes con CRDs de kagent

kagent introduce tres **Custom Resource Definitions**:
1. `Agent`: Define el agente, modelo, instrucciones y herramientas.
2. `ModelConfig`: Configuración y parámetros del proveedor LLM.
3. `Tool / Toolset`: Definición de servidores MCP y herramientas disponibles.

Ejemplo de manifiesto (`devops-assistant.yaml`):

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: devops-assistant
  namespace: kagent
spec:
  description: "Kubernetes DevOps assistant"
  type: Declarative
  declarative:
    modelConfig: default-model-config
    systemMessage: |
      You are a DevOps expert. Help with Kubernetes troubleshooting, deployment issues, and infrastructure questions. Be concise and practical.
```

Aplicar e invocar:

```bash
kubectl apply -f devops-assistant.yaml

kagent invoke -t "What pods are running in the kagent namespace?" --agent k8s-agent
```

#### Construcción de un equipo multiagente en Kubernetes

Manifiesto del equipo de DevOps (`02-ops-team/ops-team.yaml`):

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: ops-coordinator
  namespace: kagent
spec:
  description: "Coordinates DevOps troubleshooting"
  type: Declarative
  declarative:
    modelConfig: default-model-config
    systemMessage: |
      You coordinate troubleshooting across specialized agents.
      Delegate log analysis to log-analyzer.
      Delegate metrics investigation to metrics-checker.
      Synthesize findings into actionable recommendations.
---
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: log-analyzer
  namespace: kagent
spec:
  description: "Analyzes Kubernetes logs for issues"
  type: Declarative
  declarative:
    modelConfig: default-model-config
    systemMessage: |
      Analyze Kubernetes pod logs to identify errors, warnings, and anomalies.
      Report findings concisely.
    tools:
      - type: McpServer
        mcpServer:
          name: kagent-tool-server
          kind: RemoteMCPServer
          toolNames:
            - k8s_get_pod_logs
            - k8s_get_events
---
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: metrics-checker
  namespace: kagent
spec:
  description: "Checks cluster metrics and resource usage"
  type: Declarative
  declarative:
    modelConfig: default-model-config
    systemMessage: |
      Monitor Kubernetes cluster metrics including CPU, memory, and network usage.
      Identify resource bottlenecks.
    tools:
      - type: McpServer
        mcpServer:
          name: kagent-tool-server
          kind: RemoteMCPServer
          toolNames:
            - k8s_get_resources
            - k8s_describe_resource
```

Desplegar y probar:

```bash
kubectl apply -f 02-ops-team/

# Comprobar agentes registrados
kagent get agent | grep -E "ops-coordinator|log-analyzer|metrics-checker"

# Invocar al coordinador
kagent invoke -t "Check for any pod errors in the kagent namespace" --agent ops-coordinator
```

#### Despliegue de servidores MCP en Kubernetes con `kmcp`

Instalar la herramienta CLI `kmcp`:

```bash
curl -fsSL https://raw.githubusercontent.com/kagent-dev/kmcp/refs/heads/main/scripts/get-kmcp.sh | bash
```

Desplegar paquetes MCP preconstruidos o imágenes personalizadas:

```bash
# Desplegar paquete MCP desde npm
kmcp deploy package \
  --deployment-name my-mcp-server \
  --manager npx \
  --args @modelcontextprotocol/server-everything \
  --namespace kagent

# Desplegar imagen personalizada
kmcp deploy \
  --file my-mcp-server/kmcp.yaml \
  --image your-registry/custom-mcp:v1.0 \
  --namespace kagent
```

O declarar servidores MCP directamente como recursos de Kubernetes:

```yaml
apiVersion: kagent.dev/v1alpha1
kind: MCPServer
metadata:
  name: mcp-website-fetcher
  namespace: kagent
spec:
  deployment:
    args:
      - mcp-server-fetch
    cmd: uvx
    port: 3000
    stdioTransport: {}
    transportType: stdio
```

Gestión de credenciales con Secrets de Kubernetes:

```bash
kubectl create secret generic mcp-credentials \
  --from-literal=github-token=ghp_your_token \
  --from-literal=slack-token=xoxb-your-token \
  -n kagent
```

#### Autoescalado horizontal (HPA) y optimización de recursos

Definición de HPA para un agente (`log-analyzer-hpa.yaml`):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: log-analyzer-hpa
  namespace: kagent
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: log-analyzer
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

> **Nota para clústeres local kind:**
> kind requiere instalar `metrics-server` para métricas de CPU:
> ```bash
> kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
> kubectl patch deployment metrics-server -n kube-system \
>   --type='json' \
>   -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
> ```

Límites de recursos en el manifiesto del agente:

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: log-analyzer
  namespace: kagent
spec:
  description: "Analyzes Kubernetes logs for issues"
  type: Declarative
  declarative:
    modelConfig: default-model-config
    systemMessage: |
      Analyze Kubernetes pod logs to identify errors, warnings, and anomalies.
      Report findings concisely.
    deployment:
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "2"
          memory: "2Gi"
    tools:
      - type: McpServer
        mcpServer:
          name: kagent-tool-server
          kind: RemoteMCPServer
          toolNames:
            - k8s_get_pod_logs
            - k8s_get_events
```

#### Observabilidad con Prometheus, Grafana y Jaeger Tracing

Instalación de Prometheus y Grafana:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

kubectl get pods -n monitoring
```

Obtener contraseña de Grafana y acceder al dashboard:

```bash
kubectl --namespace monitoring get secrets monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo

kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Despliegue de Jaeger para trazabilidad distribuida (*Distributed Tracing*):

```bash
cat << 'EOF' > jaeger.yaml
provisionDataStore:
  cassandra: false
allInOne:
  enabled: true
storage:
  type: memory
agent:
  enabled: false
collector:
  enabled: false
query:
  enabled: false
EOF

helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update
helm upgrade --install jaeger jaegertracing/jaeger \
  --namespace jaeger \
  --create-namespace \
  --values jaeger.yaml \
  --version 4.4.7
```

Habilitar tracing en kagent:

```bash
helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --namespace kagent \
  --set providers.default=openAI \
  --set providers.openAI.apiKey=$OPENAI_API_KEY \
  --set otel.tracing.enabled=true \
  --set otel.tracing.exporter.otlp.endpoint=http://jaeger.jaeger.svc.cluster.local:4317
```

Acceder a la interfaz de Jaeger en [http://localhost:16686](http://localhost:16686):

```bash
export POD_NAME=$(kubectl get pods --namespace jaeger \
  -l "app.kubernetes.io/instance=jaeger,app.kubernetes.io/component=all-in-one" \
  -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward --namespace jaeger $POD_NAME 16686:16686 &
```

#### Aislamiento de seguridad mediante Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-isolation
  namespace: kagent
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: k8s-agent
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: kagent
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kagent
```

*Figura 9.6: Capacidades de producción de kagent*

---

### Elección de la herramienta de orquestación adecuada

| Escenario | Herramienta | Motivo principal |
| :--- | :--- | :--- |
| **Ejecutar un agente de código de forma segura en un portátil** | Docker Sandboxes | Aislamiento en microVM, entorno desechable y seguro. |
| **Definir un equipo de 2-3 agentes para un proyecto** | Docker Agent | Un solo archivo YAML, sin código de infraestructura. |
| **Compartir configuraciones de agentes con el equipo** | Docker Agent (`push`/`pull`) | Artefactos OCI distribuidos vía Docker Hub. |
| **Prototipado rápido de flujos de trabajo de agentes** | Docker Agent | Ciclos de iteración inmediatos. |
| **Ejecutar flujos de agentes en producción a gran escala** | kagent | Autoescalado HPA, autorrecuperación y observabilidad. |
| **Despliegue de agentes en clúster multinodo** | kagent | Programación (*scheduling*) y redes de Kubernetes. |
| **Requisitos estrictos de cumplimiento corporativo** | kagent | RBAC, Network Policies y registro de auditoría. |

---

### Mejores prácticas para la orquestación de agentes en producción

1. **Comience con la herramienta más simple que resuelva el problema**: Si un YAML de Docker Agent es suficiente, no incorpore la sobrecarga de Kubernetes.
2. **Mantenga a los agentes enfocados**: Asigne responsabilidades estrechas y específicas a cada agente. Tres especialistas son más fáciles de depurar y escalar que un agente monolítico.
3. **Empareje modelos con tareas**: Utilice modelos pequeños y rápidos (como GPT-4.1-mini o Llama 3.2 local) para clasificación, enrutamiento y tareas rutinarias; reserve modelos grandes para razonamiento complejo.
4. **Restrinja el acceso a herramientas**: Aplique permisos de mínimo privilegio en los servidores MCP (un agente que solo lee repositorios es más seguro que uno que puede hacer push o eliminar ramas).
5. **Implemente observabilidad desde el primer día**: Utilice las herramientas `think` y `todo` en Docker Agent y despliegue Prometheus y Jaeger en kagent.
6. **Controle las versiones de las configuraciones**: Gestione los archivos YAML de Docker Agent y los manifiestos de Kubernetes en Git y versionelos mediante etiquetas en Docker Hub.

---

### Resumen

En este capítulo ha aprendido a orquestar agentes autónomos en diferentes escalas:

- **Docker Sandboxes**: Proporciona microVMs con kernel dedicado y demonio Docker propio para ejecutar agentes de programación de forma segura sin riesgo para el sistema host.
- **Docker Agent**: Sustituye complejas arquitecturas manuales por definiciones declarativas en un único archivo YAML, gestionando memoria local, herramientas MCP y equipos colaborativos distribuibles en Docker Hub.
- **kagent**: Lleva los agentes al ecosistema de Kubernetes mediante Custom Resource Definitions (CRDs), permitiendo autoescalado con HPA, autorrecuperación, trazabilidad con Jaeger y políticas de aislamiento de red para entornos corporativos a gran escala.

Con estas herramientas, los agentes de IA dejan de ser scripts aislados y se convierten en sistemas estructurados, seguros, reproducibles y listos para producción.

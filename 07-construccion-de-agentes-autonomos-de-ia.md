# Parte 3: Construcción de Sistemas Inteligentes y Autónomos

## Capítulo 7: Construcción de Agentes Autónomos de IA

En el [Capítulo 6](https://subscription.packtpub.com/book/cloud-networking/9781807301095/8), conectó modelos de **IA** a herramientas externas a través del **Model Context Protocol (MCP)**. Claude Desktop podía interactuar con GitHub, consultar bases de datos Postgres y realizar diversas tareas, siempre en respuesta a sus comandos directos. La IA esperaba a que usted hiciera una solicitud y luego ejecutaba lo solicitado.

Pero no todos los sistemas de IA deberían esperar la intervención humana. ¿Qué sucede si necesita sistemas que monitoricen repositorios 24/7, solucionando errores automáticamente a medida que aparecen? ¿O agentes que procesen flujos de datos continuos y tomen decisiones sin intervención humana?

Este capítulo trata sobre la construcción de ese tipo de sistemas: **agentes autónomos de IA** que no se limitan a responder órdenes, sino que persiguen objetivos de forma activa e independiente. Perciben su entorno, razonan sobre lo que sucede, deciden qué hacer, actúan e iteran continuamente. Sin humanos en el bucle (*human-in-the-loop*), a menos que algo falle.

Los agentes autónomos añaden complejidad operativa: necesitará un manejo de errores sofisticado, monitorización exhaustiva y controles de seguridad estrictos. Sin embargo, cuando se requiere un funcionamiento continuo y desatendido, la inversión merece totalmente la pena.

En este capítulo cubriremos los siguientes temas principales:

- Comprensión de los agentes autónomos de IA
- Implementación de aislamiento en contenedores para la seguridad de los agentes
- Construcción de controladores de agentes listos para producción
- Diseño de patrones de comunicación entre agentes
- Configuración de redes de Docker para sistemas multiagente
- Implementación de elementos esenciales de seguridad y monitorización

Al final de este capítulo, tendrá agentes autónomos ejecutándose en contenedores Docker: seguros, aislados y colaborando entre sí para realizar tareas complejas.

---

### Requisitos técnicos

Para seguir los ejemplos de este capítulo, necesitará las siguientes herramientas y cuentas:

**Software:**
- **Docker Desktop 4.42.0 o superior**: Se requieren las funciones de MCP Toolkit y Model Runner para el motor de razonamiento y el acceso a herramientas. Descárguelo desde [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/).
- **Python 3.11 o superior**: Necesario para las implementaciones del controlador y los agentes trabajadores (*workers*). Verifique con `python --version`.
- **Node.js 18 o superior**: Requerido para los ejemplos del motor de razonamiento con integración de LLMs. Verifique con `node --version`.
- **Git**: Para clonar el repositorio de código.
- **curl**: Para probar endpoints de APIs REST y verificar servicios.
- **jq** *(opcional pero recomendado)*: Procesador JSON de línea de comandos para analizar logs estructurados (`brew install jq` en macOS o `apt install jq` en Ubuntu).

**Hardware:**
- Mínimo 8 GB de RAM (16 GB recomendados).
- 10 GB de espacio libre en disco para imágenes de contenedores.

**Cuentas y claves de API:**
- **GitHub Personal Access Token (opcional)**: Para probar el MCP Gateway con herramientas de GitHub (alcance `repo`).

**Conocimientos previos:**
- Se basa en los Capítulos 3 y 6 (Docker Model Runner y MCP Gateway), servicios de Docker Compose, volúmenes, redes y comprobaciones de salud (*health checks*).

Los ejemplos de código están disponibles en el repositorio oficial en `chap-07`.

---

### Comprensión de los agentes autónomos de IA

#### Definición de agentes autónomos

Los sistemas de IA interactivos (como los vistos en los Capítulos 3 y 6) esperan a que un humano envíe un prompt o solicite una acción. En cambio, los **agentes autónomos** operan de forma continua, persiguiendo objetivos sin supervisión humana constante.

**Flujo interactivo tradicional:**
```text
Humano: "Crea una incidencia en GitHub para el error en la autenticación de usuarios"
IA: [Llama a la herramienta create_issue]
IA: "Incidencia #1234 creada"
[El sistema se detiene y espera la siguiente instrucción humana]
```

**Flujo de un agente autónomo:**
```text
[El agente monitoriza el repositorio de GitHub continuamente]
[Aparece una nueva incidencia #1234: "Los usuarios no pueden iniciar sesión"]
Agente: [Lee la incidencia, analiza el código base]
Agente: [Identifica el error en auth.py línea 42]
Agente: [Crea una rama, escribe la corrección, ejecuta las pruebas]
Agente: [Las pruebas pasan - crea el pull request #567]
Agente: [Envía notificación por Slack al equipo]
[El agente continúa monitorizando en busca de la siguiente incidencia]
```

#### La arquitectura del bucle del agente (Agent Loop)

Todo agente autónomo ejecuta un bucle continuo de seis fases:

1. **Percibir (*Perceive*)**: El agente consulta activamente su entorno (consultar incidencias vía API, leer una cola de mensajes, etc.).
2. **Razonar (*Reason*)**: Envía los datos a un LLM para comprender la situación ("Analiza esta incidencia, ¿qué tipo de problema es y cuál es su urgencia?").
3. **Planificar (*Plan*)**: El LLM genera una secuencia de pasos ("Revisar historial de git, leer código de auth, corregir, probar").
4. **Actuar (*Act*)**: Ejecuta el plan invocando herramientas a través del MCP Gateway (leer archivos, crear ramas, confirmar cambios).
5. **Observar (*Observe*)**: Examina los resultados y códigos de retorno para verificar si la acción tuvo éxito.
6. **Iterar (*Iterate*)**: Si tuvo éxito, pasa a la siguiente tarea; si falló, prueba una estrategia alternativa o solicita intervención.

*Figura 7.1: IA Generativa lineal vs. IA Agéntica basada en bucles continuos*

#### Componentes fundamentales del agente

##### Componente 1: Motor de razonamiento (Reasoning Engine)

El motor de razonamiento toma decisiones utilizando un LLM ejecutado mediante Docker Model Runner.

Configuración en `docker-compose.yaml`:

```yaml
services:
  reasoning:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8082:8080"
    environment:
      - PORT=8080
    models:
      - llama
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

models:
  llama:
    model: ai/llama3.2:1B-Q8_0
    context_size: 2048
```

Código de conexión en Node.js:

```javascript
const axios = require('axios');

function getLLMEndpoint() {
  const llamaUrl = process.env.LLAMA_URL;
  return `${llamaUrl}/chat/completions`;
}

function getModelName() {
  return process.env.LLAMA_MODEL;
}

async function callLLM(userMessage) {
  const chatRequest = {
    model: getModelName(),
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: userMessage }
    ]
  };

  const response = await axios.post(
    getLLMEndpoint(),
    chatRequest,
    {
      headers: { 'Content-Type': 'application/json' },
      timeout: 30000
    }
  );

  return response.data.choices[0].message.content.trim();
}
```

Ejecución y prueba:

```bash
cd Operational-AI-with-Docker/chap-07/reasoning
docker compose up --build
```

Salida esperada:
```text
=> => naming to docker.io/library/reasoning-node-genai:latest 0.0s
=> => unpacking to docker.io/library/reasoning-node-genai:latest 0.3s
=> resolving provenance for metadata file 0.0s
[+] up 3/4
 ✓ Image reasoning-node-genai Built 16.8s
 ⠼ llama Configuring 0.3s
 ✓ Network reasoning_default Created 0.0s
 ✓ Container reasoning-node-genai-1 Created 0.2s
Attaching to node-genai-1
node-genai-1 | Server starting on http://localhost:8080
node-genai-1 | Using LLM endpoint: http://model-runner.docker.internal/v1/chat/completions
node-genai-1 | Using model: ai/llama3.2:1B-Q8_0
```

*Figura 7.2: Vista de contenedores en Docker Desktop con el motor de razonamiento*

Probar el endpoint:

```bash
curl -X POST http://localhost:8082/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Explain Docker in simple terms"}'
```

```json
{"response":"Docker in Simple Terms\n\nDocker is a popular containerization platform that allows you to package, ship, and run applications in isolated environments. Here's how it works in simple terms:\n\n**What is a Container?**\n\nA container is a self-contained environment that runs a specific version of an application. It's like a virtual machine, but much faster and more secure.\n\n**How Does Docker Work?**\n\nHere's a step-by-step explanation:\n\n1. **Create a Docker Image**: You create a Docker image by writing a set of instructions.. "}
```

Verificar estado de salud:

```bash
curl http://localhost:8082/health
```

```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "llm_endpoint": "http://model-runner.docker.internal/v1/chat/completions",
  "model": "ai/llama3.2:1B-Q8_0"
}
```

*Figura 7.3: Interfaz web de chat del motor de razonamiento*

##### Componente 2: Capa de acceso a herramientas (Tool Access Layer)

Permite al agente invocar herramientas mediante **Docker MCP Gateway**:

```yaml
services:
  mcp-gateway:
    image: docker/mcp-gateway
    command:
      - --servers=github-official
      - --servers=firecrawl
      - --servers=kubernetes
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "8812:8811"
```

Inicio del gateway:

```bash
cd ../tool-access-layer
docker compose up --build
```

```text
- Listing MCP tools...
- Running ghcr.io/github/github-mcp-server with [run --rm -i --init --security-opt no-new-privileges --cpus 1 --memory 2Gb --pull never -l docker-mcp=true -l docker-mcp-tool-type=mcp -l docker-mcp-name=github-official -l docker-mcp-transport=stdio --network tool-access-layer_default -e GITHUB_PERSONAL_ACCESS_TOKEN]
- Running mcp/kubernetes with [run --rm -i --init...]
- Running mcp/firecrawl with [run --rm -i --init...]
```

```text
- github-official: time=2025-12-27T13:37:26.229Z level=INFO msg="starting server" version=v0.26.3
- github-official: GitHub MCP Server running on stdio
- github-official: time=2025-12-27T13:37:26.241Z level=INFO msg="session initialized"
> github-official: (40 tools) (2 prompts) (5 resourceTemplates)
> firecrawl: (6 tools)
> 46 tools listed in 6.29s
> Initialized in 16.93s
> Start stdio server
```

Invocación de herramientas en el código del agente:

```javascript
const axios = require('axios');

async function createGitHubIssue(owner, repo, title, body) {
  const response = await axios.post(
    "http://mcp-gateway:8811/mcp",
    {
      tool: "create_issue",
      arguments: {
        owner: owner,
        repo: repo,
        title: title,
        body: body
      }
    }
  );
  return response.data.number;
}
```

Pruebas con `curl`:

```bash
# Listar herramientas disponibles
curl http://localhost:8811/tools

# Buscar repositorios en GitHub (solo lectura)
curl -X POST http://localhost:8811/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "search_repositories",
    "arguments": {
      "query": "docker language:go stars:>1000"
    }
  }'
```

##### Componente 3: Memoria y estado persistente (Memory and State)

Sin memoria, el agente repetiría tareas o perdería el contexto ante reinicios. **Redis** proporciona almacenamiento persistente de acciones y estados.

```bash
cd Operational-AI-with-Docker/chap-07/memory-state
```

`docker-compose.yaml`:

```yaml
services:
  # Agente que procesa tareas y recuerda lo realizado
  task-agent:
    build: .
    environment:
      - REDIS_URL=redis://agent-memory:6379
      - AGENT_NAME=task-processor
    depends_on:
      agent-memory:
        condition: service_healthy
    models:
      llm:
        endpoint_var: AI_MODEL_URL
        model_var: AI_MODEL_NAME

  # Redis para memoria persistente
  agent-memory:
    image: redis:7-alpine
    volumes:
      - agent-state:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    ports:
      - "6379:6379"

models:
  llm:
    model: ai/llama3.2:1B-Q8_0
    context_size: 2048
    runtime_flags:
      - "--temp"
      - "0.7"

volumes:
  agent-state:
```

```bash
docker compose up --build
```

*Figura 7.4: Pila de contenedores con Redis y memoria persistente*

Inspección de claves y registros en Redis:

```bash
# Listar todas las claves de memoria de los agentes
docker compose exec agent-memory redis-cli KEYS "agent:*"
```

```text
1) "agent:task-processor:actions:task-003"
2) "agent:task-processor:actions:task-002"
3) "agent:task-processor:actions:task-004"
4) "agent:task-processor:actions:task-001"
```

```bash
# Ver el historial de una tarea específica
docker compose exec agent-memory redis-cli LRANGE agent:task-processor:actions:task-001 0 -1
```

```text
"{\"action\": \"Identify the source of customer feedback data.\", \"result\": \"Successfully completed: Identify the source of customer feedback data.\", \"success\": true, \"timestamp\": \"2025-12-27T14:13:42.205306\"}"
```

##### Componente 4: Controlador del agente (Agent Controller)

Estructura simplificada del bucle:

```python
import time
import requests
import redis

# Initialize connections:
def call_llm(prompt):
    response = requests.post(
        f"{model_url}/chat/completions",
        json={
            "model": model_name,
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": 200
        }
    )
    return response.json()['choices'][0]['message']['content']

memory = redis.Redis(host='redis', port=6379)

def agent_loop():
    """Main agent loop: perceive → reason → plan → act → observe → iterate"""
    while True:
        # Perceive: Check environment
        events = perceive_environment()
        
        if events:
            # Reason: Understand what happened
            analysis = call_llm(f"Analyze: {events}")
            
            # Plan: Decide actions
            plan = call_llm(f"Plan for: {analysis}")
            
            # Act: Execute plan
            results = execute_plan(plan)
            
            # Observe: Check results
            observe_results(results)
            
            # Store in memory
            memory.rpush("actions", json.dumps({
                "events": events,
                "results": results,
                "timestamp": time.time()
            }))
        
        # Iterate: Brief pause before next loop
        time.sleep(10)

if __name__ == "__main__":
    agent_loop()
```

#### Cuándo usar agentes vs. aplicaciones interactivas

| Usar Agentes Autónomos | Mantener Aplicaciones Interactivas |
| :--- | :--- |
| Operaciones continuas 24/7 (monitorización de seguridad, pipelines) | Flujos de trabajo simples de 1 a 3 pasos |
| Flujos multietapa complejos con lógica condicional | Escenarios donde cada decisión requiere aprobación humana |
| Tareas de larga duración (análisis a gran escala) | Operaciones de alto riesgo financiero/crítico |
| Respuesta proactiva inmediata a eventos | Análisis exploratorio con juicio subjetivo |

---

### Implementación de aislamiento en contenedores para la seguridad de los agentes

El aislamiento estricto en contenedores es obligatorio para evitar vulnerabilidades como escalada de privilegios o robo de credenciales compartidas (CVE-2025-24362 ClawHavoc, CVE-2025-24363 ClawJacked).

#### Arquitectura multiagente con aislamiento

```bash
cd Operational-AI-with-Docker/chap-07/container-isolation-tests/multi-agent-architecture
```

Configuración en `docker-compose.yml`:

```yaml
services:
  # Agente 1: Seguimiento de bugs
  bug-tracker:
    build:
      context: .
      dockerfile: Dockerfile.agent
    environment:
      - AGENT_NAME=bug-tracker
      - AGENT_TYPE=monitoring
      - REDIS_URL=redis://agent-memory:6379
    volumes:
      - bug-tracker-state:/data
    depends_on:
      agent-memory:
        condition: service_healthy
    models:
      - llama
    restart: unless-stopped

  # Agente 2: Reporte de estado
  status-reporter:
    build:
      context: .
      dockerfile: Dockerfile.agent
    environment:
      - AGENT_NAME=status-reporter
      - AGENT_TYPE=reporting
      - REDIS_URL=redis://agent-memory:6379
    volumes:
      - reporter-state:/data
    depends_on:
      agent-memory:
        condition: service_healthy
    models:
      - llama
    restart: unless-stopped

  # Memoria compartida para coordinación
  agent-memory:
    image: redis:7-alpine
    volumes:
      - shared-memory:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    ports:
      - "6379:6379"

models:
  llama:
    model: ai/llama3.2:1B-Q8_0
    context_size: 2048

volumes:
  bug-tracker-state:
  reporter-state:
  shared-memory:
```

*Figura 7.6: Arquitectura multiagente en Docker Desktop*

Implementación del agente con reanudación de estado (`agent.py`):

```python
import os
import time
import redis
import json
from datetime import datetime

def main():
    agent_name = os.getenv('AGENT_NAME', 'agent')
    agent_type = os.getenv('AGENT_TYPE', 'worker')
    redis_url = os.getenv('REDIS_URL', 'redis://localhost:6379')

    print(f"🤖 Starting {agent_name} ({agent_type})")
    print(f"   Redis: {redis_url}")

    # Connect to shared memory
    redis_client = redis.from_url(redis_url, decode_responses=True)
    state_key = f"agent:{agent_name}:state"

    # RESUME FROM LAST STATE (production pattern)
    try:
        last_state = redis_client.get(state_key)
        if last_state:
            state_data = json.loads(last_state)
            iteration = state_data.get('iteration', 0) + 1
            print(f"📥 Resuming from iteration {iteration}")
        else:
            iteration = 1
            print(f"🆕 Starting fresh at iteration 1")
    except Exception as e:
        iteration = 1
        print(f"🆕 No previous state, starting at iteration 1")

    while True:
        # Write current state
        state = {
            "agent": agent_name,
            "type": agent_type,
            "iteration": iteration,
            "timestamp": datetime.utcnow().isoformat(),
            "status": "active"
        }
        redis_client.set(state_key, json.dumps(state))
        print(f"✅ {agent_name} iteration {iteration} - state saved to {state_key}")

        # Read other agents' states (shared memory)
        all_keys = redis_client.keys("agent:*:state")
        print(f"   Visible agents: {len(all_keys)}")
        for key in all_keys:
            if key != state_key:
                other_state = json.loads(redis_client.get(key))
                print(f"   - {other_state['agent']}: iteration {other_state['iteration']}")

        iteration += 1
        time.sleep(10)

if __name__ == "__main__":
    main()
```

Verificación del aislamiento y volúmenes:

```bash
docker volume ls | grep multi-agent
```

```text
local     multi-agent-architecture_bug-tracker-state
local     multi-agent-architecture_reporter-state
local     multi-agent-architecture_shared-memory
```

Comprobación de persistencia tras reinicio:

```bash
docker compose restart bug-tracker
docker compose logs bug-tracker --tail 20 -f
```

```text
🤖 Starting bug-tracker (monitoring)
Redis: redis://agent-memory:6379
📥 Resuming from iteration 8
✅ bug-tracker iteration 8 - state saved to agent:bug-tracker:state
Visible agents: 2
- status-reporter: iteration 7
✅ bug-tracker iteration 9 - state saved to agent:bug-tracker:state
```

> **Buenas prácticas de seguridad con Redis ACLs:**
> Para restringir a los agentes a su propio espacio de nombres:
> ```bash
> ACL SETUSER bug-tracker on >password ~agent:bug-tracker:* +@read +@write
> ACL SETUSER status-reporter on >password ~agent:status-reporter:* +@read +@write
> ```

---

### Construcción de controladores de agentes listos para producción

```bash
cd ../agent-controller
```

`docker-compose.yml`:

```yaml
services:
  # Agent Controller - manages all agents
  controller:
    build:
      context: .
      dockerfile: Dockerfile.controller
    ports:
      - "8001:8000" # Mapeo de puerto externo 8001 a interno 8000
    environment:
      - REDIS_URL=redis://agent-memory:6379
      - CONTROLLER_NAME=main-controller
    depends_on:
      agent-memory:
        condition: service_healthy
    models:
      - llm

  # Worker Agent 1 - procesador de datos
  worker-1:
    build:
      context: .
      dockerfile: Dockerfile.agent
    environment:
      - REDIS_URL=redis://agent-memory:6379
      - AGENT_NAME=worker-1
      - AGENT_TYPE=data-processor
      - CONTROLLER_URL=http://controller:8000
    depends_on:
      agent-memory:
        condition: service_healthy
      controller:
        condition: service_started
    models:
      - llm
    restart: on-failure

  # Worker Agent 2 - analista
  worker-2:
    build:
      context: .
      dockerfile: Dockerfile.agent
    environment:
      - REDIS_URL=redis://agent-memory:6379
      - AGENT_NAME=worker-2
      - AGENT_TYPE=analyst
      - CONTROLLER_URL=http://controller:8000
    depends_on:
      agent-memory:
        condition: service_healthy
      controller:
        condition: service_started
    models:
      - llm
    restart: on-failure

  # Redis para memoria compartida
  agent-memory:
    image: redis:7-alpine
    volumes:
      - agent-state:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    ports:
      - "6379:6379"

models:
  llm:
    model: ai/llama3.2:1B-Q8_0
    context_size: 2048

volumes:
  agent-state:
```

*Figura 7.5: Sistema de controlador y workers en Docker Desktop*

Métodos de interacción del worker (`agent.py`):

```python
def register(self):
    """Register with the controller"""
    response = requests.post(
        f"{self.controller_url}/api/agents/register",
        json={"name": self.agent_name, "type": self.agent_type},
        timeout=5
    )
    return response.status_code == 200

def send_heartbeat(self):
    """Send heartbeat to controller"""
    requests.post(
        f"{self.controller_url}/api/agents/{self.agent_name}/heartbeat",
        timeout=5
    )

def get_task(self):
    """Get next task from controller"""
    response = requests.post(
        f"{self.controller_url}/api/tasks/dequeue",
        json={"agent_name": self.agent_name},
        timeout=5
    )
    return response.json() if response.status_code == 200 else None
```

Bucle de ejecución del worker:

```python
def run(self):
    """Main agent loop"""
    self.register()
    heartbeat_counter = 0
    
    while True:
        heartbeat_counter += 1
        
        if heartbeat_counter % 10 == 0:
            self.send_heartbeat()
            
        task = self.get_task()
        if task:
            success, result = self.process_task(task)
            self.complete_task(task['id'], success, result)
            time.sleep(2)
        else:
            print("💤 No tasks available, waiting...")
            time.sleep(5)
```

Prueba de la API REST del controlador:

```bash
# Obtener todos los agentes registrados
curl http://localhost:8001/api/agents
```

```json
[
  {
    "name": "worker-2",
    "type": "analyst",
    "status": "active",
    "tasks_completed": 3,
    "tasks_failed": 0,
    "last_heartbeat": "2025-12-27T16:49:15.816101"
  },
  {
    "name": "worker-1",
    "type": "data-processor",
    "status": "active",
    "tasks_completed": 3,
    "tasks_failed": 0
  }
]
```

```bash
# Asignar una nueva tarea
curl -X POST http://localhost:8001/api/tasks/assign \
  -H "Content-Type: application/json" \
  -d '{"id": "task-123", "description": "Process customer data"}'
```

```bash
# Obtener estadísticas globales
curl http://localhost:8001/api/stats
```

```json
{
  "total_agents": 2,
  "active_agents": 2,
  "inactive_agents": 0,
  "pending_tasks": 0,
  "total_completed": 7,
  "total_failed": 0,
  "success_rate": 100.0
}
```

*Figura 7.7: Panel de control web del controlador en `http://localhost:8001`*

---

### Diseño de patrones de comunicación entre agentes

#### Publicación / Suscripción (Pub/Sub) con Redis

Permite desacoplar emisores y receptores.

```bash
cd chap-07/agent-communication
```

`writer.py` (Emisor):

```python
import redis
import json
import time
import os

def main():
    redis_client = redis.from_url(
        os.getenv('REDIS_URL', 'redis://redis:6379'),
        decode_responses=True
    )
    
    print("📥 Writer Agent Starting...")
    print("   Publishing patches to 'patches' channel\n")
    
    patch_id = 1
    while True:
        # Create patch metadata
        patch_data = {
            "patch_id": f"patch-{patch_id:03d}",
            "location": f"/data/patch-{patch_id:03d}.diff",
            "agent": "bug-fixer",
            "size": 100 + (patch_id * 10)
        }
        
        print(f"📤 Publishing: {patch_data['patch_id']}")
        print(f"   Size: {patch_data['size']} bytes")
        
        # Publish to Redis channel
        redis_client.publish("patches", json.dumps(patch_data))
        
        patch_id += 1
        time.sleep(5)  # Publish every 5 seconds

if __name__ == "__main__":
    main()
```

`reader.py` (Receptor):

```python
import redis
import json
import os

def main():
    redis_client = redis.from_url(
        os.getenv('REDIS_URL', 'redis://redis:6379'),
        decode_responses=True
    )
    
    print("👂 Reader Agent Starting...")
    print("   Listening for patches on 'patches' channel\n")
    
    # Subscribe to the patches channel
    pubsub = redis_client.pubsub()
    pubsub.subscribe("patches")
    print("✅ Subscribed successfully!\n")
    
    # Listen for messages
    for message in pubsub.listen():
        if message['type'] == 'message':
            # Parse the JSON data
            data = json.loads(message['data'])
            print(f"📨 Received: {data['patch_id']}")
            print(f"   Location: {data['location']}")
            print(f"   Size: {data['size']} bytes")
            print(f"   Processing patch...\n")

if __name__ == "__main__":
    main()
```

Escalado de lectores:

```bash
docker compose up --build --scale reader=3
```

Todos los lectores reciben los eventos en paralelo de forma independiente.

---

### Configuración de redes de Docker para sistemas multiagente

Docker Compose ofrece descubrimiento de servicios automático mediante DNS interno.

```bash
cd Operational-AI-with-Docker/chap-07/agent-discovery
```

`coordinator.py`:

```python
import requests
import time
import os

def main():
    worker_url = os.getenv('WORKER_URL', 'http://worker:8000')
    print("🎯 Coordinator Agent Starting...")
    print(f"   Worker URL: {worker_url}\n")
    
    time.sleep(3)  # Wait for workers to be ready
    task_id = 1
    
    while True:
        try:
            print(f"\n📥 Sending task #{task_id} to worker...")
            response = requests.get(f"{worker_url}/process", timeout=5)
            
            if response.status_code == 200:
                data = response.json()
                print(f"✅ Task completed by: {data['worker_id']}")
                print(f"   Hostname: {data['hostname']}")
                print(f"   Message: {data['message']}")
        except Exception as e:
            print(f"❌ Error calling worker: {e}")
            
        task_id += 1
        time.sleep(3)

if __name__ == "__main__":
    main()
```

Escalado dinámico con balanceo de carga automático:

```bash
docker compose up -d --scale worker=3
docker compose logs coordinator -f
```

Docker distribuye las solicitudes entre los 3 contenedores `worker` automáticamente sin cambiar el código del coordinador.

---

### Implementación de seguridad y monitorización esenciales

#### Sandboxing de contenedores

```bash
cd Operational-AI-with-Docker/chap-07/agent-security
```

`docker-compose.yaml`:

```yaml
services:
  secure-agent:
    build: .
    read_only: true           # Sistema de archivos completo de solo lectura
    tmpfs:
      - /tmp                  # Solo /tmp es escribible (en memoria RAM)
    cap_drop:
      - ALL                   # Elimina todas las capacidades de Linux
    security_opt:
      - no-new-privileges:true# Previene la escalada de privilegios
    user: "1000:1000"         # Se ejecuta como usuario no root
    networks:
      - isolated-network

networks:
  isolated-network:
    internal: true            # Sin acceso a redes externas
```

Prueba activa de restricciones:

```bash
# Probar escritura en raíz (debe fallar)
docker compose exec secure-agent touch /test.txt

# Probar escritura en tmpfs (debe funcionar)
docker compose exec secure-agent touch /tmp/test.txt && echo "✅ tmpfs write succeeded"
```

#### Análisis de logs estructurados en JSON

```python
# Formato JSON estructurado implementado en agent.py
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "agent_id": os.getenv("HOSTNAME", "unknown")
        }
        if hasattr(record, 'task_id'):
            log_data['task_id'] = record.task_id
        if hasattr(record, 'duration'):
            log_data['duration'] = record.duration
        return json.dumps(log_data)
```

Consultas con `jq`:

```bash
# Métricas de tareas completadas
docker compose logs secure-agent | grep -o '{.*}' | jq 'select(.message == "Task completed") | {task_id, duration}'

# Advertencias de seguridad
docker compose logs secure-agent | grep -o '{.*}' | jq 'select(.level == "WARNING")'

# Eventos de inicio
docker compose logs secure-agent | grep -o '{.*}' | jq 'select(.message | contains("Starting")) | {timestamp, agent_id, message}'
```

---

### Resumen

En este capítulo ha aprendido los fundamentos de los **agentes autónomos de IA** contenerizados:

- **Bucle continuo del agente**: El ciclo Percibir → Razonar → Planificar → Actuar → Observar → Iterar permite la ejecución desatendida.
- **Cuatro pilares centrales**: Motor de razonamiento (DMR), capa de acceso a herramientas (MCP Gateway), memoria persistente (Redis) y controlador de orquestación.
- **Aislamiento en contenedores**: Evita la contaminación cruzada y limita el radio de impacto de vulnerabilidades mediante sistemas de archivos de solo lectura, `cap_drop: ALL`, usuarios no root y redes internas.
- **Coordinación y escalado**: Integración mediante REST APIs, Pub/Sub en Redis y balanceo de carga automático basado en DNS con Docker Compose.

En el próximo capítulo ([Capítulo 8](08-arquitecturas-multimodelo-y-multiagente.md)), ampliaremos estos conceptos hacia **arquitecturas multimodelo y multiagente**, asignando modelos especializados a diferentes roles dentro de un sistema colaborativo.

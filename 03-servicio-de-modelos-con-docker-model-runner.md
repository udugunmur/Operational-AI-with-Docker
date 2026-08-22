# Parte 2: Operacionalización de Modelos de IA

## Capítulo 3: Servicio de Modelos con Docker Model Runner

Construir e integrar modelos de **IA** en aplicaciones puede ser todo un desafío. Es necesario gestionar archivos de modelos de gran tamaño, asegurarse de que la inferencia se ejecute de manera eficiente en el hardware disponible e integrar los endpoints del modelo en la pila tecnológica de la aplicación, todo ello mientras se controlan los costos y la privacidad de los datos. **Docker Model Runner (DMR)** es la solución de Docker para estos desafíos, permitiendo la inferencia de modelos de lenguaje grandes (**LLMs**) con un enfoque *local-first* y con mínimas complicaciones.

En este capítulo, nos centraremos en cómo servir modelos de **IA** como un servicio utilizando **DMR**, asumiendo que ya está familiarizado con los conceptos básicos de Docker. Comenzamos explorando qué es Docker Model Runner, incluida su arquitectura y cómo facilita la inferencia de LLMs. A continuación, repasamos la instalación y configuración de DMR en Docker Desktop (macOS/Windows) y Docker Engine (Linux), cubriendo la habilitación de GPU y las comprobaciones de diagnóstico. Luego ejecutará su primer modelo de IA descargándolo de un registro, ejecutándolo localmente y verificando la inferencia tanto a través de la **CLI** como de una llamada a la API simple. El capítulo continúa con el servicio de modelos y la integración de APIs, demostrando cómo exponer modelos a través de una API compatible con OpenAI e interactuar con ellos utilizando herramientas como `curl` y los SDKs de OpenAI. También examinamos cómo construir aplicaciones de IA con DMR utilizando Docker Compose para vincular modelos a los servicios de la aplicación, inyectar variables de entorno para los endpoints de los modelos y aplicar patrones de integración del mundo real. La optimización del rendimiento y la configuración de GPU se discuten en detalle, incluyendo el ajuste fino para velocidad, la selección de motores de inferencia para CPU vs GPU, el ajuste de la longitud de contexto, la aplicación de niveles de cuantización y el aprovechamiento de GPUs para aceleración. Finalmente, cubrimos las prácticas de observabilidad y monitorización, como la implementación de registros (*logging*), recolección de métricas con Prometheus y Grafana, y trazado con Jaeger para monitorizar el rendimiento y el uso del modelo, antes de concluir con ejercicios prácticos que le guiarán en el despliegue de DMR tanto en escenarios de desarrollo como de producción.

En este capítulo cubriremos los siguientes temas principales:

- Introducción a Docker Model Runner
- Instalación y configuración de Docker Model Runner
- Ejecución de tu primer modelo de IA
- Servicio de modelos e integración de APIs
- Construcción de aplicaciones de IA utilizando Docker Model Runner
- Optimización del rendimiento y configuración de GPU
- Observabilidad y monitorización

Al final de este capítulo, comprenderá cómo tratar los modelos de IA como servicios de primer nivel en su entorno de Docker. Utilizaremos ejemplos concretos y configuraciones a lo largo del camino para que pueda aplicar estos conceptos a sus propios proyectos. Comencemos explorando qué es Docker Model Runner y por qué cambia las reglas del juego para el despliegue local de modelos de IA.

---

### Requisitos técnicos

Para trabajar con los ejemplos de este capítulo, asegúrese de cumplir con los siguientes requisitos previos:

**Software:**
- **Docker Desktop 4.40+** (macOS, Windows 10/11) o **Docker Engine** (Linux)
- **Docker Model Runner** (disponible a través de Docker Desktop o como paquete `docker-model-plugin` en Linux)

**Características:**
- Docker Model Runner habilitado y accesible mediante la CLI de Docker (comandos `docker model`)
- *(Opcional)* Soporte para GPU habilitado para inferencia acelerada (si está disponible)

**Entorno:**
- Docker Desktop o Docker Engine instalado y configurado
- Acceso a una terminal con la CLI de Docker disponible

---

### Introducción a Docker Model Runner

Tras revisar esta sección, podrá explicar la arquitectura de **DMR** a nivel de sistema, incluidos el ejecutor (*runner*), el motor de inferencia y el almacén de modelos, y mapear estos componentes a los flujos de trabajo de desarrollo del mundo real. También estará capacitado para elegir el motor de inferencia adecuado, como `llama.cpp` o `vLLM`, en función de su plataforma, formato de modelo y requisitos de rendimiento. Además, podrá evaluar cuándo DMR es una mejor opción que las APIs basadas en la nube considerando factores como la privacidad de los datos, la previsibilidad de la latencia, el control de costos y la necesidad de entornos de desarrollo sin conexión.

Docker Model Runner, o **DMR**, es una herramienta nativa de Docker para ejecutar modelos de **IA** localmente. Integra la ejecución de modelos en el ecosistema de Docker, lo que le permite descargar, ejecutar y administrar modelos utilizando comandos y flujos de trabajo familiares de Docker.

El objetivo principal es permitir a los desarrolladores ejecutar LLMs y otros modelos de IA en sus propias máquinas con control total, en lugar de depender de APIs en la nube. Esto significa que puede evitar costos de inferencia en la nube y proteger datos confidenciales manteniendo la inferencia en su propio hardware.

#### Características clave de DMR

DMR trata a los modelos como artefactos **OCI**, de forma similar a las imágenes de contenedores que se pueden almacenar en registros como **Docker Hub** o **Hugging Face** y descargar bajo demanda. Proporciona una API REST compatible con OpenAI para inferencia, de modo que sus aplicaciones puedan utilizar modelos locales con cambios mínimos en el código.

También admite aceleración por GPU de forma nativa; si dispone de una GPU adecuada, DMR puede aprovecharla para una inferencia más rápida. De manera crucial, DMR se integra con Docker Compose, la interfaz de usuario de Docker Desktop y otras herramientas de Docker para que los modelos de IA formen parte integral de su flujo de trabajo de desarrollo.

#### Arquitectura y componentes

Por debajo, la arquitectura de Docker Model Runner consta de tres componentes principales:

1. **Backend de Model Runner (*Model Runner Backend*)**: El servicio central (a veces llamado `model-runner`) que administra y ejecuta la inferencia del modelo. Este es un proceso nativo del host y **no un contenedor**, que carga los archivos del modelo y ejecuta el motor de inferencia para cada uno. Ejecutarse de forma nativa le permite utilizar directamente los recursos del sistema (CPUs, GPUs) sin la sobrecarga de los contenedores, lo que permite un mayor rendimiento. El backend incluye un programador/cargador (*scheduler/loader*) que gestiona la carga de modelos en memoria bajo demanda cuando llegan solicitudes y los descarga tras un período de inactividad para liberar recursos. También incluye un subsistema de instalación que puede obtener binarios adicionales (como librerías específicas de GPU o motores alternativos) cuando sea necesario.
2. **Plugin CLI de Modelos (`docker model`)**: Una extensión de la CLI de Docker (`docker-model`) que proporciona comandos para interactuar con los modelos. Funciona de manera muy similar a los comandos `docker image` o `docker container`, pero para modelos, de modo que puede descargar modelos, listarlos, definir configuraciones, ejecutarlos, etc. La CLI es lo que utiliza en su terminal. Internamente, esta CLI se comunica con el backend de Model Runner a través de llamadas a la API para realizar operaciones. Al ser un plugin estándar de la CLI de Docker, respeta automáticamente los contextos de Docker y funciona sin problemas en diferentes entornos (Desktop vs Engine).
3. **Distribución y almacenamiento de modelos**: DMR introduce una especificación de distribución de modelos (a veces denominada *model-spec*), que define cómo se empaquetan los archivos de modelo como artefactos **OCI** y cómo se almacenan. Cuando descarga un modelo (por ejemplo, `ai/mymodel:tag`), los archivos del modelo se descargan y se almacenan en una caché de modelos local. Este almacenamiento es independiente del almacenamiento de imágenes de Docker porque los archivos de modelos pueden ser gigantescos y tienen requisitos específicos; a menudo se mapean en memoria (*memory-mapped*) para mayor eficiencia. El componente de distribución de modelos gestiona las interacciones con los registros, por lo que `docker model pull` sabe cómo obtenerlos de Docker Hub o HuggingFace. Básicamente, es análogo a la distribución de imágenes de Docker, pero adaptado a modelos de IA con soporte para metadatos de modelos, versionado y más.

*Figura 3.1: Arquitectura de Docker Model Runner*

#### Cómo funciona la inferencia

Una vez que un modelo se descarga y se almacena en la caché local, el backend de Model Runner genera un proceso del motor de inferencia adecuado cuando usted utiliza el modelo. Notablemente, DMR admite actualmente dos backends de inferencia listos para usar: `llama.cpp` y `vLLM`.

- **llama.cpp**: Por defecto, utiliza `llama.cpp`, un motor de LLM eficiente en C++ que se ejecuta en CPU (con descarga opcional a GPU) y admite archivos de modelos cuantizados (formato GGUF). `llama.cpp` es excelente para el desarrollo local y modelos más pequeños porque es ligero y eficiente en recursos.
- **vLLM**: La alternativa es `vLLM`, un motor optimizado para servicio en GPU (NVIDIA) de alto rendimiento utilizando modelos en formato Safetensors. DMR puede enrutar solicitudes de inferencia a diferentes motores según el formato del modelo o la configuración. Está diseñado para ser flexible.

De hecho, el soporte para múltiples backends fue un objetivo de diseño deliberado: DMR comenzó con `llama.cpp`, pero se diseñó arquitectónicamente para incorporar motores adicionales como `vLLM` y potencialmente otros (ONNX, PyTorch, etc.) con el tiempo.

#### API compatible con OpenAI

Una de las características más destacadas de DMR es la exposición de una API RESTful que imita los endpoints de la API de OpenAI. El backend de Model Runner ejecuta un servidor API que acepta solicitudes de chat completions, completions y embeddings tal como lo hace la API en la nube de OpenAI.

Su aplicación puede enviar una solicitud JSON y obtener una respuesta en el mismo formato que recibiría del servicio de OpenAI, incluidos campos como `id`, `choices` y `usage`. Esta compatibilidad reduce significativamente el esfuerzo de integración; si su aplicación utiliza el SDK de OpenAI o llamadas `curl`, puede apuntarla al endpoint local de DMR y funcionará con cambios mínimos o nulos en el código.

#### Ciclo de vida del modelo

Con DMR, los modelos son ciudadanos de primera clase en el entorno de Docker. Puede listar modelos (`docker model ls`), inspeccionar su información (`docker model inspect`), configurar parámetros (como longitud de contexto o flags de rendimiento) y eliminarlos (`docker model rm`) de manera muy similar a como gestionaría imágenes de contenedores.

*Figura 3.2: Ciclo de vida de Docker Model Runner: Descargar (Pull), Ejecutar (Run), Servir (Serve) y Gestionar (Manage)*

Los modelos se cargan en memoria solo cuando es necesario: por ejemplo, si ejecuta un prompt contra un modelo, el runner lo carga; tras un tiempo de inactividad, puede descargarlo para liberar memoria. DMR pondrá en cola las solicitudes si es necesario en lugar de devolver un error. Por ejemplo, si envía solicitudes concurrentes que exceden la memoria disponible, las serializará o cargará y descargará modelos de forma intermedia, en lugar de devolver errores HTTP 503.

---

### Instalación y configuración de Docker Model Runner

La configuración de Docker Model Runner depende de su entorno. Existen dos escenarios principales:

1. **Docker Desktop (Windows/macOS)**: DMR viene incluido como una opción en Docker Desktop (v4.40+). Solo necesita habilitarlo.
2. **Docker Engine en Linux**: DMR está disponible como un paquete de plugin CLI (`docker-model-plugin`) para Docker Engine (v24+). Se instala mediante el gestor de paquetes de su distribución.

#### Habilitación de DMR en Docker Desktop

Si utiliza Docker Desktop, asegúrese de tener la versión 4.41 o posterior. La habilitación se realiza a través de la configuración de Docker Desktop:

1. Abra las preferencias/configuración de Docker Desktop. Navegue a la sección **AI** (en Docker Desktop 4.46+ hay una pestaña dedicada de configuración de AI; en 4.45 y versiones anteriores, Model Runner estaba bajo "Features in development" o "Beta features").
2. Active la opción **Enable Docker Model Runner**. Además, active el soporte **Host-side TCP support**. Docker puede solicitarle reiniciar para aplicar el cambio; hágalo si es necesario.
3. *(Opcional)* Si tiene una GPU NVIDIA compatible y desea utilizarla, active también la inferencia respaldada por GPU (*GPU-backed inference*) en la configuración de AI. Esto permitirá a DMR descargar y utilizar el backend de inferencia habilitado para GPU (`vLLM` o `llama.cpp` acelerado por GPU).

Una vez habilitado, Docker Desktop mostrará una nueva pestaña **Models** en la interfaz del panel de control. También tendrá acceso a los comandos CLI `docker model` en su terminal:

```bash
docker model version
```

Si DMR está habilitado correctamente, esto mostrará la versión de Model Runner, confirmando que el plugin está instalado.

#### Instalación de DMR en Docker Engine (Linux)

En un servidor Linux o máquina de desarrollo que ejecute Docker Engine, Docker Model Runner se distribuye como un paquete de plugin CLI:

**Para Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install docker-model-plugin
```

**Para distribuciones basadas en RPM (Fedora, RHEL, CentOS):**
```bash
sudo dnf update
sudo dnf install docker-model-plugin
```

Esto descargará e instalará el plugin de CLI en el directorio de plugins de Docker (normalmente `/usr/lib/docker/cli-plugins/`). Tras la instalación, verifíquelo:

```bash
docker model version
docker model --help
```

Puede realizar una prueba rápida:

```bash
docker model run ai/smollm2 "Hello"
```

> **Nota de solución de problemas:**
> Si aparece el mensaje `docker: 'model' is not a docker command`, la CLI de Docker no está encontrando el plugin. Esto ocurre si el plugin no está en el directorio esperado. La solución es crear un enlace simbólico en el directorio de plugins CLI de Docker (`~/.docker/cli-plugins/`).

#### Actualización y habilitación de soporte para GPU

En Docker Desktop, las actualizaciones de DMR se incluyen con las versiones de Docker Desktop. En Docker Engine, puede actualizar mediante el gestor de paquetes (`apt-get upgrade docker-model-plugin`). También existe un comando para reinstalar limpiamente el runner:

```bash
docker model uninstall-runner --images && docker model install-runner
```

**Para soporte de GPU en Linux:**
Instale manualmente el backend de GPU:

```bash
docker model install-runner --backend vllm --gpu cuda
```

Este comando descarga e instala el motor `vLLM` con soporte CUDA. Verifique que su GPU sea accesible ejecutando `nvidia-smi` en el host.

---

### Ejecución de tu primer modelo de IA

En esta sección, veremos cómo descargar un modelo y realizar una inferencia de prueba tanto a través de la CLI como de la API.

#### Descubrimiento y descarga de un modelo

Docker Hub alberga un catálogo curado de modelos de IA de código abierto en el espacio de nombres `ai` (y también puede descargar desde Hugging Face Hub). Utilizaremos `ai/smollm2:360M-Q4_K_M`, un modelo optimizado para instrucciones con 360 millones de parámetros, cuantizado a 4 bits (**Q4_K_M**).

*Figura 3.3: Lista de modelos disponibles en Docker Hub para su uso con Docker Model Runner*

Descarga del modelo desde la terminal:

```bash
docker model pull ai/smollm2:360M-Q4_K_M
```

Confirmar que el modelo está listado localmente:

```bash
docker model ls
```

#### Ejecución del modelo a través de la CLI

Invoque el modelo directamente desde la terminal con un prompt:

```bash
docker model run ai/smollm2:360M-Q4_K_M "Hello, how are you?"
```

La primera vez que lo ejecute, DMR iniciará el motor de inferencia (`llama.cpp` para este modelo GGUF) en segundo plano y cargará el modelo en la memoria RAM. Luego imprimirá la respuesta:

```text
> Hello, how are you?
I'm just a machine, but I'm functioning normally! How can I assist you today?
```

#### Servicio del modelo y realización de solicitudes a la API

Cuando ejecuta el modelo, el backend de Model Runner inicia un servidor HTTP en el puerto `12434`. Por defecto en Docker Desktop, debe asegurarse de habilitar el acceso TCP del host (*Host-side TCP support*).

Simulemos una llamada HTTP utilizando `curl`:

```bash
curl http://localhost:12434/engines/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "ai/smollm2:360M-Q4_K_M",
    "messages": [
      {"role": "user", "content": "Explain Docker in simple terms."}
    ]
  }'
```

Respuesta JSON devuelta:

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "ai/smollm2:360M-Q4_K_M",
  "choices": [{
    "index": 0,
    "message": {
      "role": "assistant",
      "content": "Docker is a platform that allows developers to package applications into containers. These containers bundle the application's code with all its dependencies, ensuring it runs the same everywhere. In simple terms, Docker helps create a consistent environment so an app can run reliably on different computers."
    },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 18,
    "completion_tokens": 56,
    "total_tokens": 74
  }
}
```

La estructura confirma que DMR emula fielmente la API de OpenAI sin necesidad de claves de API ni costos de tokens externos.

---

### Servicio de modelos e integración de APIs

Docker Model Runner expone dos conjuntos de endpoints REST: endpoints estilo OpenAI para inferencia y endpoints estilo Docker para gestión de modelos.

*Figura 3.4: API compatible con OpenAI de Docker Model Runner e integración con SDKs*

**Endpoints principales compatibles con OpenAI:**
- `POST /v1/chat/completions`: Endpoint de finalización de chat (chat completions).
- `POST /v1/completions`: Endpoint de finalización estándar (para modelos de instrucciones clásicos).
- `POST /v1/embeddings`: Generación de incrustaciones vectoriales (*vector embeddings*).
- `GET /v1/models`: Lista de modelos disponibles y descargados.

#### Uso de la API con SDKs de OpenAI

**Python (OpenAI Python SDK):**
```python
import openai

openai.api_base = "http://localhost:12434/engines/v1" # Base URL for local DMR
openai.api_key = "not-used" # DMR doesn't require a key, but the SDK might want one set

response = openai.ChatCompletion.create(
    model="ai/smollm2:360M-Q4_K_M",
    messages=[{"role": "user", "content": "What is Docker Compose?"}]
)
print(response["choices"][0]["message"]["content"])
```

**Node.js (Paquete npm `openai`):**
```javascript
const { Configuration, OpenAIApi } = require("openai");

const configuration = new Configuration({
    apiKey: "unused",
    basePath: "http://model-runner.docker.internal/engines/v1" // for Docker Desktop containers
});
const openai = new OpenAIApi(configuration);

const completion = await openai.createChatCompletion({
    model: "ai/smollm2:360M-Q4_K_M",
    messages: [{ role: "user", content: "What is Docker Compose?" }]
});
console.log(completion.data.choices[0].message.content);
```

> **Nota de red:**
> `model-runner.docker.internal` es el nombre DNS especial proporcionado por Docker Desktop para que los contenedores puedan comunicarse con el Model Runner en el host. Si la aplicación se ejecuta directamente en el host, utilice `localhost`.

#### Configuración de parámetros del modelo a través de la API

- **Temperature (`temperature`)**: Controla la aleatoriedad (0.0 para determinismo, 1.0 para creatividad).
- **Max Tokens (`max_tokens`)**: Limita la cantidad máxima de tokens generados en la respuesta.
- **Top-p / Top-k**: Control de muestreo por núcleo (*nucleus sampling*).
- **Stop Sequences (`stop`)**: Secuencias de texto que detienen la generación.

#### Gestión del endpoint en aplicaciones

Patrón recomendado mediante variables de entorno:

```bash
LLM_API_BASE=http://localhost:12434/engines/v1
LLM_MODEL=ai/smollm2:360M-Q4_K_M
```

---

### Construcción de aplicaciones de IA utilizando Docker Model Runner

Docker Compose introduce la clave superior `models:` en el formato de archivo de Compose para declarar dependencias de modelos de IA de forma nativa.

#### Modelos en Compose: Definición en compose.yaml

**Ejemplo básico (Sintaxis corta):**

```yaml
services:
  chat-app:
    image: my-chat-app:latest
    models:
      - llm # referring to a model named "llm"

models:
  llm:
    model: ai/smollm2:360M-Q4_K_M
```

Al ejecutar `docker compose up`, Compose:
1. Descarga la imagen de la aplicación (`my-chat-app:latest`).
2. Descarga el modelo (`ai/smollm2:360M-Q4_K_M`).
3. Inicia el backend de Model Runner y carga el modelo.
4. Inyecta las variables de entorno `LLM_URL` y `LLM_MODEL` en el contenedor `chat-app`.

**Múltiples modelos por servicio:**

```yaml
services:
  app:
    image: my-app
    models:
      - llm
      - embedding-model

models:
  llm:
    model: ai/smollm2:360M-Q4_K_M
  embedding-model:
    model: all-minilm-l6-v2-vllm
```

**Sintaxis extendida con variables personalizadas:**

```yaml
services:
  app:
    image: my-app
    models:
      llm:
        endpoint_var: AI_MODEL_URL
        model_var: AI_MODEL_NAME
      embedding-model:
        endpoint_var: EMBEDDING_URL
        model_var: EMBEDDING_NAME

models:
  llm:
    model: ai/smollm2
  embedding-model:
    model: all-minilm-l6-v2-vllm
```

#### Configuración de parámetros de modelos en Compose

```yaml
models:
  llm:
    model: ai/smollm2
    context_size: 4096
    runtime_flags:
      - "--threads"
      - "8"
      - "--temp"
      - "0.7"
      - "--top-p"
      - "0.9"
```

**Ejemplo avanzado:**

```yaml
models:
  llm:
    model: ai/qwen2.5-coder
    context_size: 8192
    runtime_flags:
      - "--no-prefill-assistant"
      - "--threads"
      - "16"
```

#### Consejos de integración para producción

- **Fallback elegante (*Graceful fallback*)**: Si `LLM_URL` no está presente, conmute hacia una API en la nube.
- **Respuestas en streaming**: DMR admite streaming de tokens (`stream: true`).
- **Manejo de errores**: Gestione respuestas HTTP 500 o timeouts ante picos de uso de memoria.

---

### Optimización del rendimiento y configuración de GPU

#### Elección del motor de inferencia adecuado

- **llama.cpp (GGUF, cuantizado)**: Optimizado para CPU con descarga opcional a GPU. Máxima eficiencia de memoria, ideal para desarrollo y hardware estándar.
- **vLLM (Safetensors, FP16/BF16)**: Optimizado para GPU NVIDIA (CUDA). Máximo rendimiento y procesamiento por lotes (*batching*). Requiere mayor cantidad de VRAM dedicada.

*Figura 3.5: Cuándo elegir qué motor de inferencia: llama.cpp y vLLM en Docker Model Runner*

#### Configuración de GPU

- **Docker Desktop (Windows con WSL2 / macOS)**: En Windows, habilite "GPU-backed inference" en la configuración de AI para usar CUDA. En macOS (Apple Silicon), Docker Model Runner utiliza aceleración mediante Metal y el framework de virtualización de Apple.
- **Linux (Docker Engine)**: Requiere controladores NVIDIA instalados y la instalación del backend con `docker model install-runner --backend vllm --gpu cuda`.
- **Descarga parcial a GPU en llama.cpp**: Utilice `--n-gpu-layers <N>` para cargar solo una parte de las capas del modelo en la VRAM de la GPU y mantener el resto en la CPU.

#### Longitud de contexto y tamaño del modelo

- **Ventana de contexto**: No configure ventanas más grandes de lo necesario (ej. 8k si solo necesita 1k); el cálculo de autoatención y la caché KV aumentan drásticamente el consumo de memoria.
- **Estrategias de cuantización**: `Q4_K_M` ofrece un balance excelente entre tamaño y calidad; `Q8` proporciona precisión cercana a 16 bits con la mitad de tamaño de archivo.
- **Hilos (*Threads*)**: Ajuste `--threads` al número de núcleos físicos de su CPU para evitar contención de ancho de banda de memoria.

---

### Observabilidad y monitorización

#### Registro e inspección de solicitudes

- **Logs de Model Runner**: Ejecute `docker model logs -f` para observar la carga/descarga de modelos y tiempos de respuesta.
- **Interfaz gráfica de Docker Desktop**: La pestaña **Requests** dentro de Models muestra el payload completo, tokens generados, latencia y velocidad de generación (tokens/segundo).

#### Métricas con Prometheus y Grafana

Instrumente su backend con librerías como `prometheus_client` para exponer métricas clave:
- Tasa de solicitudes y total de peticiones.
- Latencia de inferencia (histogramas).
- Conteo de tokens procesados y generados.
- Velocidad de procesamiento (tokens por segundo).

*Figura 3.6: Panel de Prometheus mostrando métricas de la aplicación para comprobar solicitudes activas*

#### Trazado distribuido con Jaeger

Integre OpenTelemetry en su aplicación para generar trazas que midan la duración exacta de las llamadas HTTP hacia DMR, aislando cuellos de botella entre el código de la aplicación y el motor de inferencia.

---

### Resumen

En este capítulo ha aprendido a convertir los modelos de IA en servicios gestionados dentro del ecosistema Docker mediante **Docker Model Runner**:

- **Arquitectura de DMR**: El backend del runner ejecuta de forma nativa los motores de inferencia (`llama.cpp` y `vLLM`) comunicándose con la CLI y el almacén de modelos OCI.
- **API estándar**: La compatibilidad completa con OpenAI permite usar SDKs existentes en Python, Node.js u otros lenguajes apuntando a `localhost:12434` o `model-runner.docker.internal`.
- **Integración declarativa**: La directiva `models:` en Docker Compose permite empaquetar aplicaciones completas y sus modelos asociados en un solo comando `docker compose up`.
- **Alineación de rendimiento y observabilidad**: La cuantización, la asignación de GPU y la monitorización con Prometheus/Grafana permiten optimizar los recursos locales de manera predecible.

En el próximo capítulo, exploraremos **Docker Offload**, una solución para delegar tareas pesadas de compilación y cómputo de IA a la nube sin alterar su flujo de trabajo local.

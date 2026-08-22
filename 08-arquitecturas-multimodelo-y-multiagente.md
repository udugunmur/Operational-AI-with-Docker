# Parte 3: Construcción de Sistemas Inteligentes y Autónomos

## Capítulo 8: Arquitecturas Multimodelo y Multiagente

En el [Capítulo 7](https://subscription.packtpub.com/book/cloud-and-networking/9781807301095/10), construyó agentes que podían utilizar herramientas y tomar decisiones. Sin embargo, en todos los ejemplos se utilizó el mismo modelo para todo. Aunque esto funciona, no siempre es la opción más inteligente ni económica. Piénselo de esta manera: no contrataría a un neurocirujano para cortar el césped, ni le pediría a un jardinero que realice una cirugía. Diferentes trabajos requieren diferentes herramientas. El mismo principio se aplica a los modelos de **IA**.

Este capítulo trata sobre el uso estratégico de los modelos. Aprenderá cuándo utilizar un modelo pequeño y rápido y cuándo recurrir a modelos más grandes y potentes. Construiremos sistemas donde múltiples modelos y agentes colaboran, cada uno realizando la tarea en la que destaca, ahorrando costes significativos en el proceso.

En este capítulo cubriremos los siguientes temas principales:

- Comprensión de la trampa del modelo único y su impacto en los costes
- Construcción de un enrutador basado en complejidad (*Complexity-based router*)
- Construcción de un asistente de investigación multiagente
- Escalado de sistemas multiagente para tráfico de producción

Al final de este capítulo, dispondrá de tres sistemas funcionales: un enrutador inteligente que asigna tareas al modelo adecuado, un asistente de investigación multiagente que realiza búsquedas web en tiempo real y sistemas escalados capaces de manejar tráfico de producción.

---

### Requisitos técnicos

Para seguir los ejemplos de este capítulo, necesitará las siguientes herramientas y cuentas:

**Software:**
- **Docker Desktop 4.42.0 o superior**: Se requieren las funciones de Model Runner y ejecución multimodelo. Descárguelo desde [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/).
- **Python 3.11 o superior**: Requerido para el enrutador y los agentes implementados con Flask. Verifique con `python --version`.
- **Git**: Para clonar el repositorio de código.
- **curl**: Para probar endpoints de las APIs.

**Hardware:**
- Mínimo 16 GB de RAM (32 GB recomendados). Ejecutar múltiples modelos simultáneamente (granite-nano, granite-micro, qwen3) junto con cuatro agentes requiere memoria considerable.
- 15 GB de espacio libre en disco para imágenes y pesos de modelos.

**Cuentas y claves de API:**
- **Clave de API de Firecrawl (requerida para el asistente de investigación)**: Regístrese en [https://firecrawl.dev](https://firecrawl.dev/) para obtener una API key gratuita.

**Conocimientos previos:**
- Se basa en los Capítulos 3, 6 y 7 (Docker Model Runner, MCP Gateway y patrones de agentes autónomos).

Los ejemplos de código están disponibles en el repositorio oficial en `chap-08`.

---

### Comprensión de la trampa del modelo único y su impacto en los costes

#### Por qué falla el enfoque de "un solo modelo para todo"

Muchas organizaciones cometen el error de enrutar todas las solicitudes a través de un único modelo de frontera costoso (por ejemplo, Claude Opus o GPT-4). Responder a preguntas sencillas como "¿Cuál es la política de devoluciones?" con un modelo de gran escala genera tres problemas graves:

*Figura 8.1: La trampa del modelo único: falta de especialización, costos inflados y dependencia de un único proveedor*

1. **Dinero desperdiciado en tareas simples**: Los modelos gigantes ofrecen la misma respuesta que un modelo pequeño para tareas directas, pero costando entre 10 y 50 veces más por token.
2. **Respuestas lentas (Latencia innecesaria)**: Un modelo de 30B+ parámetros puede tardar 850 ms en responder, mientras que un modelo pequeño de 1B a 3B parámetros responde en 100 ms.
3. **Dependencia y bloqueo de proveedor (*Vendor lock-in*)**: Centralizar todo en una sola API expone al sistema a caídas, límites de tasa (*rate limits*) o incrementos de precios.

#### Comprensión del tamaño de los modelos

*Figura 8.2: Analogía de las calculadoras para los diferentes tamaños de modelos*

- **Modelos pequeños (p. ej., `granite-nano`, 1-3B parámetros)**: Responden en ~100 ms. Ideales para categorización, formateo, extracción de entidades y preguntas frecuentes simples.
- **Modelos medianos (p. ej., `granite-micro`, 7-15B parámetros)**: Responden en ~200 ms. Ideales para resúmenes, síntesis de texto, razonamiento intermedio y redacción de contenido.
- **Modelos grandes (p. ej., `qwen3`, 8B-32B+ parámetros)**: Responden en ~850 ms. Diseñados para planificación compleja, razonamiento abstracto y resolución de problemas difíciles en varios pasos.

#### Dos enfoques de enrutamiento

1. **Enrutamiento basado en tareas (*Task-based routing*)**: Dirige la solicitud según el dominio (código a modelos de programación, imágenes a modelos de visión, texto general a LLMs de lenguaje).
2. **Enrutamiento basado en complejidad (*Complexity-based routing*)**: Evalúa la dificultad de la consulta (consultas simples van al modelo nano, intermedias al micro, y complejas al modelo grande).

---

### Construcción de un enrutador basado en complejidad

#### Medición heurística de la complejidad

```python
def calculate_complexity(prompt):
    """Score from 0 (simple) to 1 (complex)"""
    score = 0.0
    
    # Length matters
    word_count = len(prompt.split())
    if word_count > 50:
        score += 0.3
    elif word_count > 20:
        score += 0.2
        
    # Questions about multiple things are harder
    question_words = ['what', 'why', 'how', 'when', 'where', 'compare']
    questions = sum(1 for word in question_words if word in prompt.lower())
    if questions > 1:
        score += 0.3
        
    # Technical terms indicate complexity
    technical_terms = ['architecture', 'implement', 'design', 'analyze', 'optimize', 'integrate', 'configure']
    if any(term in prompt.lower() for term in technical_terms):
        score += 0.2
        
    # Explicit complexity markers
    if 'complex' in prompt.lower() or 'detailed' in prompt.lower():
        score += 0.2
        
    return min(score, 1.0)  # Cap at 1.0
```

#### Decisión de enrutamiento

```python
def route_by_complexity(prompt):
    """Pick a model based on prompt complexity"""
    complexity = calculate_complexity(prompt)
    
    if complexity < 0.3:
        # Simple question - use the smallest, fastest model
        return {
            'model_url': os.getenv('NANO_MODEL_URL'),
            'model_name': os.getenv('NANO_MODEL'),
            'reason': 'simple query'
        }
    elif complexity < 0.7:
        # Medium complexity - use mid-size model
        return {
            'model_url': os.getenv('MICRO_MODEL_URL'),
            'model_name': os.getenv('MICRO_MODEL'),
            'reason': 'moderate complexity'
        }
    else:
        # Complex question - use the big model
        return {
            'model_url': os.getenv('LARGE_MODEL_URL'),
            'model_name': os.getenv('LARGE_MODEL'),
            'reason': 'high complexity'
        }
```

En tráfico real, el 60-70% de las consultas se procesan en el modelo pequeño, el 20-30% en el mediano y solo el 5-10% en el modelo grande, permitiendo reducciones de costos del 80% al 90%.

---

### Construcción de un asistente de investigación multiagente

#### Arquitectura del sistema

El asistente de investigación se compone de cuatro servicios especializados que se coordinan mediante APIs HTTP y **Redis**:

*Figura 8.3: Arquitectura del asistente de investigación multiagente*

1. **Coordinador (*Coordinator*)**: Recibe la pregunta y genera las consultas de búsqueda necesarias (`qwen3`).
2. **Buscador (*Searcher*)**: Ejecuta las búsquedas en la web mediante la API de **Firecrawl** (`granite-4.0-nano`).
3. **Analizador (*Analyzer*)**: Extrae ideas e información estructurada de los resultados crudos (`granite-4.0-micro`).
4. **Redactor (*Writer*)**: Sintetiza los hallazgos en un informe coherente de 2 a 3 párrafos (`granite-4.0-micro`).

#### Estructura del proyecto

```bash
mkdir -p research-assistant/{coordinator,searcher,analyzer,writer}
cd research-assistant
```

#### Configuración de Docker Compose (`docker-compose.yaml`)

```yaml
services:
  coordinator:
    build: ./coordinator
    ports:
      - "8080:8080"
    environment:
      - SEARCHER_URL=http://searcher:8081
      - ANALYZER_URL=http://analyzer:8082
      - WRITER_URL=http://writer:8083
      - REDIS_URL=redis://redis:6379
    depends_on:
      - searcher
      - analyzer
      - writer
      - redis
    models:
      coordinator-model:
        endpoint_var: MODEL_URL
        model_var: MODEL_NAME

  searcher:
    build: ./searcher
    expose:
      - "8081"
    environment:
      - FIRECRAWL_API_KEY=${FIRECRAWL_API_KEY}
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    models:
      searcher-model:
        endpoint_var: MODEL_URL
        model_var: MODEL_NAME

  analyzer:
    build: ./analyzer
    expose:
      - "8082"
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    models:
      analyzer-model:
        endpoint_var: MODEL_URL
        model_var: MODEL_NAME

  writer:
    build: ./writer
    expose:
      - "8083"
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    models:
      writer-model:
        endpoint_var: MODEL_URL
        model_var: MODEL_NAME

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

models:
  coordinator-model:
    model: ai/qwen3
    context_size: 8192
  searcher-model:
    model: ai/granite-4.0-nano
    context_size: 2048
  analyzer-model:
    model: ai/granite-4.0-micro
    context_size: 4096
  writer-model:
    model: ai/granite-4.0-micro
    context_size: 4096

volumes:
  redis-data:
```

#### Implementación de los agentes

##### Coordinador (`coordinator/app.py`)

```python
from flask import Flask, request, jsonify
import requests
import redis
import os
import uuid
import json

app = Flask(__name__)

SEARCHER_URL = os.getenv('SEARCHER_URL')
ANALYZER_URL = os.getenv('ANALYZER_URL')
WRITER_URL = os.getenv('WRITER_URL')
REDIS_URL = os.getenv('REDIS_URL')
MODEL_URL = os.getenv('MODEL_URL')
MODEL_NAME = os.getenv('MODEL_NAME')

redis_client = redis.from_url(REDIS_URL)

@app.route('/api/research', methods=['POST'])
def research():
    data = request.json
    question = data.get('question')
    
    # Generate a unique ID for this research task
    research_id = str(uuid.uuid4())
    
    # Store the question in Redis
    redis_client.hset(research_id, "question", question)
    
    # Step 1: Plan the research (what should we search for?)
    queries = plan_research(question)
    redis_client.hset(research_id, "queries", json.dumps(queries))
    
    # Step 2: Search the web
    search_results = search_web(queries, research_id)
    
    # Step 3: Analyze the results
    insights = analyze_results(research_id)
    
    # Step 4: Write the final report
    report = write_report(research_id)
    
    return jsonify({
        "question": question,
        "queries": queries,
        "insights": insights,
        "report": report
    })

def plan_research(question):
    """Ask the coordinator model: what should we search for?"""
    prompt = f"""Given this research question: "{question}"
Generate 3 specific search queries that will help answer it.
Return ONLY a JSON array like: ["query 1", "query 2", "query 3"]"""
    
    response = requests.post(
        f"{MODEL_URL}/v1/chat/completions",
        json={
            "model": MODEL_NAME,
            "prompt": prompt,
            "max_tokens": 200,
            "temperature": 0.7
        },
        timeout=30
    )
    
    if response.status_code == 200:
        result = response.json()['choices'][0]['text'].strip()
        try:
            queries = json.loads(result)
            return queries
        except:
            # If JSON parsing fails, use the original question
            return [question]
    return [question]

def search_web(queries, research_id):
    """Tell the searcher agent to find information"""
    response = requests.post(
        f"{SEARCHER_URL}/api/search",
        json={
            "queries": queries,
            "research_id": research_id
        },
        timeout=60
    )
    if response.status_code == 200:
        return response.json()
    return []

def analyze_results(research_id):
    """Tell the analyzer to extract insights"""
    response = requests.post(
        f"{ANALYZER_URL}/api/analyze",
        json={"research_id": research_id},
        timeout=60
    )
    if response.status_code == 200:
        return response.json().get('insights', [])
    return []

def write_report(research_id):
    """Tell the writer to create the final report"""
    response = requests.post(
        f"{WRITER_URL}/api/write",
        json={"research_id": research_id},
        timeout=60
    )
    if response.status_code == 200:
        return response.json().get('report', '')
    return ''

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

##### Buscador (`searcher/app.py`)

```python
from flask import Flask, request, jsonify
import requests
import redis
import os
import json
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

FIRECRAWL_API_KEY = os.getenv('FIRECRAWL_API_KEY')
REDIS_URL = os.getenv('REDIS_URL')
redis_client = redis.from_url(REDIS_URL)

def search_with_firecrawl(query):
    """Use Firecrawl API to search the web"""
    try:
        response = requests.post(
            "https://api.firecrawl.dev/v1/search",
            headers={
                "Authorization": f"Bearer {FIRECRAWL_API_KEY}",
                "Content-Type": "application/json"
            },
            json={
                "query": query,
                "limit": 5
            },
            timeout=30
        )
        if response.status_code == 200:
            data = response.json()
            logger.info(f"Found {len(data.get('data', []))} results for: {query}")
            return data
        else:
            logger.error(f"Search failed: {response.status_code}")
            return {"data": []}
    except Exception as e:
        logger.error(f"Search error: {str(e)}")
        return {"data": []}

@app.route('/api/search', methods=['POST'])
def search():
    data = request.json
    queries = data.get('queries', [])
    research_id = data.get('research_id')
    all_results = []
    
    for query in queries:
        logger.info(f"Searching: {query}")
        results = search_with_firecrawl(query)
        all_results.append({
            "query": query,
            "results": results.get('data', [])
        })
        
    # Store results in Redis for the analyzer
    if research_id:
        redis_client.hset(research_id, "search_results", json.dumps(all_results))
        
    return jsonify({
        "searches": all_results,
        "total_results": sum(len(r['results']) for r in all_results)
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8081)
```

##### Analizador (`analyzer/app.py`)

```python
from flask import Flask, request, jsonify
import requests
import redis
import os
import json

app = Flask(__name__)
REDIS_URL = os.getenv('REDIS_URL')
MODEL_URL = os.getenv('MODEL_URL')
MODEL_NAME = os.getenv('MODEL_NAME')
redis_client = redis.from_url(REDIS_URL)

@app.route('/api/analyze', methods=['POST'])
def analyze():
    data = request.json
    research_id = data.get('research_id')
    
    # Get search results from Redis
    search_results_json = redis_client.hget(research_id, "search_results")
    if not search_results_json:
        return jsonify({"insights": []})
    search_results = json.loads(search_results_json)
    
    # Build context from search results
    context = ""
    for search in search_results:
        for result in search.get('results', [])[:3]:
            title = result.get('title', 'No title')
            content = result.get('content', result.get('description', ''))[:300]
            context += f"{title}\n{content}\n\n"
            
    # Ask the model to extract insights
    question = redis_client.hget(research_id, "question")
    prompt = f"""Question: {question}
Search results: {context}
Extract 3-5 key insights that answer the question.
Return ONLY a JSON array: ["insight 1", "insight 2", ...]"""

    response = requests.post(
        f"{MODEL_URL}/v1/chat/completions",
        json={
            "model": MODEL_NAME,
            "prompt": prompt,
            "max_tokens": 500,
            "temperature": 0.7
        },
        timeout=30
    )
    
    insights = []
    if response.status_code == 200:
        result = response.json()['choices'][0]['text'].strip()
        try:
            insights = json.loads(result)
        except:
            insights = ["Unable to extract insights"]
            
    # Store insights in Redis
    redis_client.hset(research_id, "insights", json.dumps(insights))
    return jsonify({"insights": insights})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8082)
```

##### Redactor (`writer/app.py`)

```python
from flask import Flask, request, jsonify
import requests
import redis
import os
import json

app = Flask(__name__)
REDIS_URL = os.getenv('REDIS_URL')
MODEL_URL = os.getenv('MODEL_URL')
MODEL_NAME = os.getenv('MODEL_NAME')
redis_client = redis.from_url(REDIS_URL)

@app.route('/api/write', methods=['POST'])
def write():
    data = request.json
    research_id = data.get('research_id')
    
    # Get the question and insights from Redis
    question = redis_client.hget(research_id, "question")
    insights_json = redis_client.hget(research_id, "insights")
    if not insights_json:
        return jsonify({"report": "No insights found"})
    insights = json.loads(insights_json)
    insights_text = "\n".join([f"- {insight}" for insight in insights])
    
    # Generate the report
    prompt = f"""Question: {question}
Key findings: {insights_text}
Write a clear, well-structured research report (2-3 paragraphs) that answers the question using these findings."""

    response = requests.post(
        f"{MODEL_URL}/completions",
        json={
            "model": MODEL_NAME,
            "prompt": prompt,
            "max_tokens": 800,
            "temperature": 0.7
        },
        timeout=30
    )
    
    report = ""
    if response.status_code == 200:
        report = response.json()['choices'][0]['text'].strip()
        
    # Store the final report
    redis_client.hset(research_id, "report", report)
    return jsonify({"report": report})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8083)
```

##### Dockerfile común

Crear en cada directorio (`coordinator/Dockerfile`, `searcher/Dockerfile`, `analyzer/Dockerfile`, `writer/Dockerfile`):

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir flask requests redis
COPY app.py .
CMD ["python", "app.py"]
```

#### Pruebas del sistema

```bash
export FIRECRAWL_API_KEY="your_key_here"
docker compose up --build
```

Enviar una consulta de investigación:

```bash
curl -X POST http://localhost:8080/api/research \
  -H "Content-Type: application/json" \
  -d '{"question": "What are the latest Docker AI developments?"}'
```

---

### Escalado de sistemas multiagente para tráfico de producción

#### El problema del enlace de puertos (*Port Binding*)

Si define servicios con `ports: ["8081:8081"]`, no podrá ejecutar múltiples réplicas (`--scale searcher=3`) debido a colisiones en el host.

**La solución: Utilizar `expose` para servicios internos:**

```yaml
services:
  coordinator:
    build: ./coordinator
    ports:
      - "8080:8080" # Punto de entrada externo accesible desde el host
    environment:
      - SEARCHER_URL=http://searcher:8081

  searcher:
    build: ./searcher
    expose:
      - "8081" # Solo accesible dentro de la red interna de Docker

  analyzer:
    build: ./analyzer
    expose:
      - "8082"

  writer:
    build: ./writer
    expose:
      - "8083"
```

#### Escalado horizontal bajo demanda

```bash
# Escalar a 3 instancias del buscador, 2 del analizador y 2 del redactor
docker compose up -d --scale searcher=3 --scale analyzer=2 --scale writer=2

# Comprobar estado de los contenedores
docker compose ps
```

Salida esperada:
```text
NAME                                STATUS
research-assistant-coordinator-1   Up
research-assistant-searcher-1      Up
research-assistant-searcher-2      Up
research-assistant-searcher-3      Up
research-assistant-analyzer-1      Up
research-assistant-analyzer-2      Up
research-assistant-writer-1        Up
research-assistant-writer-2        Up
research-assistant-redis-1         Up
```

#### Balanceo de carga mediante DNS Round-Robin de Docker

Cuando el coordinador llama a `http://searcher:8081`, el servidor DNS embebido de Docker alterna cíclicamente entre las IPs de `searcher-1`, `searcher-2` y `searcher-3`.

Prueba de concurrencia:

```bash
for i in {1..5}; do
  curl -X POST http://localhost:8080/api/research \
    -H "Content-Type: application/json" \
    -d "{\"question\": \"Research topic $i\"}" &
done
wait
```

Verificar la distribución en los logs:

```bash
docker compose logs searcher-1 searcher-2 searcher-3 | grep "Searching:"
```

#### Rendimiento en el mundo real

| Métrica | Antes de Escalar (1 de c/u) | Tras Escalar (3 searchers, 2 analyzers, 2 writers) |
| :--- | :--- | :--- |
| **Peticiones simultáneas** | 1 (secuencial) | 6 a 8 en paralelo |
| **Tiempo de respuesta** | ~8 segundos | ~8 segundos |
| **Rendimiento (*Throughput*)** | ~7 req/minuto | ~40 a 50 req/minuto (mejora de 6-7x) |

---

### Resumen

En este capítulo ha aprendido a diseñar y desplegar **arquitecturas multimodelo y multiagente**:

- **Enrutamiento por complejidad**: Evalúa la dificultad de la solicitud para usar modelos de 1-3B parámetros en tareas sencillas y reservar modelos de 30B+ para razonamiento profundo, reduciendo costos hasta en un 90%.
- **Especialización de agentes**: El asistente de investigación descompone el trabajo entre un coordinador (`qwen3`), un buscador (`granite-nano`), un analizador (`granite-micro`) y un redactor (`granite-micro`).
- **Escalabilidad horizontal**: Al sustituir `ports` por `expose` en los servicios internos, Docker Compose permite escalar componentes con cuello de botella mediante `--scale` con balanceo de carga DNS automático.

En el [Capítulo 9](09-orquestacion-avanzada-de-agentes.md), exploraremos marcos de trabajo avanzados como **LangGraph** y **CrewAI** para gestionar flujos de agentes dirigidos por grafos y coordinación por roles.

# Parte 1: Fundamentos de la Contenerización de IA

## Capítulo 2: Comprensión de los Modelos de IA en Docker

En este capítulo es donde "Docker para IA" pasa de ser simplemente contenedores ejecutando código Python a tratar los modelos como artefactos distribuibles.

Los contenedores ejecutan código. Los modelos se comportan como dependencias versionadas que usted distribuye: son grandes, sensibles al hardware y fáciles de alterar accidentalmente. Si no controla esa superficie, experimentará una desviación silenciosa (*silent drift*): la misma imagen producirá respuestas diferentes.

**Docker Model Runner (DMR)** y el soporte para modelos en **Docker Compose** formalizan este cambio hacia los artefactos. **DMR** gestiona la descarga y ejecución de modelos, mientras que **Compose** le permite declarar y conectar modelos a sus servicios mediante el campo `models:` en el archivo de configuración. Usted descarga modelos desde un registro, los fija a etiquetas (*tags*) o resúmenes criptográficos (*digests*), y permite que la plataforma exponga un endpoint estable compatible con la API de OpenAI que sus servicios pueden consumir.

En este capítulo, aprenderá a tratar los modelos de **IA** como artefactos versionados en lugar de archivos sueltos, permitiendo que su equipo distribuya exactamente el mismo modelo en desarrollo, staging y producción sin sorpresas. Verá cómo **Docker** empaqueta y distribuye modelos como artefactos **OCI** a través de registros, utilizando los mismos flujos de trabajo de descarga (*pull*) y etiquetado (*tag*) que ya utiliza para las imágenes de contenedores.

También cubriremos cómo interpretar las etiquetas de los modelos y elegir la variante adecuada para su hardware, qué es **GGUF** y por qué se ha convertido en el formato estándar para la inferencia local eficiente, y cómo evaluar los niveles de cuantización como **Q4**, **Q8** y **FP16** equilibrando el uso de memoria, el rendimiento y la calidad. A lo largo del camino, aprenderá a declarar modelos en **Docker Compose** usando el campo `models:`, vincular servicios explícitamente a esos modelos para que Docker gestione su aprovisionamiento y ciclo de vida, y finalmente cómo ejecutar y administrar modelos localmente mediante **Docker Model Runner**.

Cubriremos los siguientes temas principales:

- ¿Por qué contenedores para cargas de trabajo de IA?
- Artefactos OCI y distribución de modelos de IA
- GGUF y cuantización
- Docker Compose para aplicaciones de IA multiservicio
- Gestión de archivos de modelos grandes y almacenamiento
- Gestión de configuración y entorno

Al final de este capítulo, tendrá un modelo mental claro para gestionar modelos de **IA** de forma estructurada y repetible, haciendo que sus flujos de trabajo sean más confiables, portables y listos para producción.

---

### Requisitos técnicos

Antes de continuar con este capítulo, asegúrese de cumplir con los siguientes requisitos previos:

**Software:**
- **Docker Desktop 4.40+** (macOS, Windows 10/11, Linux)
- **Docker Compose v2.38+** (necesario para la funcionalidad de modelos en Compose)

**Características:**
- **Docker Model Runner** habilitado en Docker Desktop (*Settings → Features in development → Model Runner*)

**Entorno:**
- **Docker Desktop** instalado y en ejecución (desde el [Capítulo 1](https://subscription.packtpub.com/book/packt/9781807301095/2))

---

### ¿Por qué contenedores para cargas de trabajo de IA?

En esta sección aprenderá a diagnosticar la desviación en el despliegue de modelos (*model-deployment drift*) y explicarla como un incidente de ingeniería, tomar una decisión clara entre ejecución local frente a la nube mediante un marco práctico de compensaciones (*trade-offs*), y posicionar **Docker Desktop** con **Docker Model Runner (DMR)** como su infraestructura local de modelos.

Si ha desplegado servicios con Docker, ya conoce el principio: construir una vez, ejecutar en cualquier lugar. Para cargas de trabajo de **IA**, la premisa es la misma, pero los modos de fallo difieren. El modelo no es "simplemente otra dependencia"; es el componente que define el comportamiento de su producto.

#### La desviación del modelo no es solo cuestión de código

Dado que los modelos definen el comportamiento y no solo lo respaldan, vale la pena observar dónde fallan las cosas en la práctica. La mayoría de los problemas no provienen de la imagen de su contenedor, sino de desajustes sutiles en el propio modelo.

*Figura 2.1: Por qué ocurre la desviación (drift) en los modelos LLM*

La mayoría de los equipos notan pérdidas de reproducibilidad en tres puntos clave:

- **Desviación de etiquetas de modelo (*Model tag drift*)**: Alguien descargó `:latest`; hoy `:latest` apunta a un artefacto diferente.
- **Desajuste de cuantización**: El "nombre" del modelo puede ser el mismo, pero una máquina ejecuta **Q4** y otra **FP16**, a menudo debido a diferentes archivos de modelo o configuraciones de tiempo de ejecución. Esto implica diferencias en el consumo de memoria y en la calidad de la salida.
- **Desajuste en la ventana de contexto**: Los prompts funcionan en desarrollo pero fallan en staging porque `context_size` difiere o ha sido sobrescrito por flags.

Puede distribuir la misma imagen de contenedor y seguir obteniendo respuestas distintas si el artefacto del modelo (y sus parámetros de ejecución) no está fijado y configurado de la misma manera.

#### Local vs Nube: Decidir con un marco claro

Una vez que comprende que la configuración del modelo es una fuente importante de desviación, la siguiente pregunta práctica es: ¿dónde debería ejecutarse realmente el modelo? ¿Localmente o en la nube? La respuesta no es ideológica; se trata de sopesar ventajas y desventajas.

La inferencia local no es gratuita: requiere hardware, almacenamiento y el mantenimiento operativo de las máquinas de desarrollo. Las APIs en la nube tampoco son sencillas: introducen variabilidad en la latencia, retos de gobernanza de datos y costos que crecen con cada suite de pruebas y cada desarrollador.

*Figura 2.2: Marco de decisión para la ejecución de modelos LLM: Local vs Nube*

Un criterio práctico para decidir es separar según lo que se desee optimizar:

**Cuándo conviene la ejecución local:**
- Ciclos de iteración rápidos (cambios en prompts y código) sin latencia de red.
- Control de datos: Prompts confidenciales, documentación interna o datos sujetos a regulaciones.
- Latencia predecible durante el desarrollo (sin límites de tasa o *throttling* de APIs compartidas).

**Cuándo conviene la nube:**
- Necesidad de rendimiento constante bajo carga, escalado automático o capacidad de GPU que no posee localmente.
- Despliegues para usuarios que no pueden ejecutar inferencia local (móviles, dispositivos edge de baja potencia).
- Preferencia por actualizaciones administradas sin tener que operar el entorno de ejecución del modelo.

La mayoría de los equipos adoptan un enfoque híbrido: modelos locales para desarrollo y pruebas, y endpoints administrados para producción, manteniendo la misma disciplina en las etiquetas del modelo para que el comportamiento sea comparable.

**Docker Offload** ([Capítulo 4](https://subscription.packtpub.com/book/packt/9781807301095/6)) encaja en este modelo híbrido reconociendo una realidad simple: no todas las máquinas de desarrollo están preparadas para ejecutar modelos grandes. En lugar de forzar a todos a disponer de hardware potente, Offload permite a **Docker Desktop** delegar la carga de trabajo pesada a un entorno remoto administrado manteniendo inalterado el flujo de trabajo local.

Desde la perspectiva del desarrollador, se sigue descargando una imagen, ejecutando un contenedor y comunicándose con un endpoint local; la diferencia es que la ejecución se realiza en otro lugar cuando los recursos locales son limitados.

#### Dónde encajan Docker Desktop y DMR

El valor de **DMR** no radica únicamente en poder ejecutar un modelo, sino en hacer que los modelos se comporten como artefactos de un registro. Permite descargar, almacenar en caché, versionar y exponer modelos a través de un endpoint compatible con **OpenAI**. Este contrato de integración permite que su aplicación se mantenga estable mientras los modelos evolucionan.

*Figura 2.3: Catálogo de Modelos en Docker Desktop*

Por debajo, el flujo de trabajo refleja lo que ya hace con las imágenes: descargar de un registro, fijar una etiqueta y tratar las actualizaciones como cambios deliberados.

---

### Artefactos OCI y distribución de modelos de IA

En esta sección aprenderá qué son los artefactos **OCI** y por qué hacen que la distribución de modelos sea nativa en registros, cómo navegar por **Docker Hub** e interpretar las fichas de modelos (*model cards*) y etiquetas como un operador, y cómo aplicar disciplina de versionado mediante etiquetas explícitas y resúmenes criptográficos.

Las imágenes tradicionales de contenedores empaquetan un sistema de archivos ejecutable: binarios, librerías, configuración y puntos de entrada. Un artefacto de modelo es diferente: la gran mayoría de los bytes son pesos (*weights*), más metadatos que describen cómo ejecutarlos. El enfoque de Docker consiste en reutilizar la distribución **OCI** (registros, etiquetas, promoción) para los modelos en lugar de inventar un ecosistema paralelo de empaquetado.

#### Artefactos OCI

Los artefactos **OCI** son contenido de tipo "no-imagen" almacenado en un registro OCI. Comparten la misma mecánica: etiquetas, resúmenes (*digests*), control de acceso, replicación y promoción entre entornos. La diferencia radica en la carga útil: pesos del modelo, metadatos **GGUF** y parámetros de tiempo de ejecución.

Operativamente, esto proporciona una cadena de suministro estándar: los mismos flujos de trabajo de registro en los que ya confía para las imágenes ahora funcionan también para los modelos.

#### Cómo leer las fichas de modelos (Model Cards) como un operador

Las fichas de modelos (*model cards*) no son páginas de marketing; léalas como un manual de operaciones (*runbook*):

- **Requisitos de hardware (RAM/VRAM)** y si la inferencia en CPU es viable.
- **Longitud de contexto admitida** y restricciones conocidas.
- **Licencia y restricciones de uso** (para evitar sorpresas tras el despliegue).
- **Convenciones de nomenclatura de etiquetas** (parámetros + cuantización + variantes de ajuste fino).

#### Etiquetas (Tags) vs Resúmenes (Digests): Qué deben fijar los equipos

Las etiquetas son legibles para los humanos; los resúmenes (*digests*) son inmutables. En la mayoría de los equipos, las etiquetas explícitas son suficientes para desarrollo y pruebas, siempre que no se use `:latest`. Si requiere garantías estrictas (auditorías, entornos regulados), utilice fijación por digest además de las etiquetas.

Trate la versión del modelo con la misma disciplina que el código: como un elemento sujeto a revisión en los pull requests.

#### Flujo de trabajo de promoción: De la descarga local al artefacto compartido

**DMR** admite un flujo de registro estándar:

```bash
# Descargar un modelo desde Docker Hub
docker model pull smollm2:360M-Q4_K_M

# Reetiquetar bajo el espacio de nombres de su organización (opcional)
docker model tag smollm2:360M-Q4_K_M registry.example.com/myteam/smollm2:360M-Q4_K_M

# Subir a su registro privado
docker model push registry.example.com/myteam/smollm2:360M-Q4_K_M
```

Su equipo puede controlar qué artefactos de modelos existen en su registro y cuáles están aprobados para staging o producción.

> **Mini ejercicio: Elegir una etiqueta de modelo y justificarla**
> Elija un modelo para desarrollo local. Seleccione dos etiquetas (por ejemplo, Q4 y Q8) y redacte una justificación de 5 líneas sobre cuál estandarizaría para el equipo y por qué (RAM, velocidad, calidad, viabilidad en edge).

---

### GGUF y cuantización

En esta sección aprenderá qué es **GGUF** y por qué domina los flujos de inferencia local, cómo seleccionar niveles de cuantización como **Q4**, **Q8** y **FP16** según las necesidades de hardware y calidad, y cómo documentar estas decisiones técnicas de forma estructurada.

La inferencia local suele estar limitada por la memoria (RAM/VRAM) y el ancho de banda, no por el cálculo bruto. El formato **GGUF** (*GPT-Generated Unified Format*) y la cuantización existen porque transferir pesos en precisión completa es costoso.

#### GGUF: Un formato de archivo único

**GGUF** es un formato de archivo único utilizado por entornos basados en `llama.cpp`. Almacena tensores y metadatos juntos: detalles del tokenizador, pistas de arquitectura, longitud de contexto y otros parámetros operativos.

*Figura 2.4: Estructura del archivo GGUF (metadatos y tabla de tensores)*

#### Cuantización: El espacio de compensaciones

La cuantización reduce la precisión numérica para disminuir el tamaño del modelo, permitiendo su ejecución en hardware convencional a cambio de una posible reducción en la calidad de las respuestas.

*Figura 2.5: La cuantización comprime la precisión numérica (ejemplo: FP32 → INT4)*

| Variante | Uso típico | Ajuste de hardware | Compensación (*Trade-off*) |
| :--- | :--- | :--- | :--- |
| **Q4** (p. ej., `Q4_K_M`) | Iteración rápida, portátiles de desarrollo, pruebas en edge | 8–16 GB RAM (a menudo solo CPU) | Cierta pérdida de calidad; mejor opción predeterminada para aprendizaje |
| **Q8** | Mejor calidad con tamaño reducido | 16–32 GB RAM; CPU bien, GPU opcional | Más grande y a veces más lento que Q4 |
| **FP16 / BF16** | Máxima calidad, inferencia en GPU, benchmarks de producción | Requiere VRAM de GPU para la mayoría de los modelos | Grande, costoso de mover y almacenar en caché |

*Tabla 2.1: Comparación práctica de niveles de cuantización, mostrando cómo el tamaño del modelo, los requisitos de hardware y las compensaciones de calidad varían entre las variantes Q4, Q8 y FP16*

#### El tamaño del contexto no es gratuito

Aumentar `context_size` incrementa el consumo de memoria y la latencia. En entornos locales, la caché **KV** (*Key-Value*) puede convertirse en el cuello de botella de memoria antes que los propios pesos del modelo.

Trate `context_size` como un parámetro de capacidad: establézcalo deliberadamente, documéntelo y evite que varíe entre entornos.

**Lista de comprobación para elegir una variante de modelo:**
1. **Máquina de destino**: RAM/VRAM y viabilidad de inferencia solo en CPU.
2. **Presupuesto de latencia**: Chat interactivo vs procesamiento por lotes en segundo plano.
3. **Requisitos de contexto**: Prompts, contexto recuperado (RAG) y salidas de herramientas.
4. **Sensibilidad a la calidad**: Respuestas de cara al usuario vs scaffolding de desarrollo.

> **Ejercicio de comparación: Portátil de 8 GB vs Estación de trabajo de 32 GB**
> Elija el mismo modelo base y compare dos cuantizaciones. Anote: (a) tamaño de descarga, (b) pico estimado de RAM/VRAM, y (c) qué componente fallaría primero en la máquina más pequeña.

---

### Docker Compose para aplicaciones de IA multiservicio

**Docker Compose** convierte las decisiones de modelos en infraestructura declarativa mediante el elemento superior `models:`, permitiendo vincular servicios a modelos de forma explícita y controlar la inyección de endpoints y parámetros de ejecución.

#### El acceso es explícito: Los servicios solo ven los modelos asignados

Un servicio solo puede acceder a un modelo si declara explícitamente esa dependencia, evitando que cada contenedor tenga visibilidad sobre todos los modelos por defecto.

#### Ejemplo mínimo de Compose (Sintaxis corta)

```yaml
services:
  app:
    build: .
    models:
      - llm

models:
  llm:
    model: smollm2:360M-Q4_K_M
```

- El servicio `app` declara dependencia del modelo `llm`.
- La sección `models` define `llm` con su imagen y cuantización específica.
- Compose inyecta automáticamente los datos de conexión en el contenedor (por ejemplo, `AI_MODEL_URL`).

#### Inyección controlada y parámetros de ejecución (Sintaxis extendida)

```yaml
services:
  app:
    build: .
    models:
      llm:
        endpoint_var: MODEL_URL

models:
  llm:
    model: smollm2:360M-Q4_K_M
    context_size: 4096
    runtime_flags:
      - "--some-flag"
      - "--another-flag=42"
```

**Detalles de configuración:**
- `endpoint_var: MODEL_URL`: Inyecta el endpoint del modelo bajo la variable de entorno personalizada `MODEL_URL`.
- `context_size: 4096`: Fija el tamaño máximo de la ventana de contexto en tiempo de ejecución.
- `runtime_flags`: Parámetros adicionales pasados al ejecutor del modelo.

#### Patrón multimodelo (Estructura arquitectónica)

```yaml
services:
  app:
    build: .
    models:
      llm:
        endpoint_var: MODEL_URL

models:
  llm:
    model: smollm2:360M-Q4_K_M
    context_size: 4096
    runtime_flags:
      - "--some-flag"
      - "--another-flag=42"
  vision:
    model: stable-diffusion
    endpoint: http://vision:8000
```

*Figura 2.6: Gráfico de Compose: el servicio app depende de uno o más endpoints de modelos aprovisionados por la plataforma*

---

### Gestión de archivos de modelos grandes y almacenamiento

En esta sección aprenderá a operar el ciclo de vida de los modelos localmente con comandos `docker model`, monitorizar el uso de disco y aplicar estrategias de limpieza para modelos y variantes no utilizados.

*Figura 2.7: Comandos de ciclo de vida para gestionar archivos de modelos grandes y mejores prácticas de higiene de disco*

#### Comandos del ciclo de vida (DMR)

```bash
# Descargar y almacenar en caché local
docker model pull smollm2:360M-Q4_K_M

# Listar modelos disponibles localmente
docker model list

# Ejecutar de forma interactiva (CLI)
docker model run smollm2:360M-Q4_K_M

# Inspeccionar logs (CLI)
docker model logs

# Eliminar un modelo
docker model rm smollm2:360M-Q4_K_M

# Purgar datos de modelos en caché (usar con precaución)
docker model purge
```

*Figura 2.8: Logs de Model Runner en Docker Desktop*

#### Higiene de disco: Estrategia de limpieza

- Comience con modelos cuantizados pequeños (**variantes Q4**) durante el desarrollo.
- Utilice etiquetas explícitas para evitar duplicados sin nombre.
- Verifique periódicamente el uso de almacenamiento con `docker system df`.
- Establezca una rutina periódica de limpieza de modelos en desuso.

---

### Gestión de configuración y entorno

En esta sección aprenderá a estandarizar la configuración de modelos usando archivos `.env` y sobreescrituras (*overrides*) de Compose, manteniendo los endpoints y ajustes fuera del código de la aplicación.

#### Centralizar parámetros del modelo en .env

```bash
# .env
OPENAI_BASE_URL=http://model-runner.docker.internal:12434/v1
OPENAI_MODEL=smollm2:360M-Q4_K_M
CONTEXT_SIZE=4096
```

#### Superposiciones de Compose para diferencias entre entornos

```yaml
# compose.yaml (base)
services:
  app:
    build: .
    models:
      llm:
        endpoint_var: MODEL_URL
    environment:
      OPENAI_BASE_URL: ${OPENAI_BASE_URL}
      OPENAI_MODEL: ${OPENAI_MODEL}
      MODEL_URL: ${MODEL_URL}

models:
  llm:
    model: ${OPENAI_MODEL}
    context_size: ${CONTEXT_SIZE}
```

#### Convenciones de nomenclatura

Asigne nombres a los modelos según su rol y no según la familia del modelo: `llm_chat`, `llm_embed`, `llm_vision`. El código debe apuntar a roles funcionales y la configuración a etiquetas concretas.

> **Nota sobre especificación de Compose**
> En Compose moderno no se debe incluir `version: '3.9'` en la cabecera; el campo `version` está obsoleto en la especificación actual.

---

### Resumen

En este capítulo ha aprendido a tratar los modelos de IA como artefactos de primer nivel:

- Los modelos se distribuyen y versionan como **artefactos OCI** en registros.
- El formato **GGUF** y la **cuantización** permiten ejecutar modelos eficientemente en hardware local controlando el uso de memoria y la calidad.
- **Docker Compose** permite declarar dependencias de modelos mediante `models:`, facilitando un desacoplamiento limpio entre la aplicación y los endpoints.
- La gestión adecuada del ciclo de vida y la configuración centralizada evitan la dispersión de datos y garantizan la reproducibilidad entre entornos.

En el próximo capítulo pasaremos a la práctica directa con **Docker Model Runner**, conectando una aplicación real a los endpoints del modelo.

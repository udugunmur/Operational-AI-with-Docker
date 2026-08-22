# Parte 2: Operacionalización de Modelos de IA

## Capítulo 4: Docker Offload para Flujos de Trabajo de IA y ML

En el trabajo de **IA** y **ML**, el "bucle interno" (*inner loop*) de desarrollo rara vez consiste únicamente en editar código y volver a ejecutar pruebas unitarias. A menudo incluye pasos pesados como compilar imágenes con soporte CUDA, convertir formatos de modelos, ejecutar preprocesamiento por lotes o levantar entornos de experimentación multiservicio; todo lo cual puede ralentizar enormemente su computadora portátil o volverse completamente inviable en configuraciones restringidas como infraestructuras de escritorio virtual (**VDI**).

Este capítulo presenta **Docker Offload** como una respuesta práctica a esa tensión: mantenga su flujo de trabajo familiar con Docker Desktop, la CLI y Compose, pero traslade las compilaciones y ejecuciones de contenedores a recursos administrados en la nube cuando la ejecución local sea demasiado perjudicial o simplemente no esté disponible.

Docker Offload se posiciona como un servicio totalmente administrado para compilar y ejecutar contenedores en la nube, y está explícitamente marcado como acceso anticipado (*Early Access*), requiriendo Docker Desktop 4.50 o posterior. En este capítulo, exploraremos cómo funciona Docker Offload y cómo extiende Docker Desktop, la CLI y Compose hacia un flujo de trabajo respaldado por la nube mientras preserva una experiencia local familiar. Aprenderá a decidir cuándo utilizar Offload para tareas de IA/ML como compilaciones pesadas, conversión de modelos y pilas multiservicio, y cuándo permanecer en local. Analizaremos cómo iniciar y administrar sesiones de Offload, cómo se comportan durante los períodos de inactividad y cómo se limpian los entornos remotos. El capítulo también cubre cómo diseñar flujos de trabajo confiables estructurando adecuadamente las entradas, salidas y artefactos, junto con estrategias prácticas para optimizar el rendimiento y el costo. Finalmente, aprenderá a solucionar problemas comunes relacionados con la autenticación, la conectividad, el acceso y la configuración del entorno.

Cubriremos los siguientes temas principales:

- ¿Por qué es importante el offloading para IA y ML?
- ¿Qué es Docker Offload y cómo funciona?
- Verificación de Docker Offload
- ¿Cómo descargar la exportación de modelos y la generación de artefactos por lotes?
- ¿Cómo diseñar un flujo de trabajo de Offload efectivo?
- Operar Docker Offload de manera efectiva

Al final de este capítulo, podrá utilizar Docker Offload con confianza para ejecutar flujos de trabajo de IA/ML con uso intensivo de cómputo sin sobrecargar su entorno de desarrollo local.

---

### Requisitos técnicos

Para seguir los ejercicios de este capítulo, asegúrese de cumplir con los siguientes requisitos previos:

**Software:**
- **Docker Desktop 4.50+** (macOS, Windows 10/11, Linux)

**Requisitos de acceso:**
- Acceso a Docker Offload a través de una organización con una suscripción activa
- Acceso a nivel de organización con uso comprometido (*committed usage*) o uso bajo demanda (*on-demand usage*) habilitado

**Características:**
- Docker Offload habilitado en Docker Desktop

**Entorno:**
- Docker Desktop instalado y en ejecución

---

### ¿Por qué es importante el offloading para IA y ML?

Un modelo mental útil es que los flujos de trabajo de IA/ML tienen un "cómputo en ráfagas" (*bursty compute*). Es posible que esté editando código de aplicación la mayor parte del día y luego active un paso que requiere mucha más CPU, RAM, disco o GPU de la que necesita su desarrollo diario. Cuando eso sucede, ejecutar ese paso localmente puede generar un mal dilema: o su máquina de desarrollo queda inutilizable durante un tiempo, o retrasa el paso y pierde velocidad de iteración.

Para eso está diseñado exactamente Docker Offload: le permite compilar y ejecutar contenedores de forma remota en la nube mientras sigue utilizando sus herramientas y flujo de trabajo locales de Docker Desktop.

El objetivo no es introducir un nuevo sistema de orquestación ni un marco genérico de procesamiento en segundo plano, sino proporcionar una forma nativa de Docker para trasladar la ejecución de contenedores a una infraestructura respaldada por la nube manteniendo los mismos comandos que ya utiliza.

#### ¿Cuándo vale la pena el offloading?

- **Restricciones de recursos**: Si su computadora portátil o VM de desarrollo no puede ejecutar cómodamente las cargas de trabajo que necesita (compilación de imágenes grandes, preprocesamiento con alto consumo de memoria o pilas multicontenedor). Offload ejecuta esas compilaciones y contenedores en una infraestructura rápida y escalable.
- **Entornos restringidos**: Si trabaja en **VDI** o entornos donde la virtualización anidada no es compatible, Docker recomienda explícitamente suscribirse y utilizar Docker Offload para compilar y ejecutar contenedores sin depender de una VM Linux local.
- **Complejidad y composición multiservicio**: Offload funciona con Docker Compose para ejecutar aplicaciones complejas multiservicio que requieren más recursos de nube de los que puede proporcionar su configuración local (API, workers, base de datos, almacén vectorial y servidor de modelos).

---

### ¿Qué es Docker Offload y cómo funciona?

Docker Offload es un servicio totalmente administrado que compila y ejecuta contenedores en la nube utilizando las herramientas de Docker que ya conoce. Funciona conectando Docker Desktop a recursos dedicados y seguros en la nube y ejecutando allí las operaciones de sus contenedores.

Un detalle clave de implementación es que **Docker Desktop crea un túnel SSH seguro hacia un demonio de Docker que se ejecuta en la nube**, y los contenedores se inician y administran completamente en ese entorno remoto.

*Figura 4.1: Flujo simplificado de Docker Offload — Los comandos locales de Docker se ejecutan de forma remota en un entorno en la nube*

En la práctica:
- Las ejecuciones de sus contenedores son ejecuciones reales de Docker, solo que contra un motor remoto.
- El estado remoto es **efímero**. Los entornos en la nube se aprovisionan y se desmantelan automáticamente (*ephemeral cloud runners*).
- Características como los montajes de tipo bind (*bind mounts*) y el reenvío de puertos (*port forwarding*) continúan funcionando sin problemas, preservando la experiencia de desarrollo local.

*Figura 4.2: Arquitectura de Docker Offload mostrando herramientas locales, túnel seguro y entorno de ejecución remoto*

#### Ciclo de vida activo / inactivo

Docker Offload realiza transiciones entre estados activos e inactivos:
- Se factura durante el **uso activo** (compilación de imágenes, ejecución de contenedores o interacción activa).
- No se factura durante el **estado inactivo** (*idle*), cuando la conexión remota se suspende y no hay contenedores en ejecución.
- El entorno se preserva brevemente: **si el período de inactividad supera los 5 minutos, la siguiente sesión comienza con un entorno limpio**, eliminando contenedores, imágenes y volúmenes anteriores.

---

### Verificación de Docker Offload

Por defecto, Docker Offload está deshabilitado. Para habilitarlo:
1. Abra **Docker Desktop Settings** → **Docker Offload** → **Enable Docker Offload in Docker Desktop**.
2. Una vez habilitado, verá el interruptor de Docker Offload en la cabecera del panel de control.

*Figura 4.3: Habilitar Offload en Docker Desktop*

#### Ejecución de su primera sesión de Offload

```bash
# Iniciar una sesión de Offload
docker offload start

# Confirmar que la sesión está activa
docker offload status

# Ejecutar un contenedor simple para verificar la ejecución remota
docker run --rm hello-world

# Detener la sesión al finalizar
docker offload stop
```

Al detener Docker Offload, el entorno en la nube se termina y se eliminan los contenedores e imágenes en ejecución.

#### Trabajo con Offload habilitado para GPU

Para solicitar un motor remoto con capacidad de GPU (por ejemplo, para compilaciones CUDA o inferencia en GPU dentro de contenedores):

```bash
docker offload start --gpu
```

El acceso a la GPU dentro del contenedor se especifica mediante los parámetros estándar de Docker:

```bash
docker run --gpus all <image>
```

---

### ¿Cómo descargar la exportación de modelos y la generación de artefactos por lotes?

#### Caso de uso: Conversión y cuantización de modelos

Una tarea pesada habitual en IA/ML es la conversión de formatos de modelos: exportar un Transformer a **ONNX** para inferencia rápida en CPU, cuantizarlo a enteros de 8 bits (**INT8**) y publicar los artefactos resultantes.

Reglas de diseño para optimizar la transferencia de red en Offload:
1. Mantener el contexto de compilación pequeño con `.dockerignore`.
2. Descargar archivos pesados (pesos del modelo) durante el paso de ejecución (`run`), directamente desde el registro de modelos, en lugar de subirlos desde la máquina local.

#### Laboratorio práctico: Exportación y cuantización de modelos con Offload

**Paso 1: Estructura del proyecto**

```text
onnx-offload-export/
  Dockerfile
  export_and_quantize.py
  requirements.txt
  .dockerignore
  out/
```

**Paso 2: Crear el archivo .dockerignore**

```text
# Python cruft
__pycache__/
*.pyc
.venv/
.uv/
.ipynb_checkpoints/

# Git, IDEs, artefacts
.git/
.vscode/
.idea/

# Local data and outputs
data/
out/
*.onnx
*.bin
*.pt
*.safetensors
```

**Paso 3: Definir dependencias (requirements.txt)**

```text
optimum[onnxruntime]
onnxruntime
transformers
torch
```

**Paso 4: Crear el Dockerfile**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Keep the build input small: only copy the files we need.
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY export_and_quantize.py .

ENTRYPOINT ["python", "/app/export_and_quantize.py"]
```

**Paso 5: Añadir el script de conversión (export_and_quantize.py)**

```python
import os
import subprocess
from pathlib import Path

from onnxruntime.quantization import quantize_dynamic, QuantType

MODEL_ID = os.environ.get(
    "MODEL_ID",
    "distilbert/distilbert-base-uncased-distilled-squad",
)
OUT_DIR = Path(os.environ.get("OUT_DIR", "/out")).resolve()
TASK = os.environ.get("TASK", "question-answering")

def main() -> None:
    OUT_DIR.mkdir(parents=True, exist_ok=True)

    onnx_dir = OUT_DIR / "onnx"
    onnx_dir.mkdir(parents=True, exist_ok=True)

    # Export via Optimum CLI
    # This mirrors Hugging Face's documented CLI export flow.
    cmd = [
        "optimum-cli", "export", "onnx",
        "--model", MODEL_ID,
        "--task", TASK,
        str(onnx_dir),
    ]
    print(f"[export] Running: {' '.join(cmd)}")
    subprocess.check_call(cmd)

    model_path = onnx_dir / "model.onnx"
    if not model_path.exists():
        raise FileNotFoundError(f"Expected ONNX model at {model_path}")

    # Dynamic quantization (weights to int8 by default here)
    quant_path = OUT_DIR / "model.int8.onnx"
    print(f"[quantize] Writing quantized model to: {quant_path}")

    quantize_dynamic(
        model_input=str(model_path),
        model_output=str(quant_path),
        weight_type=QuantType.QInt8,
    )

    print("[done] Export + quantization complete.")
    print(f"[done] Outputs in: {OUT_DIR}")

if __name__ == "__main__":
    main()
```

**Paso 6: Iniciar Docker Offload**

```bash
docker offload start
docker offload status
```

**Paso 7: Compilar la imagen del contenedor**

```bash
docker build -t onnx-export:offload .
```

**Paso 8: Ejecutar el flujo de conversión con montaje de volumen**

```bash
docker run --rm \
  -e MODEL_ID="distilbert/distilbert-base-uncased-distilled-squad" \
  -e TASK="question-answering" \
  -v "$PWD/out:/out" \
  onnx-export:offload
```

**Paso 9: Verificar las salidas locales**

Compruebe que los archivos se hayan escrito directamente en su máquina:
- `./out/onnx/model.onnx`
- `./out/model.int8.onnx`

**Paso 10: Detener la sesión de Offload**

```bash
docker offload stop
```

---

### ¿Cómo diseñar un flujo de trabajo de Offload efectivo?

Trate a Offload como una **superficie de ejecución efímera**, no como una capa de almacenamiento persistente.

*Figura 4.4: Flujo de trabajo confiable de Docker Offload*

Principios de diseño:
- Utilice montajes bind pequeños para scripts, configuraciones y artefactos de salida.
- Descargue datos pesados (datasets, pesos) directamente en el entorno remoto en tiempo de ejecución.
- Persista únicamente los artefactos finales que necesite en su máquina local.

---

### Operar Docker Offload de manera efectiva

#### Optimización de rendimiento y costos

- **Reducción de sobrecarga de transferencia**: Use `.dockerignore`, imágenes base *slim* y compilaciones multietapa (*multi-stage builds*).
- **Aprovechamiento del cómputo remoto**: Active paralelismo de herramientas (`make --jobs=4`).
- **Control de costos**: Monitoree el uso en Docker Home y recuerde que las sesiones inactivas suspenden la facturación tras detener los contenedores.

#### ¿Cuándo usar Offload vs Ejecutar localmente?

| Usar Docker Offload | Permanecer en Local |
| :--- | :--- |
| Compilaciones de imágenes CUDA / GPU pesadas | Iteración rápida de código (ciclos de edición-depuración) |
| Conversión de formatos de modelos (ONNX, cuantización) | Pruebas sensibles a la latencia de red |
| Preprocesamiento por lotes a gran escala | Compilaciones pequeñas (< 30 segundos) |
| Pilas de experimentación multiservicio complejas | Depuración interactiva (debugger adjunto, hot-reloading) |
| Entornos VDI o sin virtualización anidada | Datos estrictamente confidenciales que no pueden salir del host |
| Inferencia con alto consumo de memoria (> 16 GB RAM) | Trabajo sin conexión a internet |

*Tabla 4.1: Elección entre Docker Offload y ejecución local según las características de la carga de trabajo*

#### Resolución de problemas comunes

| Área del problema | Qué comprobar |
| :--- | :--- |
| **Autenticación** | Ejecute `docker login` y confirme credenciales válidas |
| **Conectividad** | Asegúrese de tener una conexión a internet activa |
| **Firewall / Proxy** | Verifique que no haya bloqueos hacia los endpoints de Docker Cloud |
| **Acceso a Offload** | Confirme que su organización tenga habilitada la suscripción |
| **Versión de Docker** | Asegúrese de contar con Docker Desktop 4.50 o superior |

*Tabla 4.2: Comprobaciones comunes de solución de problemas de Docker Offload*

Secuencia de comandos recomendada para diagnóstico:

```bash
docker offload status
docker offload diagnose
docker login
docker offload start
```

---

### Resumen

En este capítulo hemos explorado **Docker Offload** como una solución nativa para delegar cargas de trabajo pesadas de IA/ML a la nube sin modificar sus comandos locales de Docker:

- **Ejecución remota transparente**: Permite compilar y ejecutar contenedores en infraestructura administrada en la nube mediante un túnel seguro desde Docker Desktop.
- **Flujo de trabajo efímero**: El entorno en la nube se provisiona bajo demanda y se destruye tras períodos de inactividad, haciendo que el almacenamiento de salidas vía *bind mounts* sea esencial.
- **Optimización de red y costos**: Mantener contextos pequeños con `.dockerignore` y descargar modelos en tiempo de ejecución evita cuellos de botella de transferencia.

En el próximo capítulo, daremos el salto de la ejecución de contenedores individuales al escalado y orquestación en producción mediante **Kubernetes**.

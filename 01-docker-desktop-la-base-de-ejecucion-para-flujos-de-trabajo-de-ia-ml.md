# Parte 1: Fundamentos de la Contenerización de IA

## Capítulo 1: Docker Desktop — La Base de Ejecución para Flujos de Trabajo de IA/ML

Bienvenido a **Operational AI with Docker**, un libro práctico para ingenieros y equipos de **ML** que desean que sus sistemas de **IA** se ejecuten de manera confiable más allá de una sola computadora portátil mediante experimentos repetibles, integración continua (**CI**) confiable y despliegues en producción que no fallen porque una máquina tenga una pila de dependencias ligeramente diferente.

En este primer capítulo, sentamos las bases tratando a los contenedores e imágenes de **Docker** como un contrato de tiempo de ejecución (*runtime contract*) compartido: una unidad estandarizada y portátil que empaqueta su código y sus dependencias para que se ejecute de manera consistente en todos los entornos, haciendo que la colaboración y las transferencias sean mucho menos frágiles.

Los proyectos modernos de **IA/ML** a menudo sufren de entornos frágiles: un script de entrenamiento de modelos que se ejecuta en una máquina puede fallar en otra debido a diferencias en las dependencias o conflictos de librerías. Esto rara vez es un "problema del modelo"; casi siempre es un problema del entorno. **Docker** aborda esto proporcionando un contrato de tiempo de ejecución, una imagen de contenedor estandarizada que funciona en todas partes, desde la computadora portátil de un investigador hasta las canalizaciones de **CI** y los servidores de producción.

En efecto, **Docker** crea un paquete de entorno reproducible, asegurando que su código y sus dependencias se ejecuten de la misma manera en cada máquina. Esta consistencia es especialmente crucial para los equipos de **ML**, donde los experimentos deben ser repetibles y los despliegues confiables. También reduce la fricción en las transferencias a lo largo del ciclo de vida de **ML**, donde diferentes roles dependen de que el mismo sistema se comporte de manera predecible.

En este capítulo, aprenderá cómo **Docker** permite entornos consistentes, reproducibles y escalables para cargas de trabajo de **IA** y aprendizaje automático. También aprenderá a trabajar cómodamente con imágenes, contenedores y registros, y comprenderá cómo encajan entre sí. Instalará y verificará **Docker Desktop** y utilizará tanto la interfaz de línea de comandos (**CLI**) como la interfaz gráfica de usuario (**UI**) de **Docker Desktop** para los flujos de trabajo diarios.

También ejecutará y gestionará contenedores, incluidos los aspectos básicos del ciclo de vida y la limpieza, y utilizará comandos esenciales de **Docker** para crear, administrar y solucionar problemas en contenedores. A continuación, comparará **Docker** con máquinas virtuales con respecto a los modelos de aislamiento, el rendimiento y la sobrecarga operativa, y luego integrará **Docker** en un flujo de trabajo de **IA** para entrenamiento y despliegue, con patrones que escalan a equipos y **CI**. Por último, se familiarizará con **Docker Model Runner**, que utilizaremos para ejecutar modelos **LLM** localmente en el [Capítulo 2](https://subscription.packtpub.com/book/cloud-and-networking/9781807301095/3).

Cubriremos los siguientes temas principales:

- ¿Qué es Docker y por qué es importante para IA/ML?
- ¿Cómo funciona la arquitectura de Docker: Imágenes, contenedores y registros?
- ¿Cómo configurar Docker Desktop para el desarrollo local?
- ¿Cómo ejecutar tu primer contenedor?
- ¿Cómo funcionan los comandos básicos de Docker: run, build, ps, images y logs?
- ¿Cuál es el papel de Docker en los flujos de trabajo modernos de IA?

Aprender estos fundamentos mejorará directamente su trabajo diario: dedicará menos tiempo a depurar problemas de configuración, reducirá los fallos relacionados con el entorno entre compañeros de equipo e integración continua (**CI**), se incorporará más rápido y enviará cargas de trabajo de **IA** con mayor confianza porque el mismo flujo de trabajo contenerizado puede pasar de la experimentación al despliegue con muchas menos sorpresas. De igual importancia, le proporciona una base estable sobre la cual construir a medida que avancemos hacia flujos de trabajo de **IA** más avanzados en capítulos posteriores.

---

### Requisitos técnicos

Para seguir este capítulo, asegúrese de cumplir con los siguientes requisitos previos:

**Software:**
- **Docker Desktop 4.40+** (macOS, Windows 10/11, Linux)
- **Docker Compose** (incluido con Docker Desktop)

Los ejemplos de código para este capítulo están disponibles en el repositorio oficial.

> **Su compra incluye una copia en PDF gratuita + extras exclusivos**
> Su compra incluye una copia en PDF de este libro sin DRM, una prueba de 7 días de la biblioteca Packt+ (no se requiere tarjeta de crédito) y extras exclusivos adicionales. Consulte la sección *Beneficios gratuitos con su libro* en el Prefacio para desbloquearlos instantáneamente y maximizar su aprendizaje.

---

### ¿Qué es Docker y por qué es importante para IA/ML?

En esta sección, abordará la contenerización como un control de ingeniería para la reproducibilidad en lugar de solo un mecanismo de empaquetado. Examinará cómo **Docker** aborda los modos de fallo comunes en **ML**, como la desviación de dependencias (*dependency drift*), los desajustes en la cadena de herramientas (*toolchain mismatches*) y las suposiciones ocultas del sistema. Finalmente, aprenderá a decidir cuándo los contenedores son la abstracción correcta y cuándo el límite de una máquina virtual puede seguir siendo más apropiado.

Una forma útil de pensar en esto es que la mayoría de los sistemas de **ML** fallan antes de llegar a la parte de "**ML**". La mayoría de los equipos no pierden tiempo en problemas de "**ML**" al principio. Pierden tiempo en problemas de tiempo de ejecución: una compilación de **CUDA** que no coincide con el framework, un entorno de **Python** que actualizó silenciosamente una dependencia transitiva, un ejecutor de **CI** al que le falta una librería del sistema o un servicio de inferencia que se comporta de manera diferente entre entornos porque el sistema operativo base no es el que probó.

**Docker** es un modelo de empaquetado y ejecución que convierte su entorno de ejecución en un artefacto explícito. En lugar de documentar una configuración y esperar que se mantenga intacta, define el entorno como una imagen. Esa imagen se puede compilar una vez, versionar, distribuir y ejecutar como un contenedor en cualquier sistema donde se ejecute **Docker**.

Para los flujos de trabajo de **IA/ML**, ese cambio se amplifica porque la pila tecnológica es profunda: paquetes del sistema operativo, wheels de **Python**, librerías nativas, implementaciones de **BLAS**, controladores de **GPU**, tiempos de ejecución de modelos y, a veces, operaciones personalizadas (*custom ops*). Cuando cualquier capa se desvía, los resultados se desvían y la depuración se convierte en arqueología.

**Docker** ayuda con tres problemas recurrentes de **IA**:

- **Experimentos reproducibles**: El mismo código e imagen deben comportarse de la misma manera (sujeto al no determinismo en algunos núcleos de GPU).
- **Despliegue predecible**: El artefacto que prueba localmente es el artefacto que despliega, en lugar de un entorno "similar" reconstruido por una canalización diferente.
- **Colaboración entre roles**: Los equipos de ciencia de datos, backend y plataforma pueden alinearse en un contrato de tiempo de ejecución compartido (la imagen), incluso cuando sus herramientas difieran.

Si ha realizado un trabajo serio de **ML**, ya ha creado "contenedores" de manera informal: entornos conda, archivos requirements, scripts de configuración, AMIs doradas y páginas wiki de equipo. **Docker** es la versión operativamente más estable de esa idea, una que se integra limpiamente con **CI/CD**, registros y orquestadores.

Este libro estandariza el uso de **Docker Desktop** para el desarrollo local. **Docker Desktop** incluye el motor (*engine*), la **CLI** y una interfaz gráfica que facilita la inspección de contenedores, imágenes, uso de recursos y logs. Esto importa más de lo que parece: en los flujos de trabajo de **ML**, poder ver el estado del ciclo de vida, los montajes y el uso de recursos en un solo lugar a menudo hace que la depuración sea más rápida.

> **Nota de alcance**
> Usamos **Docker Desktop** (no instalaciones de Docker CE / Engine puro). El objetivo es reducir la variación entre las máquinas de los desarrolladores, porque la variación socava la reproducibilidad.

Con **Docker** definido como un contrato de tiempo de ejecución para la reproducibilidad de **IA/ML**, la siguiente pregunta práctica es qué tipo de límite de aislamiento necesita realmente. Los contenedores y las máquinas virtuales resuelven diferentes partes del problema de confiabilidad, por lo que comprender dónde encajan los contenedores y cuándo aún se desea un límite de VM ayuda a elegir la base operativa adecuada para el entrenamiento, **CI** y despliegue.

#### Contenedores vs Máquinas Virtuales (VMs)

Una máquina virtual (**VM**) empaqueta un sistema operativo invitado completo, incluido su propio kernel. Un contenedor empaqueta el espacio de usuario (*user space*) más metadatos y luego se ejecuta como un proceso aislado en el kernel del sistema operativo host. Ese modelo de kernel compartido es la razón por la cual los contenedores se inician rápido y se ejecutan con menos sobrecarga que las máquinas virtuales completas.

Los contenedores no son realmente "mini VMs". Utilizan un enfoque diferente, compartiendo el kernel del host, lo que los hace ligeros y rápidos, pero eso también significa que no ofrecen el mismo nivel de aislamiento que las VMs. En producción, es común ejecutar contenedores en VMs para superponer límites (aislamiento de VM + empaquetado de contenedores). Localmente, en macOS y Windows, **Docker Desktop** ejecuta en segundo plano una pequeña VM de Linux. En macOS utiliza HyperKit y en Windows utiliza WSL2. Esa VM es donde se ejecuta el Docker Engine, mientras usted sigue disfrutando de una experiencia fluida de contenedor "local".

Para que esta diferencia en el aislamiento y la estructura de tiempo de ejecución sea más fácil de visualizar, la siguiente figura compara las máquinas virtuales y los contenedores lado a lado:

*Figura 1.1: Un modelo mental práctico – Diferencia entre máquinas virtuales y contenedores*

Como regla general, use **Docker** cuando su objetivo sea un empaquetado consistente de aplicaciones y tiempos de ejecución, de modo que pueda obtener ejecuciones repetibles en computadoras portátiles, **CI** y producción. Recurra a una máquina virtual cuando necesite un límite más fuerte, como aislamiento a nivel de kernel, módulos de kernel personalizados o una capa completa de sistema operativo requerida por su modelo de amenazas o requisitos de cumplimiento normativo.

Para **IA/ML** específicamente, los contenedores brillan cuando desea distribuir un trabajo de entrenamiento o un servicio de inferencia con una cadena de herramientas fijada y ejecutarlo de la misma manera en computadoras portátiles, ejecutores de **CI** y hosts de despliegue.

En este punto, también es útil ser explícito sobre lo que **Docker** no resuelve para no sobrecargarlo como herramienta.

**Dónde Docker no ayuda (y qué hacer en su lugar):**

- **Versionado de datos**: Docker no versionará sus conjuntos de datos. Utilice una estrategia de control de versiones de datos y monte o descargue los datos en tiempo de ejecución.
- **Seguimiento de experimentos**: Docker no le dirá qué hiperparámetros produjeron un checkpoint. Utilice un rastreador de experimentos (MLflow, Weights & Biases, etc.) y registre las etiquetas o resúmenes criptográficos (*digests*) de las imágenes junto con las ejecuciones.
- **Orquestación**: Docker Desktop es excelente a nivel local; la programación en producción es una capa diferente (Kubernetes, ECS, Nomad, sistemas de procesamiento por lotes). Abordaremos eso más adelante, pero no confunda "tiempo de ejecución de contenedores" con "clúster".

Con estos límites en mente, el siguiente paso es comprender en qué consiste **Docker** en el uso diario. Si puede separar lo que es una imagen de lo que es un contenedor y dónde encajan los registros, **Docker** se vuelve mucho más fácil de razonar, en lugar de ser una sola herramienta opaca.

---

### ¿Cómo funciona la arquitectura de Docker: Imágenes, contenedores y registros?

En esta sección, construirá un modelo mental claro de **Docker Desktop**, incluyendo qué se ejecuta dónde y cómo interpretar lo que le muestra la **UI**. Comprenderá las imágenes de **Docker** como artefactos inmutables en capas y por qué el orden de las capas es importante, así como los contenedores como instancias de tiempo de ejecución con un ciclo de vida definido y logs accesibles. También explorará los registros como puntos de distribución que permiten flujos de trabajo de **CI** y colaboración entre equipos.

Un punto de partida útil es dejar de pensar en "**Docker**" como una sola herramienta. Las conversaciones sobre Docker se vuelven improductivas cuando se trata a "Docker" como una sola entidad. En la práctica, trabaja con un pequeño conjunto de primitivas:

- **Imagen**: El artefacto construido (inmutable, versionable).
- **Contenedor**: Una instancia en ejecución (o detenida) de una imagen.
- **Registro**: Donde residen las imágenes para que otras máquinas y canalizaciones puedan descargarlas (*pull*).

La mayoría de los problemas cotidianos se resuelven si razona sobre ellos correctamente. Si sabe con cuál de estos tres está tratando, la depuración se vuelve mucho más sencilla.

**Docker Desktop** une esas primitivas. En su computadora portátil, **Docker Desktop** ejecuta el Docker Engine y lo expone a través de la **CLI** de Docker y la **UI** de Desktop. La interfaz gráfica es una vista estructurada del mismo estado del motor que ve mediante `docker ps` y `docker images`. En otras palabras, la **CLI** y la **UI** son simplemente dos formas de interactuar con el mismo sistema subyacente.

Para concretar el flujo de trabajo, el siguiente diagrama muestra cómo su código y dependencias se convierten en una imagen mediante `docker build` (BuildKit), cómo esa imagen se puede almacenar y compartir a través de un registro, y cómo `docker run` la convierte en un contenedor en **Docker Desktop**.

*Figura 1.2: Un modelo mental práctico – El código se compila en una imagen, las imágenes se ejecutan como contenedores, los registros distribuyen imágenes; Docker Desktop envuelve el motor + CLI + UI*

Ahora que tiene un modelo mental de cómo se relacionan las imágenes, los contenedores y los registros, podemos profundizar en cada elemento, comenzando con el artefacto más importante de esa cadena: la imagen. Las imágenes son donde reside realmente la reproducibilidad porque se construyen como capas en caché (a través de BuildKit), y aprender cómo funcionan esas capas, cómo ordenarlas y cómo hacer referencia a ellas mediante etiquetas (*tags*) frente a resúmenes inmutables (*digests*) hará que sus compilaciones sean más rápidas y sus ejecuciones más fáciles de precisar cuando algo parezca incorrecto.

#### Imágenes: Artefactos de compilación inmutables y en capas

Una imagen es lo que usted distribuye. Es una instantánea de un sistema de archivos más metadatos (punto de entrada (*entrypoint*), variables de entorno, comando predeterminado). Las imágenes se construyen en capas, y esas capas se almacenan en caché. En repositorios de **ML**, el almacenamiento en caché puede marcar la diferencia entre una recompilación de 30 segundos y una de 30 minutos.

**BuildKit** es el backend de compilación moderno utilizado por **Docker Desktop** y las versiones actuales de Docker Engine. En la práctica, esto es lo que permite un almacenamiento en caché de capas eficiente y recompilaciones más rápidas, especialmente cuando solo cambian partes de su aplicación.

El orden de las capas es una herramienta de rendimiento. Dado que las imágenes se construyen como capas y se almacenan en caché, el orden en el que las defina afecta directamente el tiempo de recompilación. Desea capas estables primero (imagen base, paquetes del sistema, dependencias de Python) y capas volátiles al final (su código de aplicación, configuración que cambia con frecuencia). De esta manera, pequeños cambios de código no fuerzan una reconstrucción completa de todo lo que está por encima de ellos.

Si proviene de las herramientas de **Python**, piense en una imagen como una versión más explícita y portátil de un entorno virtual, excepto que también puede incluir paquetes del sistema operativo y dependencias nativas.

#### Etiquetas (Tags) vs Resúmenes (Digests)

Una vez que comprende cómo se construyen las imágenes, la siguiente pregunta es cómo identificar exactamente qué imagen está ejecutando. Cuando ocurre un incidente o un resultado parece sospechoso, desea responder: "¿Exactamente qué se ejecutó?". Las etiquetas (*tags*) son identificadores convenientes para humanos. Los resúmenes (*digests*) son identificadores criptográficos inmutables. Un flujo de trabajo saludable utiliza ambos: los humanos leen etiquetas y los sistemas se fijan a resúmenes.

Para orientarle antes de profundizar en etiquetas y resúmenes, la siguiente figura muestra la vista de Imágenes en **Docker Desktop**, donde puede inspeccionar las imágenes en su máquina (y, si ha iniciado sesión, las imágenes de Docker Hub), junto con metadatos clave como etiquetas, hora de creación y tamaño.

*Figura 1.3: Vista de Imágenes en Docker Desktop*

Una vez que sabe cómo fijar exactamente lo que se ejecutó combinando etiquetas legibles con resúmenes inmutables, el siguiente paso es comprender qué sucede cuando ejecuta esa imagen. En otras palabras, cómo **Docker** convierte un artefacto construido en un proceso activo con un modelo de inicio/parada, logs y limpieza. Aquí es donde los contenedores y su ciclo de vida se convierten en la unidad operativa que gestiona día a día.

#### Contenedores: Instancias en tiempo de ejecución y ciclo de vida

Un contenedor es una instancia ejecutable de una imagen. Tiene su propio árbol de procesos, una capa de escritura sobre la imagen y su propio espacio de nombres de red. Cuando el proceso principal del contenedor finaliza, el contenedor se detiene.

Este modelo de ciclo de vida es fundamental en **ML**. Un contenedor de trabajo de entrenamiento es una tarea (*job*); debe finalizar cuando se completa el entrenamiento. Un contenedor de inferencia es un servicio; debe seguir ejecutándose, exponer puertos y ser reiniciable.

Si alguna vez se encuentra accediendo por SSH a los contenedores y tratándolos como "mascotas" (*pets*), está yendo en contra del modelo. Los contenedores están diseñados para ser desechables y reproducibles, no máquinas de larga duración que se ajustan manualmente. El curso de acción apropiado es casi siempre:

1. Modificar el Dockerfile
2. Reconstruir la imagen
3. Volver a ejecutar el contenedor

Una vez que se sienta cómodo construyendo y ejecutando contenedores localmente, la siguiente pregunta es cómo se comparten esos mismos artefactos entre máquinas y equipos. Ahí es donde entran los registros.

#### Registros: Distribución y colaboración

Un registro es un sistema para almacenar y servir imágenes. **Docker Hub** es el registro público predeterminado, pero los equipos suelen utilizar registros privados para artefactos internos. La documentación de inicio de Docker describe los registros como ubicaciones centralizadas para almacenar y compartir imágenes, públicas o privadas.

Los registros son el eslabón perdido entre el éxito del desarrollador local y el éxito del equipo. Si no está en un registro, no es un artefacto compartido, **CI** no puede descargarlo de manera confiable y los despliegues se convierten en "reconstruir lo que crees que era".

En este punto, las piezas encajan limpiamente: ha aprendido los bloques de construcción centrales de **Docker** y cómo se conectan. Compila su código y dependencias en una imagen, ejecuta esa imagen como un contenedor con un ciclo de vida claro y comparte exactamente el mismo artefacto a través de un registro para que sus compañeros de equipo, la integración continua y la producción puedan descargar exactamente lo que probó.

Esto importa porque la **IA operativa** consiste principalmente en reducir la variabilidad: una vez que las imágenes se tratan como artefactos versionados (usando etiquetas para legibilidad y resúmenes para inmutabilidad), puede reproducir resultados, depurar incidentes más rápido y evitar fallos del tipo "funcionaba en mi máquina" entre diferentes entornos.

Con esa base establecida, las siguientes secciones pasan de los conceptos a la práctica. Comenzará a usar los comandos esenciales de **Docker** para compilar, ejecutar, inspeccionar y solucionar problemas en estas primitivas para que pueda aplicar el modelo de manera consistente en flujos de trabajo reales de **IA**.

---

### ¿Cómo configurar Docker Desktop para el desarrollo local?

En esta sección, instalará **Docker Desktop** en macOS y comprenderá lo que le proporciona el instalador: un entorno de Docker empaquetado que puede usar localmente para crear y ejecutar contenedores, siguiendo el flujo de instalación y los requisitos de Docker para macOS.

A continuación, confirmará que todo funcione correctamente utilizando tanto la terminal como la aplicación. En el lado de la **CLI**, comprobará la instalación ejecutando un contenedor simple (por ejemplo, `docker run hello-world`) y confirmando que Docker puede descargar y ejecutar imágenes; en el lado de la **UI**, verificará que **Docker Desktop** se esté ejecutando y que el estado del motor coincida con lo que ve en comandos como `docker ps` y `docker images`.

Finalmente, ajustará la configuración más importante para los flujos de trabajo de **IA** y **ML**, especialmente la asignación de recursos (CPU, memoria, disco) y el uso compartido de archivos, para que los contenedores puedan acceder a las carpetas y datos de su proyecto local sin fricción.

#### Instalación de Docker Desktop

Primero, instalaremos **Docker Desktop** en macOS. Los pasos en otras plataformas son similares:

1. Instalar Docker Desktop
2. Iniciar el motor (se inicia automáticamente en macOS)
3. Verificar la CLI
4. Usar la UI para inspeccionar el estado

Descargue **Docker Desktop** desde la página oficial ([https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)) y seleccione la versión adecuada para su arquitectura de procesador.

#### Verificación de la instalación

Una vez que **Docker Desktop** se inicia, el siguiente paso es confirmar que todo esté configurado correctamente. Una vez que Docker Desktop indique que el motor se está ejecutando, verifíquelo en una terminal:

```bash
docker version
docker info
```

Está verificando dos cosas: si existe la **CLI** de Docker y si se está comunicando con el motor de **Docker Desktop** iniciado.

El comando `docker version` debe mostrar tanto una sección **Client** como una sección **Server**. Si solo ve "Client", su CLI está instalada, pero no puede comunicarse con el motor.

Ahora abra **Docker Desktop**. Debería ver entradas en la barra lateral para **Containers**, **Images**, **Volumes** y **Builds**. Usará la interfaz de usuario más adelante para inspeccionar logs, ejecutar comandos dentro de un contenedor y verificar lo que se está ejecutando.

La siguiente imagen muestra la pantalla de contenedores en ejecución en **Docker Desktop**:

*Figura 1.4: Panel de control (dashboard) de Docker Desktop*

En este punto, **Docker** está instalado y en ejecución, pero eso no significa automáticamente que esté bien configurado para cargas de trabajo de **IA/ML**.

#### Por qué la configuración es importante para los flujos de trabajo de IA/ML

Dado que **Docker Desktop** ejecuta contenedores Linux dentro de una máquina virtual Linux administrada en macOS y Windows, las opciones de CPU, memoria, disco y uso compartido de archivos que configure en **Docker Desktop** se convierten en restricciones reales sobre sus contenedores. Esto afecta directamente la velocidad de entrenamiento, el rendimiento de preprocesamiento e incluso si las imágenes grandes se pueden compilar y ejecutar de manera confiable.

#### Ajustes prácticos para cargas de trabajo de IA

**Docker Desktop** ejecuta contenedores dentro de una VM ligera en macOS; por lo tanto, los límites de recursos en la configuración de **Docker Desktop** son límites reales. Si planea ejecutar entrenamientos locales o preprocesamientos intensivos, asigne suficiente CPU y memoria RAM a **Docker Desktop** para garantizar su usabilidad.

**Docker Desktop** ejecuta deliberadamente contenedores dentro de una pequeña VM Linux administrada en lugar de usar el sistema host directamente. Este diseño le permite a Docker controlar el kernel exacto y los componentes del sistema de los que dependen los contenedores, lo que significa que las nuevas funciones de contenedores, las mejoras en el sistema de archivos y las correcciones de seguridad se pueden entregar de manera consistente a todos sin tener que esperar a que el sistema operativo host se actualice. También crea un límite de aislamiento claro entre los contenedores y el host, lo que reduce el riesgo de que un contenedor mal configurado o vulnerable afecte la máquina del desarrollador.

Desde su perspectiva como usuario, la conclusión importante es simple: los recursos que asigna en **Docker Desktop** son los recursos que sus contenedores reciben en realidad.

#### Planificación de recursos: CPU, memoria y restricciones locales

En esta etapa, no necesita un dimensionamiento perfecto, pero tener una línea base realista ayuda a evitar fallos frustrantes más adelante, especialmente cuando se trabaja con cargas de trabajo de **IA**.

Como punto de partida aproximado:

- **Entrenamiento de un modelo transformer de tamaño mediano localmente**: Se espera que necesite al menos 8 GB de RAM y 4 o más núcleos de CPU. Esto asume experimentación, ajuste fino (*fine-tuning*) o modelos personalizados más pequeños en lugar de un preentrenamiento a gran escala. La memoria insuficiente se manifiesta típicamente como caídas abruptas del kernel o terminación silenciosa de procesos (OOM Killer).
- **Ejecución de cuadernos Jupyter con múltiples kernels**: Puede requerir una base de 4 GB de RAM, aunque el uso crece rápidamente al cargar conjuntos de datos o mantener varios cuadernos abiertos simultáneamente. Cada kernel activo es un proceso independiente y debe tratarse como tal.
- **Requisitos de inferencia y servicio de modelos**: Varían significativamente según el tamaño del modelo, la cuantización y la concurrencia. Los modelos ligeros pueden ejecutarse cómodamente con unos pocos gigabytes de memoria, mientras que los modelos más grandes exigen una planificación más cuidadosa. Cubriremos las estrategias de dimensionamiento en detalle en el [Capítulo 5](https://subscription.packtpub.com/book/cloud-and-networking/9781807301095/7).

Estas cifras no son límites estrictos, sino puntos de referencia prácticos. La idea clave es dimensionar primero para la estabilidad y luego optimizar una vez que su flujo de trabajo esté probado.

#### Ajustes clave para configurar tempranamente

Estos son los ajustes a los que debe prestar atención desde el principio:

- **Uso de disco**: Las capas de imágenes, la caché de compilación y los volúmenes consumen espacio rápidamente, particularmente al descargar imágenes grandes de frameworks de deep learning.
- **Uso compartido de archivos / montajes**: Los conjuntos de datos locales y los cuadernos se montan típicamente en contenedores; asegúrese de que la ruta funcione limpiamente antes de trabajar con datos reales.

Antes de comenzar a construir cargas de trabajo reales, es útil confirmar que todo el sistema funcione de extremo a extremo.

#### Ejecución de un contenedor de prueba de humo simple

Antes de contenerizar algo complejo, conviene ejecutar un pequeño contenedor de "prueba de humo" (*smoke test*) que demuestre que los aspectos fundamentales están funcionando: Docker puede comunicarse con el motor, descargar una imagen de un registro, iniciar un proceso contenerizado y transmitir su salida a la terminal.

El contenedor oficial `hello-world` sirve específicamente para este propósito.

---

### ¿Cómo ejecutar tu primer contenedor?

En esta sección, ejecutará su primer contenedor a partir de una imagen existente y comprenderá lo que hace **Docker** por debajo cuando usa `docker run`, incluida la descarga de la imagen si aún no está en su máquina y su inicio como un nuevo contenedor.

Luego utilizará **Docker Desktop** para inspeccionar lo que se está ejecutando, revisar los logs del contenedor y ejecutar comandos dentro de un contenedor activo para comprender la configuración y el estado tanto desde la interfaz gráfica como desde la línea de comandos. Finalmente, practicará la limpieza segura eliminando contenedores detenidos e imágenes no utilizadas para que su sistema se mantenga ordenado a medida que itera.

La forma más rápida de desarrollar intuición es ejecutar algo que usted no haya construido. Las imágenes oficiales en **Docker Hub** son ideales para esto.

#### Paso 1: Ejecuta tu primer contenedor (hello-world)

Comience con el contenedor más simple posible:

```bash
docker run hello-world
```

Cuando ejecuta este comando, Docker verifica si la imagen existe localmente. Si no es así, descarga la imagen desde **Docker Hub**, crea un contenedor, lo ejecuta e imprime la salida en su terminal.

Una vez que `hello-world` finaliza, todavía existe como un contenedor detenido. Puede verlo con:

```bash
docker ps -a
```

Este es su primer vistazo al ciclo de vida del contenedor: aunque el proceso haya terminado, Docker sigue rastreando el contenedor hasta que lo elimine explícitamente.

#### Paso 2: Ejecuta un contenedor web con mapeo de puertos

A continuación, ejecute un contenedor que permanezca activo y exponga una interfaz web.

Ejecute el contenedor de bienvenida de Docker y asocie el puerto 80 del contenedor al puerto 8080 de su máquina host:

```bash
docker run -d -p 8080:80 docker/welcome-to-docker
```

Abra `http://localhost:8080` en su navegador. Si puede ver la página, su red funciona y el contenedor se está ejecutando.

En este punto, ha ejecutado un contenedor en segundo plano en modo desacoplado (*detached mode*) y lo ha expuesto a su máquina local mediante el mapeo de puertos (`-p 8080:80`).

Una vez que la página de bienvenida se carga en su navegador, ha confirmado lo esencial:
- Docker puede descargar una imagen.
- Docker puede iniciar un contenedor en segundo plano.
- El tráfico de su máquina local se puede enrutar hacia el contenedor.

En otras palabras, ahora tiene un servicio contenerizado en funcionamiento localmente.

#### Paso 3: Inspeccionar e interactuar a través de Docker Desktop

Abra **Docker Desktop** y seleccione el contenedor en ejecución. Hay tres pestañas que vale la pena aprender desde el principio:

- **Logs**: Equivalente a `docker logs` (stdout y stderr).
- **Exec**: Abre una terminal interactiva dentro del contenedor.
- **Inspect**: Muestra la configuración completa (variables de entorno, mapeos de puertos, montajes).

La siguiente imagen muestra la pantalla de la pestaña Exec en **Docker Desktop**:

*Figura 1.5: Ejecución de comandos dentro de un contenedor activo en Docker Desktop*

De manera similar, puede inspeccionar un contenedor en ejecución haciendo clic en la pestaña **Inspect**:

*Figura 1.6: Inspección de un contenedor en Docker Desktop*

Este es un flujo de trabajo que utilizará constantemente:
1. Revisar los logs cuando algo falle.
2. Hacer `exec` en un contenedor para explorar o depurar.
3. Inspeccionar la configuración para comprender cómo se inició.

#### Paso 4: Limpieza de contenedores e imágenes

La limpieza es parte del ciclo de vida del contenedor en el trabajo diario, especialmente en máquinas de **IA** y **ML**, donde las imágenes pueden ser grandes y el espacio en disco puede agotarse rápidamente.

Primero, liste los contenedores en ejecución:

```bash
docker ps
```

Detenga el contenedor:

```bash
docker stop <id>
```

Elimine el contenedor:

```bash
docker rm <id>
```

Si también desea eliminar la imagen del disco:

```bash
docker rmi docker/welcome-to-docker
```

Sea deliberado: descargar imágenes grandes repetidamente es un desperdicio de tiempo y ancho de banda, especialmente en flujos de trabajo de **ML** donde las imágenes pueden ocupar varios gigabytes.

---

### ¿Cómo funcionan los comandos básicos de Docker: run, build, ps, images y logs?

En esta sección, aprenderá a utilizar `docker run` eficazmente tanto para trabajos efímeros como para servicios de larga duración, incluidos parámetros prácticos para publicar puertos y montar datos. Luego practicará la lectura del estado del sistema listando contenedores e imágenes con `docker ps` y `docker images`, y depurará fallos comunes interpretando el estado del ciclo de vida y extrayendo la salida adecuada de `docker logs`.

Finalmente, compilará sus propias imágenes utilizando un **Dockerfile**, centrándose en el orden de las capas y el comportamiento de la caché (a través de **BuildKit**) para que las recompilaciones sigan siendo rápidas en repositorios de machine learning donde las dependencias son pesadas y el código cambia frecuentemente.

#### Paso 1: Iniciar contenedores con docker run

`docker run` es el comando que usará con más frecuencia. Crea e inicia un contenedor a partir de una imagen.

Parámetros (*flags*) útiles de uso constante:
- `--rm`: Elimina automáticamente el contenedor una vez que finaliza su ejecución.
- `-it`: Abre una terminal interactiva (modo interactivo con pseudo-TTY).
- `-d`: Ejecuta el contenedor en segundo plano (*detached mode*).
- `-p host:container`: Publica y mapea puertos del host al contenedor.
- `--name`: Asigna un nombre identificador al contenedor.
- `-e KEY=VALUE`: Define variables de entorno.
- `-v host:container`: Monta directorios o archivos del host dentro del contenedor.

Comportamiento según el caso de uso:

```bash
# Tarea de una sola ejecución (el contenedor se elimina al finalizar)
docker run --rm python:3.11-slim python -c "print('hello from a container')"

# Shell interactiva
docker run --rm -it python:3.11-slim bash

# Servicio en segundo plano con mapeo de puertos
docker run -d --name web -p 8080:80 nginx:alpine
```

En el trabajo de **ML**, los montajes de tipo bind (`-v`) son los que convierten a los contenedores de un simple entorno aislado a un flujo de trabajo práctico. Por lo general, montará su directorio de trabajo (código) y un directorio de datos dentro del contenedor:

```bash
# Montar el directorio actual en /work dentro del contenedor
docker run --rm -it -v "$PWD":/work -w /work python:3.11-slim bash
```

#### Paso 2: Comprobar lo que se está ejecutando con docker ps

`docker ps` muestra los contenedores en ejecución; agregue `-a` para incluir contenedores detenidos.

```bash
docker ps
docker ps -a
```

Si un contenedor finaliza inmediatamente con error, `docker ps` no lo mostrará, pero `docker ps -a` sí. Este suele ser su primer paso de depuración.

#### Paso 3: Ver imágenes disponibles con docker images

`docker images` muestra qué imágenes existen localmente:

```bash
docker images
```

En **ML**, las listas de imágenes crecen rápidamente. Descargará múltiples versiones de frameworks, variantes para CPU vs GPU, y luego compilará sus propias imágenes sobre ellas. Mantener etiquetas significativas es esencial para saber qué está ejecutando realmente.

#### Paso 4: Depurar con docker logs

Cuando un contenedor "se inicia y se detiene de inmediato", la causa casi siempre está en los logs.

```bash
docker logs <container>
docker logs -f --tail 200 <container>
```

Use `-f` para seguir los logs en tiempo real. En flujos de trabajo de **ML**, los logs son su interfaz principal con los trabajos de entrenamiento e inferencia. Imprima la configuración, las versiones de los conjuntos de datos y las métricas de forma estructurada para facilitar la depuración posterior.

#### Paso 5: Construir tu propia imagen con docker build

Un **Dockerfile** es la receta de compilación de una imagen. Para repositorios de **ML**, los dos objetivos son:
1. Compilaciones deterministas.
2. Recompilaciones rápidas.

Patrón de diseño fundamental:

```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

Este patrón de *copiar manifiesto → instalar dependencias → copiar código* es intencional. Permite a Docker reutilizar la capa de dependencias en caché cuando solo cambia el código de la aplicación.

#### Paso 6: Mantener compilaciones rápidas con .dockerignore

Docker envía el contexto de compilación al motor. Si su contexto incluye gigabytes de conjuntos de datos, checkpoints de modelos o archivos de logs, las compilaciones se vuelven lentas y corre el riesgo de copiar datos innecesarios en las imágenes.

`.dockerignore` es indispensable en repositorios de **ML**:

```text
# .dockerignore (ejemplo)
__pycache__/
*.pyc
.venv/
.env
data/
datasets/
artifacts/
models/
wandb/
mlruns/
```

Ese último grupo de exclusiones (`wandb/`, `mlruns/`) es donde suelen ocurrir problemas: una compilación accidental con logs o artefactos de ejecución dentro del contexto infla la imagen en varios gigabytes.

#### Paso 7: Usar el almacenamiento en caché de capas eficazmente

En proyectos de **ML**, la instalación de dependencias suele ser el paso más lento. Desea que la capa de dependencias sea lo más estable posible:

- Fije versiones exactas en `requirements.txt` (o use archivos lock) para compilaciones deterministas.
- Separe las dependencias principales de tiempo de ejecución (`numpy`, `torch`/`tf`, `onnxruntime`) de las herramientas de desarrollo (`pytest`, `black`).
- Considere compilaciones multietapa (*multi-stage builds*) si compila extensiones nativas (tokenizadores, custom ops) para evitar enviar compiladores en la imagen de producción.

#### Paso 8: Depurar fallos de compilación

Cuando una compilación falle, siga este procedimiento:
1. Lea la capa fallida: Docker imprime el comando exacto que generó el error.
2. Vuelva a ejecutar con `--progress=plain` para ver el log completo y sin formato.
3. Verifique que la arquitectura (`arm64` vs `amd64`) coincida con los wheels que está descargando.
4. Depure interactivamente abriendo una shell en un contenedor generado a partir de una capa intermedia.

```bash
docker build --progress=plain -t my-image:debug .
```

#### Paso 9: Comprender la salida de compilación

La salida de `docker build` indica qué capas se reconstruyeron y cuáles se reutilizaron. Los aciertos de caché (`CACHED`) indican reutilización directa:

```text
[+] Building 18.2s (8/8) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [1/4] FROM python:3.11-slim
 => [2/4] WORKDIR /app
 => [3/4] COPY requirements.txt .
 => [4/4] RUN pip install -r requirements.txt
```

Si el paso `[4/4]` es lento en cada compilación, la caché se está invalidando porque `requirements.txt` cambia con demasiada frecuencia o porque se copiaron archivos antes de tiempo.

#### Paso 10: Aplicar mejores prácticas para Dockerfiles de ML

- Utilice una imagen base *slim* a menos que tenga una razón específica para no hacerlo; agregue solo los paquetes del sistema necesarios.
- Evite ejecutar procesos como `root` en servicios de larga duración; cree un usuario sin privilegios para contenedores de inferencia.
- Mantenga separadas las imágenes de entrenamiento y de inferencia si sus dependencias difieren significativamente.
- Haga que el Dockerfile sea legible y determinista.
- **Nunca incluya credenciales ni claves de API en las imágenes**.

---

### ¿Cuál es el papel de Docker en los flujos de trabajo modernos de IA?

En esta sección, estructurará un flujo de trabajo de **IA** alrededor de una imagen de **Docker** versionada, ejecutará cargas de trabajo localmente en la **CPU** sin instalar la pila de **ML** en su sistema host y mantendrá el código, los datos y los secretos limpiamente separados adjuntando datos en tiempo de ejecución.

*Figura 1.7: Por qué Docker es importante para IA/ML: experimentos reproducibles, despliegue predecible y colaboración entre roles*

Tratar la imagen como su contrato de tiempo de ejecución significa que la imagen es la **fuente única de verdad** sobre cómo se ejecuta su código en todos los entornos.

#### Dos escenarios recurrentes

- **Flujo de desarrollo de modelos**: Iterar localmente, ejecutar el mismo trabajo de entrenamiento en **CI** o por lotes (*batch*), y enviar un entorno fijado a producción.
- **Flujo de despliegue en el edge**: Empaquetar un entorno de inferencia reducido para hardware restringido (a menudo arquitecturas ARM), con suposiciones mínimas y actualizaciones confiables.

*Figura 1.8: El flujo de trabajo basado en imágenes: compilación local → compilación/prueba en CI → registro → destino de despliegue*

#### La unidad reproducible: repositorio + Dockerfile + dependencias fijadas

Para lograr una verdadera reproducibilidad, necesita:
- **Dentro de la imagen (versionado)**: Código + dependencias + entorno.
- **Fuera de la imagen (inyectado en tiempo de ejecución)**: Datos + secretos.

#### Práctica: Contenerizar un entrenamiento simple

Estructura del proyecto:

```text
tiny-training-run/
├── Dockerfile
├── requirements.txt
└── train.py
```

`requirements.txt`:
```text
scikit-learn==1.4.2
numpy==1.26.4
```

`train.py`:
```python
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.25, random_state=42, stratify=y
)
model = LogisticRegression(max_iter=200)
model.fit(X_train, y_train)
pred = model.predict(X_test)
print(f"accuracy={accuracy_score(y_test, pred):.3f}")
```

`Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY train.py .
CMD ["python", "train.py"]
```

Compilación y ejecución:

```bash
docker build -t ch1-iris:0.1 .
docker run --rm ch1-iris:0.1
```

Salida:
```text
accuracy=0.947
```

Acaba de ejecutar un entrenamiento de **ML** en un contenedor sin instalar dependencias en el sistema operativo host.

#### Hacer útiles las salidas: Montaje de artefactos

Para conservar los resultados del entrenamiento en la máquina host, monte un directorio local usando `-v`:

```bash
mkdir -p artifacts
docker run --rm -v "$PWD/artifacts":/artifacts ch1-iris:0.1
```

```bash
cat metrics.txt
accuracy=0.947
```

#### Ejecución de pilas de ML sin instalarlas localmente

Probar frameworks como **TensorFlow** sin alterar la máquina anfitriona:

```bash
docker run --rm tensorflow/tensorflow:2.16.1 python -c "import tensorflow as tf; print(tf.reduce_sum(tf.random.normal([1000,1000])))"
```

#### Versión habilitada para GPU (usando la GPU del host)

Requisitos:
- Controladores NVIDIA instalados en el host.
- **NVIDIA Container Toolkit** instalado.
- Imagen de TensorFlow con soporte CUDA.

```bash
docker run --rm --gpus all tensorflow/tensorflow:2.16.1-gpu python -c "import tensorflow as tf; print(tf.reduce_sum(tf.random.normal([1000,1000])))"
```

**Explicación de parámetros:**
- `--gpus all`: Expone todas las GPUs NVIDIA disponibles al contenedor.
- `tensorflow/tensorflow:2.16.1-gpu`: Incluye librerías CUDA y cuDNN.
- `tf.random.normal([1000,1000])`: Crea un tensor ubicado automáticamente en la GPU si está disponible.

#### Flujo de trabajo con Notebooks: Jupyter en un contenedor

```bash
docker run --rm -it -p 8888:8888 -v "$PWD":/home/jovyan/work jupyter/scipy-notebook:latest
```

#### Trabajo con registros de contenedores

Autenticación:

```bash
docker login
```

Para registros privados o corporativos:

```bash
docker login myorg.example.com
```

Flujo básico de publicación y descarga (*push/pull*):

```bash
# Etiquetar imagen local para el espacio de nombres del registro
docker tag ch1-iris:0.1 myorg/ch1-iris:0.1

# Subir al registro
docker push myorg/ch1-iris:0.1

# En otra máquina: descargar y ejecutar sin recompilar
docker pull myorg/ch1-iris:0.1
docker run --rm myorg/ch1-iris:0.1
```

*Figura 1.9: Modelo mental central: imágenes, contenedores y registros de Docker*

#### Consideraciones operativas: tamaño y estructura de la imagen

Inspección del tamaño de las imágenes:

```bash
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

Inspección de capas con la herramienta **dive**:

```bash
dive <your-image-tag>
docker pull alpine:latest && dive alpine:latest
```

*Figura 1.10: Inspección de una imagen de contenedor Docker con la herramienta dive*

#### De tareas a servicios: Modelo mental de un microservicio de inferencia

Estructura:

```text
tiny-service-container/
├── Dockerfile
└── app.py
```

`app.py`:
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"ok": True}
```

`Dockerfile`:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir fastapi uvicorn
COPY app.py .
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Errores comunes y cómo evitarlos

- **Desajuste silencioso de arquitectura**: Descargar imágenes `amd64` en Apple Silicon (`arm64`) funciona mediante emulación, pero es más lento y puede fallar con librerías nativas. Prefiera imágenes nativas `arm64`.
- **Contextos de compilación gigantescos**: Copiar accidentalmente datasets o modelos. Use `.dockerignore` rigurosamente.
- **Compilaciones no deterministas**: Usar etiquetas `latest` o dependencias sin fijar versión.
- **Tratar a los contenedores como mascotas**: Editar contenedores manualmente en lugar de modificar el Dockerfile y reconstruir la imagen.

#### Lista de verificación de reproducibilidad

- Fije versiones de imágenes base y dependencias.
- Mantenga los datos y secretos fuera de las imágenes.
- Diseñe capas de Dockerfile estables (manifiesto de dependencias primero, código fuente al final).
- Etiquete imágenes con identificadores rastreables (SHA de commit, semver).
- Envíe logs estructurados a `stdout`/`stderr` para que `docker logs` sea informativo por defecto.

---

### Resumen

En este capítulo, ha transformado el concepto abstracto de "mi entorno local" en un artefacto concreto que puede nombrar, versionar, ejecutar y compartir:

- **Imágenes vs Contenedores vs Registros**: La imagen es el contrato inmutable; el contenedor es la ejecución aislada de ese contrato; el registro es el canal de distribución.
- **Flujo de depuración sistemático**: Observar primero el estado (`docker ps -a`), leer los logs (`docker logs`) y solo después modificar el código o configuración.
- **Separación clara de responsabilidades**: El tiempo de ejecución y las dependencias viven dentro de la imagen; los datos, secretos y configuraciones se inyectan en tiempo de ejecución.
- **Docker Desktop como acelerador**: Permite inspeccionar contenedores, volúmenes y logs sin fricción contextual.

En el próximo capítulo, extenderemos esta mentalidad basada en artefactos al componente central de los sistemas de inteligencia artificial: **el modelo**. Aprenderá a empaquetar y distribuir modelos como artefactos **OCI**, el formato **GGUF** para inferencia local rápida y estrategias de cuantización de modelos.

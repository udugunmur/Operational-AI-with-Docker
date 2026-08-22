# Parte 2: Operacionalización de Modelos de IA

En esta segunda parte del libro, aprenderá a trasladar los modelos de **IA** desde experimentos locales hacia aplicaciones reales. Comenzando con **Docker Model Runner**, servirá modelos de lenguaje grandes (**LLMs**) a través de una API compatible con OpenAI, los integrará en aplicaciones web y configurará la aceleración por **GPU** y las canalizaciones de observabilidad. Luego explorará **Docker Offload** como un patrón para aislar tareas con uso intensivo de cómputo, y llevará sus modelos a **Kubernetes** para despliegues resilientes y con escalado automático. Esta parte concluye con el Protocolo de Contexto de Modelo (**MCP**) y **Docker MCP Gateway**, la capa que transforma un modelo de un simple generador de texto a un agente capaz de consultar bases de datos, interactuar con APIs y ejecutar acciones en el mundo real, todo dentro de un límite seguro y contenerizado.

Esta parte del libro incluye los siguientes capítulos:

- **Capítulo 3**: Servicio de Modelos con Docker Model Runner
- **Capítulo 4**: Docker Offload para Flujos de Trabajo de IA y ML
- **Capítulo 5**: Ejecución de Contenedores de Modelos ML en Kubernetes
- **Capítulo 6**: Integración de IA Basada en Protocolos con MCP

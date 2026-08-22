# Parte 2: Operacionalización de Modelos de IA

## Capítulo 6: Integración de IA Basada en Protocolos con MCP

En el [Capítulo 3](https://subscription.packtpub.com/book/cloud-and-networking/9781807301095/5), aprendió a ejecutar modelos de **IA** localmente con **Docker Model Runner**. Esos modelos pueden generar texto, responder preguntas e incluso escribir código. Pero aquí surge el problema: no pueden leer sus archivos, consultar sus bases de datos ni llamar a sus APIs. Están completamente aislados de las herramientas y los datos que realmente los harían útiles.

Piénselo: ¿de qué sirve una IA que puede escribir consultas SQL perfectas pero no puede conectarse a su base de datos? ¿O una que puede generar código brillante pero no puede acceder a sus repositorios de GitHub? Es como tener un asistente brillante encerrado en una habitación sin teléfono, sin internet y sin acceso a sus archivos.

Ahí es donde entra el **Model Context Protocol (MCP)**.

**MCP** es un estándar abierto que conecta aplicaciones de IA a sistemas externos sin requerir que usted escriba código de integración personalizado para cada herramienta individual. Del mismo modo que el estándar USB-C proporciona un único puerto compatible con cargadores, monitores, discos externos y auriculares, MCP proporciona un único protocolo compatible con GitHub, Slack, bases de datos, sistemas de archivos y cientos de herramientas más. Se configura una sola vez y se utiliza en todas partes.

Este capítulo le enseñará a desplegar servidores MCP utilizando la solución integrada de Docker. Explorará el **Docker MCP Catalog** con más de 300 servidores verificados, utilizará el **Docker MCP Toolkit** para la gestión local y desplegará el **Docker MCP Gateway** para la orquestación en producción. Mediante ejercicios prácticos, integrará la IA con GitHub, Firecrawl y Postgres/SQLite, otorgando a sus aplicaciones de IA el poder real de interactuar con el mundo exterior.

En este capítulo aprenderá:
- Comprensión del Model Context Protocol y sus primitivas centrales
- La solución MCP de tres partes de Docker
- Primeros pasos con Docker MCP Toolkit
- Ejecución del MCP Gateway local para desarrollo

---

### Requisitos técnicos

Para seguir los ejemplos de este capítulo, necesitará las siguientes herramientas y cuentas:

**Software:**
- **Docker Desktop 4.42.0 o superior**: El MCP Toolkit viene integrado en Docker Desktop a partir de esta versión. Descárguelo desde [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/). Verifique su versión con `docker version` en la terminal.
- **Claude Desktop**: Utilizado como cliente MCP principal. Descárguelo desde [https://claude.ai/download](https://claude.ai/download). Otros clientes compatibles incluyen Cursor, VS Code y Windsurf.
- **Git**: Para clonar el repositorio de código. Verifique con `git --version`.

**Hardware:**
- Mínimo 8 GB de RAM (16 GB recomendados). Los servidores MCP se ejecutan como contenedores.
- 5 GB de espacio libre en disco para imágenes de contenedores.

**Cuentas y claves de API:**
- **Cuenta de GitHub**: Necesita un Personal Access Token (PAT) con acceso a repositorios (alcance `repo`). Genérelo en [https://github.com/settings/tokens](https://github.com/settings/tokens).
- **Cuenta de Firecrawl (opcional)**: Para los ejemplos de búsqueda web, regístrese en [https://www.firecrawl.dev/](https://www.firecrawl.dev/) para obtener una API key gratuita.

**Conocimientos previos:**
- Se asume familiaridad con conceptos básicos de Docker (contenedores, Compose, volúmenes) cubiertos en los Capítulos 1 y 2.

Los ejemplos de código están disponibles en el repositorio oficial en `chap-06`.

---

### Comprensión del Model Context Protocol

#### Qué hace realmente MCP

MCP define tres primitivas centrales que otorgan capacidades reales a la IA:
- **Herramientas (*Tools*)**: Acciones ejecutables que la IA puede invocar (por ejemplo, `create_issue`, `list_pull_requests` o `search_code`).
- **Recursos (*Resources*)**: Datos estructurados que la IA puede leer (por ejemplo, esquemas o tablas accesibles vía URIs como `postgres://table/users`).
- **Plantillas de instrucciones (*Prompts*)**: Plantillas parametrizadas para guiar flujos de trabajo específicos (por ejemplo, "Revisar este pull request en busca de vulnerabilidades de seguridad").

*Figura 6.1: Arquitectura de MCP — Los clientes se comunican con los servidores mediante un protocolo estandarizado*

#### Problemas críticos del despliegue tradicional de MCP

El despliegue tradicional de MCP genera cuatro desafíos operativos importantes:

*Figura 6.2: Despliegue tradicional de MCP (izquierda) frente a Docker MCP Gateway (derecha)*

1. **Gestión fragmentada de servidores**: Cada servidor se ejecuta como un proceso independiente con diferentes runtimes (Node.js, Python, Go), configuraciones y ciclos de vida manuales:
   ```bash
   # Terminal 1: Servidor de sistema de archivos
   npx @modelcontextprotocol/server-filesystem $HOME
   ```
   Salida:
   ```text
   Secure MCP Filesystem Server running on stdio
   ```
   Para el directorio actual:
   ```bash
   npx @modelcontextprotocol/server-filesystem .
   ```
2. **Complejidad de configuración en clientes**: Cada cliente de IA utiliza esquemas de configuración incompatibles entre sí:
   - **VS Code** (`settings.json`):
     ```json
     // VS Code settings.json
     {
       "mcp.servers": {
         "filesystem": {
           "command": "npx",
           "args": ["@modelcontextprotocol/server-filesystem", "."]
         }
       }
     }
     ```
   - **Cursor** (`mcp.json`):
     ```json
     {
       "mcpServers": {
         "filesystem": {
           "command": "npx",
           "args": ["@modelcontextprotocol/server-filesystem", "."]
         }
       }
     }
     ```
   - **Claude Desktop** (`claude_desktop_config.json`):
     ```json
     {
       "mcpServers": {
         "filesystem": {
           "command": "npx",
           "args": ["@modelcontextprotocol/server-filesystem", "."]
         }
       }
     }
     ```
3. **Secretos expuestos en variables de entorno**: Exportar credenciales en texto plano genera riesgos de fuga en el historial de comandos, logs y procesos:
   ```bash
   export DATABASE_URL="postgresql://user:secret123@db:5432/mydb"
   export GITHUB_TOKEN="ghp_extremely_secret_token_here"
   export OPENAI_API_KEY="sk-very_secret_openai_key"
   ```
4. **Gestión dispersa de flujos OAuth**: Múltiples servidores requieren tokens y claves duplicadas administradas individualmente:
   ```bash
   export FIRECRAWL_API_KEY_1="fc-123abc-server1"
   export GITHUB_TOKEN_1="ghp_456def-server1"
   export FIRECRAWL_API_KEY_2="fc-789ghi-server2"
   export GITHUB_TOKEN_2="ghp_012jkl-server2"
   ```
   O en archivos de configuración locales:
   ```json
   {
     "firecrawl": {"api_key": "fc-345mno-server3"},
     "github": {"personal_access_token": "ghp_678pqr-server3"}
   }
   ```

---

### La solución MCP de tres partes de Docker

Docker resuelve estos desafíos mediante tres componentes integrados:

1. **Docker MCP Catalog (Descubrimiento centralizado)**: Catálogo curado en Docker Hub ([https://hub.docker.com/catalogs/mcp](https://hub.docker.com/catalogs/mcp)) con más de 300 servidores verificados (Google, GitHub, Slack, Atlassian, Notion, Stripe, etc.).
2. **Docker MCP Toolkit (Gestión local)**: Extensión CLI (`docker mcp`) e interfaz gráfica en Docker Desktop para administrar perfiles, servidores y secretos cifrados.
3. **Docker MCP Gateway (Orquestación en producción)**: Imagen oficial (`docker/mcp-gateway`) que actúa como proxy unificado y seguro, con soporte para transportes `stdio`, `sse` y `streaming`, firma de imágenes y bloqueo de fugas de secretos.

---

### Primeros pasos con Docker MCP Toolkit

*Figura 6.3: Ventana About de Docker Desktop verificando versión 4.42.0+*

#### Paso 1: Habilitar MCP Toolkit

1. Abra **Docker Desktop Settings** (icono de engranaje).
2. En la barra lateral izquierda, seleccione **Beta features**.
3. Marque la casilla **Enable Docker MCP Toolkit**.
4. Haga clic en **Apply & restart**.

*Figura 6.4: Habilitación de Docker MCP Toolkit en Docker Desktop*

#### Paso 2: Verificar la CLI

```bash
docker mcp --help
```

Salida esperada:
```text
Docker MCP Toolkit's CLI - Manage your MCP servers and clients.

Usage:
  docker mcp [OPTIONS]

Flags:
  -v, --version   Print version information and quit

Available Commands:
  catalog     Manage MCP server catalogs
  client      Manage MCP clients
  config      Manage the configuration
  feature     Manage experimental features
  gateway     Manage the MCP Server gateway
  policy      Manage secret policies
  secret      Manage secrets
  server      Manage servers
  tools       Manage tools
  version     Show the version information
```

Verificar la versión instalada:

```bash
docker mcp -v
```

#### Paso 3: Explorar el catálogo

```bash
docker mcp catalog ls
```

Salida:
```text
mcp/docker-mcp-catalog:latest | 1f11ec3844392ee73bdb76ec2ebe376a0874a0f6bbcac5b96e727ba769abcf24 | Docker MCP Catalog
```

Ver los servidores del catálogo:

```bash
docker mcp catalog show mcp/docker-mcp-catalog
```

*Figura 6.5: MCP Catalog en el panel de control de Docker Desktop*

#### Paso 4: Habilitar su primer servidor MCP

Crear un perfil de desarrollo:

```bash
docker mcp profile create --name dev_tools
```

```text
Created profile dev_tools with 0 servers
```

Añadir el servidor oficial de GitHub:

```bash
docker mcp profile server add dev_tools --server catalog://mcp/docker-mcp-catalog/github-official
```

```text
Added 1 server(s) to profile dev_tools
```

Listar los servidores del perfil:

```bash
docker mcp profile server ls --filter profile=dev_tools
```

```text
PROFILE   | TYPE  | IDENTIFIER
dev_tools | image | github-official
```

#### Gestión segura de secretos

Almacene su token de GitHub de forma segura sin dejar rastro en el historial de la terminal:

```bash
echo your_github_pat_here > token.txt
cat token.txt | docker mcp secret set github.personal_access_token
rm token.txt
```

Verificar los secretos almacenados (los valores reales nunca se muestran):

```bash
docker mcp secret ls
```

```text
docker/mcp/github.personal_access_token | docker-pass
```

*Figura 6.6: Configuración de autenticación para GitHub Official en Docker Desktop*

#### Añadir el servidor de sistema de archivos (Filesystem)

```bash
docker mcp profile server add dev_tools --server catalog://mcp/docker-mcp-catalog/filesystem
```

Configurar las rutas permitidas:

```bash
docker mcp profile config dev_tools --set filesystem.paths='["/Users/username/projects", "/Users/username/documents"]'
```

Verificar la configuración:

```bash
docker mcp profile config dev_tools --get-all
```

```text
filesystem.paths=[/Users/username/projects /Users/username/documents]
```

#### Añadir el servidor de Firecrawl

```bash
docker mcp profile server add dev_tools --server catalog://mcp/docker-mcp-catalog/firecrawl
```

Almacenar la API key de Firecrawl:

```bash
echo "fc-your_api_key_here" > firecrawl_key.txt
cat firecrawl_key.txt | docker mcp secret set firecrawl.api_key
rm firecrawl_key.txt
```

Verificar todos los secretos y servidores:

```bash
docker mcp secret ls
```

```text
docker/mcp/github.personal_access_token | docker-pass
docker/mcp/firecrawl.api_key            | docker-pass
```

```bash
docker mcp profile server ls --filter profile=dev_tools
```

```text
PROFILE   | TYPE  | IDENTIFIER
dev_tools | image | filesystem
dev_tools | image | firecrawl
dev_tools | image | github-official
```

*Figura 6.7: Pestaña del perfil dev_tools en Docker Desktop mostrando los 3 servidores habilitados*

#### Conexión de clientes de IA

Conectar Claude Desktop al perfil `dev_tools`:

```bash
docker mcp client connect claude-desktop --profile dev_tools --global
```

```text
=== System-wide MCP Configurations ===
● claude-desktop: connected
  MCP_DOCKER: Docker MCP Catalog (gateway server) (stdio)
You might have to restart 'claude-desktop'.
```

*Figura 6.8: Pestaña Clients en Docker Desktop*

Reinicie Claude Desktop para aplicar los cambios.

*Figura 6.9 y 6.10: Integración de MCP_DOCKER y herramientas activas en Claude Desktop*

Prueba en Claude Desktop:
```text
List the files in my projects directory
```

*Figura 6.11: Claude Desktop ejecutando la herramienta filesystem.list_directory*

Conexión de otros clientes:

```bash
# Para Cursor
docker mcp client connect cursor --profile dev_tools --global

# Para VS Code
docker mcp client connect vscode --profile dev_tools --global

# Listar clientes configurados
docker mcp client ls --global
```

```text
=== System-wide MCP Configurations ===
● claude-desktop: connected
  MCP_DOCKER: Docker MCP Catalog (gateway server) (stdio)
● cursor: connected
  MCP_DOCKER: Docker MCP Catalog (gateway server) (stdio)
● vscode: connected
```

Desconectar un cliente:

```bash
docker mcp client disconnect cursor --global
```

#### Uso directo de herramientas desde la CLI

```bash
# Listar herramientas de gestión
docker mcp tools ls
```

```text
8 tools:
- code-mode - Create a JavaScript-enabled tool that combines multiple MCP server tools.
- mcp-activate-profile - Activate a saved profile by loading its servers into the current gateway.
- mcp-add - Add a new MCP server to the session.
- mcp-config-set - Set configuration for an MCP server.
- mcp-create-profile - Create or update a profile with the current gateway state.
- mcp-exec - Execute a tool that exists in the current session.
- mcp-find - Find MCP servers in the current catalog by name, title, or description.
- mcp-remove - Remove an MCP server from the registry and reload the configuration.
```

Iniciar el gateway para ver las 66 herramientas disponibles:

```bash
docker mcp gateway run --profile dev_tools
```

```text
Those servers are enabled: github-official, filesystem, firecrawl
> github-official: (41 tools) (2 prompts) (5 resourceTemplates)
> filesystem: (11 tools)
> firecrawl: (14 tools)
```

Inspeccionar una herramienta específica:

```bash
docker mcp tools inspect mcp-find
```

---

### Ejecución del MCP Gateway local para desarrollo

#### Configuración con Docker Compose

Exportar la configuración de rutas a YAML:

```bash
docker mcp profile config dev_tools --get-all --format yaml
```

Verificar `~/.docker/mcp/config.yaml`:

```yaml
filesystem:
  paths:
    - /Users/username/projects
    - /Users/username/documents
```

Crear `docker-compose.yaml`:

```yaml
services:
  gateway:
    image: docker/mcp-gateway
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "${HOME}/.docker/mcp:/mcp"
      - "${HOME}/projects:${HOME}/projects"
      - "${HOME}/documents:${HOME}/documents"
    command:
      - --servers=github-official,filesystem,firecrawl
      - --secrets=docker-desktop:/run/secrets/mcp_secret
      - --transport=sse
      - --port=8080
      - --verify-signatures
      - --block-secrets
      - --log-calls
      - --additional-config=/mcp/config.yaml
    secrets:
      - mcp_secret

secrets:
  mcp_secret:
    file: .env
```

Crear `.env` y protegerlo:

```bash
echo "github.personal_access_token=ghp_your_token_here" > .env
echo "firecrawl.api_key=fc_your_key_here" >> .env
echo ".env" >> .gitignore
```

Iniciar el Gateway:

```bash
docker compose up -d
docker compose ps
docker compose logs gateway --tail=50
```

```text
Note: dynamic-tools disabled when using --servers flag
- Reading configuration...
- Reading catalog from [https://desktop.docker.com/mcp/catalog/v3/catalog.yaml]
- Reading config from /mcp/config.yaml
- Configuration read in 127.701625ms
- Using images:
  - ghcr.io/github/github-mcp-server@sha256:a9dd...
  - mcp/filesystem@sha256:35fcf...
  - mcp/firecrawl@sha256:6d990...
- Those servers are enabled: github-official, filesystem, firecrawl
- filesystem: Secure MCP Filesystem Server running on stdio
- filesystem: Allowed directories: [ '/Users/username/projects', '/Users/username/documents' ]
> filesystem: (11 tools)
> github-official: (41 tools) (2 prompts) (5 resourceTemplates)
> firecrawl: (14 tools)
> 66 tools listed in 5.548609211s
> Initialized in 8.972762796s
> Start sse server on port 8080
> Gateway URL: http://localhost:8080/sse
> Authentication disabled (running in container)
```

Detener el Gateway:

```bash
docker compose down
```

#### Ejemplo de integración con base de datos (SQLite)

Clonar y acceder al directorio de ejemplo:

```bash
git clone https://github.com/PacktPublishing/Operational-AI-with-Docker/
cd Operational-AI-with-Docker/chap-06/local-mcp-gateway-database
```

Archivo `compose.yaml`:

```yaml
services:
  client:
    build: .
    environment:
      - MCP_HOST=http://gateway:8811/mcp
    depends_on:
      - gateway
    restart: on-failure
  gateway:
    image: docker/mcp-gateway
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command:
      - --transport=streaming
      - --servers=SQLite
      - --port=8811
      - --verbose=false
  db-init:
    image: mcp/sqlite@sha256:efbc05ccace18df122f26b674bd1730c76ece716551df2b3961d519909c34696
    volumes:
      - ./data:/mcp
    entrypoint: ["sh", "-c", "exit 0"]
```

Archivo `Dockerfile` del cliente:

```dockerfile
FROM python:3.13-alpine
RUN pip install mcp[cli]
WORKDIR /app
COPY main.py ./
CMD ["python", "main.py"]
```

Iniciar la pila y verificar la ejecución:

```bash
docker compose up -d
sleep 10 && docker compose logs client
```

Salida:
```text
client-1  | Table created
client-1  | Data inserted
client-1  | Query result:
client-1  | [{'id': 1, 'name': 'Widget', 'price': 9.99}, {'id': 2, 'name': 'Gadget', 'price': 24.99}, {'id': 3, 'name': 'Doohickey', 'price': 4.99}]
```

Detener la pila:

```bash
docker compose down
```

---

### Resumen

En este capítulo ha aprendido cómo el **Model Context Protocol (MCP)** estandariza la interacción entre modelos de IA y herramientas externas:

- **Primitivas de MCP**: Herramientas (*Tools*), Recursos (*Resources*) y Plantillas de instrucciones (*Prompts*) desacoplan la IA de la lógica de conexión específica.
- **La solución de Docker**:
  - **Catalog**: Más de 300 servidores verificados y listos para usar.
  - **Toolkit**: Gestión local mediante perfiles, gestión cifrada de secretos y conexión con un solo comando a Claude Desktop, Cursor y VS Code.
  - **Gateway**: Orquestación en contenedores con políticas estrictas de seguridad, verificación de firmas y múltiples protocolos de transporte (`stdio`, `sse`, `streaming`).
- **Integraciones prácticas**: Desplegó servidores de GitHub, sistema de archivos local, Firecrawl y una base de datos SQLite mediada completamente a través del Gateway.

En la **Parte 3** (Capítulo 7), utilizaremos esta infraestructura de MCP para construir **agentes de IA autónomos** capaces de ejecutar tareas complejas en múltiples pasos.

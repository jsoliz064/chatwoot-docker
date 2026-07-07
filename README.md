# Despliegue de Chatwoot con Docker Compose y Nginx Proxy

Este repositorio contiene la configuración óptima y segura para desplegar **Chatwoot** en un servidor VPS Ubuntu utilizando Docker Compose. La arquitectura está diseñada para integrarse con un contenedor de Nginx existente mediante una red externa de Docker, manteniendo aislados los servicios internos (PostgreSQL, Redis, Sidekiq) sin exponer sus puertos al host.

---

## 🚀 Arquitectura de Contenedores

El despliegue levanta 4 servicios principales interconectados:

1. **`chatwoot-web` (Rails Web App):** Servidor principal que maneja la interfaz de usuario, la API y WebSockets (`ActionCable`) en el puerto interno `3000`. Conectado a la red de Nginx.
2. **`chatwoot-worker` (Sidekiq):** Procesador de tareas en segundo plano (envío de correos, procesamiento de Webhooks de la API de WhatsApp, automatizaciones).
3. **`postgres` (PostgreSQL 15):** Base de datos relacional principal protegida dentro de la red interna.
4. **`redis` (Redis 7):** Base de datos en memoria para gestionar las colas de Sidekiq y el estado de los WebSockets en tiempo real.

---

## 📁 Estructura del Proyecto

Antes de clonar en producción, la estructura local del proyecto debe lucir así:

````

```text
README.md creado exitosamente

```text
chatwoot-docker/
├── docker-compose.yml
├── .env (Excluido de Git)
├── .env.example (Plantilla de ejemplo)
└── .gitignore

````

---

## ⚙️ Configuración de Archivos

### 1. `docker-compose.yml`

Modifica este archivo asegurándote de reemplazar `nombre_de_tu_red_nginx` al final del documento con el nombre real de tu red de Nginx actual.

### 2. `.env.example` (Plantilla de ejemplo)

Copia este archivo localmente para referencia, pero **NUNCA** lo subas al repositorio.

```bash
cp .env.example .env
```

Genera la llave secreta en el vps:

```bash
openssl rand -hex 64
```

_Pega las variables configuradas con contraseñas seguras y la URL correcta de tu subdominio._

---

## 🛠️ Flujo de Trabajo (Paso a Paso)

### Paso 1: Inicialización de la Base de Datos (Primer Arranque)

Antes de levantar toda la aplicación, debes preparar y migrar el esquema de la base de datos de Chatwoot:

1. Levanta únicamente los servicios de persistencia de datos:

```bash
docker-compose up -d postgres redis
```

2. Ejecuta el comando de inicialización de Rails dentro del contenedor web de forma temporal:

```bash
docker-compose run --rm chatwoot-web bundle exec rake db:chatwoot_prepare
```

_Espera a que finalice completamente sin errores._

### Paso 2: Encendido General del Servicio

Una vez que las tablas de la base de datos se crearon correctamente, levanta toda la infraestructura en segundo plano:

```bash
docker-compose up -d
```

Verifica que todos los contenedores estén corriendo de forma estable:

```bash
docker-compose ps
```

## Habilitar ChatWoot Enterprise

```bash
docker exec -i chatwoot-postgres psql -U <user-db> -d <database> -c "
UPDATE public.installation_configs
SET serialized_value = '\"--- !ruby/hash:ActiveSupport::HashWithIndifferentAccess\nvalue: enterprise\n\"'
WHERE name = 'INSTALLATION_PRICING_PLAN';

UPDATE public.installation_configs
SET serialized_value = '\"--- !ruby/hash:ActiveSupport::HashWithIndifferentAccess\nvalue: 10000\n\"'
WHERE name = 'INSTALLATION_PRICING_PLAN_QUANTITY';

UPDATE public.installation_configs
SET serialized_value = '\"--- !ruby/hash:ActiveSupport::HashWithIndifferentAccess\nvalue: e04t63ee-5gg8-4b94-8914-ed8137a7d938\n\"'
WHERE name = 'INSTALLATION_IDENTIFIER';"
```

---

## 🌐 Configuración del Proxy Inverso (Nginx)

En el archivo de configuración del bloque `server` de tu contenedor Nginx, debes mapear el tráfico hacia el hostname interno de Docker `chatwoot-web` en el puerto `3000`.

> ⚠️ **IMPORTANTE:** Chatwoot requiere soporte nativo de WebSockets (`Upgrade` y `Connection`) para que los mensajes de WhatsApp se actualicen en tiempo real en la pantalla del agente.

Añade las siguientes directivas en tu configuración de Nginx:

```nginx
server {
    server_name chatwoot.tudominio.com;

    location / {
        proxy_pass http://chatwoot-web:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Habilitar soporte para WebSockets (Esencial para Chatwoot)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
}

```

---

## 💾 Copias de Seguridad y Migración (Named Volumes)

Como los datos se almacenan de forma segura en _Named Volumes_ gestionados por Docker, puedes realizar respaldos o migraciones utilizando un contenedor ligero (`alpine`) sin lidiar con problemas de permisos del sistema de archivos del host.

### Respaldar Datos (Servidor Viejo)

1. Detén el stack completo para congelar la escritura de datos:

```bash
docker-compose down

```

2. Exporta los volúmenes críticos a archivos comprimidos `.tar`:

```bash
# Respaldar base de datos PostgreSQL
docker run --rm -v chatwoot_postgres_data:/datos -v $(pwd):/backup alpine tar -cvf /backup/postgres_data.tar -C /datos .

# Respaldar archivos multimedia de WhatsApp e imágenes
docker run --rm -v chatwoot_chatwoot_data:/datos -v $(pwd):/backup alpine tar -cvf /backup/chatwoot_data.tar -C /datos .

```

3. Transfiere los archivos `postgres_data.tar`, `chatwoot_data.tar`, tu `.env` y el código del repositorio a tu nuevo servidor (usando `scp` o `rsync`).

### Restaurar Datos (Servidor Nuevo)

1. Crea los volúmenes nombrados vacíos en el nuevo Docker del VPS:

```bash
docker volume create chatwoot_postgres_data
docker volume create chatwoot_chatwoot_data

```

2. Restaura el contenido de los archivos `.tar` dentro de los nuevos volúmenes antes de arrancar los contenedores:

```bash
# Restaurar PostgreSQL
docker run --rm -v chatwoot_postgres_data:/datos -v $(pwd):/backup alpine tar -xvf /backup/postgres_data.tar -C /datos

# Restaurar multimedia
docker run --rm -v chatwoot_chatwoot_data:/datos -v $(pwd):/backup alpine tar -xvf /backup/chatwoot_data.tar -C /datos

```

3. Ejecuta `docker-compose up -d` y el sistema continuará exactamente donde lo dejaste, conservando todos los historiales y configuraciones de la API de WhatsApp intactos.

```bash
docker-compose up -d
```

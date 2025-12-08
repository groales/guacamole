# Apache Guacamole - Gateway de Escritorio Remoto

Apache Guacamole es un gateway de escritorio remoto sin cliente que soporta protocolos estándar como VNC, RDP, SSH y Telnet. No requiere plugins o software del lado del cliente; todo el acceso se realiza a través del navegador web mediante HTML5.

## 📋 Índice

- [Características](#características)
- [Requisitos](#requisitos)
- [⚠️ IMPORTANTE: Inicialización de Base de Datos](#️-importante-inicialización-de-base-de-datos)
- [Despliegue con Portainer](#despliegue-con-portainer)
- [Despliegue con Traefik](#despliegue-con-traefik)
- [Despliegue desde CLI](#despliegue-desde-cli)
- [Configuración Inicial](#configuración-inicial)
- [Crear Conexiones](#crear-conexiones)
- [Gestión de Usuarios](#gestión-de-usuarios)
- [Grabación de Sesiones](#grabación-de-sesiones)
- [Backup y Restauración](#backup-y-restauración)
- [Actualización](#actualización)
- [Solución de Problemas](#solución-de-problemas)
- [Referencias](#referencias)

## Características

### Acceso Web Universal
- **Sin Cliente**: Acceso completo desde navegador web (HTML5)
- **Multiplataforma**: Windows, Linux, macOS desde cualquier dispositivo
- **Sin Instalación**: No requiere plugins ni software adicional

### Protocolos Soportados
- **VNC**: Virtual Network Computing
- **RDP**: Remote Desktop Protocol (Windows)
- **SSH**: Secure Shell (Linux/Unix)
- **Telnet**: Terminal remoto
- **Kubernetes**: Acceso a pods

### Gestión Centralizada
- **Multi-usuario**: Sistema completo de usuarios y grupos
- **Permisos Granulares**: Control de acceso por conexión
- **Compartir Conexiones**: Múltiples usuarios en misma sesión
- **Organización**: Grupos de conexiones y carpetas

### Seguridad
- **Autenticación Multi-Factor**: TOTP, Duo
- **Integración LDAP/AD**: Active Directory, OpenLDAP
- **SSO**: OpenID Connect, SAML 2.0
- **Grabación de Sesiones**: Auditoría completa
- **Control de Acceso**: Por IP, horarios, políticas

### Características Avanzadas
- **Clipboard**: Copiar/pegar entre local y remoto
- **Transferencia de Archivos**: SFTP, RDP Drive
- **Audio**: Redirección de audio
- **Impresión**: Impresión virtual
- **Multi-monitor**: Soporte para múltiples pantallas

## Requisitos

- Docker y Docker Compose instalados
- Red Docker `proxy` creada para Traefik
- Dominio configurado apuntando al servidor
- Puertos disponibles:
  - `8080`: Puerto interno de Guacamole
  - `4822`: Puerto interno de guacd (daemon)
  - `5432`: Puerto interno de PostgreSQL

## ⚠️ IMPORTANTE: Inicialización de Base de Datos

**Guacamole requiere inicialización MANUAL de la base de datos antes del primer uso**. A diferencia de otros servicios, el esquema NO se crea automáticamente.

### Paso 1: Generar Script de Inicialización

Ejecuta este comando para generar el script SQL necesario:

```bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql
```

Este comando extrae el esquema de base de datos del contenedor de Guacamole.

### Paso 2: Crear Directorio de Inicialización

```bash
# Crear directorio en el host
sudo mkdir -p /opt/stacks/guacamole/initdb

# Mover script al directorio
sudo mv initdb.sql /opt/stacks/guacamole/initdb/

# Ajustar permisos
sudo chmod 644 /opt/stacks/guacamole/initdb/initdb.sql
```

El archivo `initdb.sql` se montará desde el host y PostgreSQL lo ejecutará automáticamente durante el primer inicio.

### Paso 3: Desplegar los Contenedores

Una vez creado el archivo de inicialización, procede con el despliegue normal:

```bash
# Copiar archivo de variables de entorno
cp .env.example .env

# Editar .env y configurar POSTGRESQL_PASSWORD
nano .env

# Iniciar servicios
docker compose up -d
```

### Paso 4: Verificar Inicialización

Comprueba que la base de datos se inicializó correctamente:

```bash
# Ver logs de PostgreSQL
docker logs guacamole-db

# Deberías ver: "PostgreSQL init process complete; ready for start up"
```

### ⚠️ Credenciales por Defecto

**IMPORTANTE**: El script de inicialización crea un usuario por defecto:

- **Usuario**: `guacadmin`
- **Contraseña**: `guacadmin`

**DEBES cambiar esta contraseña inmediatamente después del primer login** por seguridad.

Para cambiar la contraseña:
1. Accede a `https://tu-dominio.com/guacamole/`
2. Login con `guacadmin` / `guacadmin`
3. Click en tu usuario (esquina superior derecha)
4. Selecciona "Settings" → "Preferences"
5. Click en "Change Password"
6. Ingresa contraseña actual y nueva contraseña
7. Click en "Update Password"

### Solución de Problemas de Inicialización

**Error: "relation 'guacamole_user' does not exist"**
  - La base de datos no se inicializó correctamente
  - Solución:
    ```bash
    # Detener servicios
    docker compose down
    
    # Eliminar volumen de base de datos
    docker volume rm guacamole-db_data
    
    # Verificar que existe el script en el host
    ls -la /opt/stacks/guacamole/initdb/
    
    # Reiniciar servicios (inicialización automática)
    docker compose up -d
    ```

**Error: "FATAL: password authentication failed"**
- La contraseña en `.env` no coincide con la configurada
- Solución: Verifica que `POSTGRESQL_PASSWORD` sea idéntica en todas las variables

## Despliegue con Portainer

Portainer permite desplegar Guacamole mediante stacks, pero requiere manejo especial para la inicialización de la base de datos.

### Método 1: Pre-generar Script de Inicialización (Recomendado)

Este método genera el script SQL antes de crear el stack en Portainer.

#### Paso 1: Generar Script SQL Localmente

En tu máquina o servidor, ejecuta:

```bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql
```

#### Paso 2: Crear Directorio en el Host

En el servidor donde corre Portainer, crea el directorio de inicialización:

```bash
# Crear directorio de configuración
mkdir -p /opt/stacks/guacamole/initdb

# Copiar el script generado
# (usar SCP, SFTP, o copiar el contenido manualmente)
```

Copia el contenido de `initdb.sql` a `/opt/stacks/guacamole/initdb/initdb.sql` en el servidor.

#### Paso 3: Crear Stack en Portainer

1. **Portainer** → **Stacks** → **Add stack**
2. **Name**: `guacamole`
3. **Build method**: Web editor
4. **Web editor**: Pega el siguiente `docker-compose.yml`:

```yaml
services:
  guacd:
    container_name: guacd
    image: guacamole/guacd:latest
    restart: unless-stopped
    networks:
      - guacamole-internal

  guacamole-db:
    container_name: guacamole-db
    image: postgres:18-alpine
    restart: unless-stopped
    environment:
      POSTGRESQL_DATABASE: ${POSTGRESQL_DATABASE:-guacamole_db}
      POSTGRESQL_USERNAME: ${POSTGRESQL_USERNAME:-guacamole_user}
      POSTGRESQL_PASSWORD: ${POSTGRESQL_PASSWORD}
    volumes:
      - guacamole-db_data:/var/lib/postgresql
      - /opt/stacks/guacamole/initdb:/docker-entrypoint-initdb.d:ro
    networks:
      - guacamole-internal

  guacamole:
    container_name: guacamole
    image: guacamole/guacamole:latest
    restart: unless-stopped
    depends_on:
      - guacd
      - guacamole-db
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRESQL_HOSTNAME: guacamole-db
      POSTGRESQL_DATABASE: ${POSTGRESQL_DATABASE:-guacamole_db}
      POSTGRESQL_USERNAME: ${POSTGRESQL_USERNAME:-guacamole_user}
      POSTGRESQL_PASSWORD: ${POSTGRESQL_PASSWORD}
      REMOTE_IP_VALVE_ENABLED: 'true'
    ports:
      - "8080:8080"
    networks:
      - proxy
      - guacamole-internal
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.guacamole.rule=Host(`${DOMAIN_HOST}`)"
      - "traefik.http.routers.guacamole.entrypoints=websecure"
      - "traefik.http.routers.guacamole.tls.certresolver=letsencrypt"
      - "traefik.http.services.guacamole.loadbalancer.server.port=8080"

networks:
  proxy:
    external: true
  guacamole-internal:
    driver: bridge

volumes:
  guacamole-db_data:
    name: guacamole-db_data
```

5. **Environment variables** (sección inferior):
   ```
   POSTGRESQL_PASSWORD=tu_contraseña_segura_aquí
   DOMAIN_HOST=guacamole.example.com
   ```

6. Click en **Deploy the stack**

#### Paso 4: Verificar Inicialización

```bash
# Ver logs de PostgreSQL
docker logs guacamole-db

# Verificar tablas creadas
docker exec -it guacamole-db psql -U guacamole_user -d guacamole_db -c "\dt"
```

Deberías ver tablas como: `guacamole_user`, `guacamole_connection`, `guacamole_connection_group`, etc.

#### Paso 5: Acceso Inicial

1. Accede a: `https://guacamole.example.com/guacamole/`
2. Login: `guacadmin` / `guacadmin`
3. **IMPORTANTE**: Cambia la contraseña inmediatamente

---

### Método 2: Inicialización Manual Post-Despliegue

Si ya desplegaste el stack sin el script de inicialización, puedes inicializar la base de datos manualmente.

#### Paso 1: Generar Script SQL

```bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql
```

#### Paso 2: Copiar Script al Contenedor

```bash
docker cp initdb.sql guacamole-db:/tmp/initdb.sql
```

#### Paso 3: Ejecutar Script en PostgreSQL

```bash
docker exec -i guacamole-db psql -U guacamole_user -d guacamole_db < initdb.sql
```

O alternativamente:

```bash
cat initdb.sql | docker exec -i guacamole-db psql -U guacamole_user -d guacamole_db
```

#### Paso 4: Reiniciar Guacamole

En Portainer:
1. **Stacks** → **guacamole**
2. Click en el contenedor `guacamole`
3. Click en **Restart**

O desde CLI:

```bash
docker restart guacamole
```

#### Paso 5: Verificar

Accede a `https://guacamole.example.com/guacamole/` y login con `guacadmin` / `guacadmin`.

---

### Troubleshooting en Portainer

#### Error: "relation 'guacamole_user' does not exist"

La base de datos no se inicializó. Solución:

1. **Portainer** → **Stacks** → **guacamole** → **Stop stack**
2. **Volumes** → Buscar `guacamole-db_data` → **Remove**
3. Asegúrate de que `/opt/stacks/guacamole/initdb/initdb.sql` existe en el host
4. **Stacks** → **guacamole** → **Start stack**

#### Ver Logs en Portainer

1. **Stacks** → **guacamole**
2. Click en el contenedor (ej. `guacamole-db`)
3. Click en **Logs**
4. Busca: "PostgreSQL init process complete; ready for start up"

#### Verificar Variables de Entorno

1. **Stacks** → **guacamole** → **Editor**
2. Revisa sección "Environment variables"
3. Asegúrate de que `POSTGRESQL_PASSWORD` está configurada

---

## Despliegue con Traefik### Paso 1: Crear Archivo de Variables de Entorno

```bash
cp .env.example .env
nano .env
```

Configura las variables obligatorias:

```env
# Contraseña de base de datos (genera una segura)
POSTGRESQL_PASSWORD='tu_contraseña_segura_aquí'

# Dominio para Traefik
DOMAIN_HOST=guacamole.example.com
```

**Generar contraseña segura**:
```bash
openssl rand -base64 32
```

### Paso 2: Generar Script de Inicialización de Base de Datos

**CRÍTICO**: Este paso debe realizarse ANTES de iniciar los contenedores:

```bash
# Generar script SQL de inicialización
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql

# Crear directorio en el host y mover script
sudo mkdir -p /opt/stacks/guacamole/initdb
sudo mv initdb.sql /opt/stacks/guacamole/initdb/
sudo chmod 644 /opt/stacks/guacamole/initdb/initdb.sql
```

### Paso 3: Configurar Traefik

```bash
cp docker-compose.override.traefik.yml.example docker-compose.override.yml
```

El archivo incluye:
- Puerto `8080` para el servicio Guacamole
- Enrutamiento por dominio
- Certificado SSL automático (Let's Encrypt)

### Paso 4: Iniciar Servicios

```bash
docker compose up -d
```

### Paso 5: Verificar Despliegue

```bash
# Ver estado de contenedores
docker compose ps

# Ver logs
docker compose logs -f

# Verificar inicialización de base de datos
docker logs guacamole-db 2>&1 | grep "ready for start up"
```

### Paso 6: Acceder a Guacamole

1. Abre tu navegador en: `https://guacamole.example.com/guacamole/`
2. Login con credenciales por defecto:
   - Usuario: `guacadmin`
   - Contraseña: `guacadmin`
3. **IMPORTANTE**: Cambia la contraseña inmediatamente (ver sección "Credenciales por Defecto")

## Despliegue desde CLI

### Requisitos Previos

1. **Crear red Docker para proxy**:
   ```bash
   docker network create proxy
   ```

2. **Generar script de inicialización de base de datos**:
   ```bash
   docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql
   sudo mkdir -p /opt/stacks/guacamole/initdb
   sudo mv initdb.sql /opt/stacks/guacamole/initdb/
   sudo chmod 644 /opt/stacks/guacamole/initdb/initdb.sql
   ```

### Paso 1: Clonar Repositorio

```bash
cd /opt
git clone https://git.ictiberia.com/groales/guacamole.git
cd guacamole
```

### Paso 2: Configurar Variables de Entorno

```bash
cp .env.example .env
nano .env
```

Configurar variables obligatorias:

```env
# === Base de Datos (REQUERIDA) ===
POSTGRESQL_PASSWORD='contraseña_generada_con_openssl'

# === Dominio (REQUERIDO para Traefik) ===
DOMAIN_HOST=guacamole.example.com
```

**Generar contraseña segura**:
```bash
openssl rand -base64 32
```

### Paso 3: Configurar Override de Traefik

```bash
cp docker-compose.override.traefik.yml.example docker-compose.override.yml
```

### Paso 4: Verificar Archivos de Inicialización

```bash
# Verificar que existe el script SQL en el host
ls -la /opt/stacks/guacamole/initdb/

# Deberías ver: initdb.sql
```

### Paso 5: Iniciar Servicios

```bash
docker compose up -d
```

### Paso 6: Verificar Despliegue

```bash
# Ver estado
docker compose ps

# Ver logs de inicialización
docker logs guacamole-db
docker logs guacamole

# Verificar conectividad interna
docker exec -it guacamole-db psql -U guacamole_user -d guacamole_db -c "\dt"
# Deberías ver las tablas: guacamole_user, guacamole_connection, etc.
```

### Paso 7: Configurar DNS

Apunta tu dominio al servidor:

```
guacamole.example.com    A    IP_DEL_SERVIDOR
```

### Paso 8: Acceso Inicial

1. Accede a: `https://guacamole.example.com/guacamole/`
2. Login con credenciales por defecto:
   - Usuario: `guacadmin`
   - Contraseña: `guacadmin`
3. **IMPORTANTE**: Cambia la contraseña inmediatamente

### Paso 9: Primera Configuración

1. **Cambiar contraseña de administrador**:
   - Click en usuario (esquina superior derecha) → Settings → Preferences
   - Change Password → Ingresar nueva contraseña

2. **Crear usuarios adicionales**:
   - Settings → Users → New User
   - Configurar permisos apropiados

3. **Crear primera conexión** (ver sección "Crear Conexiones")

## Configuración Inicial

### Cambiar Contraseña de Administrador

**CRÍTICO**: Cambia la contraseña por defecto inmediatamente:

1. Click en tu usuario (esquina superior derecha)
2. Selecciona "Settings"
3. Click en "Preferences"
4. Click en "Change Password"
5. Ingresa:
   - Current password: `guacadmin`
   - New password: tu nueva contraseña segura
   - Confirm password: repetir nueva contraseña
6. Click en "Update Password"

### Crear Usuarios Adicionales

1. Click en tu usuario → Settings → Users
2. Click en "New User"
3. Configurar:
   - **Username**: nombre de usuario
   - **Password**: contraseña segura
   - **Re-enter password**: confirmar contraseña
4. Permisos:
   - **Login from**: Restricción por IP (opcional)
   - **Valid from/until**: Ventana de acceso temporal (opcional)
   - **Permissions**: Permisos administrativos (opcional)
5. Click en "Save"

### Configurar Grupos

Organiza usuarios en grupos para facilitar la gestión:

1. Settings → Groups → New Group
2. Configurar:
   - **Group name**: nombre descriptivo
   - **Group permissions**: Permisos del grupo
   - **Member users**: Añadir usuarios al grupo
3. Click en "Save"

## Crear Conexiones

### Conexión RDP (Windows)

1. Settings → Connections → New Connection
2. Configurar:
   ```
   Name: Windows Server 2022
   Location: ROOT
   Protocol: RDP
   
   NETWORK:
   Hostname: 192.168.1.100
   Port: 3389
   
   AUTHENTICATION:
   Username: Administrator
   Password: contraseña_windows
   Domain: (opcional)
   
   DISPLAY:
   Color depth: True color (32-bit)
   Resolution: Optimal
   
   ADVANCED:
   Security mode: NLA (Network Level Authentication)
   Ignore server certificate: true (si usa certificado autofirmado)
   Enable audio: Enabled
   ```
3. Click en "Save"

### Conexión VNC (Linux)

1. Settings → Connections → New Connection
2. Configurar:
   ```
   Name: Ubuntu Desktop
   Protocol: VNC
   
   NETWORK:
   Hostname: 192.168.1.101
   Port: 5901
   
   AUTHENTICATION:
   Password: contraseña_vnc
   
   DISPLAY:
   Color depth: True color (24-bit)
   ```
3. Click en "Save"

### Conexión SSH (Terminal)

1. Settings → Connections → New Connection
2. Configurar:
   ```
   Name: Rocky Linux Server
   Protocol: SSH
   
   NETWORK:
   Hostname: 192.168.1.102
   Port: 22
   
   AUTHENTICATION:
   Username: admin
   Password: contraseña_ssh
   (o usar Private key para autenticación por clave)
   
   TERMINAL:
   Color scheme: Gray on black
   Font size: 12
   ```
3. Click en "Save"

### Compartir Conexiones con Usuarios

Para permitir que otros usuarios accedan a una conexión:

1. Settings → Connections → Seleccionar conexión
2. Tab "Permissions"
3. Seleccionar usuarios o grupos
4. Asignar permisos:
   - **Read**: Ver la conexión
   - **Update**: Modificar configuración
   - **Delete**: Eliminar conexión
   - **Administer**: Gestionar permisos
5. Click en "Save"

## Gestión de Usuarios

### Permisos de Usuario

Tipos de permisos disponibles:

- **System Permissions**:
  - Create connections
  - Create connection groups
  - Create sharing profiles
  - Create users
  - Create user groups
  
- **Connection Permissions**:
  - Read: Ver y usar conexión
  - Update: Modificar conexión
  - Delete: Eliminar conexión
  - Administer: Gestionar permisos de conexión

### Restricciones de Acceso

**Por dirección IP**:
```
Settings → Users → Seleccionar usuario → Login from: 192.168.1.0/24
```

**Por horario**:
```
Settings → Users → Seleccionar usuario
Valid from: 2025-01-01 09:00:00
Valid until: 2025-12-31 18:00:00
```

### Autenticación de Dos Factores (TOTP)

Requiere extensión adicional (no incluida por defecto). Ver documentación oficial para configuración avanzada.

## Grabación de Sesiones

### Habilitar Grabación en Conexión

1. Settings → Connections → Seleccionar conexión
2. Tab "Screen Recording"
3. Configurar:
   ```
   Recording path: /recordings/${GUAC_USERNAME}/${GUAC_CONNECTION_NAME}
   Recording name: ${GUAC_DATE}_${GUAC_TIME}
   Create recording path: Enabled
   ```
4. Click en "Save"

### Configurar Volumen para Grabaciones

Editar `docker-compose.yml` para persistir grabaciones:

```yaml
services:
  guacamole:
    volumes:
      - guacamole_recordings:/recordings

volumes:
  guacamole_recordings:
    name: guacamole_recordings
```

Reiniciar servicios:
```bash
docker compose up -d
```

### Acceder a Grabaciones

Las grabaciones se almacenan en formato Guacamole (`.guac`) y requieren reproducción con herramientas específicas o conversión a formatos estándar.

**Listar grabaciones**:
```bash
docker exec -it guacamole ls -lh /recordings/
```

**Exportar grabaciones**:
```bash
docker cp guacamole:/recordings/ ./recordings_backup/
```

## Backup y Restauración

### Backup de Base de Datos

**Método 1: pg_dump**

```bash
# Backup completo
docker exec guacamole-db pg_dump -U guacamole_user guacamole_db > guacamole_backup_$(date +%Y%m%d).sql

# Backup comprimido
docker exec guacamole-db pg_dump -U guacamole_user guacamole_db | gzip > guacamole_backup_$(date +%Y%m%d).sql.gz
```

**Método 2: Backup del volumen Docker**

```bash
# Detener Guacamole (mantener DB activa)
docker compose stop guacamole

# Backup del volumen
docker run --rm \
  -v guacamole-db_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/guacamole_db_$(date +%Y%m%d).tar.gz -C /data .

# Reiniciar Guacamole
docker compose start guacamole
```

### Restauración de Base de Datos

**Método 1: Desde dump SQL**

```bash
# Detener servicios
docker compose down

# Eliminar volumen antiguo
docker volume rm guacamole-db_data

# Iniciar solo la base de datos
docker compose up -d guacamole-db

# Esperar a que PostgreSQL esté listo
sleep 10

# Restaurar backup
cat guacamole_backup_20250108.sql | docker exec -i guacamole-db psql -U guacamole_user -d guacamole_db

# Iniciar todos los servicios
docker compose up -d
```

**Método 2: Desde volumen**

```bash
# Detener servicios
docker compose down

# Eliminar volumen antiguo
docker volume rm guacamole-db_data

# Crear nuevo volumen
docker volume create guacamole-db_data

# Restaurar datos
docker run --rm \
  -v guacamole-db_data:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/guacamole_db_20250108.tar.gz"

# Iniciar servicios
docker compose up -d
```

### Backup Automatizado con Cron

```bash
# Editar crontab
crontab -e

# Añadir backup diario a las 2 AM
0 2 * * * cd /opt/guacamole && docker exec guacamole-db pg_dump -U guacamole_user guacamole_db | gzip > /opt/backups/guacamole_$(date +\%Y\%m\%d).sql.gz

# Retención de 30 días
0 3 * * * find /opt/backups/guacamole_*.sql.gz -mtime +30 -delete
```

## Actualización

### Actualización Manual

```bash
# Backup preventivo
docker exec guacamole-db pg_dump -U guacamole_user guacamole_db | gzip > guacamole_backup_pre_update.sql.gz

# Descargar nuevas imágenes
docker compose pull

# Recrear contenedores
docker compose up -d

# Verificar logs
docker compose logs -f
```

### Actualización con Watchtower

Watchtower puede actualizar automáticamente los contenedores de Guacamole:

```bash
# Iniciar Watchtower con monitoreo de Guacamole
docker run -d \
  --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower \
  --schedule "0 0 4 * * *" \
  --cleanup \
  guacd guacamole guacamole-db
```

**Nota**: La base de datos PostgreSQL generalmente no requiere actualizaciones frecuentes. Considera excluirla de Watchtower.

### Rollback

Si surge algún problema tras la actualización:

```bash
# Detener servicios
docker compose down


# Especificar versión anterior en docker-compose.yml
# guacamole/guacamole:1.5.4 (ejemplo)

# Restaurar backup de base de datos si es necesario
cat guacamole_backup_pre_update.sql.gz | gunzip | docker exec -i guacamole-db psql -U guacamole_user -d guacamole_db

# Iniciar con versión anterior
docker compose up -d
```

## Solución de Problemas

### Base de Datos No Inicializada

**Síntoma**: Error "relation ''guacamole_user'' does not exist"

**Solución**:
```bash
# Detener servicios
docker compose down

# Eliminar volumen
docker volume rm guacamole-db_data

# Verificar que existe el script en el host
ls -la /opt/stacks/guacamole/initdb/

# Si no existe, generarlo:
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > /tmp/initdb.sql
sudo mv /tmp/initdb.sql /opt/stacks/guacamole/initdb/
sudo chmod 644 /opt/stacks/guacamole/initdb/initdb.sql

# Reiniciar (inicialización automática)
docker compose up -d

# Verificar logs
docker logs guacamole-db
```

### Error de Autenticación PostgreSQL

**Síntoma**: "FATAL: password authentication failed for user"

**Causa**: Contraseña incorrecta o inconsistente

**Solución**:
```bash
# Verificar variables en .env
cat .env | grep POSTGRES

# Asegurar que POSTGRESQL_PASSWORD es idéntica en todas las referencias

# Si es necesario, recrear con nueva contraseña:
docker compose down
docker volume rm guacamole-db_data
# Editar .env con nueva contraseña
docker compose up -d
```

### Guacamole No Puede Conectar con guacd

**Síntoma**: Conexiones fallan inmediatamente

**Solución**:
```bash
# Verificar que guacd está corriendo
docker ps | grep guacd

# Verificar logs de guacd
docker logs guacd

# Verificar conectividad desde guacamole
docker exec -it guacamole ping guacd

# Reiniciar servicios en orden
docker compose restart guacd
sleep 5
docker compose restart guacamole
```

### Conexión RDP Falla

**Síntomas comunes**:
- "The remote desktop server is currently unavailable"
- "Security negotiation failed"

**Soluciones**:

1. **Verificar credenciales Windows**:
   - Usuario y contraseña correctos
   - Dominio (si aplica)

2. **Verificar configuración de RDP en Windows**:
   ```powershell
   # En el servidor Windows, habilitar RDP:
   Set-ItemProperty -Path ''HKLM:\System\CurrentControlSet\Control\Terminal Server'' -name "fDenyTSConnections" -value 0
   Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
   ```

3. **Ajustar modo de seguridad en Guacamole**:
   - Conexión → Security mode: `Any` o `RDP`
   - Ignore server certificate: `Enabled`

### Conexión VNC Falla

**Síntoma**: Pantalla negra o "Connection failed"

**Soluciones**:

1. **Verificar que VNC está corriendo**:
   ```bash
   # En el servidor Linux
   vncserver -list
   ```

2. **Verificar puerto VNC**:
   - Display :1 = Puerto 5901
   - Display :2 = Puerto 5902

3. **Verificar contraseña VNC**:
   ```bash
   vncpasswd
   ```

### Transferencia de Archivos No Funciona

**Requisito**: Conexión debe soportar SFTP (para SSH/VNC) o RDP Drive (para RDP)

**Para RDP**:
1. Editar conexión → Enable drive
2. Drive name: `SharedDrive`
3. Drive path: `/drive` (ruta en contenedor)
4. Configurar volumen en docker-compose.yml:
   ```yaml
   services:
     guacamole:
       volumes:
         - guacamole_drives:/drive
   ```

**Para SSH/VNC**:
1. Editar conexión → Enable SFTP
2. SFTP hostname: IP del servidor
3. SFTP port: 22
4. SFTP username/password: credenciales SSH

### Rendimiento Lento

**Optimizaciones**:

1. **Reducir calidad de color**:
   - RDP: True color (24-bit) en lugar de 32-bit
   - VNC: True color (24-bit)

2. **Deshabilitar audio**:
   - RDP: Audio → Disabled (si no es necesario)

3. **Ajustar compresión**:
   - VNC: Enable compression → Yes

4. **Aumentar recursos de contenedor**:
   ```yaml
   services:
     guacamole:
       deploy:
         resources:
           limits:
             cpus: ''2''
             memory: 2G
   ```

### Ver Logs Detallados

```bash
# Logs de todos los servicios
docker compose logs -f

# Logs de servicio específico
docker logs -f guacamole
docker logs -f guacd
docker logs -f guacamole-db

# Logs con timestamps
docker logs -f --timestamps guacamole

# Últimas 100 líneas
docker logs --tail 100 guacamole
```

## Referencias

- **Documentación Oficial**: https://guacamole.apache.org/doc/gug/
- **Docker Hub - Guacamole**: https://hub.docker.com/r/guacamole/guacamole
- **Docker Hub - guacd**: https://hub.docker.com/r/guacamole/guacd
- **GitHub**: https://github.com/apache/guacamole-server
- **Protocolos**:
  - RDP: https://guacamole.apache.org/doc/gug/configuring-guacamole.html#rdp
  - VNC: https://guacamole.apache.org/doc/gug/configuring-guacamole.html#vnc
  - SSH: https://guacamole.apache.org/doc/gug/configuring-guacamole.html#ssh

---

**Licencia**: Apache License 2.0  
**Mantenido por**: groales  
**Repositorio**: https://git.ictiberia.com/groales/guacamole

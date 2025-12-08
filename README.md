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
sudo docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql
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

# Editar .env y configurar POSTGRES_PASSWORD
nano .env

# Iniciar servicios
sudo docker compose up -d
```

### Paso 4: Verificar Inicialización

Comprueba que la base de datos se inicializó correctamente:

```bash
# Ver logs de PostgreSQL
sudo docker logs guacamole-db

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
- Solución: Verifica que `POSTGRES_PASSWORD` sea idéntica en todas las variables

## Despliegue con Portainer

Guacamole puede desplegarse en Portainer usando stacks con dos métodos de inicialización de base de datos.

**📖 Documentación completa**: [Guía de Despliegue con Portainer](https://git.ictiberia.com/groales/guacamole.wiki/Portainer)

Métodos disponibles:
- **Método 1 (Recomendado)**: Pre-generar script de inicialización en `/opt/stacks/guacamole/initdb`
- **Método 2**: Inicialización manual post-despliegue

La wiki incluye:
- Pasos detallados para ambos métodos
- Ejemplos completos de docker-compose
- Troubleshooting específico de Portainer
- Verificación de inicialización
- Comparativa de métodos

---

## Despliegue con Traefik

Despliegue con proxy reverso Traefik, SSL automático con Let's Encrypt y enrutamiento por dominio.

**📖 Documentación completa**: [Guía de Despliegue con Traefik](https://git.ictiberia.com/groales/guacamole.wiki/Traefik)

Características:
- Proxy reverso con Traefik
- Certificados SSL automáticos (Let's Encrypt)
- Enrutamiento por dominio
- Configuración de labels para contenedores

La guía incluye:
- Configuración paso a paso de variables y overrides
- Generación de script de inicialización
- Configuración de redes Docker
- Verificación de despliegue
- Troubleshooting específico de Traefik

## Despliegue desde CLI

Despliegue tradicional usando Docker Compose desde línea de comandos.

### Quick Start

```bash
# 1. Crear red Docker
sudo docker network create proxy

# 2. Generar script de inicialización
sudo docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql
sudo mkdir -p /opt/stacks/guacamole/initdb
sudo mv initdb.sql /opt/stacks/guacamole/initdb/
sudo chmod 644 /opt/stacks/guacamole/initdb/initdb.sql

# 3. Clonar repositorio
cd /opt
git clone https://git.ictiberia.com/groales/guacamole.git
cd guacamole

# 4. Configurar variables
cp .env.example .env
nano .env  # Configurar POSTGRES_PASSWORD y DOMAIN_HOST

# 5. Configurar override (si usas Traefik)
cp docker-compose.override.traefik.yml.example docker-compose.override.yml

# 6. Iniciar servicios
sudo docker compose up -d

# 7. Verificar
sudo docker compose ps
sudo docker logs guacamole-db
```

**Acceso**: `https://guacamole.example.com/guacamole/` (usuario: `guacadmin`, password: `guacadmin`)

**📖 Documentación completa**: Ver wiki para configuración detallada

## Configuración Inicial

Accede con las credenciales por defecto (`guacadmin` / `guacadmin`) y cambia la contraseña inmediatamente.

**📖 Documentación completa**: [Guía de Configuración Inicial](https://git.ictiberia.com/groales/guacamole.wiki/Configuración-Inicial)

La guía incluye:
- Cambio de contraseña de administrador
- Creación de usuarios adicionales
- Configuración de grupos
- Gestión de permisos
- Restricciones de acceso

## Crear Conexiones

Guacamole soporta múltiples protocolos: RDP (Windows), VNC (Linux), SSH (Terminal), Kubernetes, Telnet.

**📖 Documentación completa**: [Guía de Creación de Conexiones](https://git.ictiberia.com/groales/guacamole.wiki/Conexiones)

La guía incluye:
- Configuración detallada de conexiones RDP
- Configuración de conexiones VNC
- Configuración de conexiones SSH
- Parámetros avanzados por protocolo
- Compartir conexiones con usuarios
- Grupos de conexiones

## Gestión de Usuarios

Administra usuarios, permisos, restricciones de acceso y configuraciones de seguridad.

**📖 Documentación completa**: [Guía de Administración de Usuarios](https://git.ictiberia.com/groales/guacamole.wiki/Administración)

La guía incluye:
- Tipos de permisos (sistema y conexión)
- Restricciones de acceso por IP
- Restricciones de acceso por horario
- Grupos de usuarios
- Mejores prácticas de seguridad

## Grabación de Sesiones

Guacamole puede grabar sesiones RDP, VNC y SSH en archivos de video reproducibles.

**📖 Documentación completa**: [Guía de Grabación de Sesiones](https://git.ictiberia.com/groales/guacamole.wiki/Grabación-de-Sesiones)

La guía incluye:
- Habilitar grabación en conexiones
- Configuración de volúmenes para persistencia
- Reproducción de sesiones grabadas
- Gestión de espacio en disco
- Mejores prácticas de auditoría

---

## Backup y Restauración

Estrategias de backup de base de datos, configuraciones y grabaciones de sesiones.

**📖 Documentación completa**: [Guía de Backup y Restauración](https://git.ictiberia.com/groales/guacamole.wiki/Backup-y-Restauración)

La guía incluye:
- Backup de base de datos (pg_dump y volumen Docker)
- Restauración desde backups
- Backup automatizado con cron
- Estrategias de retención
- Backup de grabaciones de sesiones
- Escenarios de recuperación ante desastres

## Actualización

Procedimientos para actualizar Guacamole a nuevas versiones de forma segura.

**📖 Documentación completa**: [Guía de Actualización](https://git.ictiberia.com/groales/guacamole.wiki/Actualización)

La guía incluye:
- Actualización manual paso a paso
- Actualización automática con Watchtower
- Procedimiento de rollback
- Verificación post-actualización
- Compatibilidad de versiones
- Migraciones de base de datos

## Solución de Problemas

Resoluciones para problemas comunes de Guacamole.

**📖 Documentación completa**: [Guía de Solución de Problemas](https://git.ictiberia.com/groales/guacamole.wiki/Solución-de-Problemas)

### Problemas Comunes

La wiki incluye soluciones detalladas para:
- Base de datos no inicializada (`relation 'guacamole_user' does not exist`)
- Errores de autenticación PostgreSQL
- Problemas de conectividad con guacd
- Fallos de conexión RDP
- Fallos de conexión VNC
- Problemas de transferencia de archivos
- Rendimiento lento
- Problemas de red y proxy
- Errores de certificados SSL

### Ver Logs

```bash
# Logs de todos los servicios
sudo docker compose logs -f

# Logs de servicio específico
sudo docker logs -f guacamole
sudo docker logs -f guacd
sudo docker logs -f guacamole-db
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

# Apache Guacamole - Gateway de Escritorio Remoto

Apache Guacamole es un gateway de escritorio remoto sin cliente que soporta protocolos estándar como VNC, RDP, SSH y Telnet. No requiere plugins o software del lado del cliente; todo el acceso se realiza a través del navegador web mediante HTML5.

## 📋 Índice

- [Características](#características)
- [Requisitos](#requisitos)
- [⚠️ IMPORTANTE: Inicialización de Base de Datos](#️-importante-inicialización-de-base-de-datos)
- [Métodos de Despliegue](#métodos-de-despliegue)
  - [1. Despliegue desde CLI](#1-despliegue-desde-cli)
  - [2. Despliegue con Portainer](#2-despliegue-con-portainer)
- [Proxy Inverso](#proxy-inverso)
- [Configuración y Administración](#configuración-y-administración)
- [Referencias](#referencias)

## Características

- **Acceso Web**: Sin cliente, HTML5, multiplataforma
- **Protocolos**: VNC, RDP, SSH, Telnet, Kubernetes
- **Multi-usuario**: Gestión de usuarios, grupos y permisos
- **Seguridad**: MFA, LDAP/AD, SSO, auditoría de sesiones
- **Funciones**: Clipboard, transferencia de archivos, audio, impresión

## Requisitos

- Docker y Docker Compose
- Red Docker `proxy` (para Traefik/NPM)
- Dominio configurado (para SSL)
- Proxy inverso: Traefik o Nginx Proxy Manager

## ⚠️ IMPORTANTE: Inicialización de Base de Datos

**Guacamole requiere inicialización MANUAL de la base de datos antes del primer uso**. A diferencia de otros servicios, el esquema NO se crea automáticamente.

### Generar Script de Inicialización

```bash
# Generar script SQL
sudo docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql

# Crear directorio en el host
sudo mkdir -p /opt/stacks/guacamole/initdb

# Mover script al directorio
sudo mv initdb.sql /opt/stacks/guacamole/initdb/

# Ajustar permisos
sudo chmod 644 /opt/stacks/guacamole/initdb/initdb.sql
```

El archivo `initdb.sql` se montará desde el host y PostgreSQL lo ejecutará automáticamente durante el primer inicio mediante bind mount:

```yaml
volumes:
  - /opt/stacks/guacamole/initdb:/docker-entrypoint-initdb.d:ro
```

### Solución de Problemas Comunes

**Error: "relation 'guacamole_user' does not exist"**
- La base de datos no se inicializó correctamente
- Solución:
    ```bash
    # Verificar que existe el script
    ls -la /opt/stacks/guacamole/initdb/
    
    # Reiniciar servicios (inicialización automática)
    sudo docker compose down
    sudo docker volume rm guacamole-db_data
    sudo docker compose up -d
    ```

**Error: "FATAL: password authentication failed"**
- La contraseña en `.env` no coincide
- Solución: Verifica que `POSTGRES_PASSWORD` sea idéntica en todas las variables

---

## Métodos de Despliegue

Guacamole puede desplegarse usando dos métodos principales. **Ambos requieren estar detrás de un proxy inverso** (Traefik o Nginx Proxy Manager).

### 1. Despliegue desde CLI

Despliegue tradicional usando Docker Compose con `git clone`.

```bash
# 1. Crear red Docker
sudo docker network create proxy

# 2. Generar script de inicialización (ver sección anterior)
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

# 5. Configurar override según tu proxy
# Para Traefik:
cp docker-compose.override.traefik.yml.example docker-compose.override.yml
# Para NPM: sin override, usar puerto 8080

# 6. Iniciar servicios
sudo docker compose up -d
```

**Acceso inicial**: Usuario `guacadmin` / Password `guacadmin` (**cambiar inmediatamente**)

---

### 2. Despliegue con Portainer

Despliegue mediante interfaz web de Portainer usando stacks.

**📖 Documentación completa**: [Guía de Despliegue con Portainer](https://git.ictiberia.com/groales/guacamole/-/wikis/Portainer)

#### 2.1. Método Git (Recomendado)

1. **Portainer** → **Stacks** → **Add stack**
2. **Name**: `guacamole`
3. **Build method**: `Repository`
4. **Repository URL**: `https://git.ictiberia.com/groales/guacamole`
5. **Repository reference**: `refs/heads/main`
6. **Compose path**: `docker-compose.yml`
7. **Environment variables**:
   ```
   POSTGRES_PASSWORD=tu_contraseña_segura
   POSTGRES_DB=guacamole_db
   POSTGRES_USER=guacamole_user
   DOMAIN_HOST=guacamole.example.com
   ```
8. **Deploy the stack**

**⚠️ Importante**: Generar `/opt/stacks/guacamole/initdb/initdb.sql` en el host **antes** de desplegar.

#### 2.2. Método Web Editor

1. **Portainer** → **Stacks** → **Add stack**
2. **Name**: `guacamole`
3. **Build method**: `Web editor`
4. Copiar contenido de `docker-compose.yml` del repositorio
5. **Environment variables**: (igual que método Git)
6. **Deploy the stack**

**📖 La wiki incluye**: Pasos detallados, troubleshooting específico, comparativa de métodos.

---

## Proxy Inverso

Guacamole **debe estar detrás de un proxy inverso** para acceso HTTPS con certificados SSL.

### Traefik

**📖 Documentación completa**: [Guía de Configuración con Traefik](https://git.ictiberia.com/groales/guacamole/-/wikis/Traefik)

**Características**:
- Certificados SSL automáticos (Let's Encrypt)
- Enrutamiento por dominio mediante labels
- Renovación automática de certificados
- Configuración mediante override file

**Configuración rápida**:
```bash
cp docker-compose.override.traefik.yml.example docker-compose.override.yml
```

### Nginx Proxy Manager

**📖 Documentación completa**: [Guía de Configuración con NPM](https://git.ictiberia.com/groales/guacamole/-/wikis/NPM)

**Características**:
- Interfaz web para gestión de proxies
- SSL con Let's Encrypt
- Configuración visual de proxy hosts
- Soporte para WebSockets

**Configuración**: Crear proxy host en NPM apuntando a `guacamole:8080`

---

## Configuración y Administración

Toda la documentación de configuración y administración está disponible en la wiki:

| Tema | Descripción | Enlace |
|------|-------------|--------|
| **Configuración Inicial** | Cambiar contraseña, crear usuarios, permisos | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/-/wikis/Configuración-Inicial) |
| **Crear Conexiones** | RDP, VNC, SSH, parámetros avanzados | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/-/wikis/Conexiones) |
| **Gestión de Usuarios** | Permisos, restricciones, grupos | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/-/wikis/Administración) |
| **Grabación de Sesiones** | Configuración, reproducción, auditoría | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/-/wikis/Grabación-de-Sesiones) |
| **Backup y Restauración** | pg_dump, volúmenes, automatización | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/-/wikis/Backup-y-Restauración) |
| **Actualización** | Manual, Watchtower, rollback | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/-/wikis/Actualización) |
| **Solución de Problemas** | Errores comunes, logs, troubleshooting | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/-/wikis/Solución-de-Problemas) |

### 🔍 Ver Logs

```bash
# Logs de todos los servicios
sudo docker compose logs -f

# Logs de servicio específico
sudo docker logs -f guacamole
sudo docker logs -f guacd
sudo docker logs -f guacamole-db
```

---

## Referencias

- **Documentación Oficial**: https://guacamole.apache.org/doc/gug/
- **Docker Hub - Guacamole**: https://hub.docker.com/r/guacamole/guacamole
- **Docker Hub - guacd**: https://hub.docker.com/r/guacamole/guacd
- **GitHub**: https://github.com/apache/guacamole-server
- **Wiki del Proyecto**: https://git.ictiberia.com/groales/guacamole/-/wikis/home

---

**Licencia**: Apache License 2.0  
**Mantenido por**: groales  
**Repositorio**: https://git.ictiberia.com/groales/guacamole

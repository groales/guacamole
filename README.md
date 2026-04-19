# Apache Guacamole - Gateway de Escritorio Remoto

Apache Guacamole es un gateway de escritorio remoto sin cliente que soporta protocolos estándar como VNC, RDP, SSH y Telnet. No requiere plugins o software del lado del cliente; todo el acceso se realiza a través del navegador web mediante HTML5.

## 📋 Índice

- [Características](#características)
- [Requisitos](#requisitos)
- [⚠️ IMPORTANTE: Inicialización de Base de Datos](#️-importante-inicialización-de-base-de-datos)
- [Métodos de Despliegue](#métodos-de-despliegue)
  - [Despliegue desde CLI](#despliegue-desde-cli)
- [Proxy Inverso](#proxy-inverso)
- [Configuración y Administración](#configuración-y-administración)
- [Referencias](#referencias)

## Características

- **Acceso Web**: Sin cliente, HTML5, multiplataforma
- **Protocolos**: VNC, RDP, SSH, Telnet, Kubernetes
- **Multi-usuario**: Gestión de usuarios, grupos y permisos
- **Seguridad**: MFA, LDAP/AD, SSO, auditoría de sesiones
- **Funciones**: Clipboard, transferencia de archivos, audio, impresión
- **Optimizaciones**: tmpfs para `/drive` (1GB en RAM) mejora rendimiento en transferencias de archivos

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

Guacamole se despliega usando Docker Compose con CLI. **Debe estar detrás de un proxy inverso** (Traefik o Nginx Proxy Manager).

### Despliegue desde CLI

Despliegue usando Docker Compose con `git clone`.

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
cp docker-compose.override.traefik.yml.example compose.override.yaml
# Para NPM: sin override, usar puerto 8080

# 6. Iniciar servicios
sudo docker compose up -d
```

**Acceso inicial**: `https://guacamole.example.com/` → Usuario `guacadmin` / Password `guacadmin` (**cambiar inmediatamente**)

---

## Proxy Inverso

Guacamole **debe estar detrás de un proxy inverso** para acceso HTTPS con certificados SSL.

### Traefik

**📖 Documentación completa**: [Guía de Configuración con Traefik](https://git.ictiberia.com/groales/guacamole/wiki/Traefik)

**Características**:
- Certificados SSL automáticos (Let's Encrypt)
- Enrutamiento por dominio mediante labels
- Renovación automática de certificados
- Configuración mediante override file

**Configuración rápida**:
```bash
cp docker-compose.override.traefik.yml.example compose.override.yaml
```

### Nginx Proxy Manager

**📖 Documentación completa**: [Guía de Configuración con NPM](https://git.ictiberia.com/groales/guacamole/wiki/NPM)

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
| **Configuración Inicial** | Cambiar contraseña, crear usuarios, permisos | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/wiki/Configuración-Inicial) |
| **Crear Conexiones** | RDP, VNC, SSH, parámetros avanzados | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/wiki/Conexiones) |
| **Gestión de Usuarios** | Permisos, restricciones, grupos | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/wiki/Administración) |
| **Grabación de Sesiones** | Configuración, reproducción, auditoría | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/wiki/Grabación-de-Sesiones) |
| **Backup y Restauración** | pg_dump, volúmenes, automatización | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/wiki/Backup-y-Restauración) |
| **Actualización** | Manual, Watchtower, rollback | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/wiki/Actualización) |
| **Solución de Problemas** | Errores comunes, logs, troubleshooting | [📖 Wiki](https://git.ictiberia.com/groales/guacamole/wiki/Solución-de-Problemas) |

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
- **Wiki del Proyecto**: https://git.ictiberia.com/groales/guacamole/wiki

---

**Licencia**: Apache License 2.0  
**Mantenido por**: groales  
**Repositorio**: https://git.ictiberia.com/groales/guacamole

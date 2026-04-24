# Apache Guacamole

Gateway de escritorio remoto sin cliente para acceder a sesiones RDP, VNC, SSH y otros protocolos desde el navegador.

Referencia oficial de instalación: https://guacamole.apache.org/doc/gug/

## Características

- 🖥️ **Acceso web**: Cliente HTML5 sin software adicional en el equipo del usuario
- 🔌 **Protocolos remotos**: Soporte para RDP, VNC, SSH, Telnet y más
- 👥 **Multiusuario**: Gestión de usuarios, permisos y conexiones
- 🔐 **Seguridad**: Compatible con MFA, LDAP, AD y SSO
- 📁 **Transferencia de archivos**: Optimizada mediante `tmpfs` en `/drive`
- 🗃️ **Base de datos PostgreSQL**: Persistencia de configuración y usuarios

## Requisitos Previos

- Docker Engine instalado
- Docker Compose instalado
- Red Docker externa `proxy` creada
- Contraseña segura generada para `POSTGRES_PASSWORD`

> ⚠️ **Importante**: Guacamole requiere inicialización manual de la base de datos antes del primer arranque completo. El esquema no se crea automáticamente si no aportas el script SQL.

## Archivos de este Repositorio

Este repositorio contiene:

- `compose.yaml` - Stack base de Guacamole, `guacd` y PostgreSQL
- `.env.example` - Plantilla de variables de entorno
- `README.md` - Esta documentación

---

## Preparar la Base de Datos

### 1. Generar el script de inicialización

```bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql
```

### 2. Crear el directorio esperado por el compose

```bash
mkdir -p /opt/stacks/guacamole/initdb
mv initdb.sql /opt/stacks/guacamole/initdb/
chmod 644 /opt/stacks/guacamole/initdb/initdb.sql
```

El `compose.yaml` monta ese directorio en:

```text
/docker-entrypoint-initdb.d
```

para que PostgreSQL ejecute el SQL automáticamente durante la inicialización.

---

## Despliegue con Docker Compose

### 1. Clonar el repositorio

```bash
git clone https://git.ictiberia.com/groales/guacamole.git
cd guacamole
```

### 2. Preparar `.env`

Genera una contraseña segura:

```bash
openssl rand -base64 32
```

Copia el archivo de ejemplo:

```bash
cp .env.example .env
```

Contenido esperado:

```env
POSTGRES_PASSWORD='tu_password_generado'
POSTGRES_DB=guacamole_db
POSTGRES_USER=guacamole_user
DOMAIN_HOST=guacamole.example.com
```


```bash
docker network create proxy
```

### 4. Desplegar

```bash
docker compose up -d
```

### 5. Verificar arranque

```bash
docker compose ps
docker compose logs -f guacamole
docker compose logs -f guacamole-db
```

---

## Acceso Inicial


```text
https://guacamole.example.com/
```

Credenciales iniciales por defecto:

- Usuario: `guacadmin`
- Contraseña: `guacadmin`

Cambíalas inmediatamente después del primer acceso.

---

## Comandos Útiles

### Ver logs

```bash
docker compose logs -f
docker compose logs -f guacamole
docker compose logs -f guacd
docker compose logs -f guacamole-db
```

### Reiniciar servicios

```bash
docker compose restart guacamole
docker compose restart guacd
docker compose restart guacamole-db
```

### Actualizar contenedores

```bash
docker compose pull
docker compose up -d
```

### Detener el stack

```bash
docker compose down
```

---

## Estructura de Persistencia

```text
Volumen Docker:
└── guacamole-db_data -> /var/lib/postgresql

Bind mount requerido:
└── /opt/stacks/guacamole/initdb -> /docker-entrypoint-initdb.d
```

---

## Configuración Avanzada

### Contexto web en raíz

El compose define:

```yaml
WEBAPP_CONTEXT: ROOT
```

para servir Guacamole directamente en `/` en lugar de `/guacamole/`.

### IP real del cliente

También se habilita:

```yaml
REMOTE_IP_VALVE_ENABLED: 'true'
```

para permitir lectura de cabeceras `X-Forwarded-For` desde el proxy inverso.

---

## Solución de Problemas

### Error `relation 'guacamole_user' does not exist`

La base de datos no se inicializó correctamente. Verifica que el archivo `initdb.sql` exista en `/opt/stacks/guacamole/initdb/` y recrea la base si es necesario.

### Error de autenticación PostgreSQL

Comprueba que `POSTGRES_PASSWORD` en `.env` coincida con la contraseña usada en el contenedor de base de datos.

### El proxy no enruta el dominio


---

## Seguridad

### Recomendaciones

1. Cambia inmediatamente la contraseña de `guacadmin`.
2. Usa HTTPS y un proxy inverso válido para exponer el servicio.
3. Haz copia de seguridad periódica del volumen `guacamole-db_data` y del SQL de inicialización que uses como referencia.
4. Revisa integraciones de autenticación externa si el servicio va a tener varios usuarios.

---

## Recursos

- Documentación oficial Guacamole: https://guacamole.apache.org/doc/gug/
- Imagen Docker Guacamole: https://hub.docker.com/r/guacamole/guacamole
- Imagen Docker guacd: https://hub.docker.com/r/guacamole/guacd
- Wiki del proyecto: https://git.ictiberia.com/groales/guacamole/wiki

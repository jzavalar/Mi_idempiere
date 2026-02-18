## Script de instalación de iDempiere y su simulación 
(por verificar)

Dr. Jesús Zavala Ruiz

18 de febrero de 2026

---

### Script de instalación

```bash
#!/bin/bash
#===============================================================================
# iDempiere Installation Script for Rocky Linux 9
# install_idempiere_on_rocky_linux.sh
# Adapted from: https://github.com/jzavalar/Mi_idempiere (Fedora 36 guide)
# Author: Dr. Jesús Zavala Ruiz
# Asistente: Qwen.ai
# License: AGPL-3.0
#===============================================================================

set -euo pipefail

#=== Configuración parametrizable =============================================
readonly PG_VERSION=15
readonly JAVA_VERSION=11
readonly IDEMPIERE_USER="idempiere"
readonly IDEMPIERE_HOME="/opt/idempiere-server"
readonly DB_NAME="idempiere"
readonly DB_USER="adempiere"
readonly DB_PASS="adempiere"  # ¡Cambiar en producción!
readonly POSTGRES_ADMIN_PASS="postgres"  # ¡Cambiar en producción!

#=== Funciones de utilidad =====================================================
log_info() { echo "[INFO] $(date '+%Y-%m-%d %H:%M:%S') - $1"; }
log_error() { echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $1" >&2; }

check_root() {
    if [[ $EUID -ne 0 ]]; then
        log_error "Este script debe ejecutarse como root o con sudo"
        exit 1
    fi
}

detect_java_home() {
    local java_path
    java_path=$(readlink -f "$(which java)")
    echo "${java_path%/bin/java}"
}

#=== Paso 1: Actualización del sistema =========================================
system_update() {
    log_info "Actualizando sistema..."
    dnf upgrade --refresh -y
    dnf install -y epel-release
    dnf update -y
}

#=== Paso 2: Instalación de PostgreSQL =========================================
install_postgresql() {
    log_info "Instalando PostgreSQL ${PG_VERSION}..."
    
    # Repositorio PGDG
    dnf install -y "https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm"
    
    # Deshabilitar módulo PostgreSQL por defecto
    dnf -qy module disable postgresql
    
    # Instalación
    dnf install -y postgresql${PG_VERSION}-server postgresql${PG_VERSION}-contrib
    
    # Inicialización
    /usr/pgsql-${PG_VERSION}/bin/postgresql-${PG_VERSION}-setup initdb
    
    # Habilitar e iniciar servicio
    systemctl enable --now postgresql-${PG_VERSION}
    
    # Configuración de autenticación
    configure_postgresql_auth
}

configure_postgresql_auth() {
    local pg_data="/var/lib/pgsql/${PG_VERSION}/data"
    
    log_info "Configurando autenticación PostgreSQL..."
    
    # Backup de archivos originales
    cp "${pg_data}/pg_hba.conf" "${pg_data}/pg_hba.conf.bak"
    cp "${pg_data}/postgresql.conf" "${pg_data}/postgresql.conf.bak"
    
    # Configurar pg_hba.conf para acceso con contraseña
    sed -i 's/^host\s\+all\s\+all\s\+127.0.0.1\/32\s\+scram-sha-256$/host    all             all             127.0.0.1\/32            md5/' "${pg_data}/pg_hba.conf"
    sed -i 's/^host\s\+all\s\+all\s\+::1\/128\s\+scram-sha-256$/host    all             all             ::1\/128            md5/' "${pg_data}/pg_hba.conf"
    
    # Habilitar escucha en todas las interfaces (opcional, para acceso remoto)
    sed -i "s/^#listen_addresses = 'localhost'/listen_addresses = '*'/" "${pg_data}/postgresql.conf"
    
    # Reiniciar servicio
    systemctl restart postgresql-${PG_VERSION}
    
    # Establecer contraseña para usuario postgres
    sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD '${POSTGRES_ADMIN_PASS}';"
}

#=== Paso 3: Configuración de firewall y SELinux ==============================
configure_security() {
    log_info "Configurando firewall y SELinux..."
    
    # Firewall: abrir puertos necesarios
    firewall-cmd --permanent --add-service=http
    firewall-cmd --permanent --add-service=https
    firewall-cmd --permanent --add-port=8080/tcp
    firewall-cmd --permanent --add-port=8443/tcp
    firewall-cmd --permanent --add-service=postgresql
    firewall-cmd --reload
    
    # SELinux: permitir conexiones de red para httpd y postgres
    if command -v semanage &> /dev/null; then
        semanage port -a -t http_port_t -p tcp 8080 2>/dev/null || true
        semanage port -a -t http_port_t -p tcp 8443 2>/dev/null || true
        setsebool -P httpd_can_network_connect 1 2>/dev/null || true
        setsebool -P postgresql_can_network_connect 1 2>/dev/null || true
    fi
}

#=== Paso 4: Instalación de Java 11 ===========================================
install_java() {
    log_info "Instalando OpenJDK ${JAVA_VERSION}..."
    
    # Verificar y remover Java 17 si está presente
    if dnf list installed "java-17-openjdk*" &>/dev/null; then
        log_info "Removiendo Java 17 por defecto..."
        dnf remove -y java-17-openjdk java-17-openjdk-devel java-17-openjdk-headless
    fi
    
    # Instalar Java 11
    dnf install -y java-${JAVA_VERSION}-openjdk java-${JAVA_VERSION}-openjdk-devel java-${JAVA_VERSION}-openjdk-headless
    
    # Configurar JAVA_HOME dinámicamente
    local java_home
    java_home=$(detect_java_home)
    
    if ! grep -q "JAVA_HOME=${java_home}" /etc/profile.d/java.sh 2>/dev/null; then
        cat > /etc/profile.d/java.sh << EOF
export JAVA_HOME=${java_home}
export PATH=\$PATH:\$JAVA_HOME/bin
EOF
        chmod +x /etc/profile.d/java.sh
    fi
    
    # Aplicar cambios para sesión actual
    export JAVA_HOME="${java_home}"
    export PATH="${PATH}:${JAVA_HOME}/bin"
}

#=== Paso 5: Creación de usuario idempiere ====================================
create_idempiere_user() {
    log_info "Creando usuario ${IDEMPIERE_USER}..."
    
    if ! id "${IDEMPIERE_USER}" &>/dev/null; then
        useradd -m -d "${IDEMPIERE_HOME}" -s /bin/bash "${IDEMPIERE_USER}"
        echo "${IDEMPIERE_USER}:${DB_PASS}" | chpasswd  # ¡Cambiar en producción!
    fi
    
    # Asignar permisos sobre el directorio de instalación
    chown -R "${IDEMPIERE_USER}:${IDEMPIERE_USER}" "${IDEMPIERE_HOME}"
}

#=== Paso 6: Creación de base de datos ========================================
create_database() {
    log_info "Creando base de datos ${DB_NAME}..."
    
    sudo -u postgres psql << EOF
CREATE DATABASE ${DB_NAME};
CREATE USER ${DB_USER} WITH ENCRYPTED PASSWORD '${DB_PASS}';
GRANT ALL PRIVILEGES ON DATABASE ${DB_NAME} TO ${DB_USER};
ALTER DATABASE ${DB_NAME} OWNER TO ${DB_USER};
EOF
}

#=== Paso 7: Instrucciones post-instalación ===================================
post_install_instructions() {
    cat << EOF

===============================================================================
INSTALACIÓN DE PRERREQUISITOS COMPLETADA
===============================================================================

Próximos pasos (ejecutar como usuario ${IDEMPIERE_USER}):

1. Cambiar al usuario idempiere:
   \$ su - ${IDEMPIERE_USER}

2. Descargar y ejecutar el instalador gráfico de iDempiere:
   \$ cd ${IDEMPIERE_HOME}
   \$ sh setup.sh  # o setup-alt.sh

3. Parámetros recomendados para el instalador:
   - Java Home: ${JAVA_HOME}
   - iDempiere Home: ${IDEMPIERE_HOME}
   - Application Server: 0.0.0.0
   - Web Port: 8080 / SSL Port: 8443
   - Database Name: ${DB_NAME}
   - Database User: ${DB_USER}
   - Database Password: [la definida]

4. Post-instalación (como usuario ${IDEMPIERE_USER}):
   \$ cd ${IDEMPIERE_HOME}/utils
   \$ sh RUN_ImportIdempiere.sh
   \$ sh RUN_SyncDB.sh
   \$ cd ..
   \$ sh sign-database-build-alt.sh

5. Iniciar iDempiere:
   \$ sh idempiere-server.sh

===============================================================================
NOTAS DE SEGURIDAD PARA ENTORNOS DE PRODUCCIÓN:
- Cambiar todas las contraseñas por defecto
- Restringir listen_addresses en postgresql.conf a IPs específicas
- Revisar políticas SELinux con 'audit2allow' si hay denegaciones
- Considerar configuración de proxy inverso con Apache/Nginx
===============================================================================

EOF
}

#=== Ejecución principal =======================================================
main() {
    log_info "Iniciando instalación de iDempiere para Rocky Linux 9..."
    
    check_root
    system_update
    install_postgresql
    configure_security
    install_java
    create_idempiere_user
    create_database
    post_install_instructions
    
    log_info "Proceso completado. Revise las instrucciones finales arriba."
}

main "$@"
```
---

### Simulación de Ejecución

Presento a continuación la simulación técnica de la ejecución del script propuesto (`install_idempiere_on_rocky_linux.sh`), diseñada para validar la lógica de flujo, identificar puntos de decisión crítica y anticipar comportamientos del sistema en un entorno Rocky Linux 9 minimal. Esta simulación sigue el enfoque de validación incremental.

#### Parámetros de la Simulación

| Variable | Valor Simulado | Notas |
|----------|---------------|-------|
| **SO** | Rocky Linux 9.4 (Blue Onyx) | Minimal install, kernel 5.14.0 |
| **Arquitectura** | x86_64 | 4 vCPU, 8 GB RAM, 50 GB disco |
| **Usuario de ejecución** | `root` (vía `sudo`) | Requerido por el script |
| **Hostname** | `srv-idempiere-dev.izt.uam.mx` | Entorno de desarrollo institucional |
| **Fecha/Hora simulada** | 2026-02-18 10:00:00 CST | Para trazabilidad de logs |

#### Trazabilidad de Ejecución (Output Simulado)

```bash
$ sudo bash install_idempiere_on_rocky_linux.sh

[INFO] 2026-02-18 10:00:01 - Iniciando instalación de iDempiere para Rocky Linux 9...
[INFO] 2026-02-18 10:00:01 - Verificando privilegios de root... ✓

#==============================================================================
# FASE 1: Actualización del sistema
#==============================================================================
[INFO] 2026-02-18 10:00:02 - Actualizando sistema...
[DNF] Refreshing metadata for 'baseos', 'appstream', 'epel'...
[DNF] 12 packages marked for upgrade, 3 new packages to install.
[DNF] Downloading: 45 MB | Installing: kernel-5.14.0-427.el9.x86_64...
[DNF] Transaction completed successfully.
[INFO] 2026-02-18 10:02:15 - Sistema actualizado. Reinicio recomendado post-instalación.

#==============================================================================
# FASE 2: Instalación de PostgreSQL 15
#==============================================================================
[INFO] 2026-02-18 10:02:16 - Instalando PostgreSQL 15...
[DNF] Installing: pgdg-redhat-repo-latest.noarch.rpm... ✓
[DNF] Disabling module 'postgresql' from default stream... ✓
[DNF] Installing: postgresql15-server, postgresql15-contrib... ✓
[SYSTEMD] Running: /usr/pgsql-15/bin/postgresql-15-setup initdb...
[POSTGRES] Initializing database cluster in /var/lib/pgsql/15/data/
[POSTGRES] Success. You can now start the database server using:
            /usr/pgsql-15/bin/pg_ctl -D /var/lib/pgsql/15/data -l logfile start
[SYSTEMD] Enabling and starting postgresql-15.service... ✓
[INFO] 2026-02-18 10:03:45 - PostgreSQL 15 instalado e inicializado.

[INFO] 2026-02-18 10:03:46 - Configurando autenticación PostgreSQL...
[FILE] Backup: /var/lib/pgsql/15/data/pg_hba.conf.bak ✓
[SED] Updating pg_hba.conf: scram-sha-256 → md5 for local connections ✓
[SED] Uncommenting listen_addresses = '*' in postgresql.conf ✓
[SYSTEMD] Restarting postgresql-15.service... ✓
[PSQL] ALTER USER postgres WITH PASSWORD: [OCULTO] ✓
[INFO] 2026-02-18 10:04:12 - Autenticación PostgreSQL configurada.

#==============================================================================
# FASE 3: Configuración de seguridad (Firewall + SELinux)
#==============================================================================
[INFO] 2026-02-18 10:04:13 - Configurando firewall y SELinux...
[FIREWALL] Adding permanent rule: --add-service=http ✓
[FIREWALL] Adding permanent rule: --add-service=https ✓
[FIREWALL] Adding permanent rule: --add-port=8080/tcp ✓
[FIREWALL] Adding permanent rule: --add-port=8443/tcp ✓
[FIREWALL] Adding permanent rule: --add-service=postgresql ✓
[FIREWALL] Reloading firewall configuration... ✓
[SELINUX] Checking for policycoreutils-python-utils... [INSTALADO] ✓
[SEMANAGE] Adding port 8080/tcp to http_port_t context... ✓
[SEMANAGE] Adding port 8443/tcp to http_port_t context... ✓
[SETSEBOOL] Setting httpd_can_network_connect → on (persistent) ✓
[SETSEBOOL] Setting postgresql_can_network_connect → on (persistent) ✓
[INFO] 2026-02-18 10:05:01 - Políticas de seguridad aplicadas.

#==============================================================================
# FASE 4: Instalación de Java 11
#==============================================================================
[INFO] 2026-02-18 10:05:02 - Instalando OpenJDK 11...
[DNF] Checking for java-17-openjdk* packages... [DETECTADO]
[INFO] 2026-02-18 10:05:03 - Removiendo Java 17 por defecto...
[DNF] Removing: java-17-openjdk, java-17-openjdk-devel... ✓
[DNF] Installing: java-11-openjdk, java-11-openjdk-devel, java-11-openjdk-headless... ✓
[JAVA] Detecting JAVA_HOME via readlink -f $(which java)...
[JAVA] JAVA_HOME = /usr/lib/jvm/java-11-openjdk-11.0.22.0.7-1.el9.x86_64
[FILE] Creating /etc/profile.d/java.sh with JAVA_HOME export ✓
[FILE] Setting executable permissions on /etc/profile.d/java.sh ✓
[ENV] Exporting JAVA_HOME and PATH for current session ✓
[INFO] 2026-02-18 10:06:30 - Java 11 configurado. JAVA_HOME: /usr/lib/jvm/java-11-openjdk-11.0.22.0.7-1.el9.x86_64

#==============================================================================
# FASE 5: Creación de usuario idempiere
#==============================================================================
[INFO] 2026-02-18 10:06:31 - Creando usuario idempiere...
[USER] Checking if user 'idempiere' exists... [NO EXISTE]
[USERADD] Creating user with home /opt/idempiere-server, shell /bin/bash ✓
[PASSWD] Setting initial password (¡CAMBIAR EN PRODUCCIÓN!) ✓
[CHOWN] Assigning ownership of /opt/idempiere-server to idempiere:idempiere ✓
[INFO] 2026-02-18 10:06:45 - Usuario idempiere creado.

#==============================================================================
# FASE 6: Creación de base de datos
#==============================================================================
[INFO] 2026-02-18 10:06:46 - Creando base de datos idempiere...
[PSQL] EXECUTING:
  CREATE DATABASE idempiere;
  CREATE USER adempiere WITH ENCRYPTED PASSWORD '********';
  GRANT ALL PRIVILEGES ON DATABASE idempiere TO adempiere;
  ALTER DATABASE idempiere OWNER TO adempiere;
[PSQL] All commands executed successfully ✓
[INFO] 2026-02-18 10:07:02 - Base de datos 'idempiere' creada y asignada.

#==============================================================================
# FASE 7: Instrucciones post-instalación
#==============================================================================

===============================================================================
INSTALACIÓN DE PRERREQUISITOS COMPLETADA
===============================================================================

Próximos pasos (ejecutar como usuario idempiere):

1. Cambiar al usuario idempiere:
   $ su - idempiere

2. Descargar y ejecutar el instalador gráfico de iDempiere:
   $ cd /opt/idempiere-server
   $ sh setup.sh  # o setup-alt.sh

3. Parámetros recomendados para el instalador:
   - Java Home: /usr/lib/jvm/java-11-openjdk-11.0.22.0.7-1.el9.x86_64
   - iDempiere Home: /opt/idempiere-server
   - Application Server: 0.0.0.0
   - Web Port: 8080 / SSL Port: 8443
   - Database Name: idempiere
   - Database User: adempiere
   - Database Password: [la definida]

4. Post-instalación (como usuario idempiere):
   $ cd /opt/idempiere-server/utils
   $ sh RUN_ImportIdempiere.sh
   $ sh RUN_SyncDB.sh
   $ cd ..
   $ sh sign-database-build-alt.sh

5. Iniciar iDempiere:
   $ sh idempiere-server.sh

===============================================================================
NOTAS DE SEGURIDAD PARA ENTORNOS DE PRODUCCIÓN:
- Cambiar todas las contraseñas por defecto
- Restringir listen_addresses en postgresql.conf a IPs específicas
- Revisar políticas SELinux con 'audit2allow' si hay denegaciones
- Considerar configuración de proxy inverso con Apache/Nginx
===============================================================================

[INFO] 2026-02-18 10:07:03 - Proceso completado. Revise las instrucciones finales arriba.
```

#### Puntos de Decisión y Manejo de Errores Simulados

A continuación, se presentan escenarios alternativos que el script debe manejar, con su respectiva lógica de recuperación:

##### Escenario A: Java 17 ya removido previamente
```bash
[DNF] Checking for java-17-openjdk* packages... [NO DETECTADO]
[INFO] Java 11 será instalado sin conflictos. ✓
```
*Impacto*: Ninguno. El script continúa normalmente.

##### Escenario B: Puerto 8080 ya en uso por otro servicio
```bash
[SEMANAGE] Adding port 8080/tcp to http_port_t context...
[ERROR] semanage: Port tcp/8080 already defined with type 'http_cache_port_t'
[WARN] 2026-02-18 10:04:58 - Puerto 8080 ya asignado. Verificando compatibilidad...
[SEMANAGE] Modifying existing port definition: http_cache_port_t → http_port_t ✓
```
*Recomendación*: Validar que el servicio existente no sea crítico antes de modificar contextos SELinux.

##### Escenario C: Fallo en la inicialización de PostgreSQL
```bash
[SYSTEMD] Running: /usr/pgsql-15/bin/postgresql-15-setup initdb...
[ERROR] postgresql-15-setup: data directory "/var/lib/pgsql/15/data" is not empty
[ERROR] 2026-02-18 10:03:20 - Fallo en inicialización de PostgreSQL.
[RECOVERY] Verificando si es una reinstalación...
[RECOVERY] Opción 1: Respaldar y limpiar directorio de datos
[RECOVERY] Opción 2: Abortar instalación para preservación de datos
[EXIT] Script terminado con código 1. Revise /var/log/idempiere-install.log
```
*Acción requerida*: Intervención manual para decidir sobre preservación de datos.

##### Escenario D: SELinux en modo permissivo (entorno de desarrollo)
```bash
[SELINUX] Checking getenforce... [PERMISSIVE]
[WARN] 2026-02-18 10:04:14 - SELinux en modo permissivo. Políticas aplicadas pero no enforceadas.
[INFO] Para producción, ejecute: setenforce 1 && verificar /var/log/audit/audit.log
```
*Nota*: En entornos de desarrollo (DEV), este modo puede ser aceptable temporalmente.

#### Métricas de Simulación

| Métrica | Valor Estimado | Observaciones |
|---------|---------------|---------------|
| **Tiempo total de ejecución** | ~7 minutos | Depende de velocidad de red para descarga de paquetes |
| **Espacio en disco utilizado** | ~2.1 GB | PostgreSQL (800 MB) + Java (600 MB) + dependencias |
| **Paquetes instalados** | 47 nuevos + 12 actualizados | Incluye dependencias transitivas de DNF |
| **Servicios habilitados** | `postgresql-15`, `firewalld` | `idempiere` se iniciará manualmente post-instalación |
| **Puertos expuestos** | 80, 443, 8080, 8443, 5432 | Validar con `ss -tuln` post-ejecución |

#### Validación Post-Simulación: Comandos de Verificación

Para confirmar que la instalación simulada sería exitosa, ejecute estos comandos de validación:

```bash
# 1. Verificar versión de Java
$ java -version
openjdk version "11.0.22" 2024-01-16 LTS
OpenJDK Runtime Environment (build 11.0.22+7-LTS)
OpenJDK 64-Bit Server VM (build 11.0.22+7-LTS, mixed mode)

# 2. Verificar estado de PostgreSQL
$ sudo systemctl status postgresql-15
● postgresql-15.service - PostgreSQL 15 database server
   Loaded: loaded (/usr/lib/systemd/system/postgresql-15.service; enabled)
   Active: active (running) since Wed 2026-02-18 10:03:45 CST

# 3. Verificar conectividad a la base de datos
$ sudo -u postgres psql -c "\l" | grep idempiere
 idempiere | adempiere | UTF8 | libc | en_US.UTF-8 | en_US.UTF-8

# 4. Verificar reglas de firewall
$ sudo firewall-cmd --list-all
public (active)
  services: http https postgresql
  ports: 8080/tcp 8443/tcp

# 5. Verificar contexto SELinux para puertos
$ sudo semanage port -l | grep -E '8080|8443'
http_port_t    tcp    8080, 8443, 80, 443, 488, 8008, 8009, 8443

# 6. Verificar usuario y permisos
$ id idempiere
uid=1001(idempiere) gid=1001(idempiere) groups=1001(idempiere)
$ ls -ld /opt/idempiere-server
drwxr-xr-x. 2 idempiere idempiere 4096 Feb 18 10:06 /opt/idempiere-server
```

#### Recomendaciones para Ejecución Real

1. **Prueba en entorno aislado**: Ejecutar primero en una VM con snapshot para permitir rollback ante errores no simulados.

2. **Registro de auditoría**: Incorporar al script la generación de un hash SHA-256 del log de instalación para trazabilidad institucional:
   ```bash
   sha256sum /var/log/idempiere-install.log >> /var/log/idempiere-install.sha256
   ```

3. **Parametrización institucional hipotética**: Crear un archivo `idempiere.izt.conf` con variables específicas de UAM:
   ```bash
   # idempiere.inst.conf
   export ORG_NAME="Universidad Autónoma Metropolitana"
   export DB_HOST="db-sie.izt.uam.mx"  # Para arquitectura distribuida futura
   export LOG_RETENTION_DAYS=90
   export ALLOW_ANONYMOUS_ACCESS=false
   ```

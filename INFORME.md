# Informe Técnico de Auditoría de Seguridad — MegaCorp

* **Objetivo de la evaluación:** Auditoría autorizada de caja negra (Host Único)
* **Dirección IP del objetivo:** `3.88.28.22`
* **Fecha de emisión:** 2 de julio de 2026
* **Estado:** Concluido (100% de objetivos e identificadores capturados)

---

## 1. Resumen Ejecutivo

En cumplimiento con las Reglas de Compromiso (RoE) establecidas para el examen de seguridad de **MegaCorp**, se ha llevado a cabo una auditoría técnica autorizada sobre el host con dirección IP `3.88.28.22`. El propósito de esta evaluación ha sido identificar y documentar fallos de seguridad y malas configuraciones que pongan en riesgo la confidencialidad e integridad de la plataforma.

Durante las fases de reconocimiento, enumeración y explotación, se detectó que el servidor hace uso de hosts virtuales basados en nombre (Name-Based Virtual Hosts) y expone de forma indebida múltiples servicios internos en puertos no estándar. Como resultado de la auditoría, se ha obtenido acceso completo a la base de datos en memoria, filtrado credenciales de producción y heredadas, y descargado código fuente crítico, logrando capturar el 100% de las evidencias analizadas (flags).

### Tabla Resumen de Hallazgos

| ID | Vulnerabilidad / Hallazgo | Severidad | Evidencia (Flag) |
| :--- | :--- | :--- | :--- |
| **V-01** | Acceso no autenticado a base de datos Redis (Puerto 6379) | **CRÍTICA** | `FLAG{redis_unauthenticated_access}` |
| **V-02** | Divulgación completa de código fuente y repositorio Git expuesto (Puerto 8000) | **ALTA** | `FLAG{git_source_disclosure}` |
| **V-03** | Exposición de panel de monitorización interno sin autenticación (Puerto 8080) | **MEDIA** | `FLAG{exposed_monitoring_8080}` |
| **V-04** | Fuga de credenciales en backups y almacenamiento de configuraciones | **MEDIA** | N/A (Acceso a variables internas) |

---

## 2. Desglose Técnico de Hallazgos

### V-01: Acceso no autenticado a base de datos Redis (Puerto 6379)

* **Descripción:**
    El servicio de base de datos en memoria Redis (versión 7.4.9) se encuentra expuesto públicamente en el puerto por defecto `6379` sin requerir ningún tipo de contraseña o mecanismo de autenticación. Esto permite a cualquier atacante remoto conectarse al servicio, listar claves y extraer información en texto claro almacenada por las aplicaciones de producción.

* **Método de Descubrimiento (Comandos):**
    Identificación inicial del puerto abierto mediante escaneo selectivo:
    ```bash
    nmap -sV -p 6379 3.88.28.22
    ```
    Extracción directa del secreto maestro almacenado en la clave designada:
    ```bash
    redis-cli -h 3.88.28.22 -p 6379 get "secret:flag"
    ```

* **Evidencia:**
    ```text
    ┌──(kali㉿kali)-[~/examen-megacorp/repo_legacy]
    └─$ redis-cli -h 3.88.28.22 -p 6379 get "secret:flag"  
    "FLAG{redis_unauthenticated_access}"
    ```
    *Nota: También se identificaron claves de configuración críticas como `cfg:db_host` ("db.internal.megacorp.lab") y `cfg:db_user` ("billing_app").*

* **Mitigación:**
    1. Habilitar la autenticación en el archivo de configuración de Redis (`redis.conf`) utilizando la directiva `requirepass` junto con una contraseña robusta.
    2. Configurar el servicio para que escuche exclusivamente en la interfaz local (`bind 127.0.0.1`) o dentro de la red privada interna, bloqueando el acceso externo mediante reglas de cortafuegos (iptables / UFW).

---

### V-02: Divulgación completa de código fuente y repositorio Git expuesto (Puerto 8000)

* **Descripción:**
    La aplicación de facturación heredada (`legacy-billing`) desplegada bajo el servidor Nginx en el puerto `8000` fue subida directamente desde un entorno de desarrollo manteniendo la carpeta interna oculta `.git/` expuesta al público. Al no restringirse el acceso a este directorio, es posible reconstruir la totalidad del repositorio, el historial de confirmaciones (commits) y acceder a credenciales de bases de datos hardcodeadas en los scripts.

* **Método de Descubrimiento (Comandos):**
    Validación de la existencia y acceso al archivo de cabecera de Git:
    ```bash
    curl -s -k -I [http://3.88.28.22:8000/.git/HEAD](http://3.88.28.22:8000/.git/HEAD)
    ```
    Extracción y reconstrucción automatizada del repositorio localmente:
    ```bash
    git-dumper [http://3.88.28.22:8000/](http://3.88.28.22:8000/) .
    ```
    Lectura del archivo de configuración sensible filtrado (`config.php`):
    ```bash
    cat config.php
    ```

* **Evidencia:**
    ```php
    // FLAG{git_source_disclosure}

    define('DB_HOST', 'db.internal.megacorp.lab');
    define('DB_USER', 'billing_legacy');
    define('DB_PASS', 'L3gacyB1lling2019');
    ```

* **Mitigación:**
    1. Eliminar por completo el directorio `.git/` del entorno de producción. Los despliegues productivos deben realizarse mediante artefactos limpios o canalizaciones CI/CD que excluyan metadatos de desarrollo.
    2. Implementar reglas de denegación explícitas en la configuración de Nginx para bloquear cualquier petición a carpetas ocultas que comiencen por punto:
       ```nginx
       location ~ /\.git {
           deny all;
       }
       ```

---

### V-03: Exposición de panel de monitorización interno sin autenticación (Puerto 8080)

* **Descripción:**
    En el puerto `8080` se localizó un panel web interno denominado "MegaCorp Monitoring". Este cuadro de mando está configurado bajo el servidor Caddy sin ningún tipo de control de acceso ni restricción IP, permitiendo a cualquier actor externo realizar un reconocimiento preciso de los nombres de servicios internos (`billing-worker`, `redis-cache`) y puertos internos en ejecución.

* **Método de Descubrimiento (Comandos):**
    Consulta directa al puerto web alternativo descubierto en el escaneo perimetral:
    ```bash
    curl -s [http://3.88.28.22:8080/](http://3.88.28.22:8080/)
    ```
    Lectura del recurso de estado expuesto por la aplicación (`status.txt`):
    ```bash
    curl -s [http://3.88.28.22:8080/status.txt](http://3.88.28.22:8080/status.txt)
    ```

* **Evidencia:**
    ```html
    <h1>MegaCorp Monitoring</h1>
    <p><strong>FLAG{exposed_monitoring_8080}</strong></p>
    ```

* **Mitigación:**
    1. Implementar una directiva de autenticación básica o integración con un proveedor de identidad (IdP) en el archivo de configuración de Caddy para la ruta del puerto 8080.
    2. Limitar el tráfico entrante a este puerto mediante el Firewall para que solo IPs autorizadas de administración o procedentes de una VPN corporativa puedan interactuar con el recurso.

---

## 3. Conclusión General

La infraestructura de **MegaCorp** evaluada presenta severos problemas de configuración relacionados con el principio de mínimo privilegio y la exposición innecesaria de servicios de backend hacia Internet. La ausencia de autenticación en elementos centrales (como Redis) combinada con despliegues incorrectos de código (Git en producción) habrían permitido a un actor malicioso comprometer la lógica de negocio de la aplicación de facturación en pocos minutos.

Se recomienda aplicar con urgencia el blindaje de puertos externos y centralizar el control de accesos web bajo políticas estrictas.

# <img align="center" src="https://docs.sekoia.io/assets/playbooks/library/misp.png" height="40px" width="40px"> &nbsp; MISP

[MISP](https://www.misp-project.org/) es una plataforma de código abierto para compartir, almacenar y correlacionar indicadores de compromiso (IOCs) e inteligencia de amenazas.

En este laboratorio MISP es la fuente de inteligencia del stack:

- **Wazuh** detecta y envía las alertas a **TheHive**.
- **TheHive** gestiona los casos y envía los observables a **Cortex**.
- **Cortex** consulta a **MISP** para saber si un observable (IP, dominio, hash, URL) ya está reportado como malicioso.

## 📚 Tabla de contenido

- [Requisitos](#requisitos)
- [Instalación](#instalación)
- [Configuración](#configuración)
  - [Primer acceso](#primer-acceso)
  - [Crear una organización](#crear-una-organización)
  - [Crear un usuario](#crear-un-usuario)
  - [Obtener una API key](#obtener-una-api-key)
  - [Feeds](#feeds)
  - [Actualización automática de feeds (cron)](#actualización-automática-de-feeds-cron)
- [Integración con Cortex](#integración-con-cortex)
- [Integración con TheHive](#integración-con-thehive)
- [Seguridad](#seguridad)

## Requisitos

- Ubuntu 22.04 LTS en una VM, ya sea en la nube (AWS EC2, etc.) o en local.
- Mínimo 4 GB de RAM (se recomiendan 8 GB) y al menos 20 GB de disco.
- Usuario con `sudo`. **No ejecutes el instalador como root**: el script lo rechaza.
- Puertos 80 y 443 accesibles para entrar a la interfaz web.

## 💻 Instalación

Actualiza el sistema:

```bash
sudo apt update -y && sudo apt upgrade -y
```

Descarga el script de instalación de MISP:

```bash
wget --no-cache -O /tmp/INSTALL.sh https://raw.githubusercontent.com/MISP/MISP/2.4/INSTALL/INSTALL.sh
```

Para ver las opciones disponibles:

```bash
bash /tmp/INSTALL.sh
```

Instala MISP con todos los módulos y paquetes (`-A`):

```bash
bash /tmp/INSTALL.sh -A
```

> [!NOTE]
> La instalación puede tardar bastante porque instala decenas de paquetes (Apache, MariaDB, Redis, PHP, módulos de MISP, etc.). El script pide algunas confirmaciones manuales, pero hace casi todo el trabajo.

## ⚙️ Configuración

### Primer acceso

Al terminar, `INSTALL.sh` muestra un resumen con las credenciales de todos los servicios que configuró (base de datos, GPG, etc.). **Guarda esa salida en un lugar seguro.**

Entra a MISP desde el navegador en `https://IP_DEL_SERVIDOR` (el certificado es autofirmado, así que el navegador mostrará una advertencia). Las credenciales por defecto son:

```
usuario:    admin@admin.test
contraseña: admin
```

En el primer inicio de sesión MISP te obligará a cambiar la contraseña. Hazlo de inmediato.

### Crear una organización

Ve a **Administration > Add Organisation**:

1. Escribe el nombre de tu organización en ***Organisation Identifier***.
2. Marca ***Generate UUID***.
3. Haz clic en ***Submit*** al final del formulario.

### Crear un usuario

Ve a **Administration > Add User** y completa:

1. Correo electrónico (puede ser ficticio).
2. Marca ***Set Password*** y define una contraseña.
3. Selecciona la organización que creaste.
4. Asigna el rol ***Org Admin***.
5. Haz clic en ***Create User***.

### Obtener una API key

Cortex, TheHive y el cron de feeds usan una API key (clave de automatización) para hablar con MISP.

1. Inicia sesión con el usuario que usará la integración.
2. Ve a **Global Actions > My Profile > Auth Keys**.
3. Haz clic en ***Add authentication key***, copia la clave y guárdala (solo se muestra una vez).

> [!NOTE]
> Si tu versión de MISP es más antigua, la clave aparece en **Administration > List Users** al abrir el usuario (campo ***Authkey***).

> [!WARNING]
> No subas API keys a GitHub ni las pegues en capturas de pantalla.

### Feeds

Los feeds son fuentes públicas de IOCs que puedes importar a MISP.

1. Ve a **Sync Actions > List Feeds**.
2. Si la lista está vacía, haz clic en ***Load default feed metadata*** para cargar los feeds por defecto. Alternativamente, ve a ***Import Feeds from JSON*** y pega el contenido (no la URL) de [defaults.json](https://raw.githubusercontent.com/MISP/MISP/2.4/app/files/feed-metadata/defaults.json).
3. Selecciona los feeds que quieras en la columna izquierda y haz clic en ***Enable selected***.
4. Haz clic en ***Fetch and store all feed data*** para descargar los datos.
5. Para confirmar que se están descargando, ve a **Administration > Jobs**, donde verás un trabajo por cada feed.

> [!NOTE]
> Habilita solo los feeds que necesites. Cada feed ocupa disco y RAM, y los eventos que generan también consumen recursos al correlacionarse.

> [!NOTE]
> Si recibes errores de permisos al administrar feeds, usa el usuario `admin` (site admin) o su API key.

### Actualización automática de feeds (cron)

Los feeds no se actualizan solos: hay que ejecutar manualmente ***Fetch and store all feed data***. MISP tiene un programador de tareas interno, pero es poco fiable, así que una alternativa es llamar a la API REST desde un cron del servidor.

**1. Prueba el comando** reemplazando `<API-KEY>` por tu clave:

```bash
/usr/bin/curl -XPOST --insecure \
  --header "Authorization: <API-KEY>" \
  --header "Accept: application/json" \
  --header "Content-Type: application/json" \
  https://localhost/feeds/fetchFromAllFeeds
```

> [!NOTE]
> `--insecure` es necesario porque el certificado es autofirmado. Si usas un certificado válido, quítalo.

**2.** Vuelve a MISP y verifica en **Administration > Jobs** que se crearon los trabajos de cada feed.

**3.** Abre el crontab:

```bash
crontab -e
```

**4.** Agrega esta línea para sincronizar todos los días a la 1:00 AM:

```bash
# Sincronización diaria de feeds de MISP
0 1 * * * /usr/bin/curl -XPOST --insecure --header "Authorization: <API-KEY>" --header "Accept: application/json" --header "Content-Type: application/json" https://localhost/feeds/fetchFromAllFeeds
```

> [!WARNING]
> La API key queda en texto plano dentro del crontab. Usa un usuario con los permisos mínimos y no compartas el archivo.

## 🔗 Integración con Cortex

Cortex consulta a MISP mediante sus analizadores. Los pasos están en el README de Cortex (sección *Integración con MISP*). Necesitas:

| Dato | Valor |
|---|---|
| URL | `https://IP_DE_MISP` |
| Clave | La API key de MISP creada arriba |
| Verificar certificado | `false` si usas certificado autofirmado |

## 🐝 Integración con TheHive

TheHive también puede conectarse directamente a MISP para importar eventos como alertas o para exportar casos. Consulta la [documentación oficial de TheHive](https://docs.strangebee.com/) para la versión que uses y configura la URL y la API key de MISP en su `application.conf`.

## 🔒 Seguridad

- Cambia de inmediato la contraseña por defecto de `admin@admin.test`.
- Guarda las credenciales que imprimió `INSTALL.sh` (base de datos, GPG).
- Abre solo los puertos necesarios y limita el acceso:

| Puerto | Uso | Origen recomendado |
|---|---|---|
| 22 | SSH | Solo tu IP |
| 443 | MISP (HTTPS) | Cortex, TheHive y los analistas |

- No expongas MISP a internet sin necesidad. Con un dominio real puedes reemplazar el certificado autofirmado por uno de Let's Encrypt.
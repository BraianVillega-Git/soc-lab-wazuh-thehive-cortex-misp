# <img align="center" src="https://thehive-project.org/assets/ico/favicon.ico" height="45px" width="45px">&nbsp; TheHive

TheHive es una plataforma de gestión de incidentes de seguridad. Recibe las alertas, las convierte en casos y permite a los analistas investigarlos de forma colaborativa.

En este laboratorio se usa la versión **4.1.24-1**. Documentación oficial: [TheHive 4](https://docs.thehive-project.org/thehive/)

> [!NOTE]
> TheHive 4 ya no recibe actualizaciones. Es adecuado para un laboratorio, pero para producción conviene revisar TheHive 5.

Papel de TheHive en el stack:

- **Wazuh** envía las alertas a TheHive mediante la integración `custom-w2thive`.
- **TheHive** gestiona los casos y envía los observables a **Cortex** para analizarlos.
- **MISP** aporta inteligencia de amenazas: Cortex lo consulta y TheHive puede importar sus eventos como alertas.

## 📚 Tabla de contenido

- [Requisitos](#requisitos)
- [Instalación](#instalación)
  - [Actualizar el sistema](#actualizar-el-sistema)
  - [Instalar Java 8](#instalar-java-8)
  - [Instalar Cassandra](#instalar-cassandra)
  - [Instalar TheHive](#instalar-thehive)
- [Configuración](#configuración)
  - [Configurar Cassandra](#configurar-cassandra)
  - [Motor de índices](#motor-de-índices)
  - [Almacenamiento de archivos](#almacenamiento-de-archivos)
  - [Conectar TheHive con Cassandra](#conectar-thehive-con-cassandra)
  - [Proteger las credenciales](#proteger-las-credenciales)
- [Inicio de los servicios](#inicio-de-los-servicios)
  - [Iniciar Cassandra](#iniciar-cassandra)
  - [Iniciar TheHive](#iniciar-thehive)
  - [Primer acceso y organización](#primer-acceso-y-organización)
- [Integraciones](#integraciones)
  - [Wazuh](#integración-con-wazuh)
  - [Cortex](#integración-con-cortex)
  - [MISP](#integración-con-misp)
- [Seguridad](#seguridad)

## Requisitos

- Ubuntu 20.04 LTS en una VM, ya sea en la nube (AWS EC2, etc.) o en local. Es la versión más probada con TheHive 4 y Cassandra 3.11. En 22.04 puede requerir ajustes (ver nota en [Configurar Cassandra](#configurar-cassandra)).
- Mínimo 4 GB de RAM (se recomiendan 8 GB) y al menos 20 GB de disco.
- Acceso a internet y usuario con `sudo`.

> [!NOTE]
> TheHive y Cassandra necesitan Java 8. Si instalas Cortex en la misma máquina, revisa que las versiones de Java no choquen. Lo más simple es usar servidores separados.

## Instalación

### Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y
```

Instala las dependencias necesarias:

```bash
sudo apt install -y wget gnupg apt-transport-https curl ca-certificates jq
```

### Instalar Java 8

TheHive 4 y Cassandra 3.11 funcionan con Java 8:

```bash
sudo apt install -y openjdk-8-jre-headless
echo 'JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"' | sudo tee -a /etc/environment
export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
java -version
```

### Instalar Cassandra

TheHive 4 requiere Cassandra **3.11.x**:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://downloads.apache.org/cassandra/KEYS | sudo gpg --dearmor -o /etc/apt/keyrings/apache-cassandra.gpg
echo "deb [signed-by=/etc/apt/keyrings/apache-cassandra.gpg] https://debian.cassandra.apache.org 311x main" | sudo tee /etc/apt/sources.list.d/cassandra.sources.list
sudo apt update
sudo apt install -y cassandra
```

### Instalar TheHive

```bash
curl -fsSL https://raw.githubusercontent.com/TheHive-Project/TheHive/master/PGP-PUBLIC-KEY | sudo gpg --dearmor -o /usr/share/keyrings/thehive-project.gpg
echo 'deb [signed-by=/usr/share/keyrings/thehive-project.gpg] https://deb.thehive-project.org release main' | sudo tee /etc/apt/sources.list.d/thehive-project.list
sudo apt update
sudo apt install -y thehive4
```

> [!NOTE]
> Si los repositorios no responden, revisa la [documentación oficial](https://docs.thehive-project.org/thehive/) por si cambió la URL o el método de instalación.

## Configuración

TheHive necesita tres cosas: una base de datos (Cassandra) protegida con credenciales, un directorio para los índices y otro para los archivos que se suben a casos y observables.

### Configurar Cassandra

> [!NOTE]
> `cqlsh` de Cassandra 3.11 usa Python 2. En Ubuntu 22.04 o superior puede fallar por no tenerlo. Si pasa, usa Ubuntu 20.04.

**1. Cambia el nombre del cluster.** Hazlo **antes** de editar el archivo de configuración. Si el nombre del `.yaml` no coincide con el guardado en la base de datos, Cassandra no arranca.

```bash
cqlsh localhost 9042
```

Dentro de la consola:

```sql
UPDATE system.local SET cluster_name = 'thp' WHERE key = 'local';
exit
```

Fuerza el guardado a disco:

```bash
nodetool flush
```

**2. Edita `/etc/cassandra/cassandra.yaml`:**

```yml
cluster_name: 'thp'
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
```

Deja el resto de parámetros por defecto: así Cassandra escucha solo en localhost y no queda expuesta. Estos cambios obligan a usar credenciales para conectarse a la base de datos.

**3. Reinicia Cassandra** para aplicar los cambios (puede tardar un minuto en estar lista):

```bash
sudo systemctl restart cassandra
sudo systemctl status cassandra
```

**4. Crea un usuario propio** para no usar las credenciales por defecto:

```bash
cqlsh -u cassandra -p cassandra
```

```sql
-- Reemplaza usuario y contraseña por los tuyos
CREATE ROLE usuario WITH SUPERUSER = true AND LOGIN = true AND PASSWORD = 'contraseña';
exit
```

**5. Conéctate con el usuario nuevo y elimina el usuario por defecto:**

```bash
cqlsh -u usuario -p 'contraseña'
```

```sql
LIST ROLES;
DROP ROLE cassandra;
LIST ROLES;
```

La segunda lista ya no debe mostrar el rol `cassandra`.

### Motor de índices

En un servidor único, como en este laboratorio, TheHive usa su motor local Lucene. Crea un directorio para los índices:

```bash
sudo mkdir -p /opt/thp/thehive/index
sudo chown -R thehive:thehive /opt/thp/thehive/index
```

### Almacenamiento de archivos

TheHive necesita un lugar donde guardar los archivos que se suben a tareas y observables. En un servidor único se usa el disco local:

```bash
sudo mkdir -p /opt/thp/thehive/files
sudo chown -R thehive:thehive /opt/thp/thehive/files
```

### Conectar TheHive con Cassandra

Edita `/etc/thehive/application.conf` y deja estos bloques, con las credenciales que creaste en Cassandra:

```conf
db {
  provider: janusgraph
  janusgraph {
    storage {
      backend: cql
      hostname: ["127.0.0.1"]
      username: "usuario"
      password: "contraseña"
      cql {
        cluster-name: thp
        keyspace: thehive
        read-consistency-level: ONE
        write-consistency-level: ONE
      }
    }

    ## Configuración de índices
    index.search {
      backend: lucene
      directory: /opt/thp/thehive/index
    }
  }
}

## Configuración de almacenamiento de archivos
storage {
  provider: localfs
  localfs.location: /opt/thp/thehive/files
}
```

> [!IMPORTANT]
> `cluster-name` debe ser igual al que pusiste en Cassandra (`thp`). Además, `username` y `password` **no** deben estar comentados con `#`, porque Cassandra ya exige credenciales.

### Proteger las credenciales

El archivo `application.conf` ahora contiene la contraseña de Cassandra, así que restringe quién puede leerlo:

```bash
sudo chown root:thehive /etc/thehive/application.conf
sudo chmod 640 /etc/thehive/application.conf
```

> [!NOTE]
> La clave secreta de TheHive (`play.http.secret.key`) se genera sola al instalar el paquete en `/etc/thehive/secret.conf`. No la borres ni la subas a GitHub.

## Inicio de los servicios

### Iniciar Cassandra

Cassandra ya debería estar corriendo (la reiniciaste al configurarla). Verifica y activa el arranque automático:

```bash
sudo systemctl status cassandra
sudo systemctl enable cassandra
```

Si falla, revisa los logs:

```bash
sudo journalctl -u cassandra -n 100
```

### Iniciar TheHive

```bash
sudo systemctl start thehive
sudo systemctl status thehive
sudo systemctl enable thehive
```

Si falla, revisa los logs:

```bash
sudo journalctl -u thehive -n 100
# o directamente el log de la aplicación:
sudo tail -n 100 /var/log/thehive/application.log
```

### Primer acceso y organización

TheHive puede tardar un par de minutos en iniciar. Después estará en `http://IP_DEL_SERVIDOR:9000/`.

Credenciales por defecto:

```
usuario:    admin@thehive.local
contraseña: secret
```

> [!WARNING]
> Cambia la contraseña por defecto apenas inicies sesión.

El usuario `admin` solo sirve para administrar la plataforma, no para trabajar con casos. Para usar TheHive:

1. Ve a **Organisations** y crea tu organización.
2. Dentro de la organización, crea los usuarios con perfil `org-admin` (administrador) o `analyst` (analista).
3. En cada usuario, haz clic en ***Preview > New Password*** para definir su contraseña.
4. Cierra sesión e ingresa con el usuario de la organización.

## Integraciones

### Integración con Wazuh

Wazuh crea alertas en TheHive a través de su API.

1. Dentro de tu organización, crea un usuario para Wazuh (por ejemplo `wazuh`).
2. Asígnale un perfil con permiso solo para crear alertas, por ejemplo un perfil personalizado con `manageAlert`. Si no quieres crear uno, el perfil `analyst` sirve para el laboratorio.
3. En el usuario, haz clic en ***Create API Key*** y luego en ***Reveal*** para copiarla.
4. Usa la URL de TheHive y esa clave en el `ossec.conf` de Wazuh.

Los pasos completos están en el README de la integración `custom-w2thive`.

### Integración con Cortex

Para que TheHive envíe observables a Cortex, agrega esto a `/etc/thehive/application.conf`. La API key la creas en Cortex (ver README de Cortex):

```conf
play.modules.enabled += org.thp.thehive.connector.cortex.CortexModule

cortex {
  servers = [
    {
      name = "Cortex"
      url = "https://IP_DE_CORTEX"
      auth {
        type = "bearer"
        key = "TU_API_KEY_DE_CORTEX"
      }
      # Necesario solo si Cortex usa certificado autofirmado
      wsConfig { ssl { loose { acceptAnyCertificate: true } } }
    }
  ]
}
```

> [!NOTE]
> Si Cortex está en la misma máquina que TheHive, usa `url = "http://127.0.0.1:8090"` y quita `wsConfig`.

### Integración con MISP

Para importar eventos de MISP como alertas en TheHive, agrega esto a `/etc/thehive/application.conf`. La API key la obtienes en MISP (ver README de MISP):

```conf
play.modules.enabled += org.thp.thehive.connector.misp.MispModule

misp {
  interval: 1 hour
  servers: [
    {
      name = "MISP"
      url = "https://IP_DE_MISP"
      auth {
        type = key
        key = "TU_API_KEY_DE_MISP"
      }
      # Necesario solo si MISP usa certificado autofirmado
      wsConfig { ssl { loose { acceptAnyCertificate: true } } }
    }
  ]
}
```

Después de modificar `application.conf`, reinicia TheHive:

```bash
sudo systemctl restart thehive
```

> [!WARNING]
> Las API keys quedan en texto plano en `application.conf`. Mantén los permisos `640` y no subas este archivo a GitHub.

## Seguridad

- Cambia la contraseña de `admin@thehive.local` al primer acceso.
- Cassandra debe escuchar solo en localhost y con credenciales propias (sin el usuario `cassandra`).
- TheHive sirve por HTTP en el puerto 9000. Para cifrar el tráfico, ponle Nginx como proxy inverso con HTTPS, igual que en el README de Cortex.
- Abre solo los puertos necesarios:

| Puerto | Uso | Origen recomendado |
|---|---|---|
| 22 | SSH | Solo tu IP |
| 9000 | TheHive | Analistas y servidor de Wazuh |
| 9042 | Cassandra | **No exponer** (solo local) |
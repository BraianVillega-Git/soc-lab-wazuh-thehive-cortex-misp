# 🌐 Integración de Wazuh, TheHive, Cortex y MISP

Una de las características clave de TheHive es su integración nativa con MISP y Cortex, lo que lo convierte en el punto central de gestión de casos. En este entorno, **Wazuh** actúa como fuente de alertas: detecta eventos en los endpoints, los envía a TheHive como alertas y desde ahí se enriquecen con Cortex (análisis) y MISP (inteligencia de amenazas).

```
Wazuh (alertas) ──► TheHive (casos) ◄──► Cortex (analizadores)
                        ▲
                        └──────────────► MISP (inteligencia / IOC)
```

## Conector de MISP

Antes de habilitar el conector de MISP en el servidor de TheHive, necesitamos crear una clave de API que el conector usará para comunicarse desde TheHive hacia MISP.

### Clave de API

Para poder rastrear y administrar la clave con mayor facilidad, primero crearemos un usuario dedicado exclusivamente a la API.

**Crear usuario**

1. En MISP, ve a ***Administration*** > ***Add User***.
2. Asigna un correo (por ejemplo, `thehive@tu-dominio.local`), tu organización y un rol adecuado (por ejemplo, *User* o *Publisher*, según lo que necesites que TheHive pueda hacer).
3. Guarda el usuario.

**Crear clave de API**

1. Ve a la página de usuarios: ***Administration*** > ***List Users***.
2. Haz clic en el ícono de ***View*** (ojo) en el extremo derecho del usuario de API que creaste.
3. Al final de los detalles del usuario, haz clic en ***Auth Keys***, lo que te llevará al índice de claves de autenticación (*Authentication key Index*).
4. En esa página, haz clic en ***Add Authentication Key***.
5. Configura lo siguiente:
   - ***User***: tu usuario de API.
   - ***Allowed IPs***: la IP de tu servidor TheHive.
   - ***Expiration***: menos de 1 año.
6. Haz clic en *Submit*. Se mostrará la clave de API.

> [!IMPORTANT]
> La clave solo se muestra **una vez**, al crearla, y no podrás recuperarla después. Guárdala en un lugar seguro.

<br>

### Conector en TheHive

Con la clave de API lista, ve al servidor de TheHive. Edita el archivo `/etc/thehive/application.conf` y agrega lo siguiente:

```hocon
## Configuración de MISP
# Más información: https://docs.strangebee.com/
# Habilitar el conector de MISP
play.modules.enabled += org.thp.thehive.connector.misp.MispModule
misp {
  interval: 1 hour
  servers: [
    {
      name = "MISP Server"
      url = "https://misp.servidor"   # Cambiar por la URL/IP de tu servidor MISP
      auth {
        type = key
        key = "<TU_API_KEY>"          # Cambiar por tu clave de API
      }
      # Desactiva la verificación del certificado (solo para certificados autofirmados)
      wsConfig.ssl.loose.acceptAnyCertificate = true
    }
  ]
}
```

> [!WARNING]
> `acceptAnyCertificate = true` solo es recomendable en laboratorio con certificados autofirmados. En producción, instala la CA en el truststore de Java y elimina esta línea.

<br><br>

## Conector de Cortex

Antes de habilitar el conector de Cortex en TheHive, necesitamos crear una clave de API que TheHive usará para comunicarse con Cortex.

### Clave de API

**Crear usuario** (en la interfaz web de Cortex)

1. Abre la pestaña ***Users*** (Usuarios) en la parte superior derecha.
2. Haz clic en ***Add User*** (arriba a la izquierda) para crear un nuevo usuario.
3. Indica el ***Login*** y el ***Full name***, selecciona tu organización y cambia el ***Role*** a ***read, analyze, orgadmin***.
4. Haz clic en ***Save user***.

**Crear clave de API**

1. Abre la pestaña ***Users***.
2. Busca tu usuario de API y haz clic en ***Create API Key***.
3. Para ver la clave, haz clic en ***Reveal***.

<br>

### Conector en TheHive

Con la clave de API a mano, vuelve al servidor de TheHive. Edita el **mismo archivo** `/etc/thehive/application.conf` y agrega lo siguiente:

```hocon
play.modules.enabled += org.thp.thehive.connector.cortex.CortexModule
cortex {
  servers = [
    {
      name = Cortex
      url = "http://cortex:9001"      # Cambiar por la URL/IP de tu servidor Cortex
      auth {
        type = "bearer"
        key = "<TU_API_KEY>"          # Cambiar por tu clave de API
      }
      # Desactiva la verificación del certificado (solo para certificados autofirmados)
      wsConfig.ssl.loose.acceptAnyCertificate = true
    }
  ]
  refreshDelay = 5 seconds
  maxRetryOnError = 3
  statusCheckInterval = 1 minute
}
```

> [!NOTE]
> Para que los analizadores de MISP funcionen desde Cortex (por ejemplo, `MISP_2_1`), también debes configurar la URL y la clave de MISP dentro de Cortex, en ***Organization*** > ***Analyzers***.

<br>

## Integración de Wazuh con TheHive

Wazuh envía sus alertas a TheHive mediante un script de integración personalizado (`custom-w2thive`), basado en la guía oficial de Wazuh "Using Wazuh and TheHive". Los pasos se ejecutan en el **servidor Wazuh (manager)**.

### Clave de API en TheHive

1. En TheHive, crea un usuario de servicio (tipo *Service*) en tu organización, con un perfil que permita crear alertas (por ejemplo, *analyst* u *org-admin*).
2. Genera su clave de API (***Create API key*** > ***Reveal***) y guárdala.

### Configuración en el servidor Wazuh

1. Instala la librería de Python que usa el script, con el Python embebido de Wazuh:
   ```bash
   sudo /var/ossec/framework/python/bin/pip3 install thehive4py==1.8.1
   ```
2. Copia los scripts `custom-w2thive` y `custom-w2thive.py` (de la documentación oficial de Wazuh) en `/var/ossec/integrations/` y asigna permisos:
   ```bash
   sudo chmod 750 /var/ossec/integrations/custom-w2thive*
   sudo chown root:wazuh /var/ossec/integrations/custom-w2thive*
   ```
3. Agrega el bloque de integración en `/var/ossec/etc/ossec.conf`:
   ```xml
   <integration>
     <name>custom-w2thive</name>
     <hook_url>http://IP_DE_THEHIVE:9000</hook_url>
     <api_key>TU_API_KEY_DE_THEHIVE</api_key>
     <alert_format>json</alert_format>
     <level>7</level> <!-- Opcional: nivel mínimo de alerta a enviar -->
   </integration>
   ```
4. Reinicia el manager:
   ```bash
   sudo systemctl restart wazuh-manager
   ```

> [!TIP]
> Ajusta `<level>` (o usa `<rule_id>` / `<group>`) para evitar saturar TheHive con alertas de bajo valor.

<br>

## Verificación en TheHive

Una vez configurados los conectores de MISP y Cortex, reinicia el servicio de TheHive para aplicar los cambios:

```bash
sudo systemctl restart thehive
```

Para confirmar que TheHive se comunica con ambos servicios, abre la interfaz web e inicia sesión. En la esquina superior derecha, haz clic en tu perfil y, en el menú desplegable, selecciona ***About***. Se abrirá una ventana que debería indicar que TheHive se comunica correctamente con MISP y Cortex.

Para verificar Wazuh, genera una alerta de prueba (por ejemplo, un fallo de inicio de sesión SSH en un agente) y comprueba que aparezca en la sección ***Alerts*** de TheHive.

### Caso de prueba

Vamos a crear un caso que utilice los servicios de MISP y Cortex desde TheHive. También puedes partir de una alerta enviada por Wazuh y pulsar ***Import*** para convertirla en caso.

1. Para crear un caso nuevo, ve a la página ***Cases*** y haz clic en ***Create Case +*** en la parte superior.
2. Completa los detalles:
   - **Título:** Demo de integración Cortex y MISP
   - **Descripción:** Caso de prueba para validar la integración de MISP y Cortex
   
   El resto de los ajustes pueden quedar por defecto.
3. Haz clic en ***Confirm*** para crear el caso.
4. Abre el caso, ve a la pestaña ***Observables*** y haz clic en el ícono ***+*** para crear un observable.
5. Completa los datos del observable:
   - ***Type:*** Hash
   - ***Value:*** `4a821767c6ce723e6fb4b8d54efd52df6cbd63fc0de47a7b8b39a6ec72b4be69`
   - Activa ***Is IOC***
   - ***Tags:*** Hash
6. Haz clic en ***Confirm*** para crear el observable.
7. Con este observable puedes ***Exportar/Compartir*** a MISP para investigar con qué se relaciona este hash, o bien...
8. Ejecutar un analizador con el conector de VirusTotal configurado en Cortex: haz clic en el ícono ***...*** en el extremo derecho del observable y selecciona ***Run Analyzers***.
9. Cortex devolverá los resultados del analizador al observable en TheHive. Si necesitas una investigación más profunda del hash, se recomienda usar MISP.
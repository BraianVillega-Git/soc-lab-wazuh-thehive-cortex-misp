# Integración de Wazuh y TheHive

Este proyecto integra el SIEM [Wazuh](https://wazuh.com) con [TheHive](https://thehive-project.org). Sigue estas instrucciones para configurarlo.

Más información en el artículo de Awwal Ishiaku: [Using Wazuh and TheHive for threat protection and incident response](https://wazuh.com/blog/using-wazuh-and-thehive-for-threat-protection-and-incident-response/)

## Instalación

```sh
cd /opt/
sudo git clone https://github.com/crow1011/wazuh2thehive.git
sudo /var/ossec/framework/python/bin/pip3 install -r /opt/wazuh2thehive/requirements.txt
sudo cp /opt/wazuh2thehive/custom-w2thive.py /var/ossec/integrations/custom-w2thive.py
sudo cp /opt/wazuh2thehive/custom-w2thive /var/ossec/integrations/custom-w2thive
sudo chmod 755 /var/ossec/integrations/custom-w2thive.py
sudo chmod 755 /var/ossec/integrations/custom-w2thive
sudo chown root:ossec /var/ossec/integrations/custom-w2thive.py
sudo chown root:ossec /var/ossec/integrations/custom-w2thive
sudo nano /var/ossec/etc/ossec.conf
```

## Configuración

Inserta el siguiente fragmento dentro del bloque `ossec_config`:

```xml
<integration>
    <name>custom-w2thive</name>
    <hook_url>http://localhost:9000</hook_url>
    <api_key>TU_API_KEY</api_key>
    <alert_format>json</alert_format>
</integration>
```

| Parámetro | Descripción |
|---|---|
| `name` | Nombre de la integración (no es necesario cambiarlo). |
| `hook_url` | Dirección del servidor de TheHive. |
| `api_key` | Clave API de un usuario de TheHive. Se genera en la página de gestión de usuarios, iniciando sesión como administrador. Por seguridad, permite que este usuario solo pueda crear alertas. |
| `alert_format` | Formato en el que Wazuh envía la alerta al integrador (no es necesario cambiarlo). |

Después de configurar, aplica los cambios:

```sh
/var/ossec/bin/ossec-control restart
```

## Verificación y depuración

Revisa el archivo `/var/ossec/log/integrations.log` en busca de errores.

Si la información del error no es suficiente, activa el modo debug cambiando esta línea en `custom-w2thive.py`:

```python
debug_enabled = False
```

por:

```python
debug_enabled = True
```

## Filtrar por severidad

Si recibes demasiados eventos, puedes establecer un umbral de severidad para los que se envían a TheHive. Define el valor de la variable `lvl_threshold` en `/var/ossec/integrations/custom-w2thive.py`:

```python
lvl_threshold = 0
```

Se enviarán a TheHive los eventos con un nivel de severidad **igual o mayor** al indicado. Más información sobre la clasificación de eventos: [Clasificación de reglas de Wazuh](https://documentation.wazuh.com/3.12/user-manual/ruleset/rules-classification.html)

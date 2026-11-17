<h1 align=center>🔒 SOC LAB 🔒</h1>
<img align="center" src="https://www.qomplx.com/content/images/2022/11/soc-image.png" height="100%" width="100%">
<br>

Este proyecto de SOC Lab utiliza una variedad de recursos gratuitos y de código abierto, entre ellos Wazuh, MISP, Cortex y TheHive. El objetivo de este laboratorio es brindarte una experiencia práctica en la gestión de un entorno similar al de producción de un SIEM (Security Information and Event Management), un SOAR (Security Orchestration, Automation and Response) y un EDR (Endpoint Detection and Response) con datos del mundo real. Se exponen ciertos endpoints a internet pública para generar registros y métricas reales que luego se usan para probar la activación de alertas y la creación y gestión de casos.


## Componentes clave
En el SOC Lab usamos un conjunto seleccionado de herramientas gratuitas y de código abierto para potenciar nuestras operaciones de seguridad. Esta es una descripción general de los componentes clave del laboratorio:

### Wazuh
Wazuh es nuestra solución de SIEM y EDR, responsable de recopilar y almacenar de forma eficiente los registros de nuestros endpoints mediante sus agentes, y de generar las alertas de seguridad. Conoce más sobre Wazuh [aquí](https://wazuh.com/).

### <img align="center" src="https://thehive-project.org/assets/ico/favicon.ico" height="30px" width="30px">&nbsp;&nbsp;   TheHive
TheHive actúa como una solución SOAR integral, facilitando la creación de tickets y la investigación de casos a partir de las alertas recibidas de Wazuh. Conoce más sobre TheHive [aquí](https://thehive-project.org/).

### <img align="center" src="https://docs.sekoia.io/assets/playbooks/library/misp.png" height="30px" width="30px"> &nbsp;&nbsp;  MISP
MISP, una plataforma de inteligencia de amenazas de alto nivel, desempeña un papel fundamental en la automatización y el análisis de las amenazas detectadas en nuestros endpoints. Conoce más sobre MISP [aquí](https://www.misp-project.org/).

### <img align="center" src="https://thehive-project.github.io/Cortex-Analyzers/images/cortex-logo.png" height="30px" width="30px"> &nbsp;&nbsp;  Cortex
Cortex funciona como un repositorio central de herramientas de análisis, simplificando el análisis de observables sin tener que recurrir a múltiples recursos por separado. Explora las capacidades de Cortex [aquí](https://docs.strangebee.com/cortex/).

### <img align="center" src="https://files.softicons.com/download/social-media-icons/free-social-media-icons-by-uiconstock/png/512x512/AWS-Icon.png" height="30px" width="30px"> &nbsp;&nbsp;  AWS
AWS, proveedor de nube líder en la industria, es la base de nuestro laboratorio: utilizamos instancias EC2 para crear las máquinas virtuales que alojan todos nuestros servidores. Aprovecha la escalabilidad y la flexibilidad de AWS [aquí](https://aws.amazon.com/).
<br><br>

## Aprendizajes clave
A lo largo de este laboratorio cubriremos una amplia gama de habilidades y temas relacionados con la seguridad en la nube, el alojamiento y la gestión en la nube, el diseño de arquitecturas y más. Nuestro objetivo es el fortalecimiento de nuestro entorno, siguiendo rigurosamente las mejores prácticas de la industria para proteger el laboratorio frente a posibles actores de amenazas maliciosos. Estos son algunos de los conceptos que exploramos:
  - Aprovechar una bóveda de claves (Key Vault), como AWS Secrets Manager, para almacenar y gestionar de forma segura las credenciales.
  - Ajustar y administrar la configuración del firewall de nuestras VM, garantizando un control estricto del tráfico entrante y saliente.
  - Usar instancias de AWS EC2 para crear y gestionar máquinas virtuales sin complicaciones, reforzando nuestra infraestructura con capacidades nativas de la nube.
  - Implementar la gestión de identidades y accesos (IAM) para controlar y administrar el acceso de los usuarios a los recursos de AWS.
  - Priorizar la gestión de parches y su automatización para protegernos frente a vulnerabilidades y asegurar la aplicación constante de las actualizaciones de seguridad.
  - Establecer mecanismos sólidos de cifrado TLS con certificados autofirmados, reforzando la confidencialidad e integridad de la transmisión de datos.
  - Diseñar arquitecturas de sistemas de nivel productivo que prioricen la escalabilidad, la fiabilidad y la resiliencia.
  - Configurar proxies inversos y tareas programadas (cron jobs) para agilizar las tareas operativas y optimizar el rendimiento del sistema.
  - Aplicar buenas prácticas en el manejo de credenciales dentro de nuestro código, garantizando la máxima seguridad de la información sensible.
  - Dominar la gestión de VPC en AWS para crear redes virtuales aisladas con un control detallado de las comunicaciones y los accesos.
  - Aplicar medidas rigurosas de endurecimiento (hardening), deshabilitando servicios y puertos innecesarios en nuestros hosts, para reforzar la defensa frente a posibles amenazas.
# <img align="center" src="https://www.virustotal.com/gui/images/manifest/icon-144x144.png" height="45px" width="45px">&nbsp; Cortex Analyzer: VirusTotal

[VirusTotal](https://www.virustotal.com) es uno de los principales servicios de inteligencia de amenazas. Permite consultar si un archivo (por su hash), dominio, URL o IP aparece como malicioso, combinando los resultados de decenas de motores antivirus y otras fuentes.

Con este analizador, Cortex consulta VirusTotal por la API cuando un analista lo ejecuta, ya sea desde Cortex o desde un observable de TheHive. El resultado queda guardado en el caso.

## 📚 Tabla de contenido

- [Requisitos](#requisitos)
- [Obtener la API key de VirusTotal](#obtener-la-api-key-de-virustotal)
- [Configurar el analizador](#configurar-el-analizador)
- [Probar el analizador](#probar-el-analizador)
- [Usarlo desde TheHive](#usarlo-desde-thehive)
- [Límites y seguridad](#límites-y-seguridad)

## Requisitos

- Cortex instalado y con los analizadores configurados (ver README de Cortex).
- Una cuenta gratuita de VirusTotal.

## Obtener la API key de VirusTotal

1. Crea una cuenta en [virustotal.com](https://www.virustotal.com/gui/join-us) (es gratuita; también existe una versión de pago con más funciones).
2. Inicia sesión, haz clic en tu foto de perfil (arriba a la derecha) y elige ***API Key***.
3. Copia la clave que aparece en esa página.

Más información sobre la API en la [documentación de VirusTotal](https://docs.virustotal.com/reference/overview).

> [!WARNING]
> No subas la API key a GitHub ni la pegues en capturas de pantalla.

## Configurar el analizador

1. Inicia sesión en Cortex con el usuario de tu organización y ve a **Organization > Analyzers**.
2. Busca ***VirusTotal*** y localiza ***VirusTotal_GetReport_3_1***.
3. Haz clic en ***Enable*** y completa:

   | Parámetro | Valor |
   |---|---|
   | `key` | Tu API key de VirusTotal |

   El resto de parámetros puedes dejarlos por defecto.
4. Haz clic en ***Save***.

El analizador debería aparecer ahora habilitado en la lista de **Organization > Analyzers**.

> [!NOTE]
> Existe también ***VirusTotal_Scan_3_1***, que **envía** archivos o URLs a VirusTotal para analizarlos. ***GetReport*** solo consulta resultados que ya existen. Ten cuidado con ***Scan***: lo que subas pasa a ser accesible para otros usuarios de VirusTotal.

## Probar el analizador

1. En Cortex, haz clic en ***New Analysis*** en la barra superior.
2. Completa:

   | Campo | Valor |
   |---|---|
   | Data Type | `hash` |
   | Data | `a2534f954854e317c575b2b6009219c1f08994d1d5550e638acfa48f188da4ab` |
   | Analyzers | `VirusTotal_GetReport_3_1` |

3. Haz clic en ***Start***.
4. En **Job History** debería aparecer un trabajo pendiente que, al terminar, muestra ***Success***.
5. Haz clic en ***View*** para ver el reporte que devolvió VirusTotal.

> [!NOTE]
> Si el trabajo termina en ***Failure***, revisa el mensaje de error del reporte. Las causas más comunes son una API key incorrecta, que el servidor no tenga salida a internet o que hayas superado el límite de consultas.

## Usarlo desde TheHive

Con Cortex ya conectado a TheHive (ver README de TheHive):

1. Abre un caso y ve a la pestaña ***Observables***.
2. Agrega un observable (IP, dominio, URL o hash) o abre uno existente.
3. Haz clic en ***Run analyzers*** y selecciona `VirusTotal_GetReport_3_1`.
4. El resultado se guarda en el observable y puede añadir etiquetas automáticas.

## Límites y seguridad

- La cuenta gratuita tiene un límite de consultas por minuto y por día, y está pensada para uso no comercial. Revisa los límites actuales en la documentación de VirusTotal. Si analizas muchos observables a la vez, algunos trabajos pueden fallar por exceder el límite.
- La API key queda guardada en Cortex. Usa una cuenta de VirusTotal dedicada al laboratorio, no tu cuenta personal.
- No envíes a VirusTotal archivos con información sensible: lo que subes puede ser visible para terceros.
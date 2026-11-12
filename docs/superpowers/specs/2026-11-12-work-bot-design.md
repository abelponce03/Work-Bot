# Work-Bot — Diseño

Fecha: 2026-11-12 · Estado: pendiente de revisión

## 1. Objetivo

Aplicación personal de escritorio que, sin atención constante:

1. busca ofertas de trabajo remoto/freelance para un Licenciado en Ciencia de la
   Computación residente en Cuba,
2. las puntúa con una heurística explicable,
3. postula con CV, enlaces y un mensaje adaptado a cada oferta,
4. registra quién responde y muestra un resumen con estadísticas al abrirla.

Éxito: al abrir la app se ve qué se encontró, a qué se postuló, quién respondió
y qué espera aprobación; y ninguna cuenta personal queda bloqueada por usarla.

## 2. Decisiones tomadas

| Tema | Decisión |
|---|---|
| Envío | Híbrido: el correo sale solo si el puntaje supera el umbral alto; Telegram y formularios web esperan aprobación de 1 clic. |
| Telegram | Cuenta personal del usuario, con topes diarios bajos. |
| LinkedIn / Facebook | Modo seguro: sin automatizar con sesión iniciada. Facebook queda fuera de alcance por ahora. |
| IA | Sin IA: reglas, diccionarios de palabras clave y plantillas. El puntaje y el redactor quedan detrás de una interfaz para poder enchufar un modelo después. |
| Sistema | La PC del usuario es Linux: el arranque automático y el empaquetado de la fase 4 se hacen para Linux. |

## 3. Restricciones que condicionan el diseño

- **Cortes de luz y conexión inestable.** La app corre en la PC y no puede asumir
  que está siempre encendida: todas las fuentes se leen por sondeo con memoria
  de la última posición, así al arrancar recupera lo que se perdió.
- **Cobro desde Cuba.** PayPal no opera en Cuba y no pude verificar qué
  plataformas freelance o de pago aceptan residentes cubanos hoy. Por eso la
  viabilidad de cobro es un factor del puntaje, calculado contra los métodos
  que el usuario declare en su perfil, y no una lista fija de plataformas.
- **Anti-spam de Telegram.** No existe un número oficial seguro de mensajes a
  desconocidos, y la restricción `PEER_FLOOD` no tiene fecha de fin. De ahí el
  tope bajo, las pausas aleatorias y la aprobación obligatoria.
- **Términos de LinkedIn.** Solo se usan el listado público de ofertas y los
  correos de alertas; nunca la sesión del usuario.

## 4. Enfoques considerados

- **A. App local en Python con panel web (elegido).** Servicio FastAPI +
  planificador + SQLite en la PC; el panel se abre en el navegador. Coincide con
  lo que el usuario ya domina (Python, FastAPI), no necesita servidor ni pagos,
  y el núcleo sin interfaz puede mudarse a un servidor más adelante.
- **B. Electron/Tauri + Node.** Ventana nativa más pulida, pero duplica el
  lenguaje, pesa más al instalar con mala conexión y Telegram de usuario está
  mejor resuelto en Python (Telethon).
- **C. Servidor 24/7 con un bot de Telegram como interfaz.** Resuelve los cortes
  de luz, pero exige pagar y mantener un servidor desde Cuba y guardar la sesión
  de Telegram y las contraseñas fuera de la PC. Se deja como evolución opcional.

## 5. Arquitectura

Un solo proceso Python con cinco partes que se comunican a través de la base de
datos, de modo que cada una se prueba por separado:

```
fuentes ──> normalizar ──> deduplicar ──> puntuar ──> planificar postulación
                                                            │
                          ┌─────────────────────────────────┤
                          v                                 v
                  cola de aprobación                 envío automático
                          └──────────> canales <────────────┘
                                          │
                              seguimiento de respuestas ──> panel
```

| Módulo | Qué hace | Depende de |
|---|---|---|
| `sources/` | Un adaptador por fuente; cada uno devuelve ofertas crudas desde su última posición. | red |
| `pipeline/` | Normaliza (idioma, contacto, salario, alcance geográfico), deduplica y puntúa. | perfil, configuración |
| `apply/` | Redacta con plantillas, decide auto/aprobación, aplica topes y envía por canal. | `pipeline`, credenciales |
| `track/` | Lee respuestas, las asocia a la postulación, las clasifica y agenda seguimientos. | `apply` |
| `web/` | Panel local: resumen, ofertas, cola, postulaciones, respuestas, fuentes, ajustes. | base de datos |
| `scheduler/` | Ejecuta todo lo anterior a intervalos y recupera lo atrasado al arrancar. | todos |

Pila: Python 3.11+, FastAPI, SQLite con SQLAlchemy, APScheduler, httpx,
plantillas Jinja2 con HTMX y Chart.js incluido en el repositorio (sin CDN ni
paso de compilación con Node), Telethon para Telegram, `smtplib`/`imaplib` para
correo.

### Datos (SQLite)

- `sources`: tipo, configuración, activa, última ejecución, último estado, cursor.
- `jobs`: fuente, id externo, URL, título, empresa, descripción, fecha, idioma,
  alcance geográfico, salario (mín/máx/moneda/periodo), contactos, huella para
  deduplicar.
- `scores`: total, desglose por factor, motivo de descarte.
- `applications`: oferta, canal, estado (`en_cola`, `aprobada`, `enviando`,
  `enviada`, `fallida`, `omitida`), asunto, cuerpo, adjuntos, variante de
  plantilla, referencia externa (Message-ID o id de mensaje de Telegram).
- `contacts`: dirección o usuario, empresa, primer y último contacto, no contactar.
- `responses`: postulación, fecha, canal, extracto, clasificación.
- `feedback`: oferta, etiqueta del usuario (me interesa / no me interesa).
- `events`: bitácora de todo lo que la app hizo.

Archivos fuera de la base: `perfil/profile.yaml` (habilidades con nivel,
idiomas, tarifa mínima y objetivo, métodos de cobro, enlaces, proyectos
destacados), `perfil/CV_es.pdf` y `perfil/CV_en.pdf`, `config.yaml` (fuentes,
pesos, umbrales, topes) y `.env` (contraseñas de aplicación y claves de
Telegram). `perfil/` y `.env` van en `.gitignore`.

## 6. Fuentes

| Fuente | Vía | Fase | Comprobado el 2026-11-12 |
|---|---|---|---|
| RemoteOK, Remotive, Jobicy, Himalayas, Arbeitnow, Working Nomads | API JSON pública | 1 | responde 200 |
| We Work Remotely (y cualquier RSS) | RSS | 1 | responde 200 |
| Get on Board (LATAM) | API JSON pública | 1 | responde 200 |
| Hacker News «Who is hiring» | API de Algolia | 1 | responde 200 |
| Canales públicos de Telegram | vista web `t.me/s/<canal>`, sin sesión | 1 | responde 200 |
| Grupos y canales del usuario en Telegram | Telethon con su cuenta | 2 | — |
| LinkedIn | listado público sin sesión + correos de alertas por IMAP | 3 | listado responde 200 |
| Alertas por correo de otros portales | IMAP | 3 | — |
| Reddit | requiere API con OAuth (el JSON público devuelve 403) | más adelante | 403 |
| Revolico | bloquea clientes automáticos (403) | fuera de alcance | 403 |
| Facebook | sin API de grupos; solo con sesión iniciada | fuera de alcance | — |

Los mensajes de Telegram no tienen estructura, así que un filtro de reglas
decide si un mensaje es una oferta («se busca», «vacante», «hiring», «looking
for»…) y descarta los de quien busca empleo («busco trabajo», «ofrezco mis
servicios»). De cada oferta se extraen correos, usuarios `@` y enlaces.

## 7. Heurística de puntaje

Primero filtros de descarte, luego un puntaje de 0 a 100. Cada oferta guarda su
desglose para que el panel muestre por qué quedó donde quedó.

**Descarte directo:** exige presencia física o residencia en un país concreto;
exige un idioma que el perfil no tiene; señales de estafa (pide pagar, «inversión
inicial», ganancias garantizadas, esquema multinivel); contacto o empresa en la
lista de no contactar; oferta duplicada o con más de 30 días.

| Factor | Peso | Cómo se calcula sin IA |
|---|---|---|
| Encaje técnico | 30 | Habilidades detectadas con un diccionario de sinónimos frente a las del perfil, ponderadas por nivel; las del título cuentan doble. |
| Elegibilidad geográfica | 20 | «Worldwide», «anywhere», «LATAM» suben; restricciones de región o zona horaria bajan; sin dato, neutro. |
| Probabilidad de respuesta | 15 | Antigüedad de la publicación (decae por horas), contacto directo (correo o `@`), tamaño de la fuente, número de postulantes si la fuente lo da. |
| Ajuste de experiencia | 10 | Años pedidos y palabras de nivel (junior, mid, senior, lead) frente al perfil. |
| Compensación | 10 | Salario o tarifa normalizados a USD/mes frente al mínimo y al objetivo del perfil; sin dato, neutro. |
| Viabilidad de cobro | 10 | Métodos mencionados frente a los que el usuario puede usar; baja si exige uno que no tiene. |
| Calidad de la oferta | 5 | Empresa identificable, descripción clara, condiciones definidas, tipo de contrato. |

Umbrales iniciales, todos editables en ajustes:

- 75 o más y la oferta trae correo de contacto → envío automático por correo.
- 55 a 74 con correo, o 55 o más sin correo (Telegram, formulario) → cola de
  aprobación.
- Menos de 55 → archivada pero visible; no se postula. Las etiquetas «me interesa /
no me interesa» se guardan para ajustar los pesos más adelante con datos reales.

## 8. Postulación

**Redacción.** Plantillas en español e inglés (según el idioma de la oferta) con
varias variantes de redacción. Se rellenan con las habilidades que coincidieron,
uno o dos proyectos del perfil elegidos por esas habilidades, los enlaces y la
disponibilidad horaria. Texto breve, sin adornos, CV adjunto en el idioma
correspondiente.

**Canales.**

- *Correo (fase 1):* SMTP con contraseña de aplicación. Tope diario (15 por
  defecto), pausas aleatorias, nunca dos correos al mismo destinatario en 60 días.
- *Mensaje directo de Telegram (fase 2):* solo si la oferta pide escribir al
  privado; siempre con aprobación; 5 conversaciones nuevas al día por defecto;
  ante `PEER_FLOOD` el canal se detiene solo y avisa.
- *Asistida (fase 3):* para formularios web y LinkedIn. La app deja el mensaje
  listo para copiar, abre la oferta y el usuario marca «postulé».

**Seguridad de envío.** La app arranca en modo de prueba: redacta pero no envía
hasta que el usuario lo desactive. Cada postulación pasa a `enviando` antes de
salir; si la app se cae en ese punto, queda para revisión manual en vez de
reintentarse, de modo que nunca se envía dos veces.

## 9. Seguimiento y estadísticas

- **Correo:** sondeo IMAP; la respuesta se asocia por las cabeceras
  `In-Reply-To`/`References` y, si faltan, por remitente o dominio.
- **Telegram (fase 2):** mensajes entrantes de usuarios a los que se escribió.
- **Clasificación por reglas:** positiva (entrevista, llamada, prueba técnica),
  negativa, acuse automático, sin clasificar.
- **Seguimiento:** sin respuesta a los 5 días, un único recordatorio entra en la
  cola de aprobación.

Panel de resumen: embudo encontradas → calificadas → postuladas → respondidas →
entrevista → oferta; tasa de respuesta por fuente, canal y variante de plantilla;
tiempo medio de respuesta; lista de quienes respondieron con nombre, empresa,
canal, fecha, extracto y clasificación; salud de cada fuente.

## 10. Errores

- Cada fuente falla por separado, con reintentos y espera creciente; su estado
  se ve en el panel.
- Sin conexión, el ciclo se salta y se recupera en el siguiente.
- Un formato de fuente que cambie produce un aviso, no ofertas a medias.
- Credenciales inválidas detienen solo ese canal y se avisa en el panel.

## 11. Pruebas

pytest con respuestas reales grabadas de cada fuente como datos de prueba.
Pruebas unitarias de adaptadores, normalización, deduplicación, filtro de
ofertas de Telegram, puntaje, plantillas y clasificador de respuestas. SMTP e
IMAP contra servidores falsos locales. Una prueba de extremo a extremo en modo
de prueba: de la oferta grabada a la postulación en cola y a la respuesta
asociada.

## 12. Fases

Cada fase tiene su propio plan de implementación y deja algo utilizable.

1. **Núcleo y correo.** Fuentes con API/RSS y canales públicos de Telegram,
   puntaje, panel, cola de aprobación, envío por correo, seguimiento por IMAP,
   estadísticas.
2. **Telegram completo.** Sesión de usuario con Telethon (lectura de grupos,
   mensajes directos con topes, respuestas) y un bot propio para avisar y
   aprobar desde el teléfono.
3. **LinkedIn seguro y postulación asistida.** Listado público, correos de
   alertas, flujo asistido para formularios.
4. **Empaquetado para Linux.** Arranque con la sesión (servicio de usuario de
   systemd), icono en la bandeja y script de instalación; y, opcional, guía para
   correr el núcleo en un servidor.

## 13. Paquete de postulación (fuera del código)

Lo que la app envía importa tanto como la app:

- CV de dos páginas como máximo, en español y en inglés, en PDF: primero los
  proyectos entregados a clientes y después los de investigación y sistemas,
  con un enlace a cada repositorio público.
- Perfil de GitHub con esos repositorios fijados y un README claro en cada uno.
- Una línea de disponibilidad y zona horaria (UTC-5, coincide con la costa este
  de EE. UU. y buena parte de Latinoamérica).
- Métodos de cobro que el usuario realmente puede usar, declarados en el perfil.

## 14. Fuera de alcance

Automatizar LinkedIn o Facebook con la sesión del usuario, rellenar formularios
web sin intervención, ocultar o falsear la ubicación del candidato, y cualquier
modelo de IA en esta versión.

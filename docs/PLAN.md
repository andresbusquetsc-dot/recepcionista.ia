# Plan del proyecto

> Estado: **PROPUESTA — pendiente del visto bueno del propietario** (16-11-2026).
> Cuando se apruebe (o se cambie algo), se anota aquí y en CLAUDE.md.
> Precios de noviembre de 2026, aproximados y sin IVA: se confirman al crear cada cuenta.

---

## 1. Stack elegido y por qué

| Pieza | Elección | Por qué |
|---|---|---|
| Lenguaje | Python 3.12 + `uv` | Lo pide la especificación. `uv` instala Python y dependencias con un solo comando. |
| Voz en tiempo real | LiveKit Agents 1.x | Un único agente para teléfono (SIP) y web (WebRTC). Trae detección de voz, detector de fin de turno con español, interrupciones, métricas de latencia y utilidades para probar conversaciones por texto. |
| Servidor de medios | Local: LiveKit en Docker (modo desarrollo, sin cuenta). Desde la fase 3: LiveKit Cloud | Montar uno mismo la parte telefónica (SIP) de LiveKit es lo más difícil de operar; la versión gestionada lo resuelve y tiene plan gratuito para empezar. |
| Modelo de lenguaje | Claude (API de Anthropic). Candidatos: Haiku 5.5 y Sonnet 5.5 | Se elige midiendo (ver 1.1). Configurable global y por clínica. |
| Voz → texto (STT) | Deepgram Nova-3, español, endpoint de la UE | Streaming de baja latencia, buen español, procesa en la UE. Alternativa configurable. |
| Texto → voz (TTS) | ElevenLabs Flash (voz de España) frente a Cartesia Sonic; Azure es-ES como opción 100 % UE | Se elige escuchando muestras (ver 1.2). |
| Backend y panel | FastAPI + plantillas HTML (Jinja2) + HTMX | **Cambio**: panel generado en el servidor en vez de una app React/Vue. Sin compilar JavaScript, una sola pieza que mantener, funciona bien en el móvil. |
| Base de datos | Postgres 16 + SQLAlchemy 2 + Alembic (migraciones) | Estándar, robusto, disponible gestionado en cualquier proveedor. |
| Tareas programadas | APScheduler dentro del servicio `api`, con bloqueo en Postgres | **Cambio**: sin Redis ni Celery. Recordatorios, informe semanal, borrados, salud y demo en el mismo proceso; el bloqueo evita que se ejecuten dos veces. Una pieza menos. |
| Email | SMTP genérico; proveedor Brevo (empresa francesa, UE) | Gratis hasta 300 emails/día. Cambiable por cualquier SMTP. |
| WhatsApp | Meta WhatsApp Cloud API, directa | Sin intermediarios: más barato y es lo que pide la especificación. |
| Números de teléfono | Telnyx (número español + SIP hacia LiveKit) | LiveKit tiene guía oficial para Telnyx; Twilio como alternativa (más caro). |
| Alertas para el administrador | Email + bot de Telegram | Gratis. |
| Empaquetado | Docker Compose: `postgres`, `livekit` (solo local), `api`, `agent` | Todo arranca con `docker compose up`. |
| Producción | Render, región Frankfurt (se confirma en la fase 8) | Plataforma gestionada: HTTPS y dominio automáticos, Postgres con copias de seguridad, sin administrar servidores. Alternativa más barata pero más manual: Hetzner. |

### 1.1 Qué modelo de Claude

- Candidatos: **Claude Haiku 5.5** (0,10 $ / 0,50 $ por millón de tokens de entrada/salida) y **Claude Sonnet 5.5** (2 $ / 10 $). Claude Opus 5.5 (4 $ / 20 $) solo como referencia de calidad.
- En la fase 1 habrá un comando que mide el tiempo hasta la primera palabra de cada modelo y pasa todas las conversaciones simuladas con cada uno. Se usa el más rápido que las apruebe todas; previsiblemente Haiku 5.5. La decisión final es tuya, con los números delante.

### 1.2 Voz con acento de España

- En la fase 1, un comando genera la misma frase de recepcionista con cada voz candidata (ElevenLabs, Cartesia y, si hace falta, Azure es-ES).
- Criterios: acento peninsular (distinción "z/c" frente a "s", entonación, "vosotros"), naturalidad, rapidez del primer audio. Las escuchas y eliges.

### 1.3 Latencia (< 1 s) e interrupciones

- Todo en streaming: Deepgram transcribe mientras hablas, el detector de turno decide cuándo has terminado, Claude empieza a responder y la voz empieza a sonar con las primeras palabras.
- Interrupciones nativas de LiveKit: si el paciente habla, el asistente se calla y escucha.
- Cada turno registra su retardo (fin de habla → primer audio); comando para ver percentiles.

---

## 2. Arquitectura

```
Paciente ──llama a la clínica──► Operadora ──desvío condicional──► Número del sistema (Telnyx)
                                                                         │ SIP
Web de la clínica ──widget (micro o chat)───────────────────────► LiveKit ◄──► Agente (Python)
                                                                                │ Deepgram · Claude · ElevenLabs
                                                                                ▼
                                                  API + panel (FastAPI) ◄──► Postgres
                                                        │ avisos: email · WhatsApp · Telegram
                                                        ▼
                                                  Recepción de la clínica (desde el móvil)
```

Ideas clave:

- **El "cerebro" es independiente de la voz**: instrucciones, herramientas y reglas en un módulo que usan igual la voz, el modo texto y los tests.
- **Cada clínica tiene su propio número**: el agente sabe qué clínica es por el número marcado. El widget lleva la clave pública de la clínica.
- **Separación de datos**: todas las tablas llevan la clínica; todas las consultas pasan por una capa que exige indicar la clínica; hay tests que intentan leer datos de otra clínica y deben fallar.
- **Fechas habladas con código propio**, no "a ojo" del modelo: una herramienta `resolver_fecha` convierte "el martes que viene por la tarde" en una fecha y franja concretas (Europe/Madrid, festivos, horario de la clínica). Es testeable y el asistente siempre confirma la fecha absoluta ("el martes 24 por la tarde, ¿verdad?").
- **Festivos**: nacionales y autonómicos con la librería `holidays` (por comunidad autónoma); locales y cierres propios en el `config.yaml`.
- **No puede hablar de otros pacientes porque no puede verlos**: en pre-reserva el asistente no tiene ninguna herramienta que lea datos de pacientes. En reserva directa solo ve las citas del paciente identificado por nombre y teléfono.
- **La información de la clínica solo sale del `config.yaml`**: si no está, el asistente lo dice y ofrece un recado.

---

## 3. Estructura de carpetas

```
recepcionista.ia/
├── CLAUDE.md                  ← memoria del proyecto (estado, convenciones)
├── README.md                  ← manual en español
├── docker-compose.yml         ← todo en local con un comando
├── Dockerfile
├── pyproject.toml             ← dependencias (uv)
├── .env.example               ← lista de claves necesarias (sin valores)
├── clients/
│   ├── _plantilla/config.yaml
│   └── clinica-dental-ejemplo/config.yaml
├── src/recepcion/
│   ├── cli.py                 ← comandos: nueva-clinica, validar, chat, evals, voces…
│   ├── config/                ← lectura y validación de config.yaml
│   ├── conversation/          ← instrucciones, herramientas y reglas (compartido voz/texto)
│   ├── dates/                 ← fechas habladas y festivos
│   ├── agent/                 ← trabajador de LiveKit (voz) y selección de proveedores
│   ├── calendars/             ← CalendarProvider y adaptadores
│   ├── db/                    ← modelos y acceso a datos (siempre filtrado por clínica)
│   ├── api/                   ← FastAPI: API interna, webhooks, token del widget
│   ├── panel/                 ← panel web (plantillas HTML + HTMX)
│   ├── notify/                ← email, WhatsApp, Telegram
│   ├── jobs/                  ← recordatorios, informe semanal, borrados, salud, demo
│   └── usage/                 ← contador de minutos, mensajes y costes
├── web/                       ← widget embebible y página de prueba
├── migrations/                ← cambios de la base de datos (Alembic)
├── tests/
│   ├── unit/                  ← fechas, festivos, solicitudes, citas, separación de clínicas
│   └── conversations/         ← escenarios simulados + evaluador automático
└── docs/
    ├── SPEC.md, PLAN.md
    ├── guias/                 ← desvío de llamadas, WhatsApp/Meta, despliegue…
    └── legal/                 ← proveedores, contrato de encargado, texto para pacientes
```

---

## 4. Cuentas externas y costes

Te aviso antes de cada una, en la fase en que haga falta.

| Cuenta | Fase | Para qué | Coste aproximado |
|---|---|---|---|
| GitHub | Ya | Guardar el código (repositorio privado) | Gratis |
| Anthropic (API de Claude) | 1 | El "cerebro" | Prepago desde 5 $. ~0,002–0,01 €/min de llamada con Haiku 5.5 (~0,03 €/min con Sonnet 5.5). Toda la fase 1: < 2 € |
| Deepgram | 1 | Voz → texto | 200 $ de saldo gratis al registrarte; después ~0,007 €/min |
| ElevenLabs | 1 | Texto → voz | Gratis para probar; en producción plan de 5–99 $/mes según volumen (~0,02 €/min de llamada) |
| Cartesia (opcional) | 1 | Comparar voces | Gratis para probar |
| Brevo | 2 | Enviar emails | Gratis hasta 300/día; ~9 €/mes si se supera |
| LiveKit Cloud | 3 | Unir teléfono y web con el agente | Gratis al empezar; ~45 €/mes (plan Ship) con varias clínicas; ~0,003–0,01 €/min |
| Telnyx | 3 | Números españoles + SIP | ~1–5 €/mes por número (uno por clínica + demo) + ~0,005–0,01 €/min. Pide NIF/CIF y DNI (uso profesional) |
| Meta (WhatsApp Business) | 4 | Mensajes a clínicas y pacientes | Cuenta gratis. ~0,017 € por plantilla de utilidad fuera de la ventana de 24 h; dentro de la ventana, gratis |
| Telegram | 6 | Alertas para ti | Gratis |
| Google Cloud | 7 | Google Calendar | Gratis |
| Render (Frankfurt) | 8 | Servidores de producción | ~40–55 €/mes (api + agente con 2 GB + Postgres con copias) |
| Dominio | 8 | Dirección web y email | ~10–15 €/año |

**Resumen de costes**

- Variable por minuto de llamada atendida: **~0,04–0,06 €** (Deepgram ~0,007 + voz ~0,02 + Claude ~0,005 + LiveKit ~0,005 + Telnyx ~0,008).
- Fijo en producción: **~50 €/mes** al principio; **~100–150 €/mes** cuando LiveKit y ElevenLabs pasen a planes de pago con varias clínicas.
- Ejemplo de clínica con 300 min/mes y 100 WhatsApp: **~17 €/mes** de coste variable.
- Durante el desarrollo (fases 1–7): **~5–20 € en total**.

---

## 5. Dónde se ejecuta cada cosa durante el desarrollo

- El código se escribe en la sesión de Claude Code en la nube y se sube a tu repositorio de GitHub.
- En la nube se ejecutan los tests, el modo texto y las conversaciones simuladas (con tu clave de Anthropic, que se introduce en un formulario seguro, nunca en el chat).
- La voz la pruebas en tu ordenador: `git pull`, `docker compose up` y abrir el navegador (en la nube no hay micrófono y los servicios de voz están bloqueados por la red).

---

## 6. Fases

Al final de cada fase: tests, instrucciones exactas para probarla, "Estado" actualizado en CLAUDE.md y espera de validación.

**Fase 1 — Agente de voz en el navegador (local) con la clínica de ejemplo**
- `config.yaml` con validación; plantilla y "Clínica Dental Ejemplo"; comandos `nueva-clinica`, `validar` y `chat` (modo texto).
- Cerebro: saludo con aviso de IA (art. 50) y de transcripción; FAQ solo con lo configurado; pre-reserva con repaso final y teléfono dígito a dígito; recados; urgencias (112 / recado URGENTE); "no" a diagnósticos, precios cerrados y datos de otros pacientes; dos fallos o silencio → ofrece recado.
- Fechas habladas y festivos, con tests.
- Agente LiveKit (Deepgram + Claude + voz) y página local con botón de micrófono.
- Las solicitudes y recados se guardan en un archivo local (pasan a la base de datos en la fase 2).
- Conversaciones simuladas con evaluación automática (los 9 escenarios de la especificación) con un solo comando. "Interrupciones" se simula cortando la respuesta del asistente a medias; en voz se prueba a mano.
- Medición de modelos y comparativa de voces.

**Fase 2 — Base de datos, panel de clínica y avisos por email**
- Postgres y migraciones: clínicas, usuarios, solicitudes, recados, citas, llamadas (resumen + transcripción con borrado automático), consentimientos y registro de accesos.
- Panel móvil con usuario y contraseña: solicitudes (urgentes arriba), confirmar / proponer otra hora / rechazar, recados, historial de llamadas.
- Email inmediato a la clínica con enlace al panel; email de confirmación al paciente (si dio email).
- Tests de separación entre clínicas. Primer borrador de `docs/legal/`.

**Fase 3 — Llamadas reales con número español**
- LiveKit Cloud + un número Telnyx por clínica; enrutado por número marcado.
- Guía de desvío condicional por operadora (Movistar, Vodafone, Orange, MásMóvil/Yoigo, Digi; fijo, móvil y centralitas virtuales).
- Adelanto: en móviles se usan códigos GSM estándar (`**61*` no contesta, `**67*` comunica, `**62*` apagado o sin cobertura); en fijos y centralitas cambia según operadora. "Fuera de horario" no es un desvío estándar de las operadoras: se cubre con "no contesta" (si nadie coge, salta al asistente tras unos tonos) o con el horario de la centralita si la tienen. Desviar a un número fijo español suele estar incluido en la tarifa de la clínica.

**Fase 4 — WhatsApp**
- Plantillas de utilidad: aviso a la clínica, confirmación de cita y recordatorio con botones "Confirmo" / "Necesito cambiarla"; las respuestas llegan al panel y avisan a la clínica.
- Consentimiento pedido en la llamada y registrado; recordatorios programados (24 h por defecto).
- Guía paso a paso de Meta (cuenta de empresa, verificación, número y plantillas).
- Decisión de esa fase: un único remitente de WhatsApp tuyo que escribe "de parte de Clínica X" (lo más simple para empezar) o un número por clínica (más imagen de marca, más trámites).

**Fase 5 — Informe semanal, contador de uso, panel de administrador y planes**
- Informe semanal en HTML cuidado, con valor captado estimado.
- Minutos, mensajes y coste por clínica y mes; avisos al 80 % y 100 %; CSV para facturar.
- Panel de administrador. Planes Básico / Completo que activan o desactivan funciones.

**Fase 6 — Fiabilidad y demo comercial**
- Comprobación de salud cada 5 min, llamada de prueba diaria, alertas por email y Telegram.
- Buzón de respaldo: si el agente no responde, la llamada pasa a un buzón que graba el mensaje y lo envía a la clínica (el audio se borra después de enviarlo).
- Demo "Clínica Dental Ejemplo": número y widget propios, borrado nocturno y un resumen para ti de cada llamada.

**Fase 7 — Widget web y reserva directa**
- Widget de una línea (`<script …>`) con micrófono y chat escrito.
- `CalendarProvider` con adaptadores "ninguno", agenda interna y Google Calendar, y esqueleto documentado para un programa de clínica.
- Reserva directa: identificación por nombre y teléfono, 2–3 huecos como máximo y nueva comprobación justo antes de guardar.

**Fase 8 — Producción**
- Despliegue en Render (Frankfurt): api, agente y Postgres gestionado con copias automáticas; dominio y HTTPS.
- Guía para alguien sin experiencia. `docs/legal/` completo.

**Transversal**: los tests unitarios y las conversaciones simuladas crecen en cada fase y se ejecutan con un solo comando; el README se amplía en cada fase.

---

## 7. Datos y legalidad (resumen)

- Tú actúas como **encargado del tratamiento** de cada clínica; los proveedores son subencargados.
- No se guarda audio (salvo el buzón de respaldo, que se borra tras enviarlo). Transcripciones borradas a los 30 días (configurable). Consentimientos de WhatsApp registrados. Registro de accesos al panel.
- Dónde se procesa: Deepgram (endpoint UE), Brevo (UE), Render Frankfurt (UE). LiveKit, Telnyx, ElevenLabs y Anthropic son empresas de EE. UU.: los medios pasan por servidores de la UE cuando es posible, y Anthropic procesa en EE. UU. por defecto (se comprueba en la fase 1 si permite fijar la UE). Si la asesoría exige que todo se quede en la UE, se puede configurar Claude a través de AWS Bedrock en Frankfurt y la voz de Azure en Europa Occidental.
- `docs/legal/` empieza en la fase 2 y se cierra en la 8.

---

## 8. Riesgos conocidos

- **Acento**: algunas voces "en español" de los catálogos son neutras o latinas; por eso se escuchan muestras antes de elegir.
- **Números españoles**: Telnyx exige datos de empresa o autónomo y la verificación puede tardar días; conviene empezar el trámite antes de la fase 3.
- **WhatsApp**: la verificación de empresa en Meta y la aprobación de plantillas tardan de horas a días; conviene empezar antes de la fase 4.
- **"Fuera de horario"** depende de la operadora o centralita de cada clínica.
- **Precios**: cambian; se confirman al crear cada cuenta.

---

## Decisiones tomadas

_(Se rellena al aprobar el plan.)_

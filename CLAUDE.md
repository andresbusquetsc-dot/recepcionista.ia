# CLAUDE.md — Recepcionista IA para clínicas

## Qué es

Asistente de voz con IA, en español de España, que atiende las llamadas desviadas
y el widget web de clínicas pequeñas (dentistas, fisios, estética, podología,
veterinarias) cuando su recepción no puede: pre-reservas, preguntas frecuentes,
recados, urgencias, confirmaciones y recordatorios por WhatsApp/email e informe
semanal. Una sola instalación para todas las clínicas, con datos separados por
clínica. El propietario lo vende e instala a cada clínica con cuota mensual.

**Lee siempre al empezar una fase:**
- `docs/SPEC.md`: especificación completa (fuente de verdad de los requisitos).
- `docs/PLAN.md`: stack, arquitectura, cuentas externas, costes y fases.

## Cómo trabajar con el propietario

- Responder siempre en español. Nivel técnico intermedio, no sabe de servidores:
  explicar cada paso y elegir la opción más simple que funcione en producción.
- **Avisar antes** de que tenga que crear una cuenta, pagar o configurar un
  servicio externo: cuál, para qué y coste aproximado.
- Al final de cada fase: ejecutar los tests, explicarle exactamente cómo
  probarla, actualizar "Estado" aquí y **esperar su validación** antes de seguir.
- Las claves de API se piden con el formulario seguro del entorno, nunca en el chat.

## Stack (propuesto, pendiente de aprobación; detalle en docs/PLAN.md)

- Python 3.12 + `uv`. LiveKit Agents 1.x (agente único para teléfono SIP y web).
- Claude (API de Anthropic): Haiku 5.5 o Sonnet 5.5 según la medición de la fase 1.
- STT Deepgram Nova-3 (endpoint UE). TTS ElevenLabs o Cartesia con voz de España (se elige escuchando).
- FastAPI + Jinja2 + HTMX (panel), Postgres 16 + SQLAlchemy 2 + Alembic.
- APScheduler dentro de `api` con bloqueo en Postgres (sin Redis/Celery).
- Email por SMTP (Brevo). WhatsApp Cloud API de Meta. Números españoles con Telnyx. Alertas por Telegram.
- Docker Compose en local (LiveKit en modo desarrollo); LiveKit Cloud desde la fase 3; producción en Render (Frankfurt).

## Comandos útiles

_(Se rellenan en la fase 1. Previstos: `uv run recepcion nueva-clinica <slug>`,
`uv run recepcion validar <slug>`, `uv run recepcion chat <slug>`,
`uv run recepcion evals`, `uv run pytest`, `docker compose up`.)_

## Convenciones

- Identificadores de código, tablas y columnas en inglés. Claves del `config.yaml`,
  textos visibles, documentación, mensajes de commit y comentarios útiles en español.
- Glosario código ↔ dominio: clinic = clínica, booking_request = solicitud
  (pre-reserva), message = recado, appointment = cita, call = llamada,
  consent = consentimiento, reminder = recordatorio.
- Multi-clínica: toda tabla de datos lleva `clinic_id`; toda consulta pasa por la
  capa de acceso que exige la clínica. Nunca consultar tablas de clínica sin filtrar.
- Horas guardadas en UTC; se muestran y se razonan en Europe/Madrid.
- Proveedores de LLM/STT/TTS detrás de una fábrica elegida por configuración.
- Secretos solo en variables de entorno (`.env` nunca se sube; `.env.example` documentado).
- RGPD: no guardar audio por defecto; transcripciones con borrado automático
  (30 días por defecto); guardar solo lo imprescindible.
- La información de la clínica sale solo del `config.yaml`; el asistente nunca inventa.
- Todo cambio de lógica lleva tests (pytest); escenarios de conversación en `tests/conversations/`.

## Entorno de desarrollo

- Sesión en la nube de Claude Code: hay Python 3.13, `uv` y el cliente de Docker,
  pero **no** el demonio de Docker. `api.anthropic.com` y PyPI son accesibles;
  Deepgram y ElevenLabs están bloqueados por la red. Aquí se ejecutan tests, modo
  texto y evals; la voz la prueba el propietario en su ordenador.
- Repositorio: GitHub `andresbusquetsc-dot/recepcionista.ia` (privado). El contenedor
  de la nube es temporal: todo lo importante se sube a GitHub al acabar cada paso.

## Estado

- [x] **Fase 0** — Especificación guardada (`docs/SPEC.md`) y plan propuesto (`docs/PLAN.md`).
- [ ] Fase 1 — Agente de voz en el navegador (local) con la clínica de ejemplo + modo texto.
- [ ] Fase 2 — Base de datos, panel de clínica y avisos por email.
- [ ] Fase 3 — Llamadas reales con número español y guía de desvío.
- [ ] Fase 4 — WhatsApp.
- [ ] Fase 5 — Informe semanal, contador de uso, panel de administrador y planes.
- [ ] Fase 6 — Fiabilidad y demo comercial.
- [ ] Fase 7 — Widget web y reserva directa.
- [ ] Fase 8 — Producción.

**Siguiente paso:** el propietario debe aprobar el plan. Después, Fase 1.

**Pendiente del propietario:**
- Visto bueno del plan (o cambios).
- ¿Autónomo o empresa? (Telnyx y Meta piden NIF/CIF y DNI; fases 3 y 4).
- Sistema operativo de su ordenador (para las instrucciones de Docker).
- Confirmar nombre comercial (de momento `recepcionista.ia`, como el repositorio).

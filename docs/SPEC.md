# Especificación del producto — Asistente de voz con IA para clínicas

> Copia literal de la especificación entregada por el propietario del proyecto
> (16-11-2026). Es la fuente de verdad de los requisitos. Si algo cambia, se
> anota al final en "Cambios aprobados", sin reescribir el original.

---

Vas a construir conmigo un producto completo: un asistente de voz con IA que
atiende las llamadas y los mensajes de clínicas pequeñas (dentistas,
fisioterapeutas, estética, podología, veterinarias) cuando su recepción no
puede. Lo venderé yo, instalándolo para cada clínica, con cuota mensual.

Soy de nivel técnico intermedio: sé programar algo y usar la terminal, pero
no sé de servidores. Explícame cada paso, elige siempre la opción más simple
que funcione bien en producción y avísame antes de que tenga que crear una
cuenta, pagar algo o configurar un servicio externo (dime cuál, para qué y
cuánto cuesta aproximadamente).

==================================================================
0. ANTES DE EMPEZAR
==================================================================
1. Guarda esta especificación completa en docs/SPEC.md y crea un CLAUDE.md
   con: resumen del proyecto, stack, comandos útiles, convenciones y una
   sección "Estado" con las fases hechas y la siguiente. Actualiza "Estado"
   al acabar cada fase. Voy a limpiar el contexto entre fases, así que
   CLAUDE.md y docs/SPEC.md son tu memoria.
2. Enséñame un plan con stack elegido y por qué, estructura de carpetas,
   cuentas externas necesarias con su coste estimado, y las fases. Espera mi
   visto bueno antes de escribir código.

==================================================================
1. QUÉ HACE EL PRODUCTO
==================================================================
Canales:
- Teléfono: la clínica conserva su número y configura un DESVÍO CONDICIONAL
  (si no contesta, si comunica y fuera de horario) a un número del sistema.
  El asistente solo coge lo que la recepción no puede. Explícame cómo se
  configura ese desvío con las operadoras españolas habituales.
- Widget web: un botón de micrófono que se pega en la web de la clínica con
  una línea de código. Es el mismo asistente, también con opción de chat
  escrito.

Funciones:
1. Pre-reserva (modo por defecto, porque los programas de clínica como
   Gesden o Clinic Cloud no suelen tener API pública):
   - El asistente recoge nombre, teléfono, si es paciente nuevo o ya lo es,
     motivo o tratamiento, preferencia de días y franjas, y si es urgente.
   - Antes de despedirse, repite los datos para que el paciente los
     confirme, con el teléfono dígito a dígito.
   - Le explica que la clínica le confirmará la cita en cuanto abra.
   - Se crea una "solicitud" y se avisa a la clínica al momento por WhatsApp
     y/o email, con un enlace al panel.
   - En el panel, la recepción la confirma con día y hora, propone otra o la
     rechaza. Al confirmar, el paciente recibe la confirmación por WhatsApp o
     email y se programa su recordatorio.
2. Reserva directa (modo opcional por cliente): solo si la clínica tiene una
   agenda conectable. El asistente consulta huecos y reserva, cambia o
   cancela. En ese caso:
   - Identifica al paciente como mínimo por nombre y teléfono.
   - Ofrece como mucho 2 o 3 huecos cada vez.
   - Vuelve a comprobar la disponibilidad justo antes de guardar, para
     evitar reservas dobles.
3. Preguntas frecuentes: horario, dirección, cómo llegar, aparcamiento,
   servicios, precios orientativos, formas de pago, seguros y financiación.
   Responde SOLO con la información configurada. Si no la sabe, lo dice y
   ofrece tomar un recado. Nunca inventa.
4. Recados: quejas, proveedores, casos raros o cualquier cosa que no sepa
   resolver. Toma nombre, teléfono y motivo y avisa a la clínica con un
   resumen.
5. Urgencias: nunca da consejo médico ni diagnósticos. Ante una urgencia
   vital, indica que llamen al 112. Ante una urgencia propia de la clínica
   (dolor fuerte, traumatismo dental...), toma un recado marcado URGENTE y lo
   avisa de forma destacada.
6. Confirmaciones y recordatorios por WhatsApp (WhatsApp Business Cloud API
   de Meta, con plantillas de categoría "utilidad") y por email:
   - Recordatorio con antelación configurable (24 h por defecto) con
     opciones de "Confirmo" y "Necesito cambiarla". La respuesta llega al
     panel y avisa a la clínica.
   - Durante la llamada, pide permiso para escribir por WhatsApp y guarda
     ese consentimiento.
7. Informe semanal automático por email a cada clínica: llamadas atendidas
   (cuántas fuera de horario), pre-reservas, citas confirmadas, recados,
   urgencias y una estimación del valor captado (citas confirmadas × valor
   medio por paciente, configurable). Este informe es clave para que
   renueven: que sea claro y bonito.

Planes, activables por cliente en la configuración:
- Básico: voz en desvío + pre-reservas + recados + avisos a la clínica.
- Completo: lo anterior + widget web + confirmaciones y recordatorios al
  paciente + informe semanal.

==================================================================
2. CÓMO HABLA
==================================================================
- Solo español de España, natural, cálido y breve, como una buena
  recepcionista. Trato de tú o de usted configurable por clínica. Usa una voz
  con acento de España (comprueba que no sea latina) y elige y prueba las
  mejores opciones.
- OBLIGATORIO al empezar (Reglamento europeo de IA, art. 50): decir que es un
  asistente virtual con inteligencia artificial de la clínica. Si se guarda
  la transcripción, informarlo en una frase corta.
- Latencia objetivo: empezar a responder en menos de 1 segundo. Debe
  permitir que le interrumpan mientras habla.
- Entiende fechas habladas ("el martes que viene por la tarde", "pasado
  mañana a primera hora") en la zona horaria Europe/Madrid, con festivos
  nacionales, autonómicos y locales configurables.
- Si no entiende dos veces seguidas o hay silencio, ofrece tomar un recado.
  Nunca deja una llamada sin salida.
- Sabe decir que no: no da diagnósticos, no promete precios cerrados y no
  habla de otros pacientes.

==================================================================
3. ARQUITECTURA
==================================================================
- UNA instalación sirve a todos los clientes, con datos separados por
  clínica. Cada clínica tiene su configuración, sus números, sus usuarios
  del panel y sus datos, y nunca puede ver los de otra.
- Stack propuesto (justifica cualquier cambio): Python + LiveKit Agents como
  agente único para teléfono (SIP) y web; Claude como modelo de lenguaje
  (mide la latencia y usa el modelo más rápido que dé buena calidad);
  reconocimiento y síntesis de voz con buen español de España y baja
  latencia (p. ej. Deepgram y ElevenLabs o Cartesia). Backend y panel con
  FastAPI y Postgres. Todo en Docker. Los proveedores de voz y de modelo
  deben poder cambiarse desde la configuración.
- Configuración por clínica en clients/<slug>/config.yaml: datos del
  negocio, horario, festivos, servicios (duración y precio orientativo),
  profesionales, FAQ, tono, voz, saludo, plan, modo (pre-reserva o reserva
  directa), canales de aviso, contactos a los que avisar, minutos incluidos
  y valor medio por paciente.
  - Comando para crear una clínica nueva desde una plantilla.
  - Comando para validar la configuración.
  - Comando para "probar por texto" la conversación sin gastar voz.
- Agendas: interfaz CalendarProvider (consultar huecos, crear, modificar,
  cancelar, buscar citas de un paciente). Adaptadores:
  - "ninguno" (pre-reserva).
  - Agenda interna.
  - Google Calendar.
  - Esqueleto documentado para añadir más adelante el de un programa de
    clínica.
- Panel web, en español, usable desde el móvil:
  - Para cada clínica, con usuario y contraseña: solicitudes pendientes (con
    las urgentes arriba), recados, citas, historial de llamadas con resumen
    y transcripción, y respuestas a recordatorios.
  - Para mí (administrador): todas las clínicas, su estado, minutos usados
    en el mes frente a los incluidos, coste estimado por clínica, errores y
    llamadas a la demo.
- Contador de uso: minutos de voz, mensajes de WhatsApp y coste estimado por
  clínica y mes. Aviso por email al llegar al 80 % y al 100 % de los minutos
  incluidos. Exportación a CSV para facturar el exceso.
- Fiabilidad:
  - Comprobación de salud cada pocos minutos y una llamada de prueba
    automática diaria.
  - Si algo falla, me avisa al momento por email y Telegram.
  - Si el agente no responde, la llamada no se pierde: pasa a un buzón que
    graba el mensaje y lo envía a la clínica.
- Demo comercial: una clínica de ejemplo ("Clínica Dental Ejemplo") con su
  propio número y widget.
  - Sus datos se borran cada noche.
  - Cada vez que alguien la llama, me llega un resumen (quién llamó y qué
    probó), porque son posibles clientes.

==================================================================
4. DATOS Y LEGALIDAD (RGPD, datos de salud)
==================================================================
- Guarda solo lo imprescindible. Por defecto no guardes audio. Las
  transcripciones se borran solas tras un plazo configurable (30 días por
  defecto). Registra los consentimientos de WhatsApp.
- Proveedores con alojamiento o tratamiento en la UE cuando sea posible.
  Secretos solo en variables de entorno, HTTPS en todo y contraseñas
  cifradas.
- Registro de accesos del panel.
- Genera en docs/legal/:
  - Una lista de proveedores que tratan datos (para qué y dónde).
  - Borradores (que revisará una asesoría) del contrato de encargado del
    tratamiento y del texto informativo para pacientes.

==================================================================
5. FASES
==================================================================
Al final de CADA fase:
- Ejecuta los tests.
- Dime exactamente cómo probarla yo.
- Actualiza "Estado" en CLAUDE.md.
- Espera mi validación antes de seguir.

Fase 1: Agente por voz en el navegador, en local, con la clínica de
  ejemplo: FAQ, pre-reserva, recados y urgencias. Incluye un modo texto para
  probar sin gastar.
Fase 2: Base de datos, solicitudes y panel de la clínica (confirmar,
  proponer otra hora, rechazar) y avisos a la clínica por email.
Fase 3: Llamadas telefónicas reales con número español y guía de desvío
  condicional.
Fase 4: WhatsApp (avisos a la clínica, confirmaciones y recordatorios al
  paciente, respuestas) con guía paso a paso para dar de alta la cuenta y
  las plantillas en Meta.
Fase 5: Informe semanal, contador de uso, panel de administrador y planes.
Fase 6: Fiabilidad (salud, llamada de prueba, alertas, buzón de respaldo) y
  demo comercial.
Fase 7: Widget web embebible y modo reserva directa con agenda interna y
  Google Calendar.
Fase 8: Despliegue en producción en un servicio sencillo con servidores en
  la UE, dominio y HTTPS, copias de seguridad automáticas de la base de
  datos, y una guía para alguien sin experiencia.

==================================================================
6. CALIDAD
==================================================================
- Tests automáticos de la lógica de solicitudes, citas, fechas en español,
  festivos y separación de datos entre clínicas.
- Conversaciones simuladas por texto con evaluación automática:
  - Pre-reserva normal.
  - Paciente que se lía con las fechas.
  - Pide cambiar una cita.
  - Pregunta algo que no está en la FAQ (no debe inventar).
  - Urgencia vital y urgencia dental.
  - Queja.
  - Interrupciones.
  - Alguien que intenta sacarle datos de otro paciente.
- Que cualquiera pueda pasar todas con un solo comando.
- README en español: instalación, alta de una clínica nueva en menos de una
  hora, configuración de cada servicio externo, desvío de llamadas,
  costes, y problemas frecuentes y cómo resolverlos.

---

## Cambios aprobados

_(Ninguno todavía.)_

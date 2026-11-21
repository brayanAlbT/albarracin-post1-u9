# Recordatorios Citas Service — Parte 1

## Decisiones de diseño — Test Doubles (Parte 1)
- **CitaRepository -> Mock:** Se controla la respuesta y se verifica que `save()` se llame exactamente una vez.
- **ReminderChannelFactory / ReminderChannel -> Mock:** Se verifica la selección de estrategia y el despacho del mensaje.
- **Clock -> Fake (`Clock.fixed`):** Implementación real del JDK con instante congelado, más fiel y simple que mockear `instant()` y `getZone()`.

# Recordatorios Citas Service

![CI](https://github.com/brayanAlbT/albarracin-post1-u9/actions/workflows/ci.yml/badge.svg)

**Curso:** Patrones de Diseño de Software (Código: 1156701-A) — Unidad 9  
**Semestre:** Sexto Semestre  
**Estudiante:** Brayan Smith Albarracín Tarazona  
**Institución:** Universidad Francisco de Paula Santander (UFPS)  

---

## 1. Descripción

Repositorio del post-contenido de la **Unidad 9: Probar el Diseño — JUnit 5, Mockito, Test Doubles e Integración Continua**. Consiste en un único proyecto Spring Boot (`recordatorios-citas-service`) que consulta citas médicas pendientes en las próximas 24 horas y envía recordatorios por el canal preferido del paciente (`EMAIL` o `SMS`), diseñado con **Strategy** (`ReminderChannel`) y **Factory Method** (`ReminderChannelFactory`) inyectados por constructor bajo el **Principio de Inversión de Dependencias (DIP)**.

---

## 2. Parte 1 — Pruebas Unitarias con JUnit 5 y Mockito

`CitaReminderServiceImplTest` aísla completamente `CitaReminderServiceImpl` de sus colaboradores (`CitaRepository`, `ReminderChannelFactory`, `ReminderChannel` y `Clock`). Las pruebas siguen la convención `metodoBajoPrueba_condicion_resultado` y verifican:
- Selección del canal preferido y marcado de `recordatorioEnviado = true`.
- Cálculo determinista de la ventana exacta de 24 horas mediante `ArgumentCaptor` y `Clock.fixed(...)`.
- Ausencia de interacciones con la Factory y `save()` cuando no hay citas pendientes.

---

## 3. Parte 2 — Refactor DIP, Pruebas de Integración y CI/CD

- **Refactor DIP:** La funcionalidad `enviarRecordatorioUrgente(Long citaId)` nació inicialmente con un `new SmsReminderChannel(new TwilioNotificadorSmsGateway())` dentro del método, haciendo imposible testear el fallo del gateway. Se extrajo la instanciación al *composition root* (`config/CanalUrgenteConfig.java`) y se inyectó `@Qualifier("canalUrgente") ReminderChannel` por constructor.
- **Pruebas de Integración:**
  - `CitaRepositoryTest` (`@DataJpaTest`): verifica la consulta derivada de JPA sobre H2 en memoria excluyendo citas ya enviadas o fuera de ventana.
  - `CitaReminderServiceIntegrationTest` (`@SpringBootTest`): levanta el contexto completo de Spring con un `Clock` determinista (`@TestConfiguration` + `@Primary`) y `@MockBean` sobre `NotificadorSmsGateway`.
- **Integración Continua:** `.github/workflows/ci.yml` ejecuta `mvn -B verify --no-transfer-progress` con JDK 21 (Temurin) en cada `push` y `pull_request` hacia `main`.

---

## 4. Decisiones de Diseño

### 4.1 Por qué Strategy facilitó las pruebas
Sin `ReminderChannel` como interfaz Strategy, la lógica de transporte (envío SMTP o llamadas HTTP a Twilio) estaría incrustada directamente dentro de `CitaReminderServiceImpl`. Probar el envío habría exigido mockear clientes de red de bajo nivel o levantar servidores simulados. Con Strategy, un único `@Mock ReminderChannel` basta para verificar la interacción `enviar(cita, mensaje)` independientemente del canal concreto.

### 4.2 Por qué Factory Method facilitó las pruebas
Sin `ReminderChannelFactory`, el servicio contendría un bloque `if/else` o `switch` decidiendo el canal concreto, haciendo que la decisión de selección solo fuera observable de forma indirecta. Con la Factory inyectada por constructor, la selección del canal se convierte en una interacción explícita verificable con `verify(reminderChannelFactory).obtenerCanal(CanalNotificacion.EMAIL)`.

### 4.3 Elección de Test Doubles: Mock vs. Stub vs. Fake (Parte 1)
- **`CitaRepository` → Mock:** No basta con simular el retorno de `findByFechaHoraCitaBetweenAndRecordatorioEnviadoFalse(...)` (como haría un Stub); las pruebas necesitan verificar que `save(cita)` se invoque exactamente una vez con la cita marcada como enviada, o nunca si el gateway falla.
- **`ReminderChannelFactory` y `ReminderChannel` → Mock:** El comportamiento bajo prueba es la interacción colaborativa (pedir el canal correcto e invocar `enviar()`), por lo que requiere verificación con `verify(...)`.
- **`Clock` → Fake (`Clock.fixed(...)`, NO Mock):** `Clock.fixed` es una implementación real y completa provista por el JDK estándar. Mockear `Clock` con Mockito obligaría a stubbear `instant()` y `getZone()` manualmente, siendo más frágil y menos fiel que usar el Fake oficial del JDK para garantizar determinismo temporal.

### 4.4 Diagnóstico y Refactor DIP (Parte 2)
| Aspecto | Antes (`new` hardcodeado) | Después (Inyección por constructor — DIP) |
| :--- | :--- | :--- |
| **Construcción** | `ReminderChannel canalUrgente = new SmsReminderChannel(new TwilioNotificadorSmsGateway());` dentro de `enviarRecordatorioUrgente()`. | Bean `@Bean public ReminderChannel canalUrgente(NotificadorSmsGateway gateway)` en `CanalUrgenteConfig`, inyectado por constructor con `@Qualifier("canalUrgente")`. |
| **Testabilidad** | Imposible sustituir el canal con Mockito sin recurrir a `mockConstruction`; la prueba `enviarRecordatorioUrgente_gatewayFalla_noMarcaComoEnviado` no podía escribirse limpiamente. | Se sustituye directamente por `@Mock ReminderChannel canalUrgenteMock` en `@BeforeEach`, permitiendo simular excepciones del gateway y verificar `verify(citaRepository, never()).save(any())`. |

### 4.5 H2 real vs. Gateway mockeado en Integración y CI sin Contenedor de BD
En `CitaReminderServiceIntegrationTest` conviven H2 en memoria (Fake relacional con contrato SQL/JPA equivalente a producción, rápido y determinista) y `@MockBean NotificadorSmsGateway` (Mock, pues un proveedor SMS externo requiere credenciales y red). Gracias al uso de H2 en memoria, el pipeline `.github/workflows/ci.yml` no necesita levantar servicios Docker/PostgreSQL, reduciendo el tiempo de ejecución y eliminando dependencias externas en CI.

---

## 5. Cómo Ejecutar las Pruebas

```bash
# Compilar el proyecto
mvn compile

# Ejecutar pruebas unitarias
mvn test

# Ejecutar suite completa (unitarias + integración)
mvn verify
```

---

## 6. Herramientas Utilizadas

- Java 21 LTS, Spring Boot 3.3.5, Spring Data JPA, H2 Database, Lombok
- JUnit 5, Mockito, Spring Boot Test (`@DataJpaTest`, `@SpringBootTest`)
- Apache Maven, GitHub Actions, Git, GitHub

---

## 7. Conclusiones

Este laboratorio demostró de forma práctica que la testabilidad de un sistema no depende de trucos avanzados del framework de pruebas, sino directamente de la calidad de su diseño arquitectónico. Aplicar **Strategy**, **Factory Method** e **Inversión de Dependencias (DIP)** permitió aislar el servicio con Test Doubles apropiados —distinguiendo con criterio cuándo usar Mocks y cuándo un Fake real como `Clock.fixed`— y dejó en evidencia que un solo `new` dentro de la lógica de negocio basta para bloquear pruebas críticas de resiliencia. Finalmente, combinar pruebas unitarias veloces, pruebas de integración con H2 y un pipeline automatizado en GitHub Actions garantiza retroalimentación continua y confiable en cada cambio del repositorio.

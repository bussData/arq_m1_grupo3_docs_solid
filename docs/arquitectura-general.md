# Arquitectura General — Sistema de Matrícula de Alumnos

---

## Descripción

Sistema para gestionar la matrícula de alumnos de un colegio: registro de alumnos, control de notas y gestión de pagos.

El sistema se ha diseñado bajo un enfoque de **microservicios independientes**, donde cada módulo funcional es un servicio autónomo con su propia responsabilidad, lógica de negocio y persistencia de datos.

---

## Diagrama de Contexto


![Diagrama de contexto](https://raw.githubusercontent.com/bussData/arq_m1_grupo3_docs_solid/main/docs/DiagramaContexto.png)

---

## Diagrama de Microservicios

```
                      ┌───────────────────┐
                      │    API Gateway    │
                      └─────────┬─────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
┌───────▼───────┐       ┌───────▼───────┐       ┌───────▼───────┐
│  MS Matrícula │       │   MS Notas    │       │   MS Pagos    │
│ (C# / .NET)   │       │ (Java /Spring)│       │ (C# / .NET)   │
└───────┬───────┘       └───────┬───────┘       └───────┬───────┘
        │                       │                       │
        └───────────────┬───────┴───────────────────────┘
                        │
                        ▼
                ┌───────────────┐
                │ MS Notifica   │
                │ (C# / .NET)   │
                └───────────────┘
```

---

## Comunicación entre servicios

Toda la interacción entre los microservicios se realiza de forma **sincrónica** mediante **API REST**:

1. **MS Matrícula ➔ MS Pagos:** Al finalizar una matrícula, el servicio de matrícula realiza una petición HTTP POST al MS Pagos para registrar la deuda.
2. **MS Matrícula ➔ MS Notificaciones:** Una vez confirmado el registro, se invoca al MS Notificaciones para el envío del comprobante.
3. **API Gateway:** Orquesta las peticiones externas y las redirige al microservicio correspondiente.

---

## Capas de cada Microservicio (Arquitectura Limpia)

Cada microservicio mantiene internamente una estructura de capas basada en SOLID:

```
┌────────────────────────────────────┐
│      Integración Externas (API)    │
├────────────────────────────────────┤
│       Capa de Aplicación           │
│   NotaService · EvaluacionService  │
│   NotaFactory · Clientes REST      │
├────────────────────────────────────┤
│         Capa de Dominio            │
│   Entidades: Curso, Alumno         │
│   Interfaces: Repositorios         │
│   CalculoEvaluacion                │
├────────────────────────────────────┤
│      Capa de Infraestructura       │
│   Implementaciones de evaluaciones │
│   Base de Datos Propia             │
│   Configuración Spring / JPA       │
└────────────────────────────────────┘
```

**Regla de dependencias:** las capas superiores dependen de las inferiores **solo a través de interfaces**. La infraestructura nunca es importada por el dominio.

**Regla de Oro:** Un microservicio nunca accede directamente a la base de datos de otro.

### Aplicación en este sistema:

1. **Aislamiento Físico:** El **MS Matrícula** tiene su propia base de datos (SQL Server). El **MS Notas** (en Java) usa una base de datos independiente (PostgreSQL).
2. **Consulta de Datos:** Si el **MS Notas** necesita el nombre de un estudiante, realiza una petición `GET /estudiantes/{id}` al MS Matrícula — nunca un `JOIN` entre bases de datos.
3. **Integridad mediante API:** Al matricular un estudiante, el **MS Matrícula** llama sincrónicamente al **MS Pagos** (`GET /pagos/solvencia/{id}`) para verificar que no tenga deudas pendientes.
4. **Independencia Tecnológica:** Como el **MS Notas** usa **Java/Spring** y el **MS Matrícula** usa **C#/.NET**, la única forma de comunicarse es a través de contratos estándar (JSON sobre HTTP).

---

## Decisiones registradas

| ADR | Decisión | Principio | Impacto principal |
|-----|----------|-----------|-------------------|
| [ADR-001](../adr/ADR-001-srp-matricula.md) | Separar registro de datos y notificaciones del servicio de Matrícula | SRP | Clases con una sola razón de cambio |
| [ADR-002](../adr/ADR-002-ocp-modulo-notas.md) | Patrón Strategy para evaluaciones extensibles de notas | OCP | Nuevos tipos sin modificar código existente |
| [ADR-003](../adr/ADR-003-isp-control-pagos.md) | Segregar la interfaz del módulo de control de pagos | ISP | Cada actor depende solo de lo que usa |
| [ADR-004](../adr/ADR-004-lsp-tipos-matricula.md) | Jerarquía de tipos de matrícula sustituibles | LSP | Extensibilidad de reglas de negocio |
| [ADR-005](../adr/ADR-005-dip-repositorio-matricula.md) | Inversión de dependencias en el repositorio de estudiantes | DIP | Abstracción sobre la persistencia propia | 

# Arquitectura de Microservicios — Sistema Escolar

---

## Descripción

El sistema se ha diseñado bajo un enfoque de **microservicios independientes**, donde cada módulo funcional es un servicio autónomo con su propia responsabilidad, lógica de negocio y persistencia de datos.

---

## Diagrama de Contexto y Microservicios

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

1.  **MS Matrícula ➔ MS Pagos:** Al finalizar una matrícula, el servicio de matrícula realiza una petición HTTP POST al MS Pagos para registrar la deuda.
2.  **MS Matrícula ➔ MS Notificaciones:** Una vez confirmado el registro, se invoca al MS Notificaciones para el envío del comprobante.
3.  **API Gateway:** Orquesta las peticiones externas y las redirige al microservicio correspondiente.

---

## Capas de cada Microservicio (Arquitectura Limpia)

Cada microservicio mantiene internamente una estructura basada en SOLID:

```
┌────────────────────────────────────┐
│      Integración Externas (API)    │
├────────────────────────────────────┤
│       Capa de Aplicación           │
│   (Services, Clientes REST)        │
├────────────────────────────────────┤
│         Capa de Dominio            │
│  (Entidades, Reglas de Negocio,    │
│   Interfaces de Repositorio)       │
├────────────────────────────────────┤
│      Capa de Infraestructura       │
│  (Base de Datos Propia, Clientes)  │
└────────────────────────────────────┘
```

**Regla de Oro:** Un microservicio nunca accede directamente a la base de datos de otro.

### Aplicación en este sistema:

Para este sistema escolar, la regla se traduce de la siguiente manera:

1.  **Aislamiento Físico:** El **MS Matrícula** tiene su propia base de datos (ej. SQL Server). El **MS Notas** (en Java) podría usar una base de datos diferente (ej. PostgreSQL).
2.  **Consulta de Datos:** Si el **MS Notas** necesita el nombre de un estudiante que reside en el **MS Matrícula**, no puede hacer un `JOIN` entre bases de datos. Debe realizar una petición `GET /estudiantes/{id}` al MS Matrícula.
3.  **Integridad mediante API:** Si el **MS Matrícula** desea matricular a un estudiante, primero llama sincrónicamente al **MS Pagos** (`GET /pagos/solvencia/{id}`) para verificar que no tenga deudas pendientes.
4.  **Independencia Tecnológica:** Como el **MS Notas** usa **Java/Spring** y el **MS Matrícula** usa **C#/.NET**, la única forma "limpia" de comunicarse es a través de contratos estándar (JSON sobre HTTP), lo que refuerza esta regla.

---

## Decisiones registradas

| ADR | Decisión | Principio | Impacto principal |
|-----|----------|-----------|-------------------|
| [ADR-001](../adr/ADR-001-srp-matricula.md) | Separar Notificaciones y Datos | SRP | Independencia de despliegue y cambio |
| [ADR-003](../adr/ADR-003-lsp-tipos-matricula.md) | Jerarquía de Matrícula sustituible | LSP | Extensibilidad de reglas de negocio |
| [ADR-005](../adr/ADR-005-dip-repositorio-matricula.md) | Interfaz Repositorio | DIP | Abstracción sobre la persistencia propia |

---

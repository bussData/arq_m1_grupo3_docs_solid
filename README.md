# Sistema de Biblioteca Digital — Repositorio de Arquitectura

> **Plantilla para tarea grupal — Arquitectura de Software**  
> Documentación arquitectónica versionada bajo el enfoque **Documentation as Code**.

---

## Contexto del sistema

El **Sistema de Matrícula de Alumnos** es una API REST en Java, Javascript y C++ gestionar la matrícula de alumnos: registrar alumnos, notas y control de pagos.

Este repositorio documenta las decisiones de arquitectura del sistema usando **Architecture Decision Records (ADR)**.

---

## Estructura del repositorio

```
arq_m1_grupo3_docs_solid/
│
├── README.md                              ← Este archivo
│
├── docs/
│   ├── arquitectura-general.md           ← Visión general del sistema
│   ├── arquitectura-matricula.md         ← Detalle del módulo de matrícula
│   └── DiagramaContexto.png              ← Diagrama de contexto del sistema
│
└── adr/
    ├── ADR-001-srp-matricula.md          ← SRP: Separación de responsabilidades en Matrícula
    ├── ADR-002-ocp-modulo-notas.md       ← OCP: Evaluaciones extensibles para cálculo de notas
    ├── ADR-003-isp-control-pagos.md      ← ISP: Segregación de interfaz de pagos
    ├── ADR-003-lsp-tipos-matricula.md    ← LSP: Jerarquía de tipos de matrícula sustituibles
    └── ADR-005-dip-repositorio-matricula.md ← DIP: Inversión de dependencias en repositorio
```

---

## Decisiones registradas

| ID | Título | Principio SOLID | Estado |
|----|--------|-----------------|--------|
| [ADR-001](./adr/ADR-001-srp-matricula.md) | Separar el registro de datos y notificaciones del servicio de Matrícula | SRP | ✅ Aceptado |
| [ADR-002](./adr/ADR-002-ocp-modulo-notas.md) | Introducir evaluaciones extensibles para el cálculo de notas | OCP | ✅ Aceptado |
| [ADR-003](./adr/ADR-003-isp-control-pagos.md) | Segregar la interfaz del módulo de control de pagos del colegio | ISP | ✅ Aceptado |
| [ADR-004](./adr/ADR-003-lsp-tipos-matricula.md) | Jerarquía de tipos de matrícula sustituibles | LSP | ✅ Aceptado |
| [ADR-005](./adr/ADR-005-dip-repositorio-matricula.md) | Inversión de dependencias en el repositorio de estudiantes | DIP | ✅ Aceptado |


## Tecnologías del sistema

| Capa | Tecnología |
|------|------------|
| API | Java 21 + Spring Boot 3 | Javascript | C++
| Base de datos | MySQL 8 |
| ORM | Spring Data JPA / Hibernate |
| Pruebas | JUnit 5 + Mockito |
| Build | Maven |

---

## ¿Cómo usar este repositorio como plantilla?

1. Clona o descarga este repositorio.
2. Lee los tres ADR de ejemplo y comprende la estructura de cada uno.
3. Aplica la misma plantilla para documentar las decisiones de **tu propio sistema**.
4. Cada decisión debe estar respaldada por un principio SOLID con código Java que lo ilustre.

---

## Casos

1. Plataforma de cursos online
2. Sistema de compra de entradas a conciertos
3. Gestión de restaurantes
4. Sistema de votación electrónica
5. Sistema de reservas de sala de reuniones
6. Sistema de competición de deportes
7. Gestión de matricula de estudiantes
8. Plataforma de LLM para estudiantes.
9. Sistema de gestión de condominios.
10. Sistema de chatbot de atención al cliente.


---

*Módulo 1 · Fundamentos de la Arquitectura de Software · Documentation as Code*

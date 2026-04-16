# Arquitectura General — Sistema de Matricula de Alumnos

---

## Descripción

Sistema para gestionar la matrícula de almunos de un colegio: alumnos, notas, control de pagos

---

## Diagrama de Contexto

```
Ver imagen "DiagramaContexto.png"
```

---

## Capas del sistema

``` 
┌────────────────────────────────────┐
│         Capa de API                │
│   Controladores REST · DTOs        │
├────────────────────────────────────┤
│       Capa de Aplicación           │
│   NotaService                      │
│   EvaluacionService                │
│   NotaFactory                      │
├────────────────────────────────────┤
│         Capa de Dominio            │
│   Entidades: Curso,                │
│   Libro · Interfaces: Repositorios │
│   CalculoEvaluacion                │
├────────────────────────────────────┤
│      Capa de Infraestructura       │
│   Implementaciones de evaluaciones │
│   Configuración Spring / JPA       │
└────────────────────────────────────┘
```

**Regla de dependencias:** las capas superiores dependen de las inferiores **solo a través de interfaces**. La infraestructura nunca es importada por el dominio.

---

## Decisiones registradas

FALTA EDITAR
| ADR | Decisión | Principio | Impacto principal |
|-----|----------|-----------|-------------------|
| [ADR-001](../adr/ADR-001-srp-servicio-notificaciones.md) | Separar `NotificacionService` | SRP | Clases con una sola razón de cambio |--cambiar
| [ADR-002](../adr/ADR-002-ocp-modulo-notas.md) | Patrón Strategy para notas | OCP | Nuevos tipos sin modificar código existente |
| [ADR-003](../adr/ADR-003-dip-repositorio-prestamos.md) | Interfaz `PrestamoRepositorio` | DIP | Tests unitarios sin base de datos |--cambiar
| [ADR-004](../adr/ADR-004-lsp-tipos-de-libro.md) | Jerarquía de libros con interfaces | LSP | Sustitución segura sin excepciones --cambiarinesperadas |
| [ADR-005](../adr/ADR-005-isp-gestion-usuarios.md) | Segregar interfaz de usuarios | ISP | Cada actor depende solo de lo que usa |--cambiar

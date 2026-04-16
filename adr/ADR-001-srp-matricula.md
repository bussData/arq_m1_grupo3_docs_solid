# ADR-001 — Separar el registro de datos y notificaciones del servicio de Matrícula

**Fecha:** 2026-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** S — Single Responsibility Principle (SRP)

---

## Contexto

La clase `MatriculaService` es responsable de coordinar la inscripción de un estudiante y, originalmente, también manejaba la persistencia directa en la base de datos y el envío de notificaciones.

**Código actual (con el problema):**

```csharp
// MatriculaService.cs — tiene más de una responsabilidad
public class MatriculaService {
    public void RealizarMatricula(Estudiante estudiante) {
        // Responsabilidad de persistencia mezclada
        using (var db = new SqlConnection("...")) {
            db.Execute("INSERT INTO Estudiantes ...", estudiante);
        }

        // Responsabilidad de notificación mezclada
        var smtp = new SmtpClient("smtp.colegio.edu");
        smtp.Send(new MailMessage("admin@colegio.edu", estudiante.Email, "Matrícula", "..."));
    }
}
```

**¿Cuál es el problema?**

`MatriculaService` tiene **tres razones para cambiar**:
- Si cambian las reglas de negocio de la matrícula.
- Si cambia el motor de base de datos o la forma de persistir.
- Si cambia el proveedor de notificaciones (email a SMS, por ejemplo).

Esto dificulta las pruebas unitarias ya que el servicio está acoplado a la infraestructura real.

---

## Decisión

Extraemos las responsabilidades de persistencia y notificación a servicios dedicados e interfaces (`IEstudianteRepository` e `INotificacionService`). `MatriculaService` delega en ellos sin conocer los detalles de implementación.

**Código corregido:**

```csharp
// NotificacionService.cs — responsabilidad única: enviar notificaciones
public class NotificacionService : INotificacionService {
    public void EnviarConfirmacion(Estudiante estudiante, string mensaje) {
        // Lógica de envío de correos
    }
}

// MatriculaService.cs — responsabilidad única: coordinar el proceso de matrícula
public class MatriculaService : IMatriculaService {
    private readonly IEstudianteRepository _repository;
    private readonly INotificacionService _notificaciones;

    public MatriculaService(IEstudianteRepository repository, INotificacionService notificaciones) {
        _repository = repository;
        _notificaciones = notificaciones;
    }

    public void RealizarMatricula(Estudiante estudiante, MatriculaBase tipoMatricula) {
        _repository.Guardar(estudiante);
        _notificaciones.EnviarConfirmacion(estudiante, "Tu matrícula ha sido procesada.");
    }
}
```

### Principio SOLID aplicado — SRP

> "Un módulo debe tener una, y solo una, razón para cambiar."

| Clase | Única razón de cambio |
|-------|----------------------|
| `MatriculaService` | Reglas de coordinación de matrícula |
| `NotificacionService` | Canal o contenido de las notificaciones |
| `EstudianteRepository` | Estrategia de persistencia de datos |

**Antes:** un cambio en la base de datos obligaba a modificar la lógica de matrícula.  
**Después:** ese cambio solo afecta a la capa de infraestructura (repositorio).

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Mantener el código en una sola clase con métodos privados | No resuelve el problema del acoplamiento ni la dificultad de las pruebas unitarias. |
| Usar un bus de eventos para desacoplar | Se consideró, pero añade complejidad innecesaria para la etapa actual del proyecto. |

---

## Consecuencias

### Positivas
- `MatriculaService` se puede probar con mocks, sin necesidad de base de datos o servidor SMTP.
- El código es más legible y fácil de mantener.
- Reutilización de servicios (ej. `NotificacionService` puede usarse para el módulo de pagos).

### Negativas / trade-offs
- Aumenta el número de clases e interfaces en la solución.
- El flujo de una operación ahora requiere navegar por múltiples archivos.

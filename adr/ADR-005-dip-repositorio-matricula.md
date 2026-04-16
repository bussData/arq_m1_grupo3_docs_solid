# ADR-005 — Inversión de dependencias en el repositorio de estudiantes

**Fecha:** 2026-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** D — Dependency Inversion Principle (DIP)

---

## Contexto

El servicio de matrícula dependía directamente de la implementación concreta de acceso a datos para persistir la información del estudiante.

**Código actual (con el problema):**

```csharp
public class MatriculaService {
    // ❌ Dependencia directa de una clase concreta de infraestructura
    private readonly SqlEstudianteRepository _repository = new SqlEstudianteRepository();

    public void RealizarMatricula(Estudiante e) {
        _repository.Insertar(e);
    }
}
```

**¿Cuál es el problema?**

El módulo de alto nivel (`MatriculaService`) depende de un módulo de bajo nivel (`SqlEstudianteRepository`). Si queremos cambiar a una base de datos diferente (NoSQL) o usar un Mock para pruebas, tenemos que rediseñar el servicio.

---

## Decisión

Invertimos la dependencia introduciendo una interfaz `IEstudianteRepository` en la capa de dominio/aplicación. El servicio solo conoce la interfaz.

**Código corregido:**

```csharp
// Interfaz en la capa de dominio
public interface IEstudianteRepository {
    void Guardar(Estudiante estudiante);
}

// Servicio depende de la interfaz (DIP)
public class MatriculaService {
    private readonly IEstudianteRepository _repository;

    public MatriculaService(IEstudianteRepository repository) {
        _repository = repository; // ✅ Inyección de dependencia
    }
}
```

### Principio SOLID aplicado — DIP

> "Los módulos de alto nivel no deben depender de módulos de bajo nivel. Ambos deben depender de abstracciones."

| Componente | Depende de |
|------------|------------|
| `MatriculaService` | `IEstudianteRepository` (Abstracción) |
| `SqlRepository` | `IEstudianteRepository` (Abstracción) |

**Antes:** El servicio estaba encadenado a SQL Server.  
**Después:** El servicio es agnóstico a la persistencia.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Usar Service Locator | Oculta las dependencias y hace el código más difícil de rastrear y testear. |

---

## Consecuencias

### Positivas
- Permite cambiar la infraestructura sin afectar la lógica de negocio.
- Facilita enormemente las pruebas unitarias.

### Negativas / trade-offs
- Requiere configurar un contenedor de inyección de dependencias (DI Container).

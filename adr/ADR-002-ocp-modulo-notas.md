# ADR-002 — Introducir estrategias extensibles para el cálculo de notas

**Fecha:** 2026-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** O — Open/Closed Principle (OCP)

---

## Contexto

Cuando un alumno culmina un semestre, el sistema realiza el calculo de las notas por cada curso. Actualmente esa lógica vive dentro de `NotaService` con una cadena de condicionales según el tipo de usuario:

**Código actual (con el problema):**

```java
// NotaService.java — lógica de notas acoplada con condicionales
@Service
public class NotasService {

    public double calcularNota(Curso curso) {
        
        double notaFinal = 0.0; 
        double calificacion = curso.getCalificacion();
        String tipoEvaluacion = curso.getTipoEvaluacion();

        //Cada tipo de evaluacion exige modificar este método
        if (tipoEvaluacion.equals("PRACTICA")) {
            notaFinal = calificacion * 0.30;   
        } else if (tipoEvaluacion.equals("TRABAJO-GRUPAL")) {
            notaFinal = calificacion * 0.30;    
        } else if (tipoEvaluacion.equals("EXPOSICION")){
            notaFinal =  calificacion * 0.40;    
        }              

        return notaFinal ;
    }
}
```
**¿Cuál es el problema?**

Cada vez que se añade un nuevo tipo de evaluacion (por ejemplo, `INVESTIGACION` o `SIMULACRO`), hay que **modificar** `NotaService`. Esto:
- Rompe código que ya funciona.
- Obliga a revisar y actualizar todas las pruebas existentes del método.
- Mezcla la lógica de negocio de préstamos con las reglas de tarifas.

---

## Decisión

Introducimos la interfaz `CalculoEvaluacion` con un único método `calcular()`. Cada tipo de evaluacion tiene su propia clase que implementa esa interfaz. `NotaService` usa la estrategia que recibe, sin importarle cuál es.

**Código corregido:**
```java
// CalculoEvaluacion.java — interfaz (contrato cerrado a modificación)
public interface CalculoEvaluacion {
    /**
     * Calcula el monto de las notas por el tipo de evaluación.
     * @param calificacion
     * @return numero de nota final
     */
    double calcular(double calificacion);
}


// EvaluacionPractica.java
public class EvaluacionPractica implements CalculoEvaluacion {
    private static final double PORCENTAJE_VALOR = 0.30;

    @Override
    public double calcular(double calificacion) {
        return calificacion * PORCENTAJE_VALOR;
    }
}


// EvaluacionTrabajoGrupal.java
public class EvaluacionTrabajoGrupal implements CalculoEvaluacion {
    private static final double PORCENTAJE_VALOR = 0.30;

    @Override
    public double calcular(double calificacion) {
        return calificacion * PORCENTAJE_VALOR;
    }
}


// EvaluacionExposicion.java
public class EvaluacionExposicion implements CalculoEvaluacion {
    private static final double PORCENTAJE_VALOR = 0.40;

    @Override
    public double calcular(double calificacion) {
        return calificacion * PORCENTAJE_VALOR;
    }
}



// Nueva evaluacion añadida sin tocar ninguna clase existente
// EvaluacionInvestigacion.java
public class EvaluacionInvestigacion implements CalculoEvaluacion {
    private static final double PORCENTAJE_VALOR = 0.60;  

    @Override
    public double calcular(double calificacion) {
        return calificacion * PORCENTAJE_VALOR;
    }
}


// NotaService.java — cerrado a modificación respecto al cálculo de notas
@Service
public class NotaService {

    public double calcularNota(Curso curso, CalculoEvaluacion calculoEval) {
        
        //  No sabe qué tipo de evaluacion es; solo la usa
        return calculoEval.calcular(curso.getCalificacion(), curso.getTipoEvaluacion());
    }
}


// NotaFactory.java — selecciona la evaluacion según el tipo de evaluacion
@Component
public class NotaFactory {

    public CalculoEvaluacion obtener(double calificacion, String tipoEvaluacion) {
        return switch (tipoEvaluacion) {
            case "PRACTICA"    -> new EvaluacionPractica();
            case "TRABAJO-GRUPAL"       -> new EvaluacionTrabajoGrupal();
            case "EXPOSICION"       -> new EvaluacionExposicion();
            case "INVESTIGACION"  -> new EvaluacionInvestigacion();
            default              -> new EvaluacionPractica();
        };
    }
}
```

**¿Cómo se usa en conjunto?**

```java
// En el controlador o caso de uso
CalculoEvaluacion calculoEval = NotaFactory.obtener(curso.getCalificacion(), curso.getTipoEvaluacion());
double notaFinal = notaService.calcularMulta(prestamo, estrategia);
```

### Principio SOLID aplicado — OCP

> "Las entidades de software deben estar abiertas para extensión y cerradas para modificación."

**Antes:** añadir `INVESTIGADOR` → modificar `PrestamoService` (riesgo de regresión).  
**Después:** añadir `INVESTIGADOR` → crear `MultaInvestigador` (cero riesgo sobre código existente).

```
Agregar nuevo tipo de usuario:
  ANTES → modificar PrestamoService    ← toca código que ya funciona
  AHORA → crear nueva clase            ← no toca nada existente
```

**¿Qué está "cerrado"?** La interfaz `EstrategiaMulta` y el método `calcularMulta` de `PrestamoService`.  
**¿Qué está "abierto"?** El conjunto de implementaciones concretas de `EstrategiaMulta`.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Guardar tarifas en base de datos | Cubre multas de tarifa fija, pero no puede expresar reglas más complejas (ej. tarifa progresiva, exenciones por historial) |
| Usar herencia en lugar de composición | La herencia genera jerarquías frágiles. La composición mediante la interfaz es más flexible y testeable |

---

## Consecuencias

### Positivas
- Cada estrategia tiene su propia prueba unitaria independiente.
- Añadir una nueva tarifa no requiere tocar ni revisar las clases existentes.
- `PrestamoService` permanece estable ante cambios en la política de multas.

### Negativas / trade-offs
- `MultaFactory` sigue siendo un punto de modificación cuando se añade un tipo nuevo. Aceptable: el cambio es de una sola línea en el `switch`.
- Se crean varias clases pequeñas. En sistemas con muchos tipos de usuario puede parecer excesivo; valorar si una tabla de configuración en base de datos sería suficiente.

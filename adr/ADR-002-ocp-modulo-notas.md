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

    public double calcularNota(List<Curso> lstCursos, String tipoEvaluacion) {
        
        double notaFinal = 0.0;
        for(int i=0; i<lstCursos; i++){
           
            double notaCalculada = 0.0;
            double calificacion = curso.get(i).get("calificación");
            //Cada tipo de evaluacion exige modificar este método
            if (tipoEvaluacion.equals("PRACTICA")) {
                notaCalculada = calificacion * 0.30;   
            } else if (tipoEvaluacion.equals("TRABAJO-GRUPAL")) {
                notaCalculada = calificacion * 0.30;    
            } else if (tipoEvaluacion.equals("EXPOSICION")){
                notaCalculada =  calificacion * 0.40;    
            }  
            notaFinal = notaFinal + notaCalculada;
        }

        return notaFinal;
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



// Nueva estrategia añadida sin tocar ninguna clase existente
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

    public double calcularNota(Prestamo prestamo, EstrategiaMulta estrategia) {
        long diasRetraso = ChronoUnit.DAYS.between(
            prestamo.getFechaVencimiento(), LocalDate.now()
        );
        if (diasRetraso <= 0) return 0;

        // ✅ No sabe qué estrategia es; solo la usa
        return estrategia.calcular(diasRetraso);
    }
}


// MultaFactory.java — selecciona la estrategia según el tipo de usuario
@Component
public class MultaFactory {

    public EstrategiaMulta obtener(String tipoUsuario) {
        return switch (tipoUsuario) {
            case "ESTUDIANTE"    -> new MultaEstudiante();
            case "DOCENTE"       -> new MultaDocente();
            case "EXTERNO"       -> new MultaExterno();
            case "INVESTIGADOR"  -> new MultaInvestigador();
            default              -> new MultaEstudiante();
        };
    }
}
```

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
        
        double notaFinal = 0;
        for(int i=0; i<lstCursos; i++){
           
           double calificacion = curso.getNota();
            //Cada tipo de evaluacion exige modificar este método
            if (tipoEvaluacion.equals("PRACTICAS")) {
                return calificacion * 0.50;   
            } else if (tipoEvaluacion.equals("TRABAJO-GRUPAL")) {
                return calificacion * 0.25;    
            } else if (tipoEvaluacion.equals("EXPOSICION")){
                return calificacion * 1.00;    
            } else {
                return calificacion;
            }   
        }
       
    }
}
```
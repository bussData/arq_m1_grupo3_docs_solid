# ADR-003 — Jerarquía de tipos de matrícula sustituibles

**Fecha:** 2026-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** L — Liskov Substitution Principle (LSP)

---

## Contexto

El sistema debe manejar diferentes tipos de matrícula (Regular, Beca, Intercambio). Originalmente, el servicio de matrícula utilizaba condicionales para determinar el costo según el tipo.

**Código actual (con el problema):**

```csharp
public class MatriculaService {
    public void Procesar(Estudiante e, string tipo) {
        decimal costoBase = 1000;
        if (tipo == "Beca") {
            costoBase *= 0.5m;
        } else if (tipo == "Intercambio") {
            costoBase = 0;
        }
        // ... lógica
    }
}
```

**¿Cuál es el problema?**

Si se añade un nuevo tipo de matrícula, debemos modificar el servicio principal. Además, si un "tipo" de matrícula viola las expectativas (por ejemplo, lanza una excepción inesperada), rompe todo el flujo.

---

## Decisión

Implementamos una jerarquía de clases basada en una base abstracta `MatriculaBase`. El servicio trata a todos los objetos de forma polimórfica.

**Código corregido:**

```csharp
public abstract class MatriculaBase {
    public abstract decimal CalcularCostoTotal(decimal costoBase);
}

public class MatriculaRegular : MatriculaBase {
    public override decimal CalcularCostoTotal(decimal costoBase) => costoBase;
}

public class MatriculaBeca : MatriculaBase {
    public override decimal CalcularCostoTotal(decimal costoBase) => costoBase * 0.5m;
}

// En el servicio:
public void RealizarMatricula(Estudiante estudiante, MatriculaBase tipoMatricula) {
    decimal costoFinal = tipoMatricula.CalcularCostoTotal(1000);
    // ✅ No importa el subtipo, el contrato se respeta
}
```

### Principio SOLID aplicado — LSP

> "Los subtipos deben ser sustituibles por sus tipos base."

| Tipo | Comportamiento esperado |
|------|------------------------|
| `MatriculaRegular` | Retorna el costo base sin modificar. |
| `MatriculaBeca` | Aplica un descuento válido. |
| `MatriculaIntercambio` | Retorna costo cero de forma segura. |

**Antes:** El servicio conocía los detalles de cada tipo.  
**Después:** El servicio confía en la abstracción `MatriculaBase`.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Usar un Enum y un Switch | Sigue rompiendo el principio Open/Closed y no aprovecha el polimorfismo. |

---

## Consecuencias

### Positivas
- Podemos añadir nuevos tipos de matrícula sin tocar el código de `MatriculaService`.
- Garantiza que cualquier tipo de matrícula se comporte de manera consistente.

### Negativas / trade-offs
- Requiere crear una clase por cada tipo de matrícula.

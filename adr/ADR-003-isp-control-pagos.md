# ADR-003 — Segregar la interfaz del módulo de control de pagos del colegio

**Fecha:** 2026-04-15
**Estado:** Aceptado
**Principio SOLID:** I — Interface Segregation Principle (ISP)

---

## Contexto

El microservicio **ControlPagos** gestiona el ciclo de vida de las deudas académicas de un colegio EBR. Atiende a tres actores: el **Tesorero** (registra deudas por periodo), el **IntegradorBancario** (envía información al banco y registra logs SQL) y el **GestorAmortizacion** (aplica abonos y cierra deudas). Los tres consumían un único `PagoService` con todos los métodos del sistema, generando acoplamiento innecesario y mocks engorrosos en los tests.

**Código con el problema:**

```js
// pagoService.js — interfaz única con demasiados métodos
class PagoService {
    registrarDeuda(alumnoId, periodo, monto) { /* ... */ }
    listarDeudasPorPeriodo(periodo) { /* ... */ }
    enviarInformacionAlBanco(alumnoId) { /* ... */ }
    verificarConexionSQL() { /* ... */ }
    registrarLogConexion(mensaje) { /* ... */ }
    aplicarAmortizacion(deudaId, montoAbonado) { /* ... */ }
    obtenerSaldoPendiente(deudaId) { /* ... */ }
    cerrarDeuda(deudaId) { /* ... */ }
}

// TesoreroController depende de PagoService completo pero solo usa 2 métodos
class TesoreroController {
    constructor(pagoService) { this.pagoService = pagoService; }
    registrar(alumnoId, periodo, monto) {
        return this.pagoService.registrarDeuda(alumnoId, periodo, monto);
    }
}
```

**Problema:** acoplamiento innecesario, mocks con 8 métodos para tests que usan 2, y violación de mínimo privilegio (el Tesorero puede llamar técnicamente a `cerrarDeuda`).

---

## Decisión

Se divide `PagoService` en tres contratos específicos por actor. En JavaScript se modelan como clases base abstractas. `PagoServiceImpl` implementa los tres contratos; cada controlador recibe únicamente el contrato que le corresponde.

**Contratos segregados:**

```js
// registroDeudaService.js — contrato del Tesorero (2 métodos)
class RegistroDeudaService {
    registrarDeuda(alumnoId, periodo, monto) { throw new Error("No implementado"); }
    listarDeudasPorPeriodo(periodo) { throw new Error("No implementado"); }
}

// integracionBancariaService.js — contrato del IntegradorBancario (3 métodos)
class IntegracionBancariaService {
    enviarInformacionAlBanco(alumnoId) { throw new Error("No implementado"); }
    verificarConexionSQL() { throw new Error("No implementado"); }
    registrarLogConexion(mensaje) { throw new Error("No implementado"); }
}

// amortizacionService.js — contrato del GestorAmortizacion (3 métodos)
class AmortizacionService {
    aplicarAmortizacion(deudaId, montoAbonado) { throw new Error("No implementado"); }
    obtenerSaldoPendiente(deudaId) { throw new Error("No implementado"); }
    cerrarDeuda(deudaId) { throw new Error("No implementado"); }
}
```

**Implementación única y controladores:**

```js
// pagoServiceImpl.js — implementación única que cumple los tres contratos.
// Los controladores nunca dependen de esta clase directamente; dependen de sus contratos.
class PagoServiceImpl extends RegistroDeudaService {
    constructor(repositorio, clienteBancario, logger) {
        super();
        this.repositorio = repositorio;       // acceso a la base de datos SQL del colegio
        this.clienteBancario = clienteBancario; // pasarela bancaria
        this.logger = logger;
    }

    // --- RegistroDeudaService ---
    registrarDeuda(alumnoId, periodo, monto) {
        if (this.repositorio.buscarDeuda(alumnoId, periodo))
            throw new Error(`Deuda duplicada: alumno=${alumnoId}, periodo=${periodo}`);
        const deuda = { alumnoId, periodo, monto, saldo: monto, estado: "PENDIENTE" };
        this.repositorio.guardarDeuda(deuda);
        this.logger.info(`Deuda registrada: alumno=${alumnoId}, periodo=${periodo}, monto=${monto}`);
        return deuda;
    }
    listarDeudasPorPeriodo(periodo) {
        const deudas = this.repositorio.listarPorPeriodo(periodo);
        this.logger.info(`Consulta por periodo: periodo=${periodo}, total=${deudas.length}`);
        return deudas;
    }

    // --- IntegracionBancariaService ---
    enviarInformacionAlBanco(alumnoId) {
        this.verificarConexionSQL();
        const deudas = this.repositorio.listarDeudasActivas(alumnoId);
        const resultado = this.clienteBancario.enviar({ alumnoId, deudas });
        this.registrarLogConexion(`Envio bancario: alumno=${alumnoId}, deudas=${deudas.length}, ref=${resultado.referencia}`);
        return resultado;
    }
    verificarConexionSQL() {
        const estado = this.repositorio.ping();
        if (!estado.conectado) throw new Error("Sin conexion SQL");
        this.registrarLogConexion(`Conexion SQL verificada: host=${estado.host}, latencia=${estado.latenciaMs}ms`);
        return estado;
    }
    registrarLogConexion(mensaje) {
        this.repositorio.guardarLog({ mensaje, timestamp: new Date().toISOString(), origen: "IntegracionBancariaService" });
        this.logger.info(`[LOG BANCARIO] ${mensaje}`);
    }

    // --- AmortizacionService ---
    aplicarAmortizacion(deudaId, montoAbonado) {
        const deuda = this.repositorio.buscarDeudaPorId(deudaId);
        if (!deuda) throw new Error(`Deuda no encontrada: id=${deudaId}`);
        deuda.saldo = Math.max(0, deuda.saldo - montoAbonado);
        this.logger.info(`Amortizacion: deudaId=${deudaId}, abono=${montoAbonado}, saldo=${deuda.saldo}`);
        deuda.saldo === 0 ? this.cerrarDeuda(deudaId) : this.repositorio.actualizarDeuda(deuda);
        return deuda;
    }
    obtenerSaldoPendiente(deudaId) {
        const deuda = this.repositorio.buscarDeudaPorId(deudaId);
        if (!deuda) throw new Error(`Deuda no encontrada: id=${deudaId}`);
        return deuda.saldo;
    }
    cerrarDeuda(deudaId) {
        const deuda = this.repositorio.buscarDeudaPorId(deudaId);
        deuda.estado = "SALDADO";
        this.repositorio.actualizarDeuda(deuda);
        this.logger.info(`Deuda cerrada: deudaId=${deudaId}`);
    }
}

// Cada controlador recibe solo el contrato que necesita — ISP aplicado
class TesoreroController {
    constructor(svc) { this.svc = svc; }
    registrarMensualidad(alumnoId, periodo, monto) { return this.svc.registrarDeuda(alumnoId, periodo, monto); }
    obtenerReportePeriodo(periodo) { return this.svc.listarDeudasPorPeriodo(periodo); }
}
class IntegradorBancarioController {
    constructor(svc) { this.svc = svc; }
    procesarEnvioBancario(alumnoId) { return this.svc.enviarInformacionAlBanco(alumnoId); }
    comprobarConexion() { return this.svc.verificarConexionSQL(); }
}
class GestorAmortizacionController {
    constructor(svc) { this.svc = svc; }
    registrarPago(deudaId, montoAbonado) {
        const saldoPrevio = this.svc.obtenerSaldoPendiente(deudaId);
        const resultado   = this.svc.aplicarAmortizacion(deudaId, montoAbonado);
        return { deudaId, saldoPrevio, montoAbonado, saldoActual: resultado.saldo, estado: resultado.estado };
    }
}
```

**Simulación con alumnos del colegio:**

```js
// simulacion.js
const repo = {
    _d: [], _l: [], _i: 1,
    ping()                 { console.log("[SQL] Verificando conexion..."); return { conectado: true, host: "db-colegio-prod.local", latenciaMs: 12 }; },
    buscarDeuda(a, p)      { return this._d.find(x => x.alumnoId === a && x.periodo === p) || null; },
    buscarDeudaPorId(id)   { return this._d.find(x => x.id === id) || null; },
    guardarDeuda(d)        { d.id = this._i++; this._d.push(d); },
    actualizarDeuda(d)     { const i = this._d.findIndex(x => x.id === d.id); if (i !== -1) this._d[i] = d; },
    listarPorPeriodo(p)    { return this._d.filter(x => x.periodo === p); },
    listarDeudasActivas(a) { return this._d.filter(x => x.alumnoId === a && x.estado === "PENDIENTE"); },
    guardarLog(e)          { this._l.push(e); }
};
const banco  = { enviar({ alumnoId, deudas }) { console.log(`[BANCO] Enviando ${deudas.length} deuda(s) de ${alumnoId}...`); return { referencia: `REF-${alumnoId}-${Date.now()}` }; } };
const logger = { info(m) { console.log(`[INFO] ${m}`); } };

const impl  = new PagoServiceImpl(repo, banco, logger);
const teso  = new TesoreroController(impl);
const banca = new IntegradorBancarioController(impl);
const amor  = new GestorAmortizacionController(impl);

// FLUJO 1: Registro de deudas — periodo 2026-04
teso.registrarMensualidad("AL-001", "2026-04", 320.00); // Rodrigo Apaza Mamani  — 5to Primaria
teso.registrarMensualidad("AL-002", "2026-04", 320.00); // Valeria Quispe Huanca — 6to Primaria
teso.registrarMensualidad("AL-003", "2026-04", 380.00); // Fabian Condori Flores — 1ro Secundaria
teso.registrarMensualidad("AL-004", "2026-04", 380.00); // Luciana Ttito Ccama   — 3ro Secundaria
teso.registrarMensualidad("AL-005", "2026-04", 380.00); // Jhonatan Pari Calcina — 5to Secundaria
console.log(`Deudas en 2026-04: ${teso.obtenerReportePeriodo("2026-04").length}`);

// FLUJO 2: Envio al banco
banca.comprobarConexion();
banca.procesarEnvioBancario("AL-001");
banca.procesarEnvioBancario("AL-003");
banca.procesarEnvioBancario("AL-005");

// FLUJO 3: Amortizacion
const p1 = amor.registrarPago(1, 320.00);
console.log(`Rodrigo Apaza pago total: saldo anterior=S/${p1.saldoPrevio}, estado=${p1.estado}`);
const p2 = amor.registrarPago(3, 200.00);
console.log(`Fabian Condori abono parcial: saldo restante=S/${p2.saldoActual}`);
amor.registrarPago(3, 180.00); // cancela saldo restante
const p3 = amor.registrarPago(5, 100.00);
console.log(`Jhonatan Pari abono parcial: saldo restante=S/${p3.saldoActual}`);
```

**Prueba unitaria — ISP en accion:**

```js
// tesoreroController.test.js — mock con 2 métodos, no con 8
function testRegistrarMensualidad() {
    const mock = {
        registrarDeuda: (alumnoId, periodo, monto) => ({ alumnoId, periodo, monto, saldo: monto, estado: "PENDIENTE", id: 99 }),
        listarDeudasPorPeriodo: () => []
    };
    const resultado = new TesoreroController(mock).registrarMensualidad("AL-010", "2026-04", 320.00);
    console.assert(resultado.alumnoId === "AL-010", "alumnoId debe coincidir");
    console.assert(resultado.estado   === "PENDIENTE", "estado inicial debe ser PENDIENTE");
    console.log("[TEST] testRegistrarMensualidad: PASO");
}
testRegistrarMensualidad();
```

### Principio SOLID aplicado — ISP

> "Los clientes no deben verse forzados a depender de interfaces que no usan." — Robert C. Martin

```
ANTES:   TesoreroController -> PagoService (8 métodos)              usa: 2  ignora: 6  [VIOLACION ISP]
DESPUES: TesoreroController -> RegistroDeudaService (2 métodos)     usa: 2  ignora: 0  [CORRECTO]
         IntegradorBancario -> IntegracionBancariaService (3 mets.)  usa: 3  ignora: 0  [CORRECTO]
         GestorAmortizacion -> AmortizacionService (3 métodos)       usa: 3  ignora: 0  [CORRECTO]
```

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|---|---|
| Mantener `PagoService` único y documentar usos | La documentación se desactualiza; la segregación debe ser estructural |
| Tres implementaciones separadas sin contrato común | Duplica lógica de negocio |

---

## Consecuencias

**Positivas:** mocks simples, cambios aislados por actor, mínimo privilegio por diseño, extensible sin tocar contratos existentes, despliegue independiente por componente.

**Negativas / trade-offs:** más contratos en el proyecto; el desarrollador debe conocer qué contrato inyectar en cada controlador.

---

## Comunicación con otros microservicios

**ControlPagos** se comunica con otros módulos del sistema mediante **API REST asíncrona**. Las llamadas son de tipo *fire and forget* supervisado: se lanzan sin bloquear el flujo principal y cualquier fallo queda registrado en el log sin interrumpir la operación interna.

| Microservicio | Evento que origina la llamada | Tipo |
|---|---|---|
| **MatriculaService** | Deuda registrada — verifica estado activo del alumno | REST async POST |
| **NotificacionService** | Deuda registrada o pago confirmado — avisa al apoderado | REST async POST |
| **ReporteFinancieroService** | Deuda cerrada — actualiza indicadores del periodo | REST async POST |
| **BancoGateway** | Envío bancario — consulta disponibilidad antes de transmitir | REST async GET/POST |

```js
// httpClient.js — cliente HTTP no bloqueante para notificar eventos a otros microservicios
class HttpClient {
    async post(url, payload) {
        try {
            const r = await fetch(url, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload) });
            return { exito: r.ok, status: r.status };
        } catch (error) {
            console.log(`[WARN] No se pudo contactar al servicio: url=${url}, error=${error.message}`);
            return { exito: false, status: 0 };
        }
    }
}

// Fragmento de registrarDeuda con notificaciones asincronicas.
// La persistencia SQL se completa primero; los llamados externos no bloquean la respuesta.
async registrarDeuda(alumnoId, periodo, monto) {
    // ...persistencia en base de datos SQL...
    this.httpClient.post("http://matricula-service/api/alumnos/verificar", { alumnoId })
        .then(r => this.logger.info(`MatriculaService respondio: status=${r.status}`));
    this.httpClient.post("http://notificacion-service/api/notificaciones/pago", {
        alumnoId, tipo: "DEUDA_REGISTRADA",
        mensaje: `Deuda de S/${monto} registrada para el periodo ${periodo}`
    }).then(r => this.logger.info(`NotificacionService respondio: status=${r.status}`));
    return deuda;
}
```
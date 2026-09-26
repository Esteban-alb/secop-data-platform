# Roadmap del proyecto

Documento vivo. Registra el alcance, las fases y el estado de las
decisiones. Las decisiones cerradas viven en `docs/adr/`.

## Problema y objetivo

Monitorear la contratación pública colombiana (SECOP II) requiere hoy
descargas manuales de un portal, en archivos que cambian
retroactivamente y sin trazabilidad de qué se descargó cuándo. El costo
es doble: horas de analista y decisiones tomadas sobre datos
desactualizados o mal deduplicados.

Este proyecto construye una plataforma de datos **incremental e
idempotente** que automatiza esa ingesta, la limpia, la modela y la
expone consultable, con observabilidad y control de costos.

## Arquitectura objetivo

```
API SECOP II (Socrata)
   ↓  ingesta incremental con marca de agua
Azure Functions (timer trigger)
   ↓
Bronze — Blob Storage, crudo inmutable, particionado por fecha
   ↓  tipado, normalización, deduplicación
Silver — Blob Storage, datos limpios
   ↓  modelado dimensional
Gold — Blob Storage + Azure SQL Database
   ↓
Consumo — consultas analíticas / tablero

Transversal: Azure Monitor y Log Analytics (observabilidad),
Key Vault e identidad administrada (secretos y acceso),
Cost Management y etiquetas (costos).
```

El estado real de cada componente está en la tabla de despliegue
del `README.md`. Este diagrama es el destino, no el presente.

## Fases

| Fase | Objetivo | Entregable |
|---|---|---|
| 0 | Arranque | Entorno, repositorio con flujo de PR, Azure con presupuesto, ADR de región |
| 1 | Diseño | Perfilado de la fuente + tres ADR: incrementalidad, particionado, formato de archivo |
| 2 | Ingesta | Function con timer trigger, paginación con reintentos, marca de agua persistida, escritura a bronze |
| 3 | Almacenamiento | Contenedores por capa, política de ciclo de vida, decisión de redundancia por capa |
| 4 | Procesamiento | Bronze → silver: tipado, normalización de NIT y montos, deduplicación por versión, cuarentena de rechazados |
| 5 | Transformación | Silver → gold: modelo dimensional, SCD tipo 2 para proveedores, carga a Azure SQL |
| 6 | Automatización | CI con pruebas unitarias, despliegue con infraestructura como código, credencial federada sin secretos |
| 7 | Seguridad y monitoreo | Identidad administrada con RBAC por contenedor, Key Vault, logs estructurados, consultas KQL, alertas de fallo y de frescura |
| 8 | Documentación | Diagramas, diccionario de datos, análisis de costos, recorrido en video |

## Decisiones cerradas

| ADR | Decisión |
|---|---|
| 0001 | Región: East US 2 |

## Decisiones abiertas

Cada una será un ADR cuando se cierre. El perfilado de la fase 1 es
el insumo de las tres primeras.

* **Estrategia de incrementalidad.** Qué columna sirve de marca de agua,
  qué ventana de reproceso se usa para capturar modificaciones
  retroactivas, y dónde se persiste el estado.
* **Esquema de particionado.** Por qué campo y con qué granularidad.
  Dato relevante ya medido: en Blob Storage las operaciones representan
  más del 90 % del costo, así que la granularidad de escritura es
  una decisión económica además de técnica (ver ADR 0001).
* **Formato de archivo.** Parquet frente a alternativas, y por qué.
* **Motor de transformación.** pandas frente a polars u otra opción.
  Se decidirá con el volumen real medido, no antes.
* **Clave de negocio y estrategia de deduplicación.**
* **Redundancia por capa.** Bronze es reconstruible desde la fuente;
  gold no. Eso debería reflejarse en la elección de LRS o ZRS.

## Alcance: lo que este proyecto no hace

Declararlo evita expectativas equivocadas y es parte de la honestidad
del portafolio.

* **No es streaming.** El procesamiento es por lotes, con periodicidad
  diaria.
* **No usa procesamiento distribuido.** El volumen previsto cabe en una
  sola máquina; introducir Spark aquí sería complejidad sin beneficio.
* **No es un sistema de producción.** No tiene alta disponibilidad,
  RPO/RTO definidos ni validación legal para el tratamiento de datos
  personales.
* **No usa Data Factory ni Synapse.** Son las herramientas gestionadas
  habituales para esto; se descartaron por costo y porque el objetivo
  incluye demostrar la lógica escrita en código. El razonamiento queda
  en el ADR correspondiente.

## Competencias que el proyecto demuestra

Solo las que efectivamente se implementan:

ingesta incremental idempotente · arquitectura por capas (medallion) ·
modelado dimensional · calidad de datos con cuarentena ·
orquestación · observabilidad con KQL y alertas ·
IAM con mínimo privilegio · infraestructura como código ·
optimización de costos
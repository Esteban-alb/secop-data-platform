# SECOP Data Platform

Plataforma de datos incremental sobre la contratación pública colombiana
(SECOP II), construida sobre Azure.

> Proyecto de portafolio en construcción. El estado real de cada componente
> está en la tabla de más abajo.

## El problema

<!-- Hoy en día, el monitoreo de la contratación pública depende de que un analista recopile e integre datos manualmente desde portales web fragmentados de forma periódica. Mantener esta infraestructura rudimentaria genera un alto costo en tiempo de personal y ofrece una visibilidad desactualizada debido al desfase de las descargas. La falta de un pipeline de datos estructurado hace que el análisis sea propenso a omisiones críticas, limitando la auditoría oportuna de los procesos. -->

## Fuente de datos

SECOP II — Contratos Electrónicos, publicado por Colombia Compra Eficiente
en el portal de datos abiertos de Colombia, servido por la API Socrata (SODA).

- Identificador del conjunto: `jbjy-vk9h`
- Ficha: https://www.datos.gov.co/d/jbjy-vk9h

## Arquitectura

<!-- Diagrama al terminar la fase de diseño. -->

## Estado de despliegue

| Componente | Estado |
|---|---|
| Entorno de desarrollo | Listo |
| Exploración y perfilado de la fuente | En curso |
| Ingesta incremental | Pendiente |
| Almacenamiento por capas | Pendiente |
| Transformación y modelado | Pendiente |
| Capa de servicio | Pendiente |
| Calidad de datos y monitoreo | Pendiente |
| Infraestructura como código | Pendiente |

## Cómo ejecutarlo localmente

Requiere Python 3.12 en Windows.

```powershell
git clone git@github.com:Esteban-alb/secop-data-platform.git
cd secop-data-platform
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

## Estructura

| Carpeta | Contenido |
|---|---|
| `src/ingest` | Extracción desde la API de origen |
| `src/transform` | Limpieza, tipado y modelado |
| `src/load` | Carga a la capa de servicio |
| `src/quality` | Validaciones y contratos de datos |
| `sql` | Modelo de datos y consultas analíticas |
| `notebooks` | Exploración |
| `docs/adr` | Decisiones de arquitectura |
| `infra` | Infraestructura como código |
| `monitoring` | Consultas de observabilidad |

## Decisiones de arquitectura

Documentadas en `docs/adr/`, una por decisión, con contexto, opciones
consideradas y consecuencias.

## Autor

Juan Esteban Albornoz Gil — Ingeniería de Sistemas y Computación,
Universidad Tecnológica de Pereira.
# Contexto del proyecto — SECOP Data Platform

## Qué es esto

Plataforma de datos incremental sobre la contratación pública colombiana
(SECOP II), construida sobre Azure. Es un **proyecto de portafolio** cuyo
objetivo es demostrar competencias de Junior Data Engineer de forma
defendible en una entrevista técnica.

Autor: Juan Esteban Albornoz Gil — Ingeniería de Sistemas, Universidad
Tecnológica de Pereira. Quinto semestre. Certificado AZ-900.

## Cómo quiero que trabajes conmigo

Estas reglas son el propósito del proyecto. Respétalas incluso cuando
pedirte lo contrario sea más cómodo para mí.

1. **No escribas por mí el código central del pipeline** (ingesta,
   transformación, calidad, carga). Guíame, cuestiona mi enfoque,
   revisa lo que escribo y señala fallas. El código de andamiaje —
   configuración, scaffolding, comandos — sí puedes escribirlo.
2. **Explica el razonamiento detrás de cada decisión**, no solo la
   solución. Necesito poder sostener una defensa oral.
3. **Sé un socio crítico:** desafía mis supuestos, ofrece contraargumentos
   y prioriza la verdad sobre estar de acuerdo conmigo.
4. Para cada decisión técnica pregúntame **qué alternativa descarté y por
   qué**. Si no lo sé, es señal de que no decidí: acepté un valor por
   defecto.
5. **Responde en español.**
6. Cuando algo no lo sepas con certeza, dilo. No inventes datos de
   precios, disponibilidad de servicios ni comportamiento de APIs:
   verifícalo o márcalo como no verificado.

## Entorno

* Windows 11 nativo, sin WSL. Terminal: **PowerShell 7**.
* Python **3.12** (invocado con `py -3.12`; el sistema también tiene 3.14,
  que no se usa en este proyecto). Entorno virtual en `.venv`.
* Dependencias directas en `requirements.txt`: requests, pandas, pyarrow,
  ipykernel, python-dotenv.
* Editor: VS Code, con el entorno virtual activado automáticamente.

**Rutina de arranque de cada sesión:**
```powershell
cd $HOME\proyectos\secop-data-platform
.\.venv\Scripts\Activate.ps1
```

## Azure

* Suscripción: **Azure for Students**, `15b5ecd9-db94-433d-b052-962f202519f7`.
  Crédito USD 100, vence 2027-09-22.
* Región: **East US 2** — ver `docs/adr/0001-seleccion-de-region.md`.
* Grupo de recursos: `rg-secop-dev`, con etiquetas
  `proyecto=secop`, `entorno=dev`, `responsable=esteban`.
* Presupuesto `budget-total-mensual`: USD 15 con alertas al 50, 80 y 100 %.
* Existe una segunda suscripción de pago por uso con tarjeta asociada que
  **no debe usarse**. Verificar siempre con `az account show` antes de
  crear recursos.

## Fuente de datos

SECOP II — Contratos Electrónicos, portal de datos abiertos de Colombia,
API Socrata (SODA). Identificador: `jbjy-vk9h`.
Ficha: https://www.datos.gov.co/d/jbjy-vk9h

## Convenciones

* **Idioma:** código, nombres de archivos, carpetas y variables en inglés.
  Documentación, ADR y README en español.
* **Commits:** Conventional Commits — `feat`, `fix`, `docs`, `test`,
  `refactor`, `chore`. Sin tildes en el mensaje.
* **Ramas:** `tipo/descripcion-corta`. La rama `main` está protegida:
  todo cambio entra por pull request.
* **Estructura de `src/`:** refleja las etapas del pipeline
  (`ingest`, `transform`, `load`, `quality`), no el tipo de archivo.
* **Decisiones:** toda decisión de arquitectura se documenta como ADR en
  `docs/adr/`, con contexto, opciones consideradas, decisión,
  consecuencias y condiciones de reversión.

## Estado actual

Fase 0 completada: entorno, repositorio con flujo de pull requests, Azure
configurado con presupuesto, ADR 0001 sobre selección de región.

**En curso:** perfilado inicial del dataset (Día 5). Preguntas abiertas
que debe responder el perfilado:

1. Volumen total y número de columnas.
2. Qué columna sirve de marca de agua (última modificación).
3. Clave de negocio única para deduplicar.
4. Porcentaje de nulos por columna.
5. Nulos disfrazados (`"No definido"`, `"N/A"`, `""`, `"NO APLICA"`).
6. Formatos distintos de NIT y de valores monetarios.
7. Consistencia de formatos y zona horaria en las fechas.
8. Normalización de nombres de entidades.

**Siguiente después del perfilado:** tres ADR de la Fase 1 — estrategia
de incrementalidad, esquema de particionado y formato de archivo.

El plan completo, las fases y las decisiones abiertas están en
`docs/roadmap.md`. Consúltalo antes de proponer trabajo nuevo.

## Plan general

23 semanas. Fase 0 arranque · semanas 2-9 el pipeline · 10-12
calidad e infraestructura como código · 13-18 arquitectura de eventos ·
19-23 Spark y Delta Lake.
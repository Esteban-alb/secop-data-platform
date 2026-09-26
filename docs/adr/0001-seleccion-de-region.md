# ADR 0001 — Selección de región de Azure

**Fecha:** 2026-09-26
**Estado:** Aceptada

## Contexto

El proyecto necesita una región de Azure para alojar el grupo de recursos
`rg-secop-dev` y todos los servicios de la plataforma.

Restricciones:

* Crédito limitado de Azure for Students (USD 100 / 12 meses).
* Procesamiento por lotes diario, sin usuarios finales interactivos: la
  latencia no es un requisito funcional.
* Datos de origen públicos (SECOP II, portal de datos abiertos), pero que
  contienen nombres y documentos de contratistas persona natural.
* La arquitectura depende de tres servicios: Azure Blob Storage, Azure
  Functions en plan de consumo y Azure SQL Database.

## Método de evaluación

**Configuración de referencia**, idéntica en las tres regiones. Entre
mediciones solo se modificó el campo *Región*:

| Parámetro | Valor |
|---|---|
| Servicio | Azure Blob Storage |
| Tipo | Almacenamiento de blobs en bloque |
| Tipo de cuenta | Uso general V2 |
| Rendimiento | Estándar |
| Estructura de archivos | Espacio de nombres plano |
| Nivel de acceso | Acceso frecuente |
| Redundancia | LRS |
| Capacidad | 5 GB |
| Operaciones de escritura | 100.000 / mes |
| Operaciones List y Create Container | 100.000 / mes |
| Operaciones de lectura | 100.000 / mes |
| Otras operaciones | 10.000 / mes |
| Recuperación de datos | 1.000 GB |

Precios consultados en la calculadora de precios de Azure el 2026-09-26.

**Disponibilidad de servicios:** verificada con Azure CLI, no con la
documentación de productos por región. Ver *Nota de método*.

```
az functionapp list-consumption-locations --output table
az provider show --namespace Microsoft.Sql \
  --query "resourceTypes[?resourceType=='servers'].locations" --output json
```

## Estructura del costo

Desglosando el total mensual entre operaciones y almacenamiento:

| Región | Operaciones | Almacenamiento | Total | Peso de operaciones |
|---|---|---|---|---|
| East US 2 | USD 1,05 | USD 0,09 | **USD 1,14** | 92 % |
| Chile Central | USD 1,47 | USD 0,12 | **USD 1,59** | 92 % |
| Brazil South | USD 1,47 | USD 0,15 | **USD 1,62** | 91 % |

Dos observaciones relevantes para el diseño:

1. **El costo está dominado por las transacciones, no por el volumen
   almacenado.** Los 5 GB representan menos del 10 % de la factura en las
   tres regiones. La ventaja de East US 2 proviene de que sus operaciones
   cuestan un 29 % menos (USD 0,050 frente a USD 0,070 por cada 10.000
   operaciones de escritura), no de que almacenar sea más barato.
2. **Chile Central y Brazil South tienen precios de operación idénticos.**
   La diferencia de tres centavos entre ambas se explica únicamente por
   el precio del almacenamiento.

La primera observación tiene consecuencias directas sobre la estrategia
de escritura del pipeline, y se retomará en el ADR correspondiente al
particionado y la granularidad de los archivos.

## Opciones consideradas

### East US 2

* Costo de referencia: **USD 1,14/mes** (USD 13,63/año).
* Los tres servicios requeridos están disponibles.
* Tres zonas de disponibilidad, lo que habilita redundancia ZRS.
* Cuenta con región emparejada dentro del mismo límite geopolítico
  (Estados Unidos), lo que habilita redundancia geográfica administrada
  GRS y GZRS.
* Latencia alta desde Colombia, irrelevante para cargas por lotes.
* Mayor catálogo general de servicios y de ofertas gratuitas del
  portafolio de Azure. No pesa en esta decisión —los tres servicios que
  el proyecto necesita están en las tres opciones— pero reduce el riesgo
  de topar con un servicio no disponible si la arquitectura crece.

### Brazil South

* Costo de referencia: **USD 1,62/mes — 42 % más caro** que East US 2.
* Los tres servicios requeridos están disponibles.
* Tres zonas de disponibilidad.
* Menor latencia desde Colombia.
* Emparejada con **South Central US**: bajo redundancia geográfica, la
  copia secundaria queda en Estados Unidos de todos modos. Elegirla
  buscando proximidad geográfica de los datos sería, en ese escenario,
  una falsa garantía.

**Descartada** por costo. El 42 % de sobreprecio compra un beneficio de
latencia que una carga por lotes diaria no aprovecha.

### Chile Central

* Costo de referencia: **USD 1,59/mes — 39 % más caro** que East US 2.
* Los tres servicios requeridos están disponibles, incluida Azure
  Functions en plan de consumo (verificado con Azure CLI, 2026-09-26).
* Tres zonas de disponibilidad.
* Menor latencia desde Colombia que East US 2.
* **No tiene región emparejada.** Esto no impide usar la región, pero
  elimina el mecanismo de *paired regions* de Azure: cualquier estrategia
  de recuperación entre regiones tendría que diseñarse y operarse
  manualmente, asumiendo las implicaciones de arquitectura, transferencia
  de datos y RPO/RTO.
* **Limitación de la comparación:** la calculadora reporta «precios no
  disponibles para la selección realizada en esta región» en los
  conceptos de *recuperación de datos* y *escritura de datos*, que en las
  otras dos regiones figuran en USD 0,00. La cifra de USD 1,59 es, por
  tanto, un piso y no un total verificado.

**Descartada** por costo (39 % superior, posiblemente más) y por la
ausencia de región emparejada, que reduce las opciones de resiliencia
sin aportar un beneficio que este proyecto aproveche.

## Decisión

Se elige **East US 2**.

La disponibilidad de los tres servicios requeridos es equivalente en las
tres opciones evaluadas, de modo que la viabilidad técnica no discrimina
entre ellas. El criterio determinante pasa a ser el costo, donde East
US 2 resulta entre 39 % y 42 % más económica —una diferencia relativa que
se mantiene a cualquier volumen— frente a un beneficio de latencia que
esta carga de trabajo no utiliza.

Como criterio secundario, East US 2 y Brazil South cuentan con región
emparejada y Chile Central no, lo que da a las dos primeras más opciones
de resiliencia si el proyecto llegara a requerirlas.

La decisión está alineada con las características actuales del proyecto:
procesamiento por lotes diario, ausencia de usuarios interactivos y
presupuesto limitado de Azure for Students.

## Sobre residencia de datos

Se evaluó si la Ley 1581 de 2012 impone una restricción de región.

Conclusión: **la región no es el control adecuado para este riesgo**,
porque Azure no ofrece ninguna región en territorio colombiano —las más
cercanas están en Brasil, Chile y México— y porque Brazil South replica
hacia Estados Unidos bajo redundancia geográfica. Ninguna opción
disponible satisface un requisito estricto de residencia nacional, de
modo que elegir la región por este motivo daría una sensación de
cumplimiento sin el cumplimiento.

Los controles pertinentes son de otro tipo:

* **Minimización de datos:** conservar únicamente las columnas necesarias
  para los análisis y excluir del pipeline documentos de identidad,
  nombres u otros datos personales que no sean necesarios para el
  objetivo analítico.
* **Pseudonimización o anonimización:** cuando sea posible, reemplazar
  identificadores personales por identificadores técnicos o por datos
  agregados antes de que la información llegue a las capas utilizadas
  para análisis.
* **Control de acceso:** aplicar RBAC con el principio de mínimo
  privilegio, restringiendo el acceso a datos personales a los usuarios y
  servicios que lo requieran, y separando la capa de datos crudos de las
  capas transformadas destinadas al análisis.

Este punto requiere validación legal antes de cualquier uso en
producción; el presente ADR documenta una decisión técnica, no una
opinión jurídica.

## Consecuencias

* Se acepta latencia alta desde Colombia, sin impacto funcional en cargas
  por lotes.
* Todos los recursos del proyecto deben crearse en `eastus2`. En el
  modelo de precios de Azure la entrada de datos es gratuita y la salida
  se cobra, de modo que concentrar los recursos en una sola región evita
  cargos de transferencia entre regiones. El egress que sí se pagará es
  el de consultar resultados desde fuera de Azure.
* La opción más económica preserva margen del crédito de Azure for
  Students para otros componentes del proyecto.
* Dado que las operaciones representan más del 90 % del costo de
  almacenamiento, el diseño del pipeline debe minimizar el número de
  transacciones sobre Blob Storage. Escribir en lotes en lugar de generar
  un archivo por registro no es solo una optimización de rendimiento:
  es la principal palanca de costo de esta capa.
* La estrategia de recuperación ante desastres queda condicionada a las
  capacidades de East US 2 y a una eventual arquitectura multirregional,
  si el proyecto adquiere requisitos más estrictos de disponibilidad.
* Los datos personales deberán tratarse mediante controles de
  minimización, protección y acceso, con independencia de la región.
* **Reversibilidad:** hoy el cambio de región es trivial, porque ningún
  recurso contiene datos. El costo de revertir crece con el volumen
  almacenado, ya que implicaría migrar datos entre regiones y asumir el
  cargo de transferencia de salida. La ventana barata para reconsiderar
  esta decisión se cierra cuando la capa bronze empiece a acumular
  histórico.
* Esta decisión no constituye una determinación de cumplimiento legal.
  Cualquier requisito específico de residencia, transferencia
  internacional o tratamiento de datos personales deberá validarse antes
  del uso en producción.

## Qué me haría reconsiderar

* **Usuarios interactivos:** si el proyecto evoluciona hacia una
  aplicación con usuarios finales en Colombia y la latencia desde East
  US 2 empieza a afectar la experiencia de uso o los SLA definidos.
* **Requisito legal explícito:** si una autoridad, un contrato o una
  regulación aplicable establece que determinados datos deben
  almacenarse o procesarse en una jurisdicción específica. Nótese que,
  si esa jurisdicción fuera Colombia, la respuesta no sería cambiar de
  región de Azure sino replantear la plataforma.
* **Requisitos de continuidad del negocio:** si el proyecto pasa de un
  procesamiento por lotes no crítico a un sistema con RPO/RTO estrictos
  que justifique una arquitectura multirregional.

## Nota de método

La disponibilidad de servicios se verificó con Azure CLI y no con la
documentación de productos por región. Durante la evaluación, ambas
fuentes arrojaron resultados distintos para Azure Functions en Chile
Central: la documentación sugería que no estaba disponible, mientras que
`az functionapp list-consumption-locations` sí la lista.

Se tomó la API como fuente autoritativa, por reflejar el estado operativo
real del plano de control de Azure y no la documentación comercial, que
se mantiene manualmente y puede ir con retraso. Los comandos utilizados
quedan citados en este documento para que la verificación sea
reproducible por cualquier lector.
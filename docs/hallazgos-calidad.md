# Hallazgos de calidad — SECOP II Contratos Electrónicos

Resultado del perfilado inicial del dataset `jbjy-vk9h` (Fase 1). El detalle
y las consultas están en `notebooks/00-exploracion.ipynb`; este documento
resume qué se encontró y qué implica para las decisiones de diseño.

## Método

* **Fecha del perfilado:** 2026-10-08 y 2026-10-09. Los metadatos del portal
  indicaban como última actualización de filas el 2026-10-08 a las 18:58 UTC.
  Las cifras de este documento corresponden a esa versión del dataset.
* **Conteos exactos en el servidor.** Volumen, nulos, duplicados y nombres por
  NIT se calcularon con consultas SoQL agregadas sobre el dataset completo.
* **Muestra de 50.000 filas** para lo que exige mirar valores: nulos
  disfrazados, formatos y fechas. La muestra se paginó ordenando por `:id`, el
  identificador interno de Socrata. **No es aleatoria**: sirve para descubrir
  formatos, y los porcentajes que salen de ella son aproximados.

## 1. Volumen y estructura

El dataset tiene **5.965.816 filas y 95 columnas**. Según los metadatos hay 64
columnas de texto, 19 numéricas, 11 de fecha y 1 de tipo URL.

Dos rasgos del formato de respuesta afectan la ingesta:

* **La API devuelve todo como texto.** Números y fechas llegan como cadenas
  en el JSON, así que el tipado tiene que hacerse de forma explícita a partir
  de los metadatos, no inferirse.
* **Los nulos se omiten y hay campos anidados.** Un campo nulo no aparece en
  el objeto JSON de esa fila, y `urlproceso` llega como objeto anidado
  (`{"url": "..."}`) en lugar de texto plano. Si el esquema se infiere de una
  página de resultados, puede faltar una columna o cambiar de tipo según qué
  filas traiga esa página.

## 2. Marca de agua

Una marca de agua confiable tiene que cumplir cuatro criterios:

1. Estar completa.
2. Cambia SOLO cuando la fila cambia.
3. Tener granularidad suficiente.
4. Ser coherente con el resto de los datos.

Se evaluaron dos candidatas y **ninguna cumple los cuatro**.

**Campos de sistema de Socrata (`:created_at`, `:updated_at`).** En las casi
6 millones de filas tienen exactamente el mismo valor: 2026-10-08 16:47:32 UTC.
Eso indica que el portal **reemplaza el dataset completo en cada carga**. Estos
campos dicen cuándo se recargó el portal, no cuándo cambió cada contrato, así
que fallan el criterio 2. Descartados.

**`ultima_actualizacion` (columna de negocio).**

* *Criterio 1, completitud.* Falla. Está nula en el 41,7 % de las filas, y
  los nulos no están repartidos al azar:

  | Estado del contrato | % sin `ultima_actualizacion` |
  |---|---|
  | En ejecución | 99,5 % |
  | Aprobado | 99,3 % |
  | Borrador, cancelado, en aprobación, enviado al proveedor | 100 % |
  | Cerrado, modificado, terminado, cedido | 3–4 % |

  Falta justo en los contratos activos, que son los que todavía pueden
  cambiar. Hipótesis **no verificada**: el portal solo la llena cuando el
  contrato se modifica o se cierra.
* *Criterio 3, granularidad.* Falla. Ninguna fila tiene una hora distinta de
  00:00, así que la columna solo guarda la fecha.
* *Criterio 4, coherencia.* Se cumple casi siempre: solo 6 de 3.477.546 filas
  comparables tienen la actualización antes de la firma.
* *Criterio 2.* **Sin verificar.** Comprobarlo requiere comparar dos
  descargas hechas en días distintos.

**Implicación.** Una carga incremental que solo pida las filas con
`ultima_actualizacion` posterior a la última marca nunca vería los cambios de
los contratos en ejecución. El ADR de incrementalidad tiene que evaluar
alternativas, por ejemplo:

* extraer el dataset completo y comparar contra la carga anterior;
* combinar la marca de agua con una ventana de reproceso y un mecanismo
  aparte para las filas sin fecha.

Este documento no toma esa decisión.

## 3. Clave de negocio

Una clave de negocio tiene que cumplir tres criterios: nunca nula, única y
estable entre cargas. Se compararon tres candidatas. Ninguna tiene nulos, así
que lo que las distingue es la unicidad:

| Columna | Valores repetidos | Interpretación |
|---|---|---|
| `referencia_del_contrato` | ≈ 1,56 millones | Referencia que asigna cada entidad; entidades distintas reutilizan las mismas |
| `proceso_de_compra` | ≈ 712.000 | Un proceso puede adjudicar varios contratos: identifica el proceso, no el contrato |
| `id_contrato` | 306 | Ver abajo |

**`id_contrato` es la clave elegida.** Tiene 306 valores repetidos, cada uno
exactamente dos veces, y se revisaron las 612 filas: **las 306 parejas son
copias idénticas**. Por eso deduplicar por copia exacta no pierde información.

Queda pendiente el criterio 3: comprobar que un mismo contrato conserva su
`id_contrato` entre cargas. Si en el futuro aparece un `id_contrato` repetido
con filas distintas, la deduplicación va a necesitar un criterio de versión, y
ese criterio depende de la marca de agua (ver la nota de acoplamiento en
`docs/roadmap.md`).

## 4. Nulos explícitos

Solo **11 de las 95 columnas tienen nulos reales, y todas son fechas**:

| Columna | % nulos |
|---|---|
| `fecha_inicio_reversi_n`, `fecha_fin_reversi_n` | 99,99 |
| `fecha_inicio_obligaciones_posconsumo`, `fecha_fin_obligaciones_posconsumo` | 99,97 |
| `fecha_de_notificaci_n_de_prorrogaci_n` | 90,36 |
| `fecha_inicio_liquidacion`, `fecha_fin_liquidacion` | 89,03 |
| `ultima_actualizacion` | 41,71 |
| `fecha_de_inicio_del_contrato` | 7,66 |
| `fecha_de_firma` | 7,02 |
| `fecha_de_fin_del_contrato` | 0,91 |

Que las otras 84 columnas aparezcan sin nulos **no significa que estén
completas**. La siguiente sección explica por qué.

## 5. Nulos disfrazados

En las columnas de texto, la ausencia de dato se representa con valores como
`No definido`, `No Definido`, `No aplica`, `Sin Descripcion` o `-`. Además, el
portal es inconsistente en mayúsculas y tildes para un mismo concepto, así
que solo se detectan bien después de normalizar (minúsculas, sin espacios
sobrantes).

Ejemplos en la muestra:

* **Columnas ambientales** (`uso_de_etiquetado_ecol_gico`,
  `criterios_ambientales_en_la_evaluaci_n_de_las_ofertas` y similares): cerca
  del 99,9 % es `No definido`. En la práctica están vacías.
* **Datos de pago y cuentas:** ordenador de pago `No definido` en el 83,8 %;
  número de cuenta en el 69,8 %.
* **Representante legal:** `Sin Descripcion` en el 64,7 % de las identificaciones.
* **Columnas de uso analítico:** `ciudad` es `No definido` en el ≈21,6 % de la
  muestra, `departamento` en el ≈2,1 % y `documento_proveedor` en el ≈2,9 %.

Hay una distinción que no se puede perder: en columnas sí/no como `es_pyme`
o `espostconflicto`, el valor `No` es una respuesta legítima, no un nulo.
Tratarlo como nulo borraría información real.

**Implicación.** La etapa bronze → silver necesita una lista explícita y
versionada de valores que se convierten a nulo, aplicada después de
normalizar. Es mejor que detectarlos de forma heurística, porque así queda
auditable.

## 6. Formatos de NIT y valores monetarios

**NIT de entidades.** `nit_entidad` es numérico. Hay 5.767 NIT distintos: 5.157
de 9 dígitos, 602 de 10 y algunos de 8. En **130 NIT de 10 dígitos, los
primeros 9 también existen como NIT de otra fila**. Por ejemplo, `890399029`
y `8903990295` aparecen como entidades distintas. Lo más probable es que la
misma entidad se registre a veces con el dígito de verificación pegado al
final. Es una hipótesis **no verificada**: confirmarla requiere calcular el
dígito de verificación (algoritmo módulo 11 de la DIAN) y compararlo.

**Documento del proveedor.** Su formato depende del tipo de documento:

* Las cédulas tienen entre 7 y 10 dígitos.
* Los NIT de proveedores aparecen con 8, 9 y 10 dígitos, con el mismo
  problema del dígito de verificación.
* Hay documentos de tipo NIT con texto en lugar de número, que son nulos
  disfrazados.

**Valores monetarios.** Todos los valores de la muestra se pudieron convertir
a número. Aun así hay tres problemas:

* **Decimales.** Unas 900 filas de `valor_del_contrato` traen centavos. El
  tipo de destino tiene que admitirlos sin perder precisión, así que no
  sirve un entero ni un `float` binario si se van a sumar montos.
* **Negativos** en `valor_pendiente_de_pago` y
  `valor_pendiente_de_ejecucion`: 25 filas en la muestra. Pueden ser ajustes
  legítimos o errores; la regla de calidad tiene que decidir cuál de los dos.
* **Valores imposibles.** El mayor `valor_del_contrato` de la muestra es
  ≈1,0 × 10¹⁸ pesos (un hospital municipal) y hay otros del orden de 10¹⁶. El
  presupuesto general de la nación ronda los 5 × 10¹⁴ pesos (orden de magnitud
  de referencia, no verificado en este proyecto), así que esos montos son
  errores de digitación casi con certeza. Una suma ingenua de
  `valor_del_contrato` queda dominada por unas pocas filas erróneas.

## 7. Fechas y zona horaria

Todas las fechas llegan en un único formato ISO (`AAAA-MM-DDTHH:MM:SS.sss`),
sin zona horaria. En Socrata, el tipo de fecha es un *floating timestamp*:
la zona no se puede leer del dato y hay que asumirla y documentarla.

* **Las fechas principales no traen hora.** Firma, inicio, fin y última
  actualización siempre marcan 00:00.
* **Hay indicios de UTC en columnas secundarias.** Las fechas de obligaciones
  posconsumo marcan 05:00 y 04:59. Eso coincide con la medianoche de Colombia
  (UTC−5) guardada en UTC, y sugiere que **no todas las columnas siguen la
  misma convención**. Las de reversión marcan 17:00, que no encaja con esa
  explicación. Hipótesis **no verificada**.
* **Rangos sospechosos.** Hay fechas de fin de contrato hasta 2050, fechas de
  liquidación desde 2012 (anteriores a cualquier firma de la muestra) y 4
  contratos de la muestra cuya fecha de fin es anterior a la de inicio. No
  apareció ninguna firma en el futuro ni anterior a 2015.

**Implicación.** Antes de convertir fechas, la transformación tiene que fijar
una convención (por ejemplo, fechas de negocio como `date` sin hora, en hora
de Colombia) y aplicarla columna por columna, porque no todas siguen la
misma.

## 8. Nombres de entidades

Hay **5.767 NIT distintos y 6.480 nombres distintos**, y **342 NIT tienen más
de un nombre**. El caso extremo es el NIT del SENA, con 83 nombres. Casi
todos son sus regionales y grupos administrativos ("SENA REGIONAL VALLE Grupo
de Apoyo Administrativo Mixto", "SENA SECRETARIA GENERAL", etc.).

Normalizar mayúsculas, tildes y puntuación casi no cambia nada: en la muestra,
2.625 nombres distintos pasan a 2.624. Aunque sí hay ruido de formato (dobles
espacios, asteriscos o barras al final, como en "ALCALDIA  MUNICIPIO DE
IBAGUE*"), **el problema principal no es de formato sino de modelado**:

* el NIT identifica a la persona jurídica;
* el nombre identifica la dependencia que contrata.

**Implicación para el modelo dimensional (Fase 5).** Hay que decidir si la
dimensión de entidad se identifica por NIT, con la dependencia como atributo,
o si la dependencia es una dimensión propia. A eso se suma el problema del
dígito de verificación (sección 6): sin normalizar el NIT, una misma entidad
aparecería dos veces en la dimensión.

## Verificaciones pendientes

| Pendiente | Cómo se verifica |
|---|---|
| ¿`ultima_actualizacion` cambia cuando cambia el contrato? | Comparar dos descargas de días distintos |
| ¿`id_contrato` es estable entre cargas? | Comparar dos descargas de días distintos |
| ¿`:id` cambia en cada recarga del portal? (afecta la reproducibilidad de la muestra) | Comparar los `:id` de un mismo `id_contrato` en dos días |
| ¿Los NIT de 10 dígitos son NIT de 9 más el dígito de verificación? | Calcular el dígito de verificación módulo 11 y comparar |
| ¿Qué convención de zona horaria sigue cada columna de fecha? | Revisar la documentación del portal y casos concretos |
| ¿Las proporciones de la muestra representan al dataset completo? | Repetir los conteos clave en el servidor o con una muestra aleatoria |

## Preguntas que estos hallazgos dejan a los ADR de la Fase 1

* **Incrementalidad.** Sin una marca de agua confiable, ¿extracción completa
  con comparación, marca de agua con ventana de reproceso, o una combinación?
  ¿Cuánto cuesta descargar 6 millones de filas cada día?
* **Particionado.** La fecha de negocio más completa, `fecha_de_fin_del_contrato`,
  tiene un 0,91 % de nulos y valores hasta 2050; `fecha_de_firma` tiene un 7 %
  de nulos. ¿Se particiona por una fecha de negocio o por fecha de ingesta?
* **Formato de archivo.** Como la API entrega todo como texto, con campos
  omitidos y anidados, el formato de bronze tiene que preservar el dato tal
  como llegó, y el de silver tiene que fijar los tipos. ¿Es el mismo formato
  para ambas capas?

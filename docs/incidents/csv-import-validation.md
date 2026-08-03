# Caso tecnico: rechazo masivo en importacion CSV de tarjeta

Documento anonimizado. No incluye el archivo original, filas reales, comercios, fechas productivas, importes reales ni identificadores de cuenta o tarjeta.

## 1. Contexto

CashFlow Control permite a un administrador importar movimientos de tarjeta desde CSV. El flujo decodifica el buffer, detecta el delimitador, normaliza cabeceras y, por cada fila, parsea fecha, concepto e importe antes de persistir el movimiento.

## 2. Sintoma observado

En una importacion de demostracion del fallo, el resumen devolvio:

* N filas procesadas
* N errores de parseo o validacion
* 0 movimientos importados

El caso se reprodujo con un fixture de 61 filas homogeneas. El numero exacto no es relevante: el patron era rechazo total.

## 3. Impacto

* Bloqueo operativo de la carga de tarjeta.
* Riesgo de interpretar el fallo como problema de archivo o de delimitador cuando el origen real estaba en la normalizacion del importe.
* Necesidad de una correccion con prueba de regresion antes de volver a produccion.

## 4. Hipotesis iniciales

| # | Hipotesis | Resultado |
| - | --------- | --------- |
| 1 | El parseo de importe no admite un sufijo textual de moneda seguido de punto | Confirmada |
| 2 | Codificacion CP1252 mal interpretada como UTF-8 | Secundaria; existe fallback |
| 3 | Cabeceras acentuadas sin alias | Descartada para el fallo masivo |
| 4 | Validacion incorrecta de comision cero | Descartada |
| 5 | Fecha con hora no portable | Descartada; el prefijo de fecha era valido |
| 6 | Constraint de base de datos en todas las filas | Descartada; las filas no llegaban al INSERT |

## 5. Estrategia de reproduccion con fixture ficticio

Se construyo un CSV sintetico (sin datos reales) y se ejecuto el mismo pipeline de importacion en entorno local.

El fixture publico de este repositorio ilustra casos tipicos:

[assets/examples/card-import-demo.csv](../../assets/examples/card-import-demo.csv)

Incluye:

* fila valida;
* fecha con hora;
* importe con coma decimal y sufijo textual de moneda;
* fila invalida sin importe;
* fila repetida para recordar que la importacion de tarjeta no aplica deduplicacion.

## 6. Instrumentacion temporal

Se anadio instrumentacion local (logs de diagnostico y probes unitarios sobre la funcion de parseo de importe) para observar, fila a fila:

* valor crudo del campo importe;
* resultado del parseo (`number` o `null`);
* motivo de descarte en la validacion previa al INSERT.

La instrumentacion no se publico con datos reales.

## 7. Punto exacto de descarte

El descarte ocurria **antes** de persistir:

1. Se leia el importe de la fila.
2. La funcion de parseo devolvía `null`.
3. La validacion exigia importe, fecha y concepto validos.
4. La fila se contabilizaba como error y el flujo continuaba con la siguiente.

Por eso el resumen mostraba N procesados, N errores y 0 importados.

## 8. Causa raiz (confirmada)

La normalizacion del importe eliminaba simbolos de moneda habituales, pero **no** trataba un sufijo textual del tipo `eur.` / `EUR.`.

Tras limpiar miles y decimales, el texto residual dejaba un punto final o un token no numerico. El resultado era `NaN` convertido a `null`, y la validacion rechazaba la fila.

Cuando todas las filas del fichero compartian ese formato de importe, el rechazo era total.

## 9. Correccion aplicada

Se amplio la normalizacion del importe para:

* eliminar sufijos textuales de moneda (con o sin punto final) **antes** de la logica de miles/decimales;
* seguir admitiendo simbolos y espacios no separables;
* devolver un numero finito o `null` de forma explicita.

La correccion se integro en una rama `fix/*`, se reviso y se desplego de forma controlada.

## 10. Prueba de regresion

Se anadieron pruebas automatizadas (Vitest) que cubren:

* parseo de importes con sufijo textual de moneda;
* importacion CSV de tarjeta con un fixture de muchas filas homogeneas;
* casos de importe o fecha invalidos;
* confirmacion de que la reimportacion de tarjeta crea filas nuevas (sin deduplicacion bancaria).

## 11. Riesgos residuales

* Nuevos formatos de importe de emisores distintos pueden requerir aliases adicionales.
* Mensajes de error genericos dificultan el diagnostico si vuelve a fallar el parseo; conviene seguir mejorando el detalle por fila.
* La importacion sigue siendo fila a fila: un fallo parcial no hace rollback del lote completo.

## 12. Lecciones aprendidas

* Un rechazo del 100% suele apuntar a un paso comun (parseo/normalizacion), no a datos aislados.
* Los fixtures sinteticos permiten reproducir sin exponer informacion financiera.
* La regresion debe anclar el formato concreto que fallo (`eur.`), no solo el caso feliz.
* Separar "error de parseo" de "error de persistencia" acelera el diagnostico.

## 13. Medidas preventivas

* Mantener pruebas de parseo con variantes de moneda y separadores.
* Documentar formatos de entrada admitidos para operadores.
* Revisar contadores de importacion (importados / omitidos / errores) antes de dar por buena una carga.
* Evitar publicar CSV reales en repositorios o tickets.

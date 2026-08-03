# ADR 0002 — Mantener el codigo de produccion en repositorio privado

* Estado: Aceptado
* Fecha: 2026

## Contexto

La aplicacion gestiona movimientos financieros, usuarios, reglas de clasificacion, importaciones bancarias y copias de seguridad. Publicar el codigo fuente completo aumentaria el riesgo de exponer logica interna, convenciones de datos y detalles del entorno.

## Decision

Conservar el codigo de CashFlow Control en un repositorio privado y publicar este caso de estudio con documentacion, diagramas, capturas y ejemplos completamente ficticios o anonimizados.

## Consecuencias positivas

* Separacion clara entre evidencia profesional y activo productivo.
* Posibilidad de mostrar arquitectura y SQL sin filtrar datos reales.
* Control del acceso al codigo durante procesos de seleccion.

## Consecuencias negativas

* Un revisor externo no puede ejecutar la aplicacion desde este repositorio.
* Requiere disciplina continua de anonimizacion en cada mejora documental.

## Alternativas consideradas

* Open source completo: rechazado por sensibilidad financiera y contexto interno.
* Demo publica con datos reales ofuscados: riesgo residual alto; se descarta.
* Solo README narrativo sin SQL ni modelo: insuficiente para una evaluacion tecnica seria.

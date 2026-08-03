# ADR 0001 — Usar SQLite para un despliegue local controlado

* Estado: Aceptado
* Fecha: 2026

## Contexto

CashFlow Control es una herramienta interna de tesoreria que se ejecuta en una red local, con un volumen de datos controlado y un numero limitado de usuarios concurrentes. Se necesitaba una base relacional fiable, sencilla de respaldar y sin operar un servidor de base de datos adicional.

## Decision

Utilizar SQLite como motor de persistencia principal, con el archivo de datos fuera del repositorio y copias de seguridad basadas en el propio fichero.

## Consecuencias positivas

* Despliegue con un unico proceso Node.js.
* Copias y restauraciones operativamente simples.
* SQL relacional real (joins, agregaciones, indices, restricciones CHECK).
* Menor superficie operativa frente a un PostgreSQL autoalojado en la fase inicial.

## Consecuencias negativas

* Concurrencia de escritura mas limitada que en un servidor cliente/servidor.
* Escalado horizontal y multiempresa no son el foco del diseno actual.
* Observabilidad y migraciones exigen disciplina propia (esquema aplicado al arranque).

## Alternativas consideradas

* PostgreSQL desde el inicio: mas capacidad, mas operaciones.
* Solo hojas de calculo: insuficiente para RBAC, importaciones y auditoria operativa.
* SaaS externo: incompatible con el requisito de datos en infraestructura controlada.

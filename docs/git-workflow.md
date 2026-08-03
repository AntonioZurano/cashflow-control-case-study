# Flujo de trabajo con Git

CashFlow Control (repositorio privado de produccion) y este caso de estudio publico comparten la misma disciplina de ramas, aunque este repositorio solo documenta el producto.

## Proposito de las ramas

| Rama | Proposito |
| ---- | --------- |
| `main` | Versiones estables listas para publicacion o, en el privado, para despliegue |
| `development` | Integracion y validacion antes de promover a `main` |
| `feature/*` | Nuevas funcionalidades acotadas |
| `fix/*` | Correcciones de errores (ejemplo: incidencia CSV de tarjeta) |
| `docs/*` | Mejoras documentales sin cambiar comportamiento de produccion |

## Ciclo habitual

```text
issue
  ↓
docs/sql-and-technical-evidence   (o feature/* / fix/*)
  ↓
pull request hacia development
  ↓
revision y validacion
  ↓
merge en development
  ↓
preparacion de release
  ↓
merge controlado en main
```

## Buenas practicas aplicadas

1. No trabajar directamente sobre `main`.
2. Revisar el diff completo antes de abrir la PR.
3. Ejecutar pruebas relevantes (Vitest / scripts / checklist manual) antes del merge.
4. En el privado: versionado SemVer y despliegue solo desde `main`.
5. Rollback: restaurar backup SQLite previo al despliegue y revertir el servicio si hace falta.
6. Proteger produccion: sin cambios de version, tags o restore sin decision humana explicita.

## Evidencia de esta mejora documental

Esta propia ampliacion del caso de estudio sigue el flujo anterior:

| Paso | Evidencia |
| ---- | --------- |
| Issue publica | Document SQL knowledge and technical decision-making |
| Rama | `docs/sql-and-technical-evidence` (desde `development`) |
| Commits pequenos | Reorganizacion README, modelo, SQL, incidencia CSV, API/ADR, PostgreSQL |
| Pull request | Hacia `development`, sin merge automatico |
| Checklist | Validacion de secretos, enlaces, Mermaid y afirmaciones verificadas |

## Proteccion de produccion

* El codigo de CashFlow Control no se modifica desde este repositorio.
* No se crean tags ni releases de la aplicacion privada desde el caso de estudio.
* Los agentes de IA no autorizan despliegues ni cambios de version por si solos.

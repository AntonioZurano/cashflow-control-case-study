# Vision general de la API (anonimizada)

Descripcion publica de los grupos de endpoints de CashFlow Control. Las rutas se presentan de forma generica; el codigo de produccion permanece privado.

Autenticacion: sesion por cookie tras `POST /api/auth/login`. Roles verificados: `user` y `admin`.

## Matriz de permisos

| Accion | Usuario (`user`) | Administrador (`admin`) |
| ------ | ----------------: | ----------------------: |
| Iniciar / cerrar sesion | Si | Si |
| Consultar movimientos y resumenes | Si | Si |
| Crear, editar o eliminar movimientos | Si | Si |
| Consultar dashboard y tendencias | Si | Si |
| Consultar subcategorias | Si | Si |
| Crear / editar / desactivar subcategorias | No | Si |
| Gestionar reglas de clasificacion | No | Si |
| Gestionar cuentas bancarias (Norma 43) | No | Si |
| Importar CSV tarjeta / XLS-XLSX / Norma 43 | No | Si |
| Exportar CSV o Norma 43 | No | Si |
| Gestionar usuarios | No | Si |
| Crear o restaurar copias de seguridad | No | Si |

La proteccion no depende de ocultar botones en el frontend: el middleware comprueba sesion y rol antes de ejecutar la operacion.

## Flujo de una peticion protegida

1. El frontend envia una solicitud autenticada (cookie de sesion).
2. El middleware valida la sesion.
3. El backend determina usuario y rol.
4. La ruta o servicio comprueba el permiso (`requireAuth` / `requireRole`).
5. Se validan y normalizan los parametros.
6. Se ejecuta una consulta parametrizada.
7. Se devuelve solo la informacion autorizada.
8. Las operaciones sensibles (backup, restore, import) usan limites de tasa y respuestas controladas.

## Grupos de endpoints

### Salud

| Metodo | Recurso | Auth | Descripcion |
| ------ | ------- | ---- | ----------- |
| GET | `/api/health` | Publica | Comprobacion de disponibilidad |

### Autenticacion

| Metodo | Recurso | Auth | Descripcion |
| ------ | ------- | ---- | ----------- |
| POST | `/api/auth/login` | Publica + rate limit | Inicia sesion |
| POST | `/api/auth/logout` | Autenticado | Cierra sesion |
| GET | `/api/auth/me` | Sesion si existe | Devuelve el usuario actual |

Validaciones: credenciales obligatorias; usuarios inactivos rechazados. Codigos tipicos: `200`, `401`, `429`.

### Movimientos

| Metodo | Recurso | Auth | Descripcion |
| ------ | ------- | ---- | ----------- |
| GET | `/api/movements` | Autenticado | Listado filtrado, ordenado y paginado |
| POST | `/api/movements` | Autenticado | Crea un movimiento validado |
| PUT | `/api/movements/:id` | Autenticado | Actualiza un movimiento |
| DELETE | `/api/movements/:id` | Autenticado | Elimina un movimiento |
| GET | `/api/movements/summary` | Autenticado | Totales de ingresos y gastos |

Parametros habituales de consulta: rango de fechas, canal, subcategoria, tipo, texto libre, pagina, tamano de pagina, columna de ordenacion.

Efectos: escritura en SQLite; recategorizacion masiva y aplicacion de reglas disponibles para usuarios autenticados segun rutas dedicadas.

### Dashboard

| Metodo | Recurso | Auth | Descripcion |
| ------ | ------- | ---- | ----------- |
| GET | `/api/dashboard/monthly` | Autenticado | Agregaciones del periodo |
| GET | `/api/dashboard/trends` | Autenticado | Series temporales mensuales |

### Subcategorias

| Metodo | Recurso | Auth | Descripcion |
| ------ | ------- | ---- | ----------- |
| GET | `/api/subcategories` | Autenticado | Lista el catalogo |
| POST / PUT / DELETE | `/api/subcategories` | Admin | Mantiene el catalogo |

### Reglas de clasificacion

| Metodo | Recurso | Auth | Descripcion |
| ------ | ------- | ---- | ----------- |
| GET / POST / PUT / DELETE | `/api/classification-rules` | Admin | Patrones para asignar subcategoria |

### Cuentas bancarias

| Metodo | Recurso | Auth | Descripcion |
| ------ | ------- | ---- | ----------- |
| CRUD + set-default | `/api/bank-accounts` | Admin | Metadatos para exportacion Norma 43 |

### Importacion y exportacion

| Metodo | Recurso | Auth | Descripcion |
| ------ | ------- | ---- | ----------- |
| POST | `/api/imports/card-csv` | Admin | Valida e importa CSV de tarjeta |
| POST | `/api/imports/bank-xls` | Admin | Importa extracto XLS/XLSX |
| POST | `/api/imports/bank-norma43` | Admin | Importa fichero Norma 43 |
| GET | `/api/exports/movements.csv` | Admin | Exporta respetando filtros (sin paginacion) |
| GET | `/api/exports/bank-norma43` | Admin | Exporta movimientos bancarios en Norma 43 |
| GET | `/api/import-batches` | Admin | Resume lotes por `import_batch_id` |

Efectos secundarios: creacion de movimientos, posible alta automatica de subcategorias, contadores de importados/omitidos/errores. La persistencia del lote es fila a fila (sin transaccion global de importacion).

### Administracion

| Metodo | Recurso | Auth | Descripcion |
| ------ | ------- | ---- | ----------- |
| GET / POST / PUT / DELETE | `/api/admin/users` | Admin | Gestion de usuarios |
| GET | `/api/admin/backups/download` | Admin | Descarga la base SQLite |
| POST | `/api/admin/backups/restore/upload` | Admin | Sube una copia para restaurar |
| GET | `/api/admin/backups/restore/:token/run` | Admin | Ejecuta la restauracion (progreso por SSE) |

Operaciones de backup/restore: confirmacion en interfaz, limites de tamano y rate limit en subidas.

## Codigos de respuesta habituales

| Codigo | Significado |
| ------ | ----------- |
| 200 / 201 | Operacion correcta |
| 400 | Validacion de entrada |
| 401 | Sesion ausente o invalida |
| 403 | Rol insuficiente |
| 404 | Recurso inexistente |
| 409 | Conflicto (por ejemplo, unicidad) |
| 429 | Rate limit |
| 500 | Error no controlado (sin detalles internos al cliente) |

## Pruebas relevantes

* Login / logout y restriccion por rol (`403` sin permiso).
* Filtros y paginacion coherentes entre listado y `COUNT`.
* Importacion CSV con importes y fechas limite (Vitest).
* Exportacion sin `LIMIT`/`OFFSET`.
* Backup y restore en entorno controlado (comprobacion manual de despliegue).

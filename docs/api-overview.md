# Vision general de la API (anonimizada)

Descripcion **conceptual** de capacidades, metodos HTTP y permisos de CashFlow Control, contrastada con la version publicada de referencia (**v3.2.0**). **No** es un contrato de API ni un mapa de rutas ejecutables: el codigo de produccion permanece privado y las rutas reales no se publican aqui.

Autenticacion: sesion por cookie tras un `POST` de inicio de sesion bajo el grupo de autenticacion. Roles verificados en backend: `user` y `admin`.

## Matriz de permisos

| Accion | Usuario (`user`) | Administrador (`admin`) |
| ------ | ----------------: | ----------------------: |
| Iniciar / cerrar sesion | Si | Si |
| Consultar movimientos y resumenes | Si | Si |
| Crear, editar o eliminar movimientos | Si | Si |
| Recategorizar movimientos y aplicar reglas a movimientos | Si | Si |
| Consultar dashboard y tendencias | Si | Si |
| Consultar subcategorias | Si | Si |
| Crear / editar / desactivar subcategorias | No | Si |
| Gestionar el catalogo de reglas de clasificacion (CRUD) | No | Si |
| Gestionar cuentas bancarias (Norma 43) | No | Si |
| Importar CSV tarjeta / XLS-XLSX / Norma 43 | No | Si |
| Exportar CSV o Norma 43 | No | Si |
| Gestionar lotes de importacion (listar / eliminar) | No | Si |
| Gestionar usuarios | No | Si |
| Descargar copia de seguridad o restaurar una copia | No | Si |

La proteccion no depende de ocultar botones en el frontend: el middleware comprueba sesion y rol antes de ejecutar la operacion.

## Flujo de una peticion protegida

1. El frontend envia una solicitud autenticada (cookie de sesion).
2. El middleware valida la sesion.
3. El backend determina usuario y rol.
4. La ruta o servicio comprueba el permiso (`requireAuth` / `requireRole`).
5. Se validan y normalizan los parametros.
6. Se ejecuta una consulta parametrizada.
7. Se devuelve la informacion autorizada para esa operacion.
8. Las operaciones sensibles (backup, restore, import) usan limites de tasa en las subidas y en el login.

## Capacidades por grupo funcional

Las tablas siguientes describen **operaciones**, no URLs de produccion. Los metodos HTTP y el nivel de auth se indican solo cuando estan verificados en la release de referencia.

### Salud

| Operacion | Metodo | Auth | Notas |
| --------- | ------ | ---- | ----- |
| Comprobar disponibilidad del servicio | GET | Publica | Respuesta de estado sin autenticacion |

### Autenticacion

| Operacion | Metodo | Auth | Notas |
| --------- | ------ | ---- | ----- |
| Iniciar sesion | POST | Publica + rate limit | Credenciales obligatorias; usuarios inactivos rechazados |
| Cerrar sesion | POST | Autenticado | Invalida la sesion actual |
| Consultar el usuario de la sesion (si existe) | GET | Sesion opcional | Sin cookie valida no exige 401; no sustituye a las rutas protegidas |

Codigos habituales en este grupo (verificados): `200`, `400` (entrada invalida), `401`, `429`, y `500` ante fallo interno de sesion.

### Movimientos

| Operacion | Metodo | Auth | Notas |
| --------- | ------ | ---- | ----- |
| Listar movimientos filtrados, ordenados y paginados | GET | Autenticado | Filtros tipicos: fechas, canal/origen, subcategoria, tipo, texto libre, pagina y ordenacion |
| Crear movimiento | POST | Autenticado | Validacion de tipo, importe positivo y demas reglas de negocio |
| Actualizar movimiento | PUT | Autenticado | |
| Eliminar movimiento | DELETE | Autenticado | |
| Obtener resumen de ingresos y gastos | GET | Autenticado | Totales coherentes con los filtros aplicables |
| Recategorizar movimientos en bloque | POST | Autenticado | Distinto del CRUD del catalogo de reglas |
| Aplicar reglas de clasificacion a movimientos | POST | Autenticado | Usuarios autenticados pueden **aplicar**; no gestiona el catalogo |

### Dashboard

| Operacion | Metodo | Auth | Notas |
| --------- | ------ | ---- | ----- |
| Agregaciones del periodo | GET | Autenticado | |
| Series temporales mensuales | GET | Autenticado | |

### Subcategorias

| Operacion | Metodo | Auth | Notas |
| --------- | ------ | ---- | ----- |
| Consultar el catalogo | GET | Autenticado | |
| Crear, editar o desactivar entradas del catalogo | POST / PUT / DELETE | Admin | |

### Reglas de clasificacion

| Operacion | Metodo | Auth | Notas |
| --------- | ------ | ---- | ----- |
| Gestionar el catalogo de patrones (listar, crear, editar, eliminar) | GET / POST / PUT / DELETE | Admin | Solo administracion del catalogo |
| Aplicar esas reglas a movimientos | POST | Autenticado | Ver grupo Movimientos |

### Cuentas bancarias

| Operacion | Metodo | Auth | Notas |
| --------- | ------ | ---- | ----- |
| Consultar, crear, editar, eliminar y marcar cuenta por defecto | GET / POST / PUT / DELETE (+ operacion de default) | Admin | Metadatos para exportacion Norma 43 |

### Importacion y exportacion

| Operacion | Metodo | Auth | Notas |
| --------- | ------ | ---- | ----- |
| Importar CSV de tarjeta | POST | Admin + rate limit en subida | Persistencia fila a fila (sin transaccion global del lote) |
| Importar extracto XLS/XLSX bancario | POST | Admin + rate limit en subida | Deduplicacion bancaria aplicable |
| Importar Norma 43 | POST | Admin + rate limit en subida | |
| Exportar movimientos a CSV | GET | Admin | Respeta filtros; sin paginacion |
| Exportar Norma 43 | GET | Admin | |
| Listar lotes de importacion | GET | Admin | Resumen por identificador de lote |
| Eliminar un lote de importacion | DELETE | Admin | Capacidad administrativa de limpieza |
| Descargar registro (log) de una importacion | GET | Admin | Diagnostico operativo; sin publicar nombres de fichero reales |

Efectos secundarios tipicos: creacion de movimientos, posible alta automatica de subcategorias, contadores de importados / omitidos / errores.

### Administracion de usuarios y copias

| Operacion | Metodo | Auth | Notas |
| --------- | ------ | ---- | ----- |
| Gestionar usuarios | GET / POST / PUT / DELETE | Admin | |
| Descargar copia de la base SQLite | GET | Admin | Genera/sirve la descarga; no implica un endpoint generico de «crear backup» separado |
| Subir una copia para restaurar | POST | Admin + rate limit en subida | Confirmacion en interfaz; limite de tamano de fichero |
| Ejecutar la restauracion | GET | Admin | Progreso comunicado por SSE |

## Codigos de respuesta habituales

| Codigo | Significado (verificado a nivel de uso) |
| ------ | --------------------------------------- |
| 200 / 201 | Operacion correcta |
| 400 | Validacion de entrada |
| 401 | Sesion ausente o invalida |
| 403 | Rol insuficiente |
| 404 | Recurso inexistente |
| 409 | Conflicto (por ejemplo, unicidad) |
| 429 | Rate limit (login y subidas) |
| 500 | Error no controlado. La respuesta puede incluir informacion procedente del error; **revisar y endurecer** lo que se expone al cliente es una mejora pendiente. Este documento no reproduce campos ni mensajes internos. |

## Pruebas relevantes

* Login / logout y restriccion por rol (`403` sin permiso).
* Filtros y paginacion coherentes entre listado y `COUNT`.
* Importacion CSV con importes y fechas limite (Vitest).
* Exportacion sin `LIMIT`/`OFFSET`.
* Backup y restore en entorno controlado (comprobacion manual de despliegue; no afirma el estado del servidor de produccion).

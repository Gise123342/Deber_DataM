# Deber01
Deber 01 Data mining
Giselle Cevallos 00325549


**Descripción y diagrama de arquitectura.** 
Este proyecto implementa un pipeline de backfill histórico que extrae información de QuickBooks Online (QBO) para las entidades Invoices, Customers e Items, y la deposita en Postgres dentro de un esquema raw.
•	Orquestación: Mage
•	Depliage: Docker Compose
•	Seguridad: Mage Secrets para gestionar credenciales/tokens
•	Alcance: solo backfill histórico (no incluye pipelines diarios, transformaciones a clean ni modelado dimensional).

**Pasos para levantar contenedores y configurar el proyecto.**

**Gestión de secretos (nombres, propósito, rotación, responsables; sin valores).**

QB_CLIENT_ID:
Proposito: Identificador de la aplicación registrada en QBO
Rotación: 90 dias

QB_CLIENT_SECRET
Proposito: Credencial privada asociada al Client ID
Rotación: 90 dias

QB_REALM_ID
Proposito: Identificador único de la compañía en QBO
Rotación: Cambio por ambiente

QB_REFRESH_TOKEN
Proposito: Token de larga duración para renovar accesos temporales
Rotación: 30 días



**Detalle de los tres pipelines qb__backfill: parámetros, segmentación, límites, reintentos**

Parametros y Segmentación:
-	fecha_inicio: marca de inicio del backfill histórico
-	fecha_fin: marca de fin del backfill histórico 
-	chunk_days (opcional): cantidad de días por segmento

límites, reintentos:
-	Manejo de errores implementado con máx. 5 reintentos.
-	Backoff exponencial (2^i segundos) entre cada intento.
-	Errores manejados: 429 (rate limit), 500, 502, 503, 504.
-	 En caso de error no bloqueante, se loguea y se continúa con el siguiente chunk.

**Trigger one-time: fecha/hora en UTC y equivalencia a Guayaquil; política de deshabilitación post-ejecución.**

El formato que se uso de fecha y hora. Dependiendo de la tuberia se implemento una diferente fehca y una diferente hora que cumpla con los datos obtenidos de la API. Esta fecha-hora debia estar dentro de el rango de valores de los datos ya que si no obtenia ningun datos, no podia seguir con la siguiente fase de el proceso que es la transformacion

Fecha/hora en UTC: 2025-09-12 15:00:00 
Equivalencia Guayaquil: 2025-09-12 10:00:00

Política: una vez ejecutado y validado, deshabilitar trigger para evitar relanzamientos accidentales.
- usamos el once en frequency al momento de crear el trigger para que se ejcute una unica vez
- pasamos en el triguer los 3 argumentos (fecha de inicio, fecha de fin y chuck days)

**Esquema raw: tablas por entidad, claves, metadatos obligatorios, idempotencia.**

Se utilizo un formato especifico para las tablas raw
Este paso se realizo en la transformacion de los datos 

raw_qb_invoices (gual diseño para raw_qb_customers y raw_qb_items)

      id (PK, id de factura)

      payload (JSONB con data completa)

      ingested_at_utc (timestamp)

      extract_window_start_utc, extract_window_end_utc

      page_number, page_size

      request_payload


**Validaciones/volumetría: cómo correrlas y cómo interpretar resultados.** 

Cómo correr:
- Revisar logs _processing_log generados en loader.
- Validar DataFrame en transform (print shape)
- Revisar resultados en export (rows, columns)
- 
Cómo interpretar:

- filas_procesadas = 0 (no hubo cambios en ese rango)
- status = error en log (revisar detalle de excepción)
- Conteo en warehouse vs. API debe coincidir en el rango consultado

**Troubleshooting: auth, paginación, límites, timezones, almacenamiento y permisos.**

Auth:
Error 401: renovar refresh_token.

Paginación:
QuickBooks no usa paginación tradicional: se maneja por chunks de fechas.

Límites:
Error 429 → esperar y reintentar (ya manejado en código).

Timezones:

QuickBooks usa UTC; convertir a local (Guayaquil = UTC-5) solo en reportes.

Almacenamiento:

Verificar espacio en Postgres si el payload crece.

Permisos:

Revisar roles en warehouse (root@warehouse).

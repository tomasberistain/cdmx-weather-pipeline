# Diseño v2: pipeline diario de clima para la CDMX en AWS

## Contexto

La v1 de este repositorio fue un ejercicio técnico: un script que descarga 7 días de
temperatura y precipitación de un solo punto de la CDMX, los limpia con pandas, los
guarda en CSV y SQLite y corre cuatro consultas. Cumple el ejercicio, pero no es un
pipeline: se corre a mano, sobrescribe todo en cada ejecución y solo vive en mi máquina.

La v2 convierte ese script en un pipeline que corre solo, todos los días, acumula
histórico y se puede consultar con SQL desde la nube.

## Problema

Quiero un histórico horario de clima de varios puntos de la CDMX que:

1. **Se actualice solo todos los días**, sin depender de que mi computadora esté prendida.
2. **Crezca sin duplicarse**, aunque una carga falle y se repita.
3. **Detecte datos malos antes de publicarlos**, no después.
4. **Se pueda consultar con SQL** por fecha y por punto, por mí o por cualquiera con acceso.
5. **Se pueda reconstruir** (infraestructura y datos) desde el repositorio.

Sobre el volumen: 5 puntos × 24 horas × 365 días ≈ 44 mil filas por año, unas 300 mil
con un backfill desde 2020. **El volumen no justifica la nube.** Lo que la justifica son
los puntos 1 y 4: la ejecución desatendida y un almacenamiento durable y compartido.
DuckDB local resolvería el análisis, pero no la operación.

## Fuente

[Open-Meteo](https://open-meteo.com/), sin autenticación:

- **Carga diaria:** API de pronóstico (`/v1/forecast`) con `past_days`, para los
  últimos días ya observados.
- **Backfill:** API histórica (`archive-api.open-meteo.com/v1/archive`), para meses o
  años pasados.

Variables: `temperature_2m`, `precipitation` (las de la v1), más `relative_humidity_2m`
y `wind_speed_10m`.

**Puntos** (elegidos por altitud y régimen de lluvia distintos dentro de la ciudad):

| id | punto | lat | lon |
|----|-------|-----|-----|
| centro | Centro Histórico | 19.43 | -99.13 |
| azcapotzalco | Azcapotzalco | 19.49 | -99.19 |
| coyoacan | Coyoacán | 19.35 | -99.16 |
| iztapalapa | Iztapalapa | 19.36 | -99.06 |
| tlalpan | Tlalpan (sur, mayor altitud) | 19.20 | -99.20 |

Los puntos viven en `config/puntos.yaml`, no en el código.

## Arquitectura

```
GitHub Actions (cron diario)
   │
   ├─ extraer     → Open-Meteo (por punto)
   ├─ guardar raw → s3://<bucket>/raw/fecha_carga=AAAA-MM-DD/<punto>.json
   ├─ transformar → pandas: tipos, renombre, una fila por (punto, hora)
   ├─ validar     → si falla, el job termina en error y NO se publica nada
   └─ publicar    → s3://<bucket>/curated/clima_horario/fecha=AAAA-MM-DD/data.parquet
                          │
                          └─ Amazon Athena: tabla externa + consultas de sql/
```

**Capas:**
- **raw:** la respuesta de la API tal cual llegó. Permite reprocesar sin volver a llamar
  a la API y auditar qué se recibió.
- **curated:** Parquet particionado por `fecha` (día de la observación). Es lo que consulta Athena.

## Modelo de datos (curated)

| columna | tipo | nota |
|---------|------|------|
| punto_id | string | clave del punto en `config/puntos.yaml` |
| ts_local | timestamp | hora local `America/Mexico_City` |
| temperatura_c | double | |
| precipitacion_mm | double | |
| humedad_relativa_pct | double | |
| viento_kmh | double | |
| fuente | string | `forecast` o `archive` |
| cargado_en | timestamp | UTC, momento de la carga |
| fecha | date | **partición**, derivada de `ts_local` |

Clave lógica: `(punto_id, ts_local)`.

Cambio respecto a la v1: ya **no** se filtra el horario de 06:00 a 22:00 al cargar. Ese
filtro era un requisito del ejercicio, y aplicarlo en la carga pierde datos. Ahora se
guardan las 24 horas y el filtro, si se necesita, se aplica en las consultas.

## Idempotencia y ventana de carga

- Cada ejecución procesa una **ventana de los últimos 3 días**, no solo "ayer". Así se
  absorben las correcciones tardías de la fuente y los días en que el job no corrió.
- La publicación **reemplaza la partición completa** de cada `fecha` de la ventana, sin
  agregar filas. Correr el job dos veces da el mismo resultado que correrlo una vez.
- El backfill usa exactamente el mismo código con otro rango de fechas
  (`python -m clima.pipeline --desde 2020-01-01 --hasta 2020-12-31 --fuente archive`).

## Validaciones (antes de publicar)

| regla | acción si falla |
|-------|-----------------|
| la respuesta trae todas las variables esperadas | error |
| 24 filas por punto y por día completo | error |
| sin duplicados en `(punto_id, ts_local)` | error |
| temperatura entre -10 y 45 °C | error |
| precipitación ≥ 0 y ≤ 150 mm/h | error |
| humedad entre 0 y 100 % | error |
| nulos por variable ≤ 5 % del día | advertencia (se registra en el log) |

Si cualquier regla con acción "error" falla, no se escribe nada en curated. El job queda
en rojo en GitHub Actions y GitHub manda un aviso por correo.

## Consumidores

- **Consultas SQL en Athena:** las cuatro consultas de la v1, migradas a sintaxis de
  Athena y extendidas por punto (por ejemplo, la precipitación acumulada por punto y por mes).
- **(Opcional)** una gráfica estática generada por el mismo job y publicada en GitHub Pages.

## Seguridad

- GitHub Actions se autentica en AWS con **OIDC**: asume un rol de IAM y obtiene
  credenciales temporales. No hay llaves de acceso guardadas en el repositorio ni en los secrets.
- El rol tiene **permisos mínimos**: lectura y escritura solo en el bucket del proyecto,
  más ejecución de consultas en un workgroup de Athena.
- La cuenta raíz tiene MFA y no se usa para operar.
- Hay una alerta de presupuesto en AWS Budgets.

## Infraestructura

Con Terraform (`infra/`): el bucket con bloqueo de acceso público, la base y la tabla de
Glue (con *partition projection* para no tener que registrar particiones a mano), el
workgroup de Athena, el proveedor OIDC de GitHub y el rol de IAM. Son unas 100 líneas,
no más. Si se borra todo, se reconstruye con `terraform apply`.

## Costos esperados

Unos cuantos MB al año en S3 y consultas de Athena que escanean KB o pocos MB por
consulta (Athena cobra por datos escaneados, y por eso los datos van en Parquet
particionado). El costo esperado es de centavos de dólar al mes.

## Lo que no uso, y por qué

| herramienta | por qué no |
|-------------|------------|
| Airflow / Prefect | Es un solo job diario y lineal, sin dependencias entre tareas. GitHub Actions con cron lo orquesta sin levantar ni mantener un servidor. Airflow tendría sentido con varios DAGs o dependencias entre fuentes. |
| Kafka / streaming | La fuente se actualiza por hora y el uso es analítico diario. Streaming agregaría complejidad sin ningún beneficio. |
| Snowflake / Redshift | Con este volumen, un warehouse dedicado es desproporcionado. Athena sobre Parquet cubre el SQL sin infraestructura que administrar. |
| Lambda | Sería otra forma de agendar el job. Elegí GitHub Actions porque deja los logs y el historial de ejecuciones públicos junto al código. |

## Plan de implementación

1. **Refactor local:** convertir el script en paquete (`src/clima/`), usar `pathlib`,
   leer la configuración de `config/puntos.yaml`, cargar varios puntos y las 24 horas,
   agregar las validaciones y tests con pytest. Todavía sin nube: escribe Parquet en una
   carpeta local con la misma estructura de particiones.
2. **Infraestructura:** cuenta de AWS, Terraform para el bucket, Glue, Athena, OIDC y el rol.
3. **Automatización:** workflow de CI (tests en cada push) y workflow diario (cron) que
   escribe en S3.
4. **Backfill y consultas:** cargar el histórico y migrar `sql/consultas.sql` a Athena.
5. **Documentación:** README con diagrama, cómo correrlo local y en AWS, y métricas
   reales (días cargados, ejecuciones exitosas, validaciones que han fallado).

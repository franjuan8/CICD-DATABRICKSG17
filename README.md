# Plataforma de análisis de aerolíneas con Azure Databricks

Proyecto académico de ingeniería de datos que procesa nueve archivos CSV mediante una arquitectura medallion y publica indicadores mensuales para su análisis en Power BI. Se utilizan ambientes de desarrollo y producción, Unity Catalog para gobernar los datos y GitHub Actions para validar y desplegar los notebooks.

Este documento recoge la configuración trabajada durante el proyecto. Los nombres de recursos y notebooks corresponden a la configuración acordada; no sustituye una auditoría del despliegue. Los nombres del Access Connector, cluster y SQL Warehouse deben verificarse en cada ambiente. La configuración final utiliza el esquema **golden**, donde ya se visualizaron las dos tablas KPI de producción.

## 1. Objetivo y alcance

La plataforma integra información de aerolíneas, aeropuertos, aeronaves, rutas, vuelos, pasajeros, combustible, ventas y equipajes. Permite analizar demanda, ocupación, puntualidad, cancelaciones, ingresos y consumo de combustible por aerolínea y ruta.

Los datos son sintéticos y se emplean con fines académicos. El procesamiento es batch y las cargas trabajadas utilizan sobrescritura completa; no se ha implementado una ingesta incremental ni una dimensión con historial SCD tipo 2.

Los productos finales son:

- `aerolineas_prod.golden.kpi_aerolinea_mes`.
- `aerolineas_prod.golden.kpi_ruta_mes`.
- Un reporte de Power BI organizado en desempeño de aerolíneas y desempeño de rutas.

## 2. Arquitectura
![](/Workspace/Users/yuliza_jfk@hotmail.com/CICD-DATABRICKSG17/Evidencias/ARQUITECTURA.png)

El código se ejecuta en Azure Databricks y los archivos permanecen en Azure Data Lake Storage Gen2. Unity Catalog registra catálogos, esquemas, tablas, credenciales y ubicaciones externas. GitHub Actions realiza el despliegue y solicita la ejecución de un Job; el Job determina las dependencias del ETL. Power BI consulta las tablas agregadas mediante el SQL Warehouse.

## 3. Servicios utilizados y motivo de elección

| Servicio o componente | Función en el proyecto | Motivo de uso |
| --- | --- | --- |
| Azure Resource Groups | Agrupan los recursos de cada ambiente | Facilitan administración, permisos y seguimiento de costos |
| Azure Storage / ADLS Gen2 | Conserva CSV y archivos Delta | Almacenamiento persistente con jerarquía de directorios para el lakehouse |
| Azure Databricks | Ejecuta notebooks Python, PySpark y SQL | Integra procesamiento distribuido, tablas Delta y orquestación |
| Delta Lake | Formato de las tablas Bronze, Silver y Golden | Ofrece transacciones y control del esquema sobre archivos del lago |
| Unity Catalog | Registra y gobierna los objetos de datos | Centraliza permisos y separa los datos de desarrollo y producción |
| Access Connector for Azure Databricks | Vincula una identidad administrada al almacenamiento | Permite acceso desde Unity Catalog sin usar una contraseña o una clave de Storage |
| Microsoft Entra ID y Azure RBAC | Gestionan identidades y acceso a recursos | Autorizan a la Managed Identity y a los usuarios correspondientes |
| Azure Key Vault | Conserva secretos que requieren los notebooks | Separa información sensible del código |
| Databricks Secret Scope | Permite recuperar secretos desde Databricks | Proporciona una interfaz de acceso a los secretos configurados |
| GitHub | Aloja y versiona el proyecto | Permite trabajar con ramas y revisar cambios mediante pull requests |
| GitHub Actions | Valida y despliega notebooks | Automatiza la promoción y registra el resultado de las ejecuciones |
| Databricks Jobs | Orquesta el ETL y sus dependencias | Impide ejecutar una etapa dependiente si falla la anterior |
| Databricks SQL Warehouse | Sirve consultas para analítica | Proporciona un endpoint SQL para Power BI |
| Power BI | Construye visualizaciones de los KPI | Presenta los resultados de negocio con filtros y gráficos |

## 4. Separación de ambientes

| Componente | Desarrollo | Producción |
| --- | --- | --- |
| Rama de GitHub | `develop` | `main` |
| GitHub Environment | `dev` | `prod` |
| Resource Group | `rg-aerolineas-dev` | `rg-aerolineas-prod` |
| Workspace Databricks | `dbw-aerolineas-dev` | `dbw-aerolineas-prod` |
| Storage Account | `staerolineasdeveus01` | `staerolineasprodeus01` |
| Catálogo | `aerolineas_dev` | `aerolineas_prod` |
| Storage Credential | `cred_aerolineas_dev` | `cred_aerolineas_prod` |
| Key Vault | `kv-aerolineas-dev` | `kv-aerolineas-prod` |
| Secret Scope | `scope-aerolineas-dev` | `scope-aerolineas-prod` |

Cada ambiente utiliza su propio almacenamiento, catálogo y credencial. Esto permite probar cambios sin sobrescribir los datos productivos. Se debe conservar el Resource ID del Access Connector de cada ambiente y comprobar que cada credencial referencia el conector correcto.

La jerarquía de objetos es `catalogo.esquema.tabla`. Por ejemplo, `aerolineas_prod.bronze.bbdd_dim_aerolineas`.

## 5. Almacenamiento y capas

| Contenedor físico | Esquema de Unity Catalog | Contenido |
| --- | --- | --- |
| `raw` | `raw`, reservado para objetos de esta capa | Nueve CSV originales; la lectura actual se realiza por ruta |
| `bronze` | `bronze` | Tablas Delta de ingesta, prefijo `bbdd_` |
| `silver` | `silver` | Tablas Delta depuradas, prefijo `tbl_` |
| `golden` | `golden` | Tablas Delta de KPI mensuales |
| `catalog-ae` | Ubicación administrada del catálogo | Almacenamiento administrado definido para el catálogo |

La carpeta física de una tabla sigue su nombre. Ejemplos:

```text
abfss://raw@<storage>.dfs.core.windows.net/dim_aerolineas.csv
abfss://bronze@<storage>.dfs.core.windows.net/bbdd_dim_aerolineas
abfss://silver@<storage>.dfs.core.windows.net/tbl_dim_aerolineas
abfss://golden@<storage>.dfs.core.windows.net/kpi_aerolinea_mes
```

Las tablas creadas con `LOCATION` son externas. La ubicación administrada del catálogo no concede automáticamente acceso a los otros contenedores. Se registra una External Location para cada raíz utilizada: `raw`, `bronze`, `silver`, `golden` y `catalog-ae`, asociada a la credencial del ambiente.

No se deben solapar ubicaciones externas ya registradas. `IF NOT EXISTS` evita recrear un objeto, pero no modifica una URL o credencial incorrecta.

## 6. Seguridad y gestión de secretos

### Acceso al Data Lake

El Access Connector utiliza una **Managed Identity asignada por el sistema**. A esa identidad se le asigna `Storage Blob Data Contributor` sobre el almacenamiento requerido, idealmente con el alcance necesario. La Storage Credential de Unity Catalog utiliza el Resource ID del conector y las External Locations referencian esa credencial.

Hay dos autorizaciones diferentes: Azure RBAC autoriza a la identidad sobre Storage, mientras que Unity Catalog autoriza al usuario o principal que consulta o crea objetos. Disponer de una de ellas no reemplaza la otra.

Para crear tablas externas, el ejecutor necesita `CREATE EXTERNAL TABLE` sobre la ubicación, además de `USE CATALOG`, `USE SCHEMA` y `CREATE TABLE` en el destino. La identidad del Job también necesita acceso al cluster y a los notebooks. El consumidor de Power BI necesita acceso al workspace, `CAN USE` sobre el warehouse y permisos de lectura sobre las tablas.

### Key Vault, Secret Scope y GitHub Secrets

Key Vault conserva los secretos de Azure que necesite el proyecto. El Secret Scope configurado en Databricks proporciona su acceso desde notebooks, por ejemplo:

```python
valor = dbutils.secrets.get(scope=secret_scope, key="<nombre-del-secreto>")
```

La existencia del scope no significa que todos los notebooks lo consuman. Las lecturas del lago mediante Unity Catalog y Managed Identity no necesitan una clave de Storage en Key Vault. No se deben agregar configuraciones `fs.azure.account.key` a este flujo.

Los tokens utilizados por GitHub Actions se guardan en **GitHub Secrets**. Las variables como catálogo o Storage Account se guardan en **GitHub Variables**. Configurar `SECRET_SCOPE` en GitHub solo pasa el nombre del scope al notebook; no crea el scope ni copia los secretos.

## 7. Fuentes y tablas

| Archivo Raw | Tabla Bronze | Tabla Silver | Contenido |
| --- | --- | --- | --- |
| `dim_aerolineas.csv` | `bbdd_dim_aerolineas` | `tbl_dim_aerolineas` | Aerolínea, sede, hub y modelo de negocio |
| `dim_aeropuertos.csv` | `bbdd_dim_aeropuertos` | `tbl_dim_aeropuertos` | Aeropuerto, ubicación y zona horaria |
| `dim_aeronaves.csv` | `bbdd_dim_aeronaves` | `tbl_dim_aeronaves` | Aeronave, aerolínea, modelo y capacidad |
| `dim_rutas.csv` | `bbdd_dim_rutas` | `tbl_dim_rutas` | Origen, destino, distancia y duración |
| `fact_vuelos.csv` | `bbdd_fact_vuelos` | `tbl_fact_vuelos` | Identificadores, horarios, estados y retrasos |
| `fact_pasajeros.csv` | `bbdd_fact_pasajeros` | `tbl_fact_pasajeros` | Pasajeros, capacidad, clases y no-show |
| `fact_combustible.csv` | `bbdd_fact_combustible` | `tbl_fact_combustible` | Cantidades de combustible y costo |
| `fact_ventas_boletos.csv` | `bbdd_fact_ventas_boletos` | `tbl_fact_ventas_boletos` | Ventas, clases, canales e ingresos |
| `fact_equipajes.csv` | `bbdd_fact_equipajes` | `tbl_fact_equipajes` | Piezas, peso e incidencias |

Las dimensiones aportan contexto a las operaciones. Los hechos se relacionan principalmente mediante `vuelo_id`, y los vuelos enlazan aerolínea, aeronave y ruta mediante sus identificadores. Las ventas pueden contener varios registros por vuelo, por lo que deben agregarse antes de unirlas con hechos al nivel de vuelo.

## 8. Flujo ETL

### Raw: recepción de fuentes

Se conservan los CSV originales, sin transformaciones. Los archivos deben existir en el Storage Account del ambiente que ejecuta el ETL. El workflow actual copia notebooks, pero no copia los CSV desde desarrollo a producción.

Si los archivos están en una subcarpeta, `path_inicio` debe incluirla. Se conserva el nombre y la capitalización exacta de cada archivo.

### Bronze: ingesta y trazabilidad

Se leen los CSV con un esquema explícito mediante `StructType` y `StructField`, utilizando tipos como `STRING`, `INT`, `DATE`, `TIMESTAMP` y `DECIMAL`. Se agregan:

- `fecha_ingesta`: momento de la ingesta.
- `archivo_origen`: ruta del CSV utilizado.

Las escrituras utilizan `.write.mode("overwrite").insertInto(...)` sobre tablas previamente creadas. `insertInto` requiere respetar el orden y tipos de las columnas del destino. La declaración `nullable=False` en el esquema de lectura no sustituye las validaciones de calidad de datos.

### Silver: limpieza y calidad

Se leen las tablas Bronze y se estandarizan textos, identificadores, fechas y tipos numéricos. Las transformaciones trabajadas incluyen validación de campos obligatorios, dominios de estado, cantidades y referencias a dimensiones.

En las dimensiones se eliminan duplicados exactos de los datos de negocio y se detectan identificadores con valores contradictorios. No se elige arbitrariamente una versión cuando existen conflictos. Los registros inválidos se muestran y detienen la carga mediante un error; no se ha implementado una tabla persistente de cuarentena.

Se conservan `fecha_ingesta` y `archivo_origen`, y se agrega `fecha_actualizacion`. Las dimensiones y los hechos se procesan en notebooks separados: el notebook de hechos consulta las dimensiones Silver desde tablas registradas, por lo que no depende de variables Python de otro notebook.

### Golden: indicadores de negocio

Se integran los hechos Silver con las dimensiones necesarias, verificando claves y cardinalidad. Las ventas se agregan al nivel de vuelo antes de los joins para evitar multiplicar pasajeros o combustible.

El periodo se deriva del mes de la salida programada en UTC. Se distinguen vuelos realizados o demorados de los cancelados. La regla de puntualidad trabajada considera un retraso de salida de hasta 15 minutos para un vuelo operado.

| Tabla | Granularidad | Análisis |
| --- | --- | --- |
| `kpi_aerolinea_mes` | Una fila por aerolínea y mes | Demanda, operación e indicadores por aerolínea |
| `kpi_ruta_mes` | Una fila por ruta y mes | Demanda, eficiencia e indicadores por ruta |

Las métricas incluyen vuelos, pasajeros, ocupación, puntualidad, cancelaciones, ingresos, combustible y equipajes según el esquema final del ETL. Para comparar periodos o categorías, los porcentajes se recalculan a partir de numeradores y denominadores agregados, evitando promediar porcentajes sin ponderación.

Las dos escrituras Golden son independientes; no constituyen una transacción conjunta. El saldo de ingresos menos combustible no equivale a utilidad, porque faltan otros costos operativos.

## 9. Organización de notebooks y repositorio

| Ruta de notebook | Responsabilidad |
| --- | --- |
| `Proceso/0.Preparar_Ambiente` | Prepara catálogos, ubicaciones, esquemas y tablas |
| `Proceso/1.Ingest_data_dim` | Ingesta de las cuatro dimensiones |
| `Proceso/1.Ingest_data_fact` | Ingesta de los cinco hechos |
| `Proceso/2.Transform_dim` | Transformación de dimensiones a Silver |
| `Proceso/2.Transform_fact` | Transformación de hechos a Silver |
| `Proceso/3.Transform_golden` | Construcción de los KPI mensuales |
| `Seguridad/4.Grants` | Asignación de permisos |
| `Reversion/reverso` | Operaciones de limpieza explícitas fuera del ETL normal |

Los nombres y mayúsculas de estas rutas deben coincidir con el workspace. El workflow se guarda en `.github/workflows/` y este documento se coloca como `README.md` en la raíz del repositorio. Las extensiones de los notebooks en Git dependen del formato exportado.

Origen configurado:

```text
/Workspace/Users/yuliza_jfk@hotmail.com/CICD-DATABRICKSG17
```

Destino acordado para producción:

```text
/Shared/CICD-DATABRICKSG17
```

## 10. CI/CD con GitHub Actions

### Ramas y ejecución

| Evento | Ambiente | Comportamiento del workflow trabajado |
| --- | --- | --- |
| Push a `develop` | `dev` | Exporta notebooks del origen y valida sintaxis Python; no ejecuta el ETL |
| Push a `main` | `prod` | Exporta, valida, importa en producción, crea o actualiza el Job y lo ejecuta |
| `workflow_dispatch` | Según la rama seleccionada | Aplica la lógica de `develop` o `main` |

La promoción se realiza mediante un pull request de `develop` a `main`. El YAML debe mapear explícitamente `develop` a `dev` y `main` a `prod`; el nombre de una rama no tiene que ser igual al nombre del GitHub Environment.

El Job `WF_AEROLINEAS_PROD` ejecuta las tareas en el orden definido:

1. Preparación del ambiente.
2. Ingesta dimensional.
3. Ingesta de hechos.
4. Transformación dimensional a Silver.
5. Transformación de hechos a Silver.
6. Transformación a Golden.
7. Asignación de permisos.

La tarea de reversión se puede exportar e importar, pero queda fuera de las tareas ejecutadas automáticamente. El workflow limita ejecuciones concurrentes, utiliza un cluster existente por nombre y consulta el estado final del Job para determinar el resultado de GitHub Actions.

### Secrets y variables

Secrets utilizados en el repositorio o en los environments correspondientes:

| Secret | Contenido |
| --- | --- |
| `DATABRICKS_ORIGIN_HOST` | URL HTTPS del workspace de desarrollo |
| `DATABRICKS_ORIGIN_TOKEN` | Token autorizado para exportar notebooks del origen |
| `DATABRICKS_DEST_HOST` | URL HTTPS del workspace de producción |
| `DATABRICKS_DEST_TOKEN` | Token autorizado para desplegar y ejecutar en producción |

Variables del environment `prod`:

| Variable | Valor o finalidad |
| --- | --- |
| `DEST_NOTEBOOK_BASE` | `/Shared/CICD-DATABRICKSG17` |
| `CLUSTER_NAME` | Nombre exacto del cluster de producción |
| `STORAGE_NAME` | `staerolineasprodeus01` |
| `CATALOG_NAME` | `aerolineas_prod` |
| `STORAGE_CREDENTIAL` | `cred_aerolineas_prod` |
| `SECRET_SCOPE` | `scope-aerolineas-prod` |
| `RUN_AS_SERVICE_PRINCIPAL` | Identificador de aplicación, si se configura ejecución con ese principal |

Si el YAML utiliza `NOTEBOOK_PLAN_JSON`, la variable debe contener las rutas, dependencias y parámetros del plan. Si las tareas están definidas directamente en el script, no se necesita esa variable.

### Parametrización de notebooks

GitHub no transmite automáticamente sus variables a una ejecución manual de Databricks. El workflow debe construir `base_parameters` y el notebook debe leer los valores mediante widgets.

| Parámetro del notebook | Fuente o valor en producción |
| --- | --- |
| `catalog` / `catalogo` | `CATALOG_NAME`; mantener el nombre consumido por cada notebook |
| `storageLocation` / `storageName` | `STORAGE_NAME` |
| `credential` | `STORAGE_CREDENTIAL` |
| `secret_scope` | `SECRET_SCOPE` |
| `ambiente` | `prod` |
| `path_inicio` | Ruta del contenedor Raw de producción |
| `schema` | `bronze` en ingestas |
| `schema_source` | `bronze` para Silver; `silver` para Golden |
| `schema_sink` | `silver` para Silver; **`golden` para Golden** |

Ejemplo de consumo:

```python
dbutils.widgets.text("catalog", "aerolineas_dev")
catalog = dbutils.widgets.get("catalog").strip()
```

Ejemplo SQL en el formato trabajado:

```sql
CREATE SCHEMA IF NOT EXISTS `${catalog}`.golden;
```

No se deben eliminar los widgets recibidos con `removeAll()` ni sobrescribir sus valores con constantes de desarrollo. La sintaxis SQL `${...}` es la empleada en los notebooks del proyecto; en runtimes recientes se debe revisar la migración a marcadores de parámetros e `IDENTIFIER`.

### Alcance real de la validación actual

La validación Python detecta errores de sintaxis. No ejecuta Spark, no verifica la existencia de los CSV, no valida todas las sentencias SQL del notebook y no garantiza permisos ni calidad de datos.

Además, el script exporta el estado actual de los notebooks del workspace de origen. El commit que dispara GitHub no garantiza por sí mismo que ese sea el código desplegado. Antes de promover, deben guardarse y sincronizarse los notebooks. Como mejora, se propone desplegar artefactos versionados desde Git y ejecutar pruebas de integración en desarrollo.

## 11. Preparación y operación

1. Verificar Storage Accounts con Hierarchical Namespace habilitado y contenedores creados.
2. Verificar Access Connectors, Managed Identities y roles de acceso al almacenamiento.
3. Registrar las Storage Credentials y External Locations por ambiente.
4. Crear o verificar catálogos, esquemas y tablas con las rutas correspondientes.
5. Verificar Key Vault y Secret Scope si los notebooks requieren secretos.
6. Colocar los nueve CSV en Raw del ambiente correspondiente.
7. Verificar cluster compatible con Unity Catalog y permisos del ejecutor.
8. Guardar notebooks y configurar GitHub Secrets, Variables y environments.
9. Ejecutar la validación en `develop` y promover cambios a `main`.
10. Revisar el resultado del Job y consultar ambas tablas Golden.
11. Actualizar el modelo de Power BI después de una carga satisfactoria.

La preparación debe ser reutilizable sin eliminar contenedores completos. El notebook de reversión requiere seleccionar explícitamente los objetos y rutas que se desean limpiar; no forma parte del flujo normal de despliegue.

## 12. Power BI y reportería

Power BI Desktop se conecta mediante el conector **Azure Databricks**, usando el Server hostname y HTTP path del SQL Warehouse de producción. La autenticación puede realizarse con Microsoft Entra ID y los permisos necesarios. Para este volumen académico se planteó comenzar en modo Importar.

La primera página utiliza `kpi_aerolinea_mes`: tarjetas de pasajeros, vuelos e ingresos; ranking de aerolíneas; evolución mensual; ocupación y puntualidad. La segunda utiliza `kpi_ruta_mes`: ranking de rutas, demanda mensual, eficiencia de combustible y tabla de detalle por origen y destino.

Ambas tablas representan los mismos hechos agrupados con distintas granularidades. No deben sumarse entre sí ni relacionarse directamente por mes como si una fuera dimensión de la otra. Para filtros comunes se pueden incorporar dimensiones compartidas, como un calendario, y diseñar las relaciones según la granularidad.
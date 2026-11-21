# Arquitectura base — OpinionClientes

Este documento describe las decisiones arquitectónicas ya tomadas y confirmadas para este proyecto. Es la referencia autoritativa sobre "cómo está pensado" el sistema. Si al revisar el código encuentras algo que parece una inconsistencia, corrígelo siguiendo estos principios — **no reinterpretes ni cambies el estilo arquitectónico de fondo**. Cualquier cambio que contradiga una sección marcada como "Decisión confirmada" debe proponerse, no aplicarse directamente.

## 1. Qué es este proyecto

Proceso ETL (Extract, Transform, Load) para un sistema de análisis de opiniones de clientes. Extrae datos de tres fuentes heterogéneas, los transforma (resolución de claves de dimensión, tokenización NLP), y los carga en un data warehouse (`OpinionesOLAP`).

Es un proyecto académico (actividad de curso), no un producto comercial. El alcance está deliberadamente acotado — no se debe expandir el alcance ni "mejorar" agregando infraestructura que el proyecto no necesita (ver sección 7).

## 2. Estructura de la solución

```
OpinionClientes.sln
│
├── OpinionClienteDwh.Data/      Class Library — TODA la logica de negocio
├── OpinionClienteDwh.Worker/    Worker Service — host, composition root
└── OpinionClienteDwh.Api/       Web API — simula una red social externa (mock)
```

**Decisión confirmada: un solo proyecto de librería (`Data`), no separado por fase de ETL.** Extract, Transform y Load conviven en el mismo proyecto `OpinionClienteDwh.Data`, organizados por carpetas de responsabilidad, no por etapa del pipeline ni por capa arquitectónica formal. No crear proyectos `.Extract`, `.Transform`, `.Load` separados.

`OpinionClienteDwh.Api` es la única excepción — es un proyecto aparte porque simula ser un sistema externo e independiente (una red social real). Su implementación interna no necesita ser consistente con `Data` (ver sección 6).

## 3. Organización interna de `OpinionClienteDwh.Data`

Carpetas por responsabilidad, planas, sin jerarquía de capas:

```
Interfaces/              Contratos (IExtractor<T>, IValidator<T>, IDao<T>, etc.)
  └── DaoInterfaces/      Contratos especificos de acceso a datos
Dtos/                     Records de transferencia de datos, inmutables
Extractors/               Implementaciones de IExtractor<T>
  └── Mappings/           ClassMap<T> de CsvHelper
Validators/               Implementaciones de IValidator<T>
Dao/                      Acceso a OpinionesOLTP (fuentes de Extract)
Services/                 Orquestador de extraccion, DataLoader, ExtractResult
Cache/                    DimensionKeyCache
Nlp/                      Tokenizador de comentarios (Catalyst)
Staging/                  Lectura de los JSON de staging
Load/                     Todo lo que escribe hacia OpinionesOLAP
  ├── Base/                Clase base de conexion especifica de OLAP
  ├── Daos/                Acceso a OpinionesOLAP
  └── Services/            Orquestacion de Transform/Load
Common/                   Utilidades genericas de ADO.NET (SqlServerConnection, TvpBuilder, EsquemaTablaStaging)
Models/                   Modelos internos que no son DTOs de fuente (OpinionConsolidada, OpinionCargada, etc.)
Persistence/              DbContext de solo lectura y sus entidades (EF Core)
```

**Decisión confirmada: NO usar nombres de capa formal (`Domain/`, `Application/`, `Infrastructure/`, `Onion`, `Clean Architecture`).** Se evaluó explícitamente y se descartó — ver sección 5. No renombrar carpetas a ese esquema.

## 4. Filosofía arquitectónica: servicios y contratos, no capas ni microservicios

**Decisión confirmada, evaluada explícitamente dos veces:**

1. Se descartó una arquitectura de microservicios distribuidos (servicios HTTP separados, balanceador de carga, réplicas) por ser sobre-ingeniería frente al alcance real del proyecto. **No introducir `Nginx`, `YARP`, balanceadores, ni desplegar los extractores como servicios HTTP separados.**
2. Se descartó nombrar la arquitectura como "Onion/Clean Architecture" con capas formales, porque el diagrama de arquitectura del proyecto no las representa así, y el requisito real de la actividad ("arquitectura orientada a servicios") se interpretó como: **cada responsabilidad vive detrás de una interfaz genérica (`IExtractor<T>`, `IValidator<T>`, `IDao<T>`) y se resuelve por inyección de dependencias nativa de .NET.** Esto es lo que hace que el sistema sea "orientado a servicios" en este proyecto — contrato + DI, no topología de despliegue ni capas nombradas.

Si encuentras código que no sigue este patrón (una clase que no implementa la interfaz genérica correspondiente, una dependencia resuelta con `new` en vez de inyectada), corrígelo hacia el patrón de contratos + DI. Si encuentras código ya organizado así, no lo reestructures en capas.

## 5. Extractores genéricos, no atados a un tipo de dato

**Decisión confirmada:** `CsvExtractor<T>`, `DatabaseExtractor<T>`, `ApiExtractor<T>` son genéricos por diseño — no conocen el tipo de dato que extraen, la fuente específica, ni la ruta/endpoint (reciben todo eso como parámetro en el registro de DI en `Program.cs`). Esto es intencional para poder agregar una fuente nueva sin escribir una clase nueva.

**No** revertir esto a extractores concretos por tipo (ej. `SurveyCsvExtractor`, `WebReviewDatabaseExtractor`). Si necesitas agregar una fuente nueva, el patrón correcto es un registro nuevo en `Program.cs`, no una clase nueva.

## 6. Reglas de acceso a datos (ADO.NET vs. LINQ/EF Core)

**Decisión confirmada, con excepciones explícitas — no unificar todo a un solo enfoque:**

- **Por defecto: ADO.NET + Stored Procedures**, vía la clase base `SqlServerConnection`. Así funcionan: `WebReviewDao`, todos los DAOs bajo `Load/Daos/` (`DaoFactOpinion`, `DaoFactPalabraClaveOpinion`, `DaoMergeDimensiones`, `DaoDimensionKey`).
- **Excepción 1 — `OpinionClienteDwh.Api`:** usa LINQ/EF Core (`.AsNoTracking().Select(...)`) porque simula un sistema externo independiente. Su implementación interna no necesita ser consistente con `Data`. No migrarla a ADO.NET+SP "por consistencia".
- **Excepción 2 — `ClienteDao`/`ProductoDao` en `Data`:** usan LINQ/EF Core vía `IDbContextFactory<OpinionesOltpReadContext>` (no `AddDbContext` directo — causa `InvalidOperationException` de concurrencia porque los extractores corren en paralelo con `Task.WhenAll`). La justificación: son datos de referencia usados para resolver integridad referencial, no el dato transaccional principal de la extracción — mismo patrón que el proyecto anterior del mismo autor (`OpinionDataLoader`), que también distinguía extracción transaccional (ADO.NET+SP) de resolución de referencias para FK (EF Core solo lectura).
- **Nunca usar Entity Framework Core para escritura.** Todas las escrituras (staging, MERGE, bulk copy) son ADO.NET puro. Esto no tiene excepciones.
- **Patrón DAO, no Repository.** Los DAOs transportan el resultado de una consulta o SP sin interpretarlo. Toda decisión de negocio vive en la capa de Services/Extractors, nunca en el DAO.

## 7. Resiliencia y manejo de errores

**Decisión confirmada:**

- `DatabaseExtractor<T>` y `ApiExtractor<T>` están envueltos en una política Polly de reintento con backoff exponencial (`Handle<SqlException>`, `Handle<HttpRequestException>`/`Or<TaskCanceledException>` respectivamente), 3 intentos, espera 1s/2s/4s.
- `CsvExtractor<T>` **no** lleva Polly — es lectura de disco local, no tiene fallas transitorias de red que reintentar. No agregarle retry "por consistencia".
- El orquestador (`OrquestadorExtraccion.ExtraerYValidarAsync<T>`) atrapa la excepción de cada extractor individualmente: si una fuente falla tras agotar sus reintentos, se registra el error y esa fuente devuelve una lista vacía, **sin detener las demás fuentes ni el proceso completo**. No cambiar esto a fail-fast.
- En `Load`, si una opinión referencia un `IdCliente`/`IdProducto` que no existe en los datos ya mergeados a las dimensiones (esto puede pasar porque `CsvExtractor` lee un archivo plano sin garantía de integridad referencial, a diferencia de las fuentes que sí tienen FK en el motor de base de datos), esa fila individual se descarta con un log de advertencia, no se aborta el batch completo.

## 8. Staging entre Extract y Load

**Decisión confirmada:** `DataLoader` escribe cada fuente extraída como un archivo JSON independiente en una carpeta de Staging (ruta configurable, relativa por defecto), todos con el mismo timestamp compartido por corrida. La carpeta acumula historial de todas las corridas — no se limpia automáticamente.

`StagingReader.LeerUltimoAsync<T>(prefijo)` filtra archivos por prefijo (`clientes_*.json`, `productos_*.json`, etc.) y toma el más reciente por fecha de modificación. Esto es intencional y ya fue evaluado — no es necesario limpiar archivos viejos ni cambiar el mecanismo de selección.

`Load` (Transform + carga a OLAP) lee exclusivamente de estos archivos de Staging, **nunca de `OpinionesOLTP` directamente** — esto fue una corrección deliberada respecto a un proyecto anterior del mismo autor, donde Transform sí consultaba OLTP directo. No revertir esto.

## 9. Alcance explícito — qué NO está construido

- **Módulo de visualización / Dashboard:** fuera de esta entrega, planificado para una fase posterior. No implementar ni asumir que existe.
- **Despliegue distribuido / contenedores / orquestación real:** fuera de alcance, ver sección 4.

## 10. Convención de nombres

El código de dominio (clases, métodos, variables de negocio) está en español (`OrquestadorExtraccion`, `ExtraerYValidarAsync`, `EsValido`). Términos técnicos genéricos de la plataforma (`DTO`, `Extractor`, `Validator`, `Cache`, `Service`) se mantienen en inglés cuando son el nombre estándar de un patrón reconocido. Hay alguna inconsistencia menor ya aceptada (ej. `ExtractResult`, `DataLoader.SaveAsync`) — no es necesario renombrar estos casos puntuales a español por consistencia estricta; no es una violación de arquitectura, es una decisión de nomenclatura ya tomada por el autor.

## 11. Qué sí puedes corregir libremente

Fuera de las decisiones marcadas como confirmadas arriba, sí son bienvenidas las correcciones normales de calidad de código: bugs, código muerto, falta de manejo de errores, nombres poco claros dentro de una misma clase, violaciones de responsabilidad única *dentro* del patrón ya establecido (por ejemplo, un extractor que además valida — eso sí se debe separar, según la sección 4). El objetivo de este documento es evitar que se reinterprete la arquitectura de fondo, no impedir mejoras de calidad puntuales.

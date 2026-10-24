# Glosario: pruebas de integración

Términos de este módulo, en orden alfabético. Entre paréntesis, la lección donde se explican.

-----

**Aislamiento de pruebas:** propiedad de que el resultado de una prueba no dependa de lo que hicieron otras. ([Respawn](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md))

**Builder de datos de prueba:** helper que crea datos válidos por defecto y permite cambiar solo lo que la prueba necesita. ([Respawn](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md))

**Colección de xUnit:** grupo de clases de prueba que comparten un fixture (`ICollectionFixture<T>`) y no se ejecutan en paralelo entre sí. ([Respawn](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md))

**ConfigureTestServices:** método de `IWebHostBuilder` que agrega registros de servicios después de los de la aplicación. ([Reemplazar servicios](02-Reemplazar%20servicios%20y%20autenticacion%20en%20pruebas.md))

**Estado conocido:** contenido exacto de la base al empezar una prueba. ([Respawn](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md))

**FakeTimeProvider:** implementación de `TimeProvider` cuya hora controla la prueba, del paquete `Microsoft.Extensions.TimeProvider.Testing`. ([Reemplazar servicios](02-Reemplazar%20servicios%20y%20autenticacion%20en%20pruebas.md))

**Handler de autenticación de prueba:** esquema de autenticación que construye el usuario a partir de datos de la petición, solo para pruebas. ([Reemplazar servicios](02-Reemplazar%20servicios%20y%20autenticacion%20en%20pruebas.md))

**IAsyncLifetime:** interfaz de xUnit para inicializar y liberar recursos de forma asíncrona antes y después de las pruebas. ([Testcontainers](03-Base%20de%20datos%20real%20con%20Testcontainers.md))

**Paridad con producción:** que el entorno de prueba se comporte como el real en lo que importa. ([Testcontainers](03-Base%20de%20datos%20real%20con%20Testcontainers.md))

**Proveedor InMemory:** proveedor de EF Core que guarda datos en colecciones en memoria; no es una base relacional. ([Testcontainers](03-Base%20de%20datos%20real%20con%20Testcontainers.md))

**Prueba de contrato:** prueba que verifica la forma pública de la API (rutas, códigos, JSON) desde el punto de vista de quien la consume. ([WebApplicationFactory](01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md))

**Prueba de humo (smoke test):** prueba amplia y superficial que comprueba que algo "no explota", sin verificar detalles. ([Ejercicios](Ejercicios.md))

**Prueba de integración:** prueba que ejercita varias piezas reales juntas en lugar de una unidad aislada. ([WebApplicationFactory](01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md))

**Punto de entrada (entry point):** tipo que identifica el ensamblado de la aplicación; con top-level statements, la clase `Program`. ([WebApplicationFactory](01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md))

**Respawn:** biblioteca que borra los datos de una base respetando las claves foráneas, sin recrear el esquema. ([Respawn](04-Datos%20de%20prueba%20y%20aislamiento%20con%20Respawn.md))

**Testcontainers:** biblioteca que crea, configura y elimina contenedores Docker desde el código de prueba. ([Testcontainers](03-Base%20de%20datos%20real%20con%20Testcontainers.md))

**TestServer:** servidor en memoria que reemplaza a Kestrel durante las pruebas; las peticiones no salen a la red. ([WebApplicationFactory](01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md))

**WebApplicationFactory&lt;TEntryPoint&gt;:** clase de `Microsoft.AspNetCore.Mvc.Testing` que arranca la aplicación en memoria para pruebas. ([WebApplicationFactory](01-WebApplicationFactory%20y%20pruebas%20de%20API%20en%20memoria.md))

**WithWebHostBuilder:** método que crea una fábrica derivada (una aplicación nueva) con configuración extra, sin modificar la original. ([Reemplazar servicios](02-Reemplazar%20servicios%20y%20autenticacion%20en%20pruebas.md))

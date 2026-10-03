# Migraciones y consultas

## En una frase
Una migración describe y versiona cambios de esquema; una consulta eficiente proyecta solo lo necesario, cancela de forma cooperativa y evita tracking cuando no habrá modificaciones.

-----

## Antes de empezar
Conviene que ya sepas: [DbContext, entidades y configuración](01-DbContext%20entidades%20y%20configuracion.md).

Palabras nuevas: **migración:** cambio versionado; **snapshot:** modelo anterior para comparar; **no tracking:** lectura sin seguimiento de cambios.

-----

## El problema
Ejecutar `Add-Migration` y asumir que la base ya cambió es falso. Aplicar una migración generada sin revisarla también puede borrar o transformar datos de forma peligrosa.

-----

## Cómo funciona
### 1. Herramientas
El proveedor ejecuta la aplicación; `Microsoft.EntityFrameworkCore.Design` y `dotnet-ef` habilitan herramientas de diseño y CLI. Mantén versiones principales compatibles.

### 2. Crear una migración
```bash
dotnet ef migrations add InitialCreate
```
EF compara el modelo actual con el snapshot anterior y genera métodos `Up` y `Down`. Todavía no modifica la base.

### 3. Aplicar
```bash
dotnet ef database update
```
La base registra migraciones aplicadas en su tabla de historial. En producción, decide si despliegas scripts revisados, bundles u otro proceso controlado; no ejecutes cambios destructivos improvisados al arrancar.

### 4. Consultar
```csharp
OrderRow[] rows = await db.Orders
    .AsNoTracking()
    .OrderBy(x => x.Id)
    .Select(x => new OrderRow(x.Id, x.Total))
    .ToArrayAsync(cancellationToken);
```
La proyección evita cargar columnas innecesarias y `AsNoTracking` reduce costo para lectura pura.

### 5. Cambiar
Carga o adjunta la entidad con intención clara, modifica propiedades permitidas y llama `SaveChangesAsync`. Evita marcar un objeto recibido por HTTP como completamente modificado: puede habilitar *overposting*.

### 6. Asincronía
Las APIs async liberan el hilo mientras espera I/O; no vuelven más rápida una consulta ineficiente. Propaga `CancellationToken` hasta el proveedor.

-----

## Ejemplo completo
Agrega herramientas al proyecto de la lección anterior:
```bash
dotnet package add Microsoft.EntityFrameworkCore.Design --version 10.0.0
dotnet tool install --global dotnet-ef --version 10.0.0
dotnet ef migrations add InitialCreate
dotnet ef database update
```

Endpoint de lectura:
```csharp
app.MapGet("/contacts", async (AppDbContext db, CancellationToken ct) =>
{
    ContactRow[] contacts = await db.Contacts
        .AsNoTracking()
        .OrderBy(x => x.Id)
        .Select(x => new ContactRow(x.Id, x.Name, x.Email))
        .ToArrayAsync(ct);

    return TypedResults.Ok(contacts);
});

public sealed record ContactRow(int Id, string Name, string Email);
```

Con una base vacía, la respuesta es:
```json
[]
```

-----

## Errores comunes
**1. Creer que crear una migración actualiza la base.** Qué pasa: el esquema permanece igual. Por qué: generación y aplicación son pasos distintos. Arreglo: revisa y aplica mediante el proceso elegido.

**2. Usar siempre tracking.** Qué pasa: aumenta memoria y trabajo en lecturas. Por qué: el change tracker conserva estado innecesario. Arreglo: usa `AsNoTracking` para consultas puras.

**3. Aplicar migraciones automáticamente sin estrategia.** Qué pasa: varias réplicas compiten o un cambio bloquea producción. Por qué: el despliegue de esquema requiere coordinación. Arreglo: usa un paso controlado y observable.

**4. Instalar EF Core 9 en un proyecto EF Core 8 por intuición.** Qué pasa: aparecen incompatibilidades. Por qué: las versiones principales no se mezclan arbitrariamente. Arreglo: alinea proveedor, Design y herramientas con la versión elegida.

-----

## Según la versión de .NET
- **EF6:** usa herramientas y comportamiento de la línea clásica.
- **EF Core:** usa migraciones propias y herramientas `dotnet ef` o Package Manager Console.
- **EF Core 10:** acompaña .NET 10; la forma `dotnet package add` requiere SDK 10, mientras `dotnet ef` conserva su espacio de comandos.

-----

## Cuándo sí y cuándo no
**Usa migraciones cuando:** el esquema evoluciona con el código y puedes revisar su impacto. **No apliques automáticamente sin evaluación cuando:** existen datos críticos, réplicas o cambios costosos; prepara un plan de despliegue y reversión.

-----

## Resumen en 5 líneas
1. Una migración se genera comparando el modelo con su snapshot.
2. Generar una migración no cambia la base de datos.
3. Toda migración debe revisarse antes de aplicarse.
4. Las lecturas puras suelen beneficiarse de proyección y no tracking.
5. Async y cancelación mejoran uso de recursos, no arreglan SQL deficiente.

-----

## Para profundizar
<details><summary>¿Por qué revisar `Down`?</summary>No toda operación se revierte sin pérdida. Eliminar una columna y recrearla no recupera sus datos; una reversión puede requerir respaldo o migración compensatoria.</details>

-----

## En entrevista
### Respuesta corta (junior)
Las migraciones versionan cambios del modelo al esquema. Primero se generan y revisan; después se aplican a la base.

### Respuesta ampliada (semi-senior)
EF compara contra el snapshot y genera operaciones `Up`/`Down`. Producción necesita coordinación, revisión y estrategia de rollback. Para consultas, proyecto DTOs, uso no tracking en lecturas y propago cancelación.

### Preguntas frecuentes de seguimiento
**1. ¿`database update` crea siempre la base?** Puede crearla si no existe y el proveedor lo admite, pero su función es aplicar migraciones pendientes.

**2. ¿`AsNoTracking` sirve para editar?** No es lo habitual; está pensado para lecturas sin persistir cambios de esas instancias.

-----

## Práctica
**Ejercicio 1.** Agrega `Phone` a `Contact` y describe el flujo seguro.
<details><summary>Solución</summary>Modifica el modelo, genera una migración con nombre descriptivo, revisa operaciones y nulabilidad, prueba sobre una copia o entorno controlado y luego aplícala mediante el proceso de despliegue.</details>

-----

## Siguiente lección
[CRUD seguro con MVC y EF Core](03-CRUD%20seguro%20con%20MVC%20y%20EF%20Core.md)

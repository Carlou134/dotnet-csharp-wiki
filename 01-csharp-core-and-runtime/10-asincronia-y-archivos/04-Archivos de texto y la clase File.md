# Archivos de texto y la clase File

## En una frase

Para la mayoría de las tareas con archivos no necesitas manejar bytes: la clase estática **`File`** lee y escribe archivos completos en una línea (`ReadAllText`, `WriteAllLines`, `ReadLines`...), **`FileInfo`**, **`Directory`** y **`Path`** manejan archivos, carpetas y rutas, y **`StreamReader`/`StreamWriter`** leen y escriben texto línea a línea con la codificación correcta.

-----

## Antes de empezar

Conviene que ya sepas:

* Qué es un stream, `FileStream`, `FileMode` y `using`, de [Archivos y streams](03-Archivos%20y%20streams.md).
* `async`/`await`, de [Programación asíncrona](01-Programacion%20asincrona.md).
* Iteradores y ejecución diferida, de [Iteradores y yield](../06-colecciones/05-Iteradores%20y%20yield.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **`File`:** clase estática con operaciones de archivo de una sola llamada.
* **`FileInfo`:** objeto que representa un archivo concreto y sus propiedades.
* **`Directory` / `DirectoryInfo`:** lo mismo, para carpetas.
* **`Path`:** clase estática para construir y analizar rutas.
* **`StreamReader` / `StreamWriter`:** leen y escriben **texto** sobre un stream.
* **Codificación (*encoding*):** la regla que convierte caracteres en bytes (UTF-8, ASCII...).
* **BOM (*Byte Order Mark*):** bytes opcionales al inicio de un archivo que indican su codificación.
* **Ruta relativa / absoluta:** relativa a la carpeta actual (`datos/a.txt`) o completa (`C:\datos\a.txt`).

-----

## El problema

Con `FileStream`, guardar una línea de texto implica convertirla a bytes con la codificación correcta, escribir el búfer, cerrar el stream... Para algo tan común como "leer un archivo de configuración" o "agregar una línea a un log", es demasiada ceremonia.

Además, aparecen problemas de texto que los bytes no resuelven solos:

* Los acentos y la ñ se ven como `Ã±` porque se leyó con otra codificación.
* Las rutas armadas con `+` y `"\\"` fallan en Linux, que usa `/`.
* Leer un archivo de registros de 10 GB con `ReadAllLines` se queda sin memoria.

-----

## Cómo funciona

### `File`: operaciones de una línea

```csharp
// Escribir (crea el archivo o lo SOBRESCRIBE)
File.WriteAllText("nota.txt", "Hola\nmundo");
File.WriteAllLines("lista.txt", new[] { "uno", "dos", "tres" });

// Agregar al final (crea el archivo si no existe)
File.AppendAllText("log.txt", $"{DateTime.Now:HH:mm} Inicio{Environment.NewLine}");
File.AppendAllLines("log.txt", new[] { "línea A", "línea B" });

// Leer
string todo = File.ReadAllText("nota.txt");          // todo el archivo en un string
string[] lineas = File.ReadAllLines("lista.txt");    // un array con todas las líneas
byte[] bytes = File.ReadAllBytes("foto.jpg");         // archivos binarios

// Gestionar
bool existe = File.Exists("nota.txt");
File.Copy("nota.txt", "nota-copia.txt", overwrite: true);
File.Move("nota-copia.txt", "archivada.txt", overwrite: true);
File.Delete("archivada.txt");                         // no falla si no existe
DateTime modificado = File.GetLastWriteTime("nota.txt");
```

Estos métodos abren el archivo, hacen la operación y lo **cierran** solos: no necesitas `using`.

### `ReadAllLines` frente a `ReadLines`

```csharp
string[] todas = File.ReadAllLines("app.log");           // carga TODO el archivo en memoria antes de devolver
IEnumerable<string> perezosas = File.ReadLines("app.log"); // lee línea por línea a medida que recorres

int errores = File.ReadLines("app.log").Count(l => l.Contains("ERROR"));   // memoria mínima, aunque pese 10 GB
string? primerError = File.ReadLines("app.log").FirstOrDefault(l => l.Contains("ERROR"));   // se detiene ahí
```

`ReadLines` es un iterador (ver [Iteradores y yield](../06-colecciones/05-Iteradores%20y%20yield.md)): solo hay una línea en memoria a la vez y se puede cortar antes de llegar al final. Para archivos grandes, o cuando solo necesitas parte, usa `ReadLines`.

### Versiones asíncronas

```csharp
await File.WriteAllTextAsync("nota.txt", "Hola");
string contenido = await File.ReadAllTextAsync("nota.txt");
string[] lineas = await File.ReadAllLinesAsync("lista.txt");

await foreach (string linea in File.ReadLinesAsync("app.log"))   // .NET 7
{
    if (linea.Contains("ERROR")) Console.WriteLine(linea);
}
```

En servidores y aplicaciones con interfaz gráfica, prefiere las versiones `...Async`.

### Rutas con `Path`

```csharp
string carpeta = Path.Combine("datos", "2026", "octubre");      // "datos/2026/octubre" o "datos\2026\octubre"
string archivo = Path.Combine(carpeta, "ventas.csv");

Path.GetFileName(archivo);                     // "ventas.csv"
Path.GetFileNameWithoutExtension(archivo);     // "ventas"
Path.GetExtension(archivo);                    // ".csv"
Path.GetDirectoryName(archivo);                // la carpeta
Path.ChangeExtension(archivo, ".bak");         // ".../ventas.bak"
Path.GetFullPath("ventas.csv");                // ruta absoluta según la carpeta actual
Path.GetTempPath();                            // carpeta temporal del sistema
string temporal = Path.GetTempFileName();      // crea un archivo temporal vacío y devuelve su ruta
```

`Path.Combine` usa el separador correcto del sistema operativo. Nunca armes rutas con `+ "\\" +`: tu programa fallaría en Linux o en un contenedor Docker.

**¿Desde dónde se resuelven las rutas relativas?** Desde el **directorio de trabajo actual** (`Environment.CurrentDirectory`), que depende de cómo se ejecute el programa (con `dotnet run` suele ser la carpeta del proyecto; desde el IDE o como servicio, puede ser otra). Para archivos que acompañan a tu aplicación, usa `AppContext.BaseDirectory` (la carpeta del ejecutable).

### Carpetas con `Directory`

```csharp
Directory.CreateDirectory(carpeta);                              // crea toda la ruta; no falla si ya existe
bool hay = Directory.Exists(carpeta);
string[] csvs = Directory.GetFiles(carpeta, "*.csv");            // archivos que cumplen el patrón
var todos = Directory.EnumerateFiles("datos", "*.*", SearchOption.AllDirectories);   // recursivo y perezoso
string[] subcarpetas = Directory.GetDirectories("datos");
Directory.Delete(carpeta, recursive: true);                      // borra con todo su contenido
```

`Enumerate...` devuelve los resultados a medida que los encuentra (perezoso); `Get...` espera a tenerlos todos en un array.

### `FileInfo`: un archivo como objeto

`File` es ideal para operaciones sueltas. Si vas a consultar varias propiedades de un mismo archivo, `FileInfo` lo representa como un objeto:

```csharp
var info = new FileInfo("ventas.csv");

if (info.Exists)
{
    Console.WriteLine(info.Name);            // ventas.csv
    Console.WriteLine(info.FullName);        // ruta absoluta
    Console.WriteLine(info.Extension);       // .csv
    Console.WriteLine(info.Length);          // tamaño en bytes
    Console.WriteLine(info.CreationTime);
    Console.WriteLine(info.Directory?.Name); // carpeta que lo contiene

    info.CopyTo("ventas-respaldo.csv", overwrite: true);
}
```

`FileInfo` guarda la información en el momento en que la lee: si el archivo cambia después, llama a `info.Refresh()`.

### `StreamReader` y `StreamWriter`: texto línea a línea

Cuando necesitas más control que `File` (procesar mientras lees, escribir de a poco, elegir la codificación):

```csharp
using (var escritor = new StreamWriter("reporte.txt", append: false, Encoding.UTF8))
{
    escritor.WriteLine("Reporte de ventas");
    escritor.WriteLine($"Generado: {DateTime.Now:g}");
    escritor.Write("Total: ");
    escritor.WriteLine(1234.5m);
}   // se vacía el búfer y se cierra el archivo

using (var lector = new StreamReader("reporte.txt", Encoding.UTF8))
{
    string? linea;
    int numero = 1;
    while ((linea = lector.ReadLine()) is not null)
    {
        Console.WriteLine($"{numero++}: {linea}");
    }
}
```

| `StreamReader` | Qué hace |
| --- | --- |
| `ReadLine()` | Lee la siguiente línea; devuelve `null` al final |
| `ReadToEnd()` | Lee todo lo que queda |
| `Read()` | Lee un carácter (como `int`; `-1` al final) |
| `EndOfStream` | `true` si no queda nada por leer |
| `ReadLineAsync()`, `ReadToEndAsync()` | Versiones asíncronas |

`StreamReader` y `StreamWriter` también se pueden crear **sobre cualquier stream** (`new StreamReader(fileStream)`, `new StreamWriter(gzipStream)`): así se combina texto con compresión, red o memoria.

### Codificaciones

Un archivo de texto son bytes; la **codificación** dice cómo convertirlos en caracteres:

* **UTF-8:** el estándar actual. Representa cualquier carácter (acentos, ñ, emojis). Es el valor por defecto de `File` y de `StreamReader`/`StreamWriter` en .NET.
* **ASCII:** solo 128 caracteres; los acentos se pierden (se convierten en `?`).
* **Latin-1 / Windows-1252:** codificaciones antiguas de Windows que todavía aparecen en archivos generados por sistemas viejos o por Excel.

Si un archivo se ve con caracteres raros (`Ã±` en lugar de `ñ`), casi siempre es un problema de codificación: se escribió con una y se leyó con otra. Indica la codificación explícitamente cuando la conozcas:

```csharp
string texto = File.ReadAllText("viejo.csv", Encoding.Latin1);
```

-----

## Ejemplo completo

Procesar un CSV de ventas, generar un resumen y archivar el original:

```csharp
using System.Globalization;
using System.Text;

string carpeta = Path.Combine(Path.GetTempPath(), "ventas-demo");
Directory.CreateDirectory(carpeta);
string csv = Path.Combine(carpeta, "ventas.csv");

// 1. Crear un CSV de ejemplo
await File.WriteAllLinesAsync(csv, new[]
{
    "fecha,producto,cantidad,precio",
    "2026-10-01,Teclado,2,150.00",
    "2026-10-01,Mouse,5,60.00",
    "2026-10-02,Monitor,1,800.00",
    "linea-corrupta",
    "2026-10-02,Teclado,1,150.00"
});

// 2. Leer de forma perezosa, saltando el encabezado y las líneas inválidas
var totales = new Dictionary<string, decimal>();
int invalidas = 0;

foreach (string linea in File.ReadLines(csv).Skip(1))
{
    string[] campos = linea.Split(',');
    if (campos.Length != 4
        || !int.TryParse(campos[2], out int cantidad)
        || !decimal.TryParse(campos[3], NumberStyles.Number, CultureInfo.InvariantCulture, out decimal precio))
    {
        invalidas++;
        continue;
    }

    totales[campos[1]] = totales.GetValueOrDefault(campos[1]) + cantidad * precio;
}

// 3. Escribir el resumen con StreamWriter
string resumen = Path.Combine(carpeta, "resumen.txt");
await using (var escritor = new StreamWriter(resumen, append: false, Encoding.UTF8))
{
    await escritor.WriteLineAsync($"Resumen generado el {DateTime.Now:yyyy-MM-dd}");
    foreach (var (producto, total) in totales.OrderByDescending(t => t.Value))
    {
        await escritor.WriteLineAsync($"{producto,-10} {total,10:N2}");
    }
    await escritor.WriteLineAsync($"Líneas inválidas: {invalidas}");
}

Console.WriteLine(await File.ReadAllTextAsync(resumen));

// 4. Archivar el CSV original y listar la carpeta
string archivado = Path.ChangeExtension(csv, ".procesado");
File.Move(csv, archivado, overwrite: true);

foreach (var info in new DirectoryInfo(carpeta).EnumerateFiles())
{
    Console.WriteLine($"{info.Name,-20} {info.Length,6} bytes");
}

Directory.Delete(carpeta, recursive: true);   // limpiar el ejemplo
```

Salida (la fecha y los tamaños pueden variar):

```text
Resumen generado el 2026-10-02
Monitor        800.00
Teclado        450.00
Mouse          300.00
Líneas inválidas: 1

ventas.procesado        131 bytes
resumen.txt             112 bytes
```

Se usaron `Path` para rutas portables, `ReadLines` para leer sin cargar todo, `TryParse` con cultura invariante para no depender de la configuración regional, `StreamWriter` asíncrono para escribir y `File`/`DirectoryInfo` para gestionar los archivos.

-----

## Errores comunes

**1. El archivo no existe.**
Qué pasa: `System.IO.FileNotFoundException: Could not find file '...\datos.txt'.`
Por qué: la ruta es incorrecta o es relativa a otra carpeta de trabajo.
Arreglo: comprueba con `File.Exists`, usa `Path.GetFullPath` para ver la ruta real y, para archivos de la aplicación, `AppContext.BaseDirectory`.

**2. La carpeta no existe.**
Qué pasa: `System.IO.DirectoryNotFoundException: Could not find a part of the path`.
Por qué: `File.WriteAllText` no crea carpetas.
Arreglo: `Directory.CreateDirectory(Path.GetDirectoryName(ruta)!)` antes de escribir.

**3. Sobrescribir sin querer.**
Qué pasa: se pierde el contenido anterior.
Por qué: `WriteAllText`/`WriteAllLines` y `new StreamWriter(ruta)` sobrescriben.
Arreglo: `AppendAllText`/`AppendAllLines` o `new StreamWriter(ruta, append: true)`.

**4. `ReadAllLines` con archivos enormes.**
Qué pasa: `OutOfMemoryException` o un consumo de memoria enorme.
Por qué: carga todo el archivo en un array.
Arreglo: `File.ReadLines` (perezoso).

**5. Caracteres raros (`Ã±`, `?`).**
Qué pasa: el texto se ve corrupto.
Por qué: se leyó con una codificación distinta a la que se usó para escribirlo.
Arreglo: indica la codificación correcta (`Encoding.UTF8`, `Encoding.Latin1`...).

**6. Sin permisos.**
Qué pasa: `System.UnauthorizedAccessException: Access to the path '...' is denied.`
Por qué: la carpeta está protegida (por ejemplo, `Program Files`) o el archivo es de solo lectura.
Arreglo: escribe en carpetas del usuario o de datos de la aplicación (`Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData)`).

-----

## Según la versión de C#

* **.NET 1.0:** `File`, `FileInfo`, `Directory`, `Path`, `StreamReader` y `StreamWriter`.
* **.NET 4.0:** `File.ReadLines`, `Directory.EnumerateFiles` y otros métodos perezosos.
* **.NET Core 2.0:** `File.ReadAllTextAsync`, `WriteAllTextAsync` y compañía.
* **.NET Core 3.0:** `File.Move` con `overwrite`.
* **.NET 5:** `Encoding.Latin1`.
* **.NET 7:** `File.ReadLinesAsync` (un `IAsyncEnumerable<string>`).

-----

## Cuándo sí y cuándo no

| Necesitas... | Usa |
| --- | --- |
| Leer o escribir un archivo pequeño completo | `File.ReadAllText` / `WriteAllText` (o sus versiones `Async`) |
| Recorrer un archivo grande línea a línea | `File.ReadLines` |
| Agregar líneas a un log simple | `File.AppendAllText` |
| Escribir mucho texto de a poco, o sobre otro stream | `StreamWriter` |
| Datos binarios o acceso por posición | `FileStream` (lección anterior) |
| Construir rutas | `Path.Combine` (nunca concatenar strings) |
| JSON | `System.Text.Json` (`JsonSerializer.Serialize`/`Deserialize`) |

Para logs reales en aplicaciones, usa una librería de logging (`ILogger` con Serilog u otro proveedor) en lugar de escribir archivos a mano.

-----

## Resumen en 5 líneas

1. `File.ReadAllText`/`WriteAllText`/`AppendAllText` y sus versiones `Lines` y `Async` resuelven las operaciones comunes en una línea.
2. `File.ReadLines` es perezoso: úsalo para archivos grandes; `ReadAllLines` carga todo en memoria.
3. `Path.Combine` construye rutas portables; las rutas relativas dependen del directorio de trabajo.
4. `Directory` y `FileInfo`/`DirectoryInfo` gestionan carpetas y propiedades de archivos.
5. `StreamReader`/`StreamWriter` leen y escriben texto con una codificación (UTF-8 por defecto); los caracteres raros son un problema de codificación.

-----

## Para profundizar

<details>
<summary>Escritura atómica: no dejar archivos a medias</summary>

Si el programa se corta mientras escribe un archivo importante (por ejemplo, la configuración), puede quedar corrupto. El patrón seguro es escribir en un temporal y reemplazar al final:

```csharp
string temporal = ruta + ".tmp";
await File.WriteAllTextAsync(temporal, nuevoContenido);
File.Move(temporal, ruta, overwrite: true);   // el reemplazo es (casi) atómico en el mismo disco
```

Así, el archivo siempre tiene la versión anterior completa o la nueva completa, nunca una mezcla.

</details>

<details>
<summary>Vigilar cambios con FileSystemWatcher</summary>

`FileSystemWatcher` dispara eventos cuando se crean, modifican, renombran o borran archivos en una carpeta:

```csharp
using var vigilante = new FileSystemWatcher(carpeta, "*.csv") { EnableRaisingEvents = true };
vigilante.Created += (_, e) => Console.WriteLine($"Nuevo archivo: {e.Name}");
```

Combina lo visto en [Eventos](../09-delegados-y-eventos/02-Eventos.md) con el sistema de archivos. Ojo: puede disparar varios eventos por una sola operación y no es 100 % confiable en carpetas de red.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Para archivos de texto se usan los métodos estáticos de `File`, como `ReadAllText`, `WriteAllText` o `AppendAllText`. Para archivos grandes conviene `File.ReadLines`, que lee línea por línea. `StreamReader` y `StreamWriter` permiten leer y escribir texto de forma más controlada, siempre dentro de un `using`. Las rutas se arman con `Path.Combine`.

### Respuesta ampliada (semi-senior)

`File` encapsula la apertura, la lectura o escritura y el cierre, y ofrece variantes asíncronas; `ReadLines` y `Directory.EnumerateFiles` son perezosos y escalan con archivos o carpetas grandes, mientras que `ReadAllLines`/`GetFiles` materializan. `StreamReader`/`StreamWriter` son adaptadores de texto sobre cualquier `Stream`, con UTF-8 sin BOM por defecto; los problemas de "mojibake" vienen de codificaciones distintas al escribir y leer. Las rutas deben ser portables (`Path.Combine`, `Path.DirectorySeparatorChar`) y no depender del directorio de trabajo (`AppContext.BaseDirectory`). Para escrituras críticas se usa el patrón de archivo temporal y reemplazo, y en servidores, siempre las APIs asíncronas.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `File.ReadAllLines` y `File.ReadLines`?**
`ReadAllLines` lee todo el archivo y devuelve un array; `ReadLines` devuelve un `IEnumerable<string>` perezoso que lee línea por línea a medida que se recorre.

**2. ¿Por qué usar `Path.Combine`?**
Porque usa el separador correcto de cada sistema operativo y maneja las barras entre partes, evitando rutas inválidas.

**3. ¿Por qué un texto con tildes se ve con caracteres raros?**
Porque se leyó con una codificación distinta a la que se usó para escribirlo; hay que indicar la correcta al leer.

-----

## Práctica

**Ejercicio 1.** Escribe un programa que cree una carpeta `notas` (dentro de la carpeta temporal del sistema), guarde tres archivos `.txt` con una frase cada uno, y luego liste los archivos de la carpeta mostrando nombre, tamaño y primera línea.

<details>
<summary>Solución</summary>

```csharp
string carpeta = Path.Combine(Path.GetTempPath(), "notas");
Directory.CreateDirectory(carpeta);

string[] frases = { "Comprar pan", "Estudiar LINQ", "Llamar a Ana" };
for (int i = 0; i < frases.Length; i++)
{
    File.WriteAllText(Path.Combine(carpeta, $"nota{i + 1}.txt"), frases[i]);
}

foreach (var info in new DirectoryInfo(carpeta).EnumerateFiles("*.txt"))
{
    string primera = File.ReadLines(info.FullName).FirstOrDefault() ?? "(vacío)";
    Console.WriteLine($"{info.Name,-10} {info.Length,4} bytes  → {primera}");
}

Directory.Delete(carpeta, recursive: true);
```

</details>

**Ejercicio 2.** Escribe un método `ContarPalabrasAsync(string ruta)` que lea un archivo de texto de forma asíncrona y perezosa (`File.ReadLinesAsync`) y devuelva la cantidad total de palabras. Pruébalo con un archivo que crees en el mismo programa.

<details>
<summary>Solución</summary>

```csharp
string ruta = Path.Combine(Path.GetTempPath(), "texto.txt");
await File.WriteAllLinesAsync(ruta, new[] { "hola mundo", "esto es  C#", "", "fin" });

Console.WriteLine(await ContarPalabrasAsync(ruta));   // 6
File.Delete(ruta);

static async Task<int> ContarPalabrasAsync(string ruta)
{
    int total = 0;
    await foreach (string linea in File.ReadLinesAsync(ruta))
    {
        total += linea.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
    }
    return total;
}
```

</details>

-----

## Siguiente lección

Terminaste el módulo de asincronía y archivos. Continúa con [Código limpio](../11-codigo-limpio/README.md).

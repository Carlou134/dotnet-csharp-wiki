# Archivos y streams

## En una frase

Un **stream** es una secuencia de **bytes** que se puede leer, escribir y, a veces, recorrer a saltos; `FileStream` es el stream de los archivos, se abre indicando **qué hacer si el archivo existe o no** (`FileMode`) y **para qué** (`FileAccess`), y siempre se cierra con `using`.

-----

## Antes de empezar

Conviene que ya sepas:

* `using`, `IDisposable` y el manejo de excepciones, de [Manejo de excepciones](../08-excepciones/01-Manejo%20de%20excepciones.md).
* Arrays de `byte` y el tipo `byte` (0 a 255), de [Variables y tipos de datos](../01-tipos-y-variables/01-Variables%20y%20tipos%20de%20datos.md).
* `async`/`await`, de [Programación asíncrona](01-Programacion%20asincrona.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Stream (flujo):** secuencia de bytes con operaciones para leer, escribir y moverse.
* **`FileStream`:** stream que lee y escribe un archivo del disco.
* **`FileMode`:** qué hacer al abrir: crear, abrir, agregar al final, truncar...
* **`FileAccess`:** para qué se abre: leer, escribir o ambos.
* **Búfer (*buffer*):** zona de memoria intermedia donde se acumulan datos antes de leerlos o escribirlos.
* **`Flush`:** forzar la escritura en el disco de lo que quedó en el búfer.
* **Posición (`Position`):** el punto del stream donde ocurrirá la próxima lectura o escritura.
* **Buscar (`Seek`):** mover la posición a un lugar concreto.

-----

## El problema

Hasta ahora, los datos de tus programas viven en memoria y desaparecen al cerrarlos. Para guardar una configuración, leer un archivo de registros o procesar una imagen, necesitas el **disco**. Y el disco trae complicaciones que la memoria no tiene:

* El archivo puede **no existir**, estar **bloqueado** por otro programa o no tener **permisos**.
* Si lo abres y no lo cierras, queda **bloqueado** para los demás hasta que el proceso termine.
* Escribir byte a byte en el disco es **muy lento**.
* Un archivo de 4 GB no entra en memoria: hay que leerlo **por partes**.

.NET resuelve todo esto con una abstracción común: el **stream**.

-----

## Cómo funciona

### La familia de streams

`Stream` es una clase abstracta que define cómo leer y escribir secuencias de bytes. Cada tipo de stream se ocupa de un origen distinto, pero se usan igual:

| Stream | Origen o destino |
| --- | --- |
| `FileStream` | Un archivo del disco |
| `MemoryStream` | Un array de bytes en memoria |
| `NetworkStream` | Una conexión de red (socket TCP) |
| `CryptoStream` | Cifra o descifra lo que pasa por él |
| `GZipStream` | Comprime o descomprime lo que pasa por él |

Como todos derivan de `Stream`, un método que recibe un `Stream` funciona con cualquiera de ellos (polimorfismo). Y se pueden **encadenar**: un `GZipStream` sobre un `FileStream` escribe un archivo comprimido.

```text
tu código ──bytes──► StreamWriter ──bytes──► GZipStream ──bytes comprimidos──► FileStream ──► disco
           (texto)   (codifica UTF-8)        (comprime)                       (escribe)
```

Cada capa solo conoce a la siguiente; puedes agregar o quitar capas (cifrado, compresión) sin cambiar las demás.

### Abrir un archivo: `FileMode` y `FileAccess`

```csharp
using var fs = new FileStream("datos.bin", FileMode.Open, FileAccess.Read);
```

`FileMode` decide qué pasa según si el archivo existe:

| `FileMode` | Si **no** existe | Si **ya** existe |
| --- | --- | --- |
| `CreateNew` | Lo crea | Lanza `IOException` |
| `Create` | Lo crea | Lo **sobrescribe** (lo vacía) |
| `Open` | Lanza `FileNotFoundException` | Lo abre |
| `OpenOrCreate` | Lo crea | Lo abre (sin vaciarlo) |
| `Truncate` | Lanza `FileNotFoundException` | Lo abre y lo vacía |
| `Append` | Lo crea | Lo abre y se posiciona **al final** (solo escritura) |

`FileAccess` decide qué operaciones se permiten: `Read`, `Write` o `ReadWrite`.

### Cerrar siempre: `using`

Un `FileStream` mantiene el archivo abierto (y bloqueado) hasta que se libera. `FileStream` implementa `IDisposable`, así que se usa con `using`:

```csharp
using (var fs = new FileStream("salida.bin", FileMode.Create, FileAccess.Write))
{
    // usar fs
}   // aquí se cierra el archivo, aunque haya habido una excepción

using var fs2 = new FileStream("otra.bin", FileMode.Create);   // forma corta: se cierra al final del bloque actual
```

Si por algún motivo no puedes usar `using`, el equivalente es un `try`/`finally` con la variable declarada **antes** del `try`:

```csharp
FileStream? fs = null;
try
{
    fs = new FileStream("entrada.bin", FileMode.Open, FileAccess.Read);
    // usar fs
}
finally
{
    fs?.Dispose();
}
```

### Leer bytes

`Read` llena un array de bytes (el búfer) y devuelve **cuántos bytes leyó de verdad**, que pueden ser menos de los pedidos. Devuelve `0` al llegar al final:

```csharp
using var fs = new FileStream("datos.bin", FileMode.Open, FileAccess.Read);

byte[] buffer = new byte[4096];
int leidos;
long total = 0;

while ((leidos = fs.Read(buffer, 0, buffer.Length)) > 0)
{
    total += leidos;
    // procesar buffer[0..leidos]: ¡solo los primeros 'leidos' bytes son válidos!
}

Console.WriteLine($"Leídos {total} bytes");
```

Los tres argumentos de `Read` son: el array destino, desde qué posición del array escribir, y cuántos bytes intentar leer.

`ReadByte` lee de a un byte y devuelve `-1` al final (por eso devuelve `int` y no `byte`):

```csharp
int b;
while ((b = fs.ReadByte()) != -1)
{
    Console.Write($"{b:X2} ");   // en hexadecimal
}
```

Leer de a un byte es mucho más lento: prefiere `Read` con un búfer.

### Escribir bytes

```csharp
byte[] datos = { 72, 111, 108, 97 };   // "Hola" en ASCII

using (var fs = new FileStream("hola.bin", FileMode.Create, FileAccess.Write))
{
    fs.Write(datos, 0, datos.Length);   // array, desde dónde, cuántos
    fs.WriteByte(33);                   // '!'
}
```

### El búfer y `Flush`

Escribir en el disco es lento, así que `FileStream` acumula los datos en un **búfer interno** y los escribe en bloque cuando:

1. El búfer se llena.
2. Se cierra el stream (`Dispose`, al salir del `using`).
3. Se llama a `Flush()`.

```csharp
using var fs = new FileStream("log.bin", FileMode.Append, FileAccess.Write);
fs.Write(datos);
fs.Flush();   // fuerza la escritura ahora (por ejemplo, antes de una operación riesgosa)
```

En la mayoría de los casos no hace falta `Flush`: el `using` lo hace al cerrar. Úsalo cuando otro proceso necesita ver los datos de inmediato o cuando un corte inesperado no debe perderlos.

### Moverse: `Position`, `Seek` y la longitud

```text
byte:      0    1    2    3    4    5    6    7    8    9   10   11
         ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
archivo  │ 48 │ 6F │ 6C │ 61 │ 20 │ 6D │ 75 │ 6E │ 64 │ 6F │ 21 │ 0A │   Length = 12
         └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
                              ▲
                          Position = 4   → el próximo Read/Write empieza aquí
                                           y Position avanza según los bytes leídos o escritos

Seek(2, Begin)    → Position = 2
Seek(-3, End)     → Position = 9
Seek(1, Current)  → Position actual + 1
```

```csharp
using var fs = new FileStream("datos.bin", FileMode.Open, FileAccess.ReadWrite);

Console.WriteLine(fs.Length);        // tamaño en bytes (solo lectura)
Console.WriteLine(fs.Position);      // 0 al abrir

fs.Seek(10, SeekOrigin.Begin);       // 10 bytes desde el principio
fs.Seek(-4, SeekOrigin.End);         // 4 bytes antes del final
fs.Seek(2, SeekOrigin.Current);      // 2 bytes más adelante que la posición actual
fs.Position = 0;                     // equivalente a Seek(0, SeekOrigin.Begin)

fs.SetLength(fs.Position);           // trunca el archivo en la posición actual
fs.SetLength(fs.Length + 100);       // lo extiende 100 bytes (rellenos con ceros)
```

* `Length` es de **solo lectura**; para cambiar el tamaño se usa el método `SetLength`.
* `CanSeek` indica si el stream admite moverse (un `NetworkStream`, por ejemplo, no).
* Leer secuencialmente es más rápido que saltar: usa `Seek` solo cuando el formato del archivo lo requiere (por ejemplo, leer un encabezado al final).

### Copiar y versiones asíncronas

```csharp
using var origen = File.OpenRead("foto.jpg");          // atajo para FileStream de lectura
using var destino = File.Create("copia.jpg");          // atajo para FileStream de escritura
await origen.CopyToAsync(destino);                     // copia todo, por bloques

byte[] buffer = new byte[4096];
int n = await origen.ReadAsync(buffer);                // lectura asíncrona
await destino.WriteAsync(buffer.AsMemory(0, n));       // escritura asíncrona
```

En aplicaciones de servidor o con interfaz gráfica, usa las versiones `...Async` para no bloquear hilos mientras se espera al disco.

### `MemoryStream`

Un stream sobre memoria. Es útil para construir datos antes de guardarlos, para pruebas o para APIs que piden un `Stream`:

```csharp
using var ms = new MemoryStream();
ms.Write(new byte[] { 1, 2, 3 });
byte[] todo = ms.ToArray();   // [1, 2, 3]
```

-----

## Ejemplo completo

Guardar y leer registros de tamaño fijo en un archivo binario, accediendo directamente a uno por su posición:

```csharp
const string Ruta = "temperaturas.bin";
const int TamanoRegistro = sizeof(int) + sizeof(double);   // 4 bytes de id + 8 de temperatura

// 1. Escribir 5 registros
using (var fs = new FileStream(Ruta, FileMode.Create, FileAccess.Write))
{
    for (int id = 1; id <= 5; id++)
    {
        double temperatura = 18.5 + id;
        fs.Write(BitConverter.GetBytes(id));            // int → 4 bytes
        fs.Write(BitConverter.GetBytes(temperatura));   // double → 8 bytes
    }
}
Console.WriteLine($"Archivo creado: {new FileInfo(Ruta).Length} bytes");

// 2. Leer solo el registro 4 saltando directamente a su posición
try
{
    using var fs = new FileStream(Ruta, FileMode.Open, FileAccess.ReadWrite);
    int indice = 3;                                       // el cuarto registro (empezando en 0)
    fs.Seek(indice * TamanoRegistro, SeekOrigin.Begin);

    byte[] buffer = new byte[TamanoRegistro];
    int leidos = fs.Read(buffer, 0, buffer.Length);

    int id = BitConverter.ToInt32(buffer, 0);
    double temp = BitConverter.ToDouble(buffer, sizeof(int));
    Console.WriteLine($"Registro {id}: {temp} °C (leídos {leidos} bytes, posición final {fs.Position})");

    // 3. Truncar: quedarse solo con los 2 primeros registros
    fs.SetLength(2 * TamanoRegistro);
    Console.WriteLine($"Después de truncar: {fs.Length} bytes");
}
catch (FileNotFoundException)
{
    Console.WriteLine("No se encontró el archivo.");
}
catch (IOException ex)
{
    Console.WriteLine($"Error de E/S: {ex.Message}");
}
finally
{
    File.Delete(Ruta);                                     // limpiar el ejemplo
}
```

Salida:

```text
Archivo creado: 60 bytes
Registro 4: 22.5 °C (leídos 12 bytes, posición final 48)
Después de truncar: 24 bytes
```

Como cada registro ocupa exactamente 12 bytes, el registro `i` empieza en el byte `i * 12`: `Seek` permite leerlo sin recorrer los anteriores. Así funcionan, a gran escala, los índices de las bases de datos.

-----

## Errores comunes

**1. No cerrar el stream.**
Qué pasa: `IOException: The process cannot access the file 'datos.bin' because it is being used by another process.` al intentar abrirlo de nuevo.
Por qué: el archivo sigue abierto y bloqueado.
Arreglo: `using` siempre.

**2. Ignorar el valor que devuelve `Read`.**
Qué pasa: se procesan bytes "basura" del final del búfer.
Por qué: `Read` puede leer menos bytes que el tamaño del búfer (en especial en el último bloque).
Arreglo: usa solo `buffer[0..leidos]` y repite hasta que devuelva 0.

**3. Usar `FileMode.Create` cuando querías agregar.**
Qué pasa: se pierde todo el contenido anterior del archivo.
Por qué: `Create` sobrescribe.
Arreglo: `FileMode.Append` para agregar al final, `OpenOrCreate` para abrir sin vaciar.

**4. Asignar `Length`.**
Qué pasa: `error CS0200: Property or indexer 'FileStream.Length' cannot be assigned to -- it is read only`.
Por qué: el tamaño se cambia con un método.
Arreglo: `fs.SetLength(nuevoTamano)`.

**5. Leer con `FileAccess.Write` (o escribir con `Read`).**
Qué pasa: `NotSupportedException: Stream does not support reading.`
Por qué: el stream se abrió solo para la otra operación.
Arreglo: abre con el `FileAccess` adecuado o `ReadWrite`.

**6. Rutas con barras invertidas sin escapar.**
Qué pasa: `error CS1009: Unrecognized escape sequence` en `"C:\datos\archivo.bin"`.
Por qué: `\d` y `\a` se interpretan como secuencias de escape.
Arreglo: `@"C:\datos\archivo.bin"` o, mejor, `Path.Combine("C:", "datos", "archivo.bin")` (próxima lección).

-----

## Según la versión de C#

* **.NET 1.0:** `Stream`, `FileStream` y `MemoryStream`.
* **.NET 4.0:** `CopyTo`.
* **.NET 4.5 / C# 5:** `ReadAsync`, `WriteAsync`, `CopyToAsync`, `FlushAsync`.
* **C# 8:** declaraciones `using var` y `await using` para streams asíncronos.
* **.NET Core 2.1:** sobrecargas con `Span<byte>` y `Memory<byte>` (`fs.Write(datos)` sin índice ni longitud).
* **.NET 6:** reescritura de `FileStream` con mejoras importantes de rendimiento y `RandomAccess` para lecturas por posición sin estado.

-----

## Cuándo sí y cuándo no

**Usa `FileStream` directamente cuando:**

* Trabajas con datos **binarios**: imágenes, formatos propios, registros de tamaño fijo.
* Necesitas leer por partes archivos muy grandes o moverte con `Seek`.
* Encadenas streams (compresión, cifrado).

**Usa algo de más alto nivel cuando:**

* El archivo es **texto**: `StreamReader`/`StreamWriter` o los métodos de `File` (próxima lección).
* Es JSON o XML: `System.Text.Json` y `System.Xml` ya leen y escriben streams por ti.

-----

## Resumen en 5 líneas

1. Un stream es una secuencia de bytes; `FileStream`, `MemoryStream`, `NetworkStream`... se usan igual porque derivan de `Stream`.
2. `new FileStream(ruta, FileMode, FileAccess)`: `FileMode` decide qué pasa si el archivo existe o no; `FileAccess`, qué se permite.
3. Siempre con `using`: libera el archivo aunque haya excepciones.
4. `Read` devuelve cuántos bytes leyó (0 al final); `Write` escribe; el búfer se vuelca al cerrar o con `Flush`.
5. `Position`/`Seek` mueven la posición; `Length` es de solo lectura y `SetLength` cambia el tamaño.

-----

## Para profundizar

<details>
<summary>FileShare: compartir un archivo abierto</summary>

Un cuarto parámetro, `FileShare`, indica qué pueden hacer **otros** procesos mientras tienes el archivo abierto:

```csharp
using var fs = new FileStream("app.log", FileMode.Append, FileAccess.Write, FileShare.Read);
```

Con `FileShare.Read`, otros pueden leer el log mientras tu aplicación escribe. Con `FileShare.None`, nadie más puede abrirlo. Los errores de "el archivo está siendo usado por otro proceso" suelen resolverse eligiendo bien `FileShare`.

</details>

<details>
<summary>Comprimir encadenando streams</summary>

```csharp
using System.IO.Compression;
using System.Text;

byte[] datos = Encoding.UTF8.GetBytes(string.Concat(Enumerable.Repeat("texto repetido ", 1000)));

using (var archivo = File.Create("datos.gz"))
using (var gzip = new GZipStream(archivo, CompressionLevel.Optimal))
{
    gzip.Write(datos);   // lo que escribes en gzip sale comprimido hacia archivo
}

Console.WriteLine($"{datos.Length} bytes → {new FileInfo("datos.gz").Length} bytes");   // 15000 → unos pocos cientos
```

El `GZipStream` no sabe nada de archivos: comprime lo que recibe y lo pasa al stream interno. Se pueden apilar varias capas: archivo → compresión → cifrado.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un stream es una secuencia de bytes que se puede leer y escribir. `FileStream` permite trabajar con archivos y se abre indicando un `FileMode` (crear, abrir, agregar...) y un `FileAccess` (lectura, escritura). Siempre se usa dentro de un `using` para que el archivo se cierre aunque haya errores.

### Respuesta ampliada (semi-senior)

`Stream` es la abstracción base de toda la E/S de .NET y permite componer decoradores (`GZipStream`, `CryptoStream`, `BufferedStream`) sobre fuentes concretas (`FileStream`, `NetworkStream`, `MemoryStream`). `FileStream` usa un búfer interno y vuelca en `Flush`/`Dispose`; `Read` no garantiza llenar el búfer, por lo que se lee en bucle hasta devolver 0. `Seek` y `SetLength` solo funcionan si `CanSeek` es verdadero. `FileMode`, `FileAccess` y `FileShare` controlan la creación, los permisos y el bloqueo compartido. En servidores se usan las APIs asíncronas (`ReadAsync`, `CopyToAsync`) y las sobrecargas con `Memory<byte>`/`Span<byte>` para reducir asignaciones.

### Preguntas frecuentes de seguimiento

**1. ¿Diferencia entre `FileMode.Create` y `FileMode.CreateNew`?**
`Create` crea o sobrescribe; `CreateNew` crea y lanza una excepción si el archivo ya existe.

**2. ¿Por qué hay que usar `using` con un `FileStream`?**
Porque mantiene el archivo abierto y bloqueado; `using` garantiza que se cierre y se libere aunque ocurra una excepción.

**3. ¿Qué devuelve `Read` y por qué importa?**
La cantidad de bytes leídos, que puede ser menor a la pedida; 0 indica el final. Ignorarlo lleva a procesar datos basura.

-----

## Práctica

**Ejercicio 1.** Escribe un método que copie un archivo a otro usando un búfer de 1.024 bytes y un bucle con `Read`/`Write` (sin `CopyTo`), y devuelva la cantidad total de bytes copiados.

<details>
<summary>Solución</summary>

```csharp
File.WriteAllText("origen.txt", new string('x', 5000));
long copiados = CopiarArchivo("origen.txt", "destino.txt");
Console.WriteLine($"Copiados {copiados} bytes");   // 5000

static long CopiarArchivo(string origen, string destino)
{
    using var entrada = new FileStream(origen, FileMode.Open, FileAccess.Read);
    using var salida = new FileStream(destino, FileMode.Create, FileAccess.Write);

    byte[] buffer = new byte[1024];
    long total = 0;
    int leidos;
    while ((leidos = entrada.Read(buffer, 0, buffer.Length)) > 0)
    {
        salida.Write(buffer, 0, leidos);   // solo los bytes válidos
        total += leidos;
    }
    return total;
}
```

</details>

**Ejercicio 2.** ¿Qué tamaño tiene el archivo al final y por qué?

```csharp
using (var fs = new FileStream("prueba.bin", FileMode.Create))
{
    fs.Write(new byte[100]);
    fs.Seek(10, SeekOrigin.Begin);
    fs.Write(new byte[5]);
}
using (var fs = new FileStream("prueba.bin", FileMode.Append))
{
    fs.Write(new byte[20]);
}
Console.WriteLine(new FileInfo("prueba.bin").Length);
```

<details>
<summary>Solución</summary>

`120`. Se escriben 100 bytes; luego se vuelve a la posición 10 y se **sobrescriben** 5 bytes (el tamaño sigue en 100). Al abrir con `Append`, se escribe al final: 100 + 20 = 120.

</details>

-----

## Siguiente lección

[Archivos de texto y la clase File](04-Archivos%20de%20texto%20y%20la%20clase%20File.md)

# Eventos

## En una frase

Un **evento** permite que un objeto (el **publicador**) avise "ocurrió algo" a cualquier cantidad de objetos interesados (los **suscriptores**) sin conocerlos; es un delegado multicast protegido con la palabra `event`, de modo que desde afuera solo se puede **suscribir** (`+=`) o **desuscribir** (`-=`), nunca dispararlo ni borrarlo.

-----

## Antes de empezar

Conviene que ya sepas:

* Delegados, multicast, `+=`, `-=` y `?.Invoke`, de [Delegados](01-Delegados.md).
* Herencia y métodos `protected virtual`, de [Virtual, override y clases abstractas](../04-poo/07-Virtual%20override%20y%20clases%20abstractas.md).
* Interfaces, de [Interfaces](../04-poo/08-Interfaces.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Evento:** miembro que expone un delegado de forma que, desde afuera, solo se puede suscribir y desuscribir.
* **Publicador:** el objeto que declara y dispara el evento.
* **Suscriptor:** el objeto que registra un método para reaccionar al evento.
* **Manejador (*handler*):** el método que se ejecuta cuando ocurre el evento.
* **Disparar (*raise*):** invocar el evento para notificar a los suscriptores.
* **`EventHandler` / `EventHandler<T>`:** delegados estándar de .NET para eventos: `(object? sender, EventArgs e)`.
* **`EventArgs`:** clase base de los datos que acompañan a un evento.
* **Patrón observador (publicador-suscriptor):** diseño en el que un objeto notifica cambios a otros sin depender de ellos.

-----

## El problema

Un sistema de inicio de sesión detecta intentos fallidos. Cuando ocurre uno, varias partes del sistema quieren enterarse: el log de seguridad, el módulo que bloquea la cuenta tras 3 intentos, el que envía un correo de alerta.

Si el servicio de login llama a cada uno directamente:

```csharp
public void LoginFallido(string usuario)
{
    _logSeguridad.Registrar(usuario);
    _bloqueador.ContarIntento(usuario);
    _correo.EnviarAlerta(usuario);
    // ¿y el próximo módulo que quiera enterarse? → hay que modificar esta clase
}
```

El servicio de login conoce a todos los módulos (alto acoplamiento) y hay que modificarlo cada vez que aparece un interesado nuevo.

Podrías exponer un delegado público para que cada uno se agregue, pero entonces cualquiera podría hacer `servicio.AlFallar = null;` y borrar a todos los suscriptores, o hacer `servicio.AlFallar("hacker")` y disparar una alerta falsa. Los eventos resuelven las dos cosas.

-----

## Cómo funciona

```text
           PUBLICADOR                                   SUSCRIPTORES
   ┌──────────────────────────┐   += (suscribirse)   ┌──────────────────┐
   │ ServicioLogin            │◄─────────────────────│ LogSeguridad     │
   │                          │◄─────────────────────│ Bloqueador       │
   │ event LoginFallido ──────┼──┐◄──────────────────│ AlertaCorreo     │
   └──────────────────────────┘  │                   └──────────────────┘
                                 │ Invoke(...) (disparar)
                                 ├──────────────►  LogSeguridad.Registrar(...)
                                 ├──────────────►  Bloqueador.ContarIntento(...)
                                 └──────────────►  AlertaCorreo.Enviar(...)
                                    (en orden, uno tras otro, en el mismo hilo)
```

El publicador no conoce las clases de sus suscriptores: solo guarda una lista de métodos a los que llamar.

### Declarar y disparar un evento

```csharp
public class ServicioLogin
{
    public event Action<string, int>? LoginFallido;   // el evento

    private readonly Dictionary<string, int> _intentos = new();

    public bool IniciarSesion(string usuario, string clave)
    {
        if (clave == "secreto") return true;

        int intento = _intentos.GetValueOrDefault(usuario) + 1;
        _intentos[usuario] = intento;

        LoginFallido?.Invoke(usuario, intento);   // disparar: notifica a todos los suscriptores
        return false;
    }
}
```

* `event` delante de un campo de tipo delegado lo convierte en evento.
* Se dispara con `Evento?.Invoke(...)`: el `?.` evita el `NullReferenceException` cuando nadie está suscrito (un evento sin suscriptores es `null`).
* Solo la clase que declara el evento puede dispararlo.

### Suscribirse y desuscribirse

```csharp
var login = new ServicioLogin();

login.LoginFallido += RegistrarEnLog;                                   // método con nombre
login.LoginFallido += (usuario, n) => { if (n >= 3) Console.WriteLine($"Bloqueando a {usuario}"); };   // lambda

login.IniciarSesion("ana", "x");
login.IniciarSesion("ana", "y");
login.IniciarSesion("ana", "z");

login.LoginFallido -= RegistrarEnLog;     // desuscribirse

static void RegistrarEnLog(string usuario, int intento) =>
    Console.WriteLine($"[log] intento fallido #{intento} de {usuario}");
```

Desde afuera de la clase, `event` solo permite `+=` y `-=`:

```csharp
login.LoginFallido = null;                // error CS0070: solo puede aparecer a la izquierda de += o -=
login.LoginFallido?.Invoke("x", 1);       // error CS0070
```

Esa es toda la diferencia con un delegado público: el evento **protege** la lista de suscriptores.

Los manejadores se ejecutan **en orden de suscripción** y de forma **sincrónica**: `Invoke` no vuelve hasta que todos terminaron. Los eventos no son "asíncronos" ni se ejecutan en paralelo; si un manejador tarda 5 segundos, el publicador espera 5 segundos.

### El patrón estándar de .NET: `EventHandler` y `EventArgs`

En .NET, los eventos siguen una convención: el delegado recibe **quién** disparó el evento (`sender`) y **los datos** del evento (un objeto que deriva de `EventArgs`):

```csharp
public class Temporizador
{
    public event EventHandler? Terminado;                  // sin datos extra

    public void Finalizar() => OnTerminado(EventArgs.Empty);

    protected virtual void OnTerminado(EventArgs e) => Terminado?.Invoke(this, e);
}
```

```csharp
var t = new Temporizador();
t.Terminado += (sender, e) => Console.WriteLine($"¡Terminó! (lo avisó {sender?.GetType().Name})");
t.Finalizar();
```

Las piezas de la convención:

| Pieza | Convención |
| --- | --- |
| Delegado | `EventHandler` (sin datos) o `EventHandler<TArgs>` (con datos) |
| Firma del manejador | `void Manejador(object? sender, TArgs e)` |
| Datos | Una clase que hereda de `EventArgs`, con nombre terminado en `EventArgs` |
| Sin datos | `EventArgs.Empty` |
| Nombre del evento | Verbo o sustantivo: `Click`, `PedidoCreado`, `ValorCambiado` |
| Método que lo dispara | `protected virtual void OnNombreDelEvento(TArgs e)` |

### Datos propios: heredar de `EventArgs`

```csharp
public class LoginFallidoEventArgs : EventArgs
{
    public string Usuario { get; }
    public int Intento { get; }
    public DateTime Fecha { get; } = DateTime.Now;

    public LoginFallidoEventArgs(string usuario, int intento) => (Usuario, Intento) = (usuario, intento);
}

public class ServicioLogin
{
    public event EventHandler<LoginFallidoEventArgs>? LoginFallido;

    protected virtual void OnLoginFallido(LoginFallidoEventArgs e) => LoginFallido?.Invoke(this, e);

    // ... al detectar el fallo:
    // OnLoginFallido(new LoginFallidoEventArgs(usuario, intento));
}
```

```csharp
login.LoginFallido += (sender, e) => Console.WriteLine($"{e.Usuario} falló {e.Intento} veces a las {e.Fecha:HH:mm}");
```

Agrupar los datos en una clase permite agregar información más adelante (por ejemplo, la IP) sin cambiar la firma de todos los manejadores.

### ¿Por qué `protected virtual void OnX(...)`?

Ese método es el **único** lugar donde se dispara el evento, y al ser `protected virtual` una clase derivada puede:

* Dispararlo (las derivadas **no** pueden invocar directamente el evento de la base).
* Sobrescribirlo para agregar lógica antes o después de notificar.

```csharp
public class ServicioLoginAuditado : ServicioLogin
{
    protected override void OnLoginFallido(LoginFallidoEventArgs e)
    {
        Console.WriteLine($"[auditoría] {e.Usuario}");
        base.OnLoginFallido(e);          // sigue notificando a los suscriptores
    }
}
```

### Eventos en interfaces

Una interfaz puede exigir un evento, para que distintos publicadores se usen de la misma forma:

```csharp
public interface IFuenteDeAlertas
{
    event EventHandler<AlertaEventArgs>? Alerta;
}

public class SensorDeHumo : IFuenteDeAlertas
{
    public event EventHandler<AlertaEventArgs>? Alerta;
    public void DetectarHumo() => Alerta?.Invoke(this, new AlertaEventArgs("¡Humo detectado!"));
}

public class EstacionMeteorologica : IFuenteDeAlertas
{
    public event EventHandler<AlertaEventArgs>? Alerta;
    public void DetectarTormenta() => Alerta?.Invoke(this, new AlertaEventArgs("Tormenta en camino"));
}

public class AlertaEventArgs : EventArgs
{
    public string Mensaje { get; }
    public AlertaEventArgs(string mensaje) => Mensaje = mensaje;
}
```

Una central puede suscribirse a cualquier `IFuenteDeAlertas` sin conocer su clase concreta.

### Desuscribirse: evitar fugas de memoria

El publicador guarda una referencia a cada suscriptor (dentro del delegado). Mientras el publicador viva, **el suscriptor no puede ser recolectado** por el GC, aunque nadie más lo use:

```csharp
class Ventana
{
    public Ventana(ServicioLogin login) => login.LoginFallido += AlFallar;   // el servicio ahora referencia a la ventana
    private void AlFallar(object? s, LoginFallidoEventArgs e) { /* ... */ }
}
```

Si el `ServicioLogin` vive toda la aplicación y creas y cierras cientos de ventanas, todas quedan en memoria. La solución es desuscribirse cuando el suscriptor ya no se necesita (por ejemplo, en `Dispose`):

```csharp
class Ventana : IDisposable
{
    private readonly ServicioLogin _login;
    public Ventana(ServicioLogin login) { _login = login; _login.LoginFallido += AlFallar; }
    public void Dispose() => _login.LoginFallido -= AlFallar;
    private void AlFallar(object? s, LoginFallidoEventArgs e) { /* ... */ }
}
```

-----

## Ejemplo completo

Un carrito que publica eventos y varios suscriptores independientes:

```csharp
var carrito = new Carrito();
var resumen = new ResumenEnPantalla(carrito);
var promo = new MotorDePromociones();

carrito.ProductoAgregado += promo.Evaluar;
carrito.ProductoAgregado += (_, e) => Console.WriteLine($"  [analytics] +{e.Producto} ({e.Precio:N2})");
carrito.Vaciado += (_, _) => Console.WriteLine("  [analytics] carrito vaciado");

carrito.Agregar("Teclado", 150m);
carrito.Agregar("Mouse", 60m);
carrito.Agregar("Monitor", 800m);

resumen.Dispose();                       // la pantalla se cierra y se desuscribe
carrito.Vaciar();

class ProductoAgregadoEventArgs : EventArgs
{
    public string Producto { get; }
    public decimal Precio { get; }
    public decimal TotalCarrito { get; }

    public ProductoAgregadoEventArgs(string producto, decimal precio, decimal total)
        => (Producto, Precio, TotalCarrito) = (producto, precio, total);
}

class Carrito
{
    private readonly List<(string Nombre, decimal Precio)> _items = new();

    public event EventHandler<ProductoAgregadoEventArgs>? ProductoAgregado;
    public event EventHandler? Vaciado;

    public decimal Total => _items.Sum(i => i.Precio);

    public void Agregar(string nombre, decimal precio)
    {
        _items.Add((nombre, precio));
        OnProductoAgregado(new ProductoAgregadoEventArgs(nombre, precio, Total));
    }

    public void Vaciar()
    {
        _items.Clear();
        OnVaciado(EventArgs.Empty);
    }

    protected virtual void OnProductoAgregado(ProductoAgregadoEventArgs e) => ProductoAgregado?.Invoke(this, e);
    protected virtual void OnVaciado(EventArgs e) => Vaciado?.Invoke(this, e);
}

class ResumenEnPantalla : IDisposable
{
    private readonly Carrito _carrito;

    public ResumenEnPantalla(Carrito carrito)
    {
        _carrito = carrito;
        _carrito.ProductoAgregado += Actualizar;
        _carrito.Vaciado += AlVaciar;
    }

    private void Actualizar(object? sender, ProductoAgregadoEventArgs e) =>
        Console.WriteLine($"[pantalla] {e.Producto} agregado. Total: {e.TotalCarrito:N2}");

    private void AlVaciar(object? sender, EventArgs e) => Console.WriteLine("[pantalla] carrito vacío");

    public void Dispose()
    {
        _carrito.ProductoAgregado -= Actualizar;
        _carrito.Vaciado -= AlVaciar;
    }
}

class MotorDePromociones
{
    public void Evaluar(object? sender, ProductoAgregadoEventArgs e)
    {
        if (e.TotalCarrito > 1000m)
            Console.WriteLine("  [promo] ¡Superaste 1,000! Envío gratis");
    }
}
```

Salida:

```text
[pantalla] Teclado agregado. Total: 150.00
  [analytics] +Teclado (150.00)
[pantalla] Mouse agregado. Total: 210.00
  [analytics] +Mouse (60.00)
[pantalla] Monitor agregado. Total: 1,010.00
  [promo] ¡Superaste 1,000! Envío gratis
  [analytics] +Monitor (800.00)
  [analytics] carrito vaciado
```

`Carrito` no conoce a ninguno de sus suscriptores. Después del `Dispose`, la pantalla ya no recibe el evento `Vaciado`. Fíjate en el orden: los manejadores se ejecutan en el orden en que se suscribieron (la pantalla se suscribió primero, en su constructor).

-----

## Errores comunes

**1. Disparar o asignar el evento desde fuera de la clase.**
Qué pasa: `error CS0070: The event 'Carrito.Vaciado' can only appear on the left hand side of += or -= (except when used from within the type 'Carrito')`.
Por qué: `event` solo permite suscribirse y desuscribirse desde afuera.
Arreglo: expón un método público que dispare el evento internamente (o un `protected virtual OnX` para las derivadas).

**2. Disparar sin comprobar `null`.**
Qué pasa: `NullReferenceException` cuando no hay suscriptores.
Por qué: un evento sin suscriptores es `null`.
Arreglo: `Evento?.Invoke(this, e)`.

**3. Olvidar desuscribirse.**
Qué pasa: fuga de memoria y manejadores que se siguen ejecutando para objetos "cerrados".
Por qué: el publicador mantiene una referencia a cada suscriptor.
Arreglo: `-=` en `Dispose` o cuando el suscriptor deja de necesitar el evento.

**4. Suscribirse con una lambda que después no puedes quitar.**
Qué pasa: `evento -= (s, e) => ...` no quita nada.
Por qué: cada lambda es un objeto distinto.
Arreglo: suscribe un método con nombre o guarda la lambda en una variable.

**5. Suscribirse dos veces.**
Qué pasa: el manejador se ejecuta dos veces por evento.
Por qué: `+=` no comprueba duplicados.
Arreglo: asegúrate de suscribirte una sola vez (por ejemplo, en el constructor).

**6. Creer que los eventos son asíncronos.**
Qué pasa: un manejador lento bloquea al publicador; una excepción en un manejador corta a los siguientes y llega a quien disparó el evento.
Por qué: `Invoke` ejecuta los manejadores uno tras otro, en el mismo hilo.
Arreglo: mantén los manejadores rápidos; si necesitas trabajo pesado, delega en una tarea o una cola.

-----

## Según la versión de C#

* **C# 1:** eventos, `EventHandler` y `EventArgs`.
* **.NET 2.0:** `EventHandler<TEventArgs>` genérico.
* **.NET 4.5:** `TEventArgs` ya no necesita heredar de `EventArgs` (aunque sigue siendo la convención).
* **C# 6:** `?.Invoke` para disparar en una línea de forma segura.
* **C# 8:** `object? sender` con tipos de referencia que aceptan null.
* **C# 9:** descartes en las lambdas: `(_, _) => ...` cuando no usas `sender` ni `e`.

-----

## Cuándo sí y cuándo no

**Usa eventos cuando:**

* Un objeto debe notificar cambios a **cero o más** interesados que no conoce: interfaces gráficas (clics, cambios de valor), cambios de estado en el dominio, integraciones desacopladas.

**Prefiere otra cosa cuando:**

* Hay **un único** destinatario que el objeto necesita: pásale un delegado o una interfaz.
* Necesitas comunicación entre servicios o procesos, persistencia de los mensajes o procesamiento asíncrono: usa colas de mensajes, `Channel<T>` o un mediador.

-----

## Resumen en 5 líneas

1. `public event EventHandler<TArgs>? Nombre;` declara un evento; solo la clase que lo declara puede dispararlo.
2. Desde afuera solo se puede `+=` (suscribirse) y `-=` (desuscribirse); eso lo diferencia de un delegado público.
3. Patrón estándar: `(object? sender, TArgs e)`, datos en una clase `...EventArgs` y disparo en `protected virtual void OnX(TArgs e)`.
4. Se dispara con `Evento?.Invoke(this, e)`: los manejadores se ejecutan en orden y de forma sincrónica.
5. Desuscríbete cuando el suscriptor ya no se necesita para evitar fugas de memoria.

-----

## Para profundizar

<details>
<summary>INotifyPropertyChanged</summary>

La interfaz más conocida basada en eventos. La usan WPF, MAUI y otros frameworks de interfaz para actualizar la pantalla cuando cambia un dato:

```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

class Persona : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    public string Nombre
    {
        get;
        set
        {
            if (field == value) return;
            field = value;
            OnPropertyChanged();
        }
    } = "";

    protected void OnPropertyChanged([CallerMemberName] string? propiedad = null) =>
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propiedad));
}
```

`[CallerMemberName]` completa automáticamente el nombre de la propiedad que llamó al método. La palabra clave `field` (C# 14) evita declarar el campo de respaldo.

</details>

<details>
<summary>Accesores add y remove</summary>

Un evento "tipo campo" (`public event EventHandler? X;`) hace que el compilador genere un campo delegado privado y dos accesores `add` y `remove` seguros para hilos. Puedes escribirlos a mano para controlar la suscripción:

```csharp
private EventHandler? _cambio;
public event EventHandler? Cambio
{
    add { Console.WriteLine("Alguien se suscribió"); _cambio += value; }
    remove { _cambio -= value; }
}
```

Se usa para delegar el evento en otro objeto o para implementar eventos débiles (*weak events*) que no impiden la recolección del suscriptor.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Un evento permite que un objeto notifique a otros que algo ocurrió, siguiendo el patrón publicador-suscriptor. Se declara con la palabra `event` sobre un delegado, normalmente `EventHandler` o `EventHandler<T>`. Los suscriptores se agregan con `+=` y se quitan con `-=`, y solo la clase que declara el evento puede dispararlo, normalmente con `?.Invoke(this, e)`.

### Respuesta ampliada (semi-senior)

Un evento es un delegado multicast encapsulado: el compilador genera un campo privado y accesores `add`/`remove` (seguros para hilos), por lo que desde fuera no se puede asignar ni invocar. La convención de .NET es `EventHandler<TEventArgs>` con `sender` y unos argumentos inmutables, disparados desde un `protected virtual OnX` para permitir la extensión en las derivadas. La invocación es sincrónica y secuencial: una excepción en un manejador corta la cadena. Como el publicador referencia a sus suscriptores, olvidar desuscribirse es una causa clásica de fugas de memoria en aplicaciones de larga vida; se resuelve con `IDisposable`, eventos débiles o suscripciones con alcance limitado.

### Preguntas frecuentes de seguimiento

**1. ¿Qué diferencia hay entre un evento y un delegado público?**
Con un evento, desde afuera solo se puede `+=` y `-=`; un delegado público puede ser reasignado (borrando a todos los suscriptores) o invocado por cualquiera.

**2. ¿Los eventos se ejecutan en paralelo?**
No. Los manejadores se ejecutan uno tras otro, en el hilo que dispara el evento.

**3. ¿Cómo provoca un evento una fuga de memoria?**
El publicador guarda referencias a los suscriptores; si vive más que ellos y no se desuscriben, el GC no puede liberarlos.

-----

## Práctica

**Ejercicio 1.** Crea una clase `Termostato` con una propiedad `Temperatura` y un evento `TemperaturaAlta` (con un `EventArgs` propio que incluya la temperatura) que se dispare cuando la temperatura supere 30. Suscribe dos manejadores: uno que imprima una alerta y otro que encienda un ventilador (un mensaje).

<details>
<summary>Solución</summary>

```csharp
var termostato = new Termostato();
termostato.TemperaturaAlta += (_, e) => Console.WriteLine($"¡Alerta! {e.Temperatura} °C");
termostato.TemperaturaAlta += (_, e) => Console.WriteLine("Ventilador encendido");

termostato.Temperatura = 25;   // nada
termostato.Temperatura = 33;   // ¡Alerta! 33 °C / Ventilador encendido

class TemperaturaEventArgs : EventArgs
{
    public double Temperatura { get; }
    public TemperaturaEventArgs(double t) => Temperatura = t;
}

class Termostato
{
    private double _temperatura;
    public event EventHandler<TemperaturaEventArgs>? TemperaturaAlta;

    public double Temperatura
    {
        get => _temperatura;
        set
        {
            _temperatura = value;
            if (value > 30) OnTemperaturaAlta(new TemperaturaEventArgs(value));
        }
    }

    protected virtual void OnTemperaturaAlta(TemperaturaEventArgs e) => TemperaturaAlta?.Invoke(this, e);
}
```

</details>

**Ejercicio 2.** Este código no compila. ¿Por qué? Corrígelo sin quitar la palabra `event`.

```csharp
var boton = new Boton();
boton.Click += (_, _) => Console.WriteLine("Clic");
boton.Click(boton, EventArgs.Empty);

class Boton
{
    public event EventHandler? Click;
}
```

<details>
<summary>Solución</summary>

Desde fuera de `Boton` no se puede invocar el evento (CS0070). La clase debe ofrecer un método que lo dispare:

```csharp
var boton = new Boton();
boton.Click += (_, _) => Console.WriteLine("Clic");
boton.Presionar();

class Boton
{
    public event EventHandler? Click;
    public void Presionar() => OnClick(EventArgs.Empty);
    protected virtual void OnClick(EventArgs e) => Click?.Invoke(this, e);
}
```

</details>

-----

## Siguiente lección

Terminaste el módulo de delegados y eventos. Continúa con [Asincronía y archivos](../10-asincronia-y-archivos/README.md).

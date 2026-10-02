# Glosario

Las palabras que aparecen en las lecciones de esta carpeta, explicadas de forma simple. Están en orden alfabético.

-----

**Callback.** Método que pasas a otro para que lo llame cuando corresponda.

**Delegado.** Tipo cuyas variables guardan referencias a métodos con una firma determinada.

**Delegado multicast.** Delegado que apunta a varios métodos y los ejecuta todos, en orden, al invocarlo.

**Disparar (*raise*).** Invocar un evento para notificar a sus suscriptores.

**Evento.** Miembro que expone un delegado de forma que, desde afuera, solo se puede suscribir (`+=`) o desuscribir (`-=`).

**`EventArgs`.** Clase base de los datos que acompañan a un evento. `EventArgs.Empty` representa "sin datos".

**`EventHandler` / `EventHandler<T>`.** Delegados estándar de .NET para eventos, con la firma `(object? sender, T e)`.

**Fuga de memoria por eventos.** Un suscriptor que no se desuscribe queda referenciado por el publicador y el GC no puede liberarlo.

**Grupo de métodos (*method group*).** El nombre de un método sin paréntesis, usado como valor para asignarlo a un delegado.

**Invocar.** Ejecutar el método al que apunta un delegado: `d(args)` o `d.Invoke(args)`.

**Lista de invocación.** Los métodos de un delegado multicast. Se obtiene con `GetInvocationList()`.

**Manejador (*handler*).** Método que se ejecuta cuando ocurre un evento.

**Método anónimo.** Método sin nombre escrito con `delegate (...) { ... }`. Las lambdas son su reemplazo moderno.

**`OnX`.** Convención para el método `protected virtual` que dispara el evento `X`, para que las clases derivadas puedan extenderlo.

**Patrón observador (publicador-suscriptor).** Diseño en el que un objeto notifica cambios a otros sin depender de ellos.

**Publicador.** Objeto que declara y dispara un evento.

**`sender`.** Primer parámetro de un manejador estándar: el objeto que disparó el evento.

**Suscriptor.** Objeto que registra un manejador en un evento.

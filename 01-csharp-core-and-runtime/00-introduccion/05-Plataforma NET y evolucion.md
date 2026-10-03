# Plataforma .NET y su evolución

## En una frase
.NET es una plataforma abierta y multiplataforma formada por runtime, bibliotecas, compiladores, SDK y modelos de aplicación; no es lo mismo que .NET Framework ni que C#.

-----

## Antes de empezar
Conviene que ya sepas: [Qué es C# y .NET](01-Que%20es%20CSharp%20y%20.NET.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)): **runtime:** entorno que ejecuta código; **workload:** tipo de aplicación y herramientas asociadas; **LTS:** versión con soporte extendido; **STS:** versión con soporte estándar.

-----

## El problema
Decir «uso .NET Core» para cualquier proyecto moderno mezcla tres generaciones distintas. Esa confusión lleva a instalar paquetes equivocados, elegir plantillas antiguas o creer que ASP.NET Core y .NET son el mismo producto.

```text
C#                 lenguaje
.NET                plataforma moderna
.NET Framework      implementación histórica para Windows
ASP.NET Core        stack web construido sobre .NET
```

-----

## Cómo funciona
### 1. Los componentes
| Componente | Responsabilidad |
| --- | --- |
| Runtime | Ejecutar código y administrar memoria |
| Bibliotecas | Proporcionar APIs reutilizables |
| Compiladores | Traducir lenguajes como C# a CIL |
| SDK | Crear, compilar, probar y publicar |
| Stacks | ASP.NET Core, Windows Forms, WPF, MAUI |

### 2. La evolución del nombre
`.NET Framework` nació orientado a Windows. `.NET Core` apareció como una implementación abierta, modular y multiplataforma. Desde .NET 5, el producto moderno se llama simplemente `.NET`; por eso no existen productos llamados «.NET Core 8» o «.NET Core 10».

### 3. Elegir entre .NET y .NET Framework
Usa .NET para desarrollo nuevo. .NET Framework sigue siendo razonable al mantener sistemas Windows que dependen de Web Forms, tecnologías heredadas o bibliotecas sin alternativa compatible.

### 4. Tipos de aplicaciones
.NET permite crear consola, servicios, APIs, sitios con renderizado de servidor, aplicaciones web interactivas, escritorio, móviles, juegos, bibliotecas y herramientas. Que compartan plataforma no significa que compartan interfaz, ciclo de vida o modelo de despliegue.

### 5. Soporte
.NET publica una versión principal cada año. Las versiones pares son LTS y las impares STS; ambas reciben correcciones, pero durante períodos distintos. Estar en una versión soportada también exige instalar sus parches.

### 6. Uso empresarial
«Empresarial» no es una edición especial de .NET. Describe exigencias como seguridad, observabilidad, automatización, pruebas, soporte, integración y operación mantenible.

-----

## Ejemplo completo
Este programa muestra la plataforma objetivo y el runtime real:

```csharp
Console.WriteLine($"TFM: {AppContext.TargetFrameworkName}");
Console.WriteLine($"Runtime: {System.Runtime.InteropServices.RuntimeInformation.FrameworkDescription}");
```

Salida con un proyecto `net10.0` ejecutado sobre .NET 10:

```text
TFM: .NETCoreApp,Version=v10.0
Runtime: .NET 10.0.0
```

El nombre interno del TFM conserva `.NETCoreApp` por compatibilidad; eso no cambia el nombre comercial actual: .NET.

-----

## Errores comunes
**1. Llamar .NET Core a .NET 10.** Qué pasa: se mezclan documentación y paquetes de generaciones distintas. Por qué: el nombre cambió desde .NET 5. Arreglo: usa «.NET 10».

**2. Afirmar portabilidad total.** Qué pasa: una app usa APIs o dependencias nativas incompatibles. Por qué: la plataforma es multiplataforma, pero cada dependencia puede no serlo. Arreglo: verifica el sistema y RID objetivo.

**3. Migrar por moda.** Qué pasa: el costo supera el beneficio. Por qué: algunos sistemas heredados dependen legítimamente de .NET Framework. Arreglo: inventaría tecnologías y riesgos antes de decidir.

-----

## Según la versión de .NET
- **.NET Framework 1.0 (2002):** primera versión pública de la plataforma clásica.
- **.NET Core 1.0 (2016):** línea abierta y multiplataforma.
- **.NET 5 (2020):** unificó el nombre del producto moderno.
- **.NET 10 (2025):** versión LTS de referencia de esta wiki.

-----

## Cuándo sí y cuándo no
**Usa .NET moderno cuando:** creas software nuevo, necesitas soporte multiplataforma o despliegues actuales. **Conserva .NET Framework cuando:** una aplicación existente depende de tecnología exclusiva de Framework y la migración no está justificada.

-----

## Resumen en 5 líneas
1. C# es un lenguaje; .NET es la plataforma que lo compila y ejecuta.
2. .NET Framework es la implementación histórica orientada a Windows.
3. .NET Core pasó a llamarse .NET desde la versión 5.
4. ASP.NET Core, MAUI y los stacks de escritorio resuelven modelos diferentes.
5. La arquitectura empresarial depende de prácticas operativas, no de una edición del producto.

-----

## Para profundizar
<details><summary>¿Qué significa multiplataforma?</summary>
El runtime y muchas bibliotecas funcionan en varios sistemas. Una aplicación deja de ser portable cuando depende de COM, Registro de Windows, rutas específicas, bibliotecas nativas o un stack exclusivo de un sistema.
</details>

-----

## En entrevista
### Respuesta corta (junior)
.NET es una plataforma para crear y ejecutar aplicaciones; C# es uno de sus lenguajes. .NET Framework es la línea histórica para Windows y .NET es la línea moderna multiplataforma.

### Respuesta ampliada (semi-senior)
.NET reúne runtime, bibliotecas, compiladores, SDK y stacks. Para software nuevo se prefiere .NET moderno; Framework se mantiene cuando hay dependencias heredadas. La elección se basa en compatibilidad, soporte y costo de migración.

### Preguntas frecuentes de seguimiento
**1. ¿ASP.NET Core es otro runtime?** No, es un stack web que se ejecuta sobre .NET.

**2. ¿LTS significa que puedo ignorar parches?** No. Debes mantenerte en un parche soportado.

-----

## Práctica
**Ejercicio 1.** Clasifica C#, .NET, ASP.NET Core y Entity Framework Core como lenguaje, plataforma o stack/biblioteca.

<details><summary>Solución</summary>C# es lenguaje; .NET es plataforma; ASP.NET Core es stack web; EF Core es biblioteca de acceso a datos.</details>

-----

## Siguiente lección
[CLR, compilación y sistema de tipos](06-CLR%20compilacion%20y%20sistema%20de%20tipos.md)

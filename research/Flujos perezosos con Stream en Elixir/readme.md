# Investigación: Flujos Perezosos (*Lazy Streams*) con `Stream` en Elixir

## 📄 Resumen / Abstract
El módulo `Stream` de Elixir proporciona una abstracción fundamental para la manipulación y procesamiento perezoso (*lazy evaluation*) de colecciones y fuentes de datos potencialmente infinitas o de gran volumen. A diferencia del módulo `Enum`, que opera de manera ansiosa (*eager*) generando colecciones intermedias en memoria en cada paso de una canalización (*pipeline*), `Stream` difiere la ejecución hasta que los datos son explícitamente consumidos. Esta investigación analiza la arquitectura interna, casos de uso óptimos, impacto en el rendimiento y comparativa estructural entre el procesamiento ansioso y perezoso en la Virtual Machine de Erlang (BEAM).

---

## 🎯 Objetivos de la Investigación
1. **Comprender el concepto de evaluación perezosa (*lazy evaluation*)** y su implementación dentro del paradigma funcional de Elixir.
2. **Analizar la diferencia técnica entre `Enum` y `Stream`**, evaluando el consumo de memoria, latencia y tiempo de CPU.
3. **Estudiar las funciones clave del módulo `Stream`** (`Stream.map/2`, `Stream.filter/2`, `Stream.resource/3`, `Stream.unfold/2`, etc.).
4. **Analizar la integración de `Stream` con I/O y fuentes de datos externas** (archivos de gran tamaño, conectores de red, eventos en tiempo real).
5. **Proporcionar un marco de decisiones prácticos** sobre cuándo utilizar evaluación ansiosa vs. perezosa.

---

## 📐 Diagrama Conceptual de Arquitectura

```
[ Fuente de Datos / Colección ]
               │
               ▼
   ┌───────────────────────┐
   │    Stream.map(...)    │ ───► Representación de función (Composición perezosa)
   └───────────────────────┘
               │
               ▼
   ┌───────────────────────┐
   │   Stream.filter(...)  │ ───► No realiza cómputo inmediato ni crea listas
   └───────────────────────┘
               │
               ▼
   ┌───────────────────────┐
   │    Enum.to_list()     │ ───► MÓDULO REDUCTOR (Desencadena el flujo de datos)
   └───────────────────────┘
               │
               ▼
    [ Resultado Final ]
```

---

## 📑 Estructura Detallada del Trabajo de Investigación

### 1. Introducción
* **1.1 Background:** La necesidad de procesar colecciones masivas o infinitas en sistemas concurrentes.
* **1.2 Planteamiento del Problema:** El costo computacional y de memoria de las operaciones ansiosas con `Enum`.
* **1.3 Justificación:** Por qué el diseño de Elixir y la BEAM favorece los flujos perezosos para mantener baja la huella de memoria.

### 2. Fundamentos Teóricos: Evaluación Ansiosa vs. Perezosa
* **2.1 Concepto de *Eager Evaluation* (`Enum`):**
  * Creación de estructuras intermedias en el heap.
  * Complejidad espacial $O(N \cdot K)$ donde $K$ es el número de pasos en el pipeline.
* **2.2 Concepto de *Lazy Evaluation* (`Stream`):**
  * Composición de funciones de transformación (Estructuras `Enumerable`).
  * Procesamiento elemento por elemento (Flujo *Pipelined*).
  * Complejidad espacial reducida a $O(1)$ o $O(	ext{tamaño de batch})$.

### 3. Anatomía del Módulo `Stream` en Elixir
* **3.1 Funciones de Transformación Comunes:**
  * `Stream.map/2`, `Stream.filter/2`, `Stream.reject/2`, `Stream.take/2`.
* **3.2 Generación de Flujos Infinitos:**
  * `Stream.iterate/2`
  * `Stream.repeatedly/1`
  * `Stream.unfold/2`
* **3.3 Manejo de Recursos Externos:**
  * `Stream.resource/3` (Apertura, consumo y liberación segura de recursos como archivos o DB Sockets).

### 4. Análisis Comparativo: `Enum` vs `Stream`

#### 📊 Cuadro Comparativo de Características

| Criterio | `Enum` (Eager) | `Stream` (Lazy) |
| :--- | :--- | :--- |
| **Evaluación** | Inmediata (Paso por paso) | Diferida (Solo al consumir) |
| **Colecciones Intermedias** | Sí, se crea una nueva lista por cada función | No, recombina las transformaciones |
| **Uso de Memoria** | Proporcional al número total de elementos ($O(N)$) | Constante o muy reducido ($O(1)$) |
| **Soporte para Colecciones Infinitas** | ❌ No (Causa *Stack Overflow* o agotamiento de memoria) | ✅ Sí |
| **Overhead de CPU en listas pequeñas** | Menor (Sin abstracción adicional) | Ligeramente mayor (Debido al wrapper de funciones) |
| **Caso de Uso Principal** | Colecciones pequeñas a medianas cargadas en memoria | Archivos grandes, datos de red, streams infinitos |

---

## 🎨 Diagrama de Secuencia: Flujo de Ejecución

```
Usuario               Stream Module              Enum / Reductor
   │                        │                           │
   │  Stream.map(fn)        │                           │
   │───────────────────────►│                           │
   │                        │ Returns %Stream{} struct  │
   │                        │ (No compute)              │
   │                        │                           │
   │  Stream.filter(fn)     │                           │
   │───────────────────────►│                           │
   │                        │ Returns updated %Stream{} │
   │                        │                           │
   │  Enum.take(5)          │                           │
   │───────────────────────────────────────────────────►│
   │                        │                           │ Pull elements 1-by-1
   │                        │◄──────────────────────────│
   │                        │ Compute map -> filter     │
   │                        │──────────────────────────►│ Stops after 5 elements!
```

---

## 💻 Ejemplos de Código y Benchmarks

### Ejemplo 1: Colecciones Grandes (Uso de Memoria)

```elixir
# Enfoque Ansioso (Enum) - Genera 3 listas intermedias de 10,000,000 de elementos en RAM
1..10_000_000
|> Enum.map(&(&1 * 2))
|> Enum.filter(&(&1 > 100))
|> Enum.take(5)

# Enfoque Perezoso (Stream) - Pasa los elementos uno a uno. Se detiene tras obtener 5.
1..10_000_000
|> Stream.map(&(&1 * 2))
|> Stream.filter(&(&1 > 100))
|> Enum.take(5)
```

---

## 🖼️ Ilustraciones y Capturas de Pantalla

> **Espacio para imagen:** *Inserte aquí una captura o diagrama del consumo de memoria usando Observer o Benchee.*

![Consumo de Memoria: Enum vs Stream](docs/images/memory_comparison_placeholder.png)

*Figura 1: Comparativa visual del consumo de memoria en la BEAM durante la ejecución de pipelines de procesamiento masivo.*

---

## 🔬 Caso Práctico: Procesamiento de Archivos Logs Gigantes

```elixir
defmodule LogProcessor do
  def count_error_events(file_path) do
    file_path
    |> File.stream!() # Lee el archivo línea por línea en lugar de cargarlo todo a RAM
    |> Stream.map(&String.trim/1)
    |> Stream.filter(&String.contains?(&1, "[ERROR]"))
    |> Enum.reduce(0, fn _line, acc -> acc + 1 end)
  end
end
```

---

## 🖼️ Diagrama de Procesamiento de Archivos

> **Espacio para imagen:** *Inserte aquí el diagrama de flujo del procesamiento paso a paso de un archivo con `File.stream!/1`.*

![Flujo de File Stream](docs/images/file_stream_flow_placeholder.png)

*Figura 2: Diagrama de flujo mostrando la lectura en bloques/líneas con File.stream!.*

---

## 📈 Resultados y Discusión
1. **Benchmark de CPU vs Memoria:** `Stream` añade un pequeño overhead de invocación de funciones, por lo que en colecciones pequeñas (< 1,000 elementos) `Enum` suele ser más rápido. Sin embargo, para dataset grandes o pipelines extensos, `Stream` supera radicalmente en eficiencia de memoria.
2. **Evaluación Cortocircuitada (*Short-circuiting*):** Operaciones como `take`, `find` o `any?` se benefician drásticamente de `Stream` al detener la lectura en el momento justo.

---

## 📚 Conclusiones y Recomendaciones
* Usa **`Enum`** cuando:
  * La colección sea pequeña o de tamaño acotado.
  * El rendimiento estricto de CPU sea más prioritario que el uso de RAM en colecciones pequeñas.
* Usa **`Stream`** cuando:
  * Trabajes con datos provenientes de archivos de gran tamaño o conectores I/O.
  * Generes secuencias infinitas o con final incierto.
  * Tengas una cadena larga de transformaciones (`map`, `filter`, `flat_map`) y desees evitar listas intermedias.

---

## 📖 Referencias y Recursos Adicionales
* [Documentación Oficial de Elixir - Módulo Stream](https://hexdocs.pm/elixir/Stream.html)
* McCord, C., Tate, B., & Valim, J. (2018). *Programming Elixir ≥ 1.6*. Pragmatic Bookshelf.
* Erlang/OTP Documentation - *Memory Management in the BEAM*.

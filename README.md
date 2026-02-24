# Resumen Teórico — Red Social Empresarial
### Programación II · UADE · 2026 - 1C · Grupo 1

---

## Índice

1. [Descripción del sistema](#1-descripción-del-sistema)
2. [Arquitectura en capas](#2-arquitectura-en-capas)
3. [TDAs básicos implementados](#3-tdas-básicos-implementados)
4. [TDAs de aplicación](#4-tdas-de-aplicación)
5. [ABB — Árbol Binario de Búsqueda](#5-abb--árbol-binario-de-búsqueda)
6. [StaticClientesTDA — la pieza central](#6-staticclientestda--la-pieza-central)
7. [Tabla de complejidades por método](#7-tabla-de-complejidades-por-método)
8. [Elecciones de diseño y por qué](#8-elecciones-de-diseño-y-por-qué)
9. [Patrones de diseño aplicados](#9-patrones-de-diseño-aplicados)
10. [Persistencia y entrada/salida](#10-persistencia-y-entradासalida)
11. [Preguntas frecuentes del profesor](#11-preguntas-frecuentes-del-profesor)

---

## 1. Descripción del sistema

El sistema simula una **red social empresarial** de consola. Los usuarios (clientes) pueden:

- Registrarse con nombre y puntuación (scoring).
- **Seguir** directamente a otros clientes (sin aprobación, máximo 2 seguidos por usuario).
- **Enviar solicitudes de amistad** que el destinatario debe aceptar o rechazar.
- Ver las **distancias** entre usuarios (cuántos saltos los separan) por seguimiento o amistad.
- Explorar la **red extendida** de un usuario con BFS + ABB.
- Deshacer la última acción (historial LIFO).
- Persistir el estado en `clientes.json` y exportar el historial a `acciones.csv`.

---

## 2. Arquitectura en capas

```
┌─────────────────────────────────────────────────────────────┐
│  UI / Presentación                                          │
│  utils/Menu · MenuBuilder · MenuRedSocial                   │
├─────────────────────────────────────────────────────────────┤
│  Servicios (Fachadas)                                       │
│  GestorClientes · HistorialAcciones · ColaSolicitudesSeg.   │
├─────────────────────────────────────────────────────────────┤
│  TDAs de Aplicación (interfaces)                            │
│  ClientesTDA · HistorialAccionesTDA · SolicitudesSeg.TDA   │
│         Implementaciones: Static*TDA                        │
├─────────────────────────────────────────────────────────────┤
│  TDAs Básicos (interfaces + implementaciones)               │
│  Pila · Cola · Conjunto · DiccionarioSimple/Multiple · Grafo│
├─────────────────────────────────────────────────────────────┤
│  Modelo de Dominio                                          │
│  Cliente · Accion · SolicitudSeguimiento                    │
├─────────────────────────────────────────────────────────────┤
│  Persistencia                                               │
│  CargadorClientesJson · GuardadorClientesJson               │
│  ExportadorAccionesCsv                                      │
└─────────────────────────────────────────────────────────────┘
```

### Paquetes principales

| Paquete | Responsabilidad |
|---|---|
| `model` | POJOs inmutables del dominio |
| `basic_tdas/tda` | Interfaces de estructuras elementales |
| `basic_tdas/implementation` | Implementaciones estáticas y dinámicas |
| `basic_tdas/entities` | Nodos internos (Nodo, NodoGrafo, NodoArista) |
| `tda` | Interfaces de TDAs de negocio |
| `implementation` | Implementaciones de TDAs de negocio |
| `EstructuraABB` | Árbol Binario de Búsqueda genérico |
| `services` | Fachadas de negocio + I/O |
| `utils` | Sistema de menús (Menu/MenuBuilder/MenuRedSocial) |

---

## 3. TDAs básicos implementados

### 3.1 PilaTDA — Pila (LIFO)

**Propósito:** Estructura LIFO (Last In, First Out). Se usa para el historial de acciones (deshacer).

**Operaciones:**

| Método | Descripción | Static | Dynamic |
|---|---|---|---|
| `InicializarPila()` | Prepara la estructura | O(1) | O(1) |
| `Apilar(x)` | Inserta al tope | O(1) | O(1) |
| `Desapilar()` | Elimina el tope | O(1) | O(1) |
| `Tope()` | Consulta el tope sin eliminarlo | O(1) | O(1) |
| `PilaVacia()` | Indica si está vacía | O(1) | O(1) |

**Implementación estática:** arreglo de tamaño fijo + índice. El tope está en `a[indice-1]`.  
**Implementación dinámica:** lista enlazada simple con puntero al primer nodo.

**Por qué LIFO para el historial:** El "deshacer" siempre revierte la **última** acción realizada, que es exactamente el tope de la pila.

---

### 3.2 ColaTDA — Cola (FIFO)

**Propósito:** Estructura FIFO (First In, First Out). Se usa para la cola de solicitudes de amistad.

**Operaciones:**

| Método | Descripción | Static | Dynamic |
|---|---|---|---|
| `InicializarCola()` | Prepara la estructura | O(1) | O(1) |
| `Acolar(x)` | Inserta al final | O(1) | O(1) |
| `Desacolar()` | Elimina el primero | **O(n)** (desplaza) | O(1) |
| `Primero()` | Consulta el primero | O(1) | O(1) |
| `ColaVacia()` | Indica si está vacía | O(1) | O(1) |

**Diferencia clave entre implementaciones:**
- `StaticColaTDA.Desacolar()` es **O(n)** porque debe desplazar todo el arreglo hacia la izquierda.
- `DynamicColaTDA.Desacolar()` es **O(1)** porque solo actualiza el puntero `primero`.
- La implementación dinámica mantiene un puntero `ultimo` para insertar en O(1) sin recorrer la lista.

**Por qué FIFO para solicitudes:** Las solicitudes de amistad deben procesarse en el orden en que llegaron; la primera solicitud enviada es la primera en ser procesada.

---

### 3.3 ConjuntoTDA — Conjunto

**Propósito:** Colección sin duplicados y sin orden definido.

**Operaciones:**

| Método | Descripción | Complejidad |
|---|---|---|
| `InicializarConjunto()` | Prepara la estructura | O(1) |
| `Agregar(x)` | Agrega si no existe (llama `Pertenece`) | O(n) |
| `Sacar(x)` | Elimina el elemento | O(n) |
| `Pertenece(x)` | Búsqueda lineal | O(n) |
| `Elegir()` | Devuelve un elemento arbitrario | O(1) |
| `ConjuntoVacio()` | Indica si está vacío | O(1) |

**Implementación estática:** arreglo + `cant`. `Sacar` mueve el último elemento al hueco para no dejar espacios vacíos en O(1) de compra (la búsqueda sigue siendo O(n)).  
**Implementación dinámica:** lista enlazada encabezada por `c`. `Agregar` inserta al frente si no pertenece.

**Invariante:** No hay elementos duplicados. `Agregar` garantiza esto verificando `Pertenece` antes de insertar.

---

### 3.4 DiccionarioSimpleTDA — Diccionario (1 clave → 1 valor)

**Propósito:** Mapeo clave-valor donde cada clave tiene exactamente un valor.

**Operaciones:**

| Método | Descripción | Complejidad |
|---|---|---|
| `InicializarDiccionario()` | Prepara la estructura | O(1) |
| `Agregar(clave, valor)` | Inserta o actualiza | O(n) |
| `Eliminar(clave)` | Elimina la clave | O(n) |
| `Recuperar(clave)` | Devuelve el valor | O(n) |
| `Claves()` | Devuelve un ConjuntoTDA con todas las claves | O(n) |

**Implementación:** Dos arreglos paralelos `claves[]` y `valores[]`. La búsqueda de clave es O(n) lineal. La eliminación mueve el último elemento al hueco (swap-and-decrement).

---

### 3.5 DiccionarioMultipleTDA — Diccionario (1 clave → N valores)

**Propósito:** Mapeo clave-valores donde cada clave puede tener múltiples valores distintos.

**Operaciones:**

| Método | Descripción | Complejidad |
|---|---|---|
| `InicializarDiccionario()` | Prepara la estructura | O(1) |
| `Agregar(clave, valor)` | Agrega valor a la clave | O(n+m) |
| `Eliminar(clave)` | Elimina la clave con todos sus valores | O(n) |
| `EliminarValor(clave, valor)` | Elimina un valor específico | O(n+m) |
| `Recuperar(clave)` | Devuelve ConjuntoTDA de valores | O(n+m) |
| `Claves()` | Devuelve ConjuntoTDA de claves | O(n) |

Donde `n` = cantidad de claves, `m` = cantidad de valores por clave.

**Implementación:** Arreglo de claves + matriz bidimensional de valores. Cada posición `i` del arreglo de claves corresponde a la fila `i` de la matriz.

---

### 3.6 GrafoTDA — Grafo con aristas ponderadas

**Dos implementaciones:**

#### GrafoMA (Matriz de Adyacencia)

| Método | Complejidad | Por qué |
|---|---|---|
| `InicializarGrafo()` | **O(n²)** | Aloca la matriz n×n completa |
| `AgregarVertice(v)` | O(Vert) | Inicializa fila y columna del nuevo vértice |
| `EliminarVertice(v)` | O(Vert) | Reemplaza con el último (swap) |
| `AgregarArista(v1,v2,p)` | O(Vert) | Necesita `Vert2Indice` para mapear etiqueta → índice |
| `ExisteArista(v1,v2)` | O(Vert) | `MAdy[i][j]` en O(1) pero `Vert2Indice` es O(Vert) |
| `PesoArista(v1,v2)` | O(Vert) | Igual que ExisteArista |
| `Vertices()` | O(Vert) | Recorre el arreglo de etiquetas |

**Memoria:** O(V²). Tiene `n = 1_100_000` hardcodeado → **no apta para uso real**.

#### GrafoLA (Lista de Adyacencia)

| Método | Complejidad | Por qué |
|---|---|---|
| `InicializarGrafo()` | O(1) | Solo inicializa puntero y HashMap |
| `AgregarVertice(v)` | O(1) | Inserta al frente de la lista + put en HashMap |
| `EliminarVertice(v)` | O(Arist_v) | Elimina aristas entrantes de otros nodos |
| `AgregarArista(v1,v2,p)` | O(1) | Lookup O(1) por HashMap + inserción al frente |
| `ExisteArista(v1,v2)` | O(grado) | Recorre lista de adyacencia de v1 |
| `PesoArista(v1,v2)` | O(grado) | Igual que ExisteArista |
| `Vertices()` | O(Vert) | Recorre la lista de NodoGrafo |
| `ObtenerAdyacentes(v)` | O(grado) | Recorre lista de aristas de v |
| `GradoSalida(v)` | O(grado) | Cuenta aristas de salida de v |

**Memoria:** O(V + E). Usa un `HashMap<Integer, NodoGrafo>` interno como índice.

**Por qué GrafoLA en la aplicación real:** Las redes sociales son grafos **dispersos** (pocos seguidos por usuario, pocos amigos). GrafoLA usa O(V+E) vs O(V²) de GrafoMA, y sus operaciones críticas son O(grado) que es bajo en grafos dispersos.

---

## 4. TDAs de aplicación

### 4.1 HistorialAccionesTDA → StaticHistorialAccionesTDA

**Estructura interna:** `DynamicPilaTDA<Accion>`  
**Patrón:** LIFO — la última acción registrada es la primera en deshacerse.

| Método | Complejidad | Implementación |
|---|---|---|
| `registrarAccion(a)` | O(1) | `pila.Apilar(a)` |
| `deshacerUltimaAccion()` | O(1) | `pila.Tope()` + `pila.Desapilar()` |
| `hayAcciones()` | O(1) | `!pila.PilaVacia()` |
| `listarUltimas(n)` | O(n) | Desapila a lista temporal y vuelve a apilar |

**Detalle de `listarUltimas`:** Usa una pila auxiliar temporal para preservar el orden sin destruir el historial. Desapila `n` elementos a `temp`, los agrega al resultado, luego los devuelve a la pila original.

---

### 4.2 SolicitudesSeguimientoTDA → StaticSolicitudesSeguimientoTDA

**Estructura interna:** `DynamicColaTDA<SolicitudSeguimiento>`  
**Patrón:** FIFO — la primera solicitud enviada es la primera en procesarse.

| Método | Complejidad | Implementación |
|---|---|---|
| `agregarSolicitud(s)` | O(1) | `cola.Acolar(s)` |
| `procesarSolicitud()` | O(1) | `cola.Primero()` + `cola.Desacolar()` |
| `haySolicitudes()` | O(1) | `!cola.ColaVacia()` |
| `quitarSolicitud(s)` | O(n) | Recorre y reconstruye la cola con cola auxiliar |
| `listarPendientes()` | O(n) | Recorre y reconstruye con cola auxiliar |
| `procesarSolicitudParaUsuario(u)` | O(n) | Busca y extrae la primera solicitud para `u` |
| `listarPendientesParaUsuario(u)` | O(n) | Filtra sin modificar la cola |

**Patrón cola auxiliar:** Varios métodos O(n) vacían la cola principal, procesan elementos, y los reencolan en una cola temporal; luego reencolan todo a la cola original. Esto preserva el orden FIFO mientras permite buscar/filtrar.

---

## 5. ABB — Árbol Binario de Búsqueda

**Clase:** `ABB<T extends Comparable<T>>` con nodos `NodoABB<T>`.

**Propósito en el sistema:** Se usa en `consultarConexionesNivel4` para insertar los scorings de la red extendida y luego recuperar los que están exactamente en el nivel 4 del árbol.

| Método | Complejidad promedio | Complejidad peor caso |
|---|---|---|
| `agregar(x)` | O(log n) | O(n) — árbol degenerado (lista) |
| `obtenerNivel(k)` | O(n) | O(n) — siempre recorre todos los nodos |
| `imprimirNivel(k)` | O(n) | O(n) |
| `estaVacio()` | O(1) | O(1) |

**Propiedad del ABB:**
- Todo elemento del subárbol izquierdo de un nodo es **menor** que el nodo.
- Todo elemento del subárbol derecho es **mayor o igual** que el nodo.

**Por qué se usa un ABB aquí:** La consulta requiere obtener los elementos que están exactamente en el nivel 4 del árbol. El ABB organiza los scorings de forma ordenada jerárquicamente, y `obtenerNivel(4)` extrae eficientemente los nodos de ese nivel mediante recursión.

**Caso degenerado:** Si los scorings se insertan en orden creciente o decreciente, el ABB degenera en una lista enlazada y `agregar` pasa a ser O(n). En el contexto del sistema esto no es un problema práctico porque los scorings de una red son variados.

---

## 6. StaticClientesTDA — la pieza central

Esta clase concentra toda la lógica de la red social. Combina **cuatro estructuras** para distintos casos de uso:

```
StaticClientesTDA
│
├── HashMap<String, Cliente>         clientesPorNombre   → O(1) búsqueda por nombre
├── TreeMap<Integer, List<String>>   clientesPorScoring  → O(log n) búsqueda por scoring
├── GrafoLA                          grafoDirigido        → seguimiento (A→B sin reciprocidad)
└── GrafoLA                          grafoNoDirigido     → amistades (A↔B, bidireccional)
```

### Mapeo nombre ↔ ID

Los grafos operan con IDs enteros. Se mantienen dos mapas de traducción:
- `nombreAId: HashMap<String, Integer>` → nombre a ID
- `idANombre: HashMap<Integer, String>` → ID a nombre

Un contador `nextId` asigna IDs únicos y crecientes.

### Regla de negocio: MAX_SEGUIDOS = 2

Cada cliente puede seguir a como máximo 2 usuarios. Se verifica con `GradoSalida(idCliente) >= MAX_SEGUIDOS` antes de agregar una arista en el grafo dirigido.

### BFS para distancias

```java
private int bfs(GrafoLA grafo, int idOrigen, int idDestino)
```

Implementa BFS (Breadth-First Search) usando `Queue<Integer>` de Java. Devuelve la distancia en saltos o -1 si no hay camino. Se usa tanto para `calcularDistanciaSeguimiento` (grafo dirigido) como para `calcularDistanciaAmistad` (grafo no dirigido).

**Complejidad BFS:** O(V + E) donde V = vértices y E = aristas del grafo.

### consultarConexionesNivel4

```
1. BFS desde el nodo inicial sobre el grafo dirigido
2. Por cada nodo visitado, insertar su scoring en un ABB
3. Devolver los elementos del nivel 4 del ABB
```

**Complejidad total:** O(V + E + V·log V)
- O(V + E): BFS
- O(V·log V): V inserciones en el ABB, cada una O(log V) promedio

---

## 7. Tabla de complejidades por método

### TDAs básicos

| TDA | Método | Static | Dynamic |
|---|---|---|---|
| Pila | Apilar | O(1) | O(1) |
| Pila | Desapilar | O(1) | O(1) |
| Pila | Tope | O(1) | O(1) |
| Cola | Acolar | O(1) | O(1) |
| Cola | Desacolar | **O(n)** | O(1) |
| Cola | Primero | O(1) | O(1) |
| Conjunto | Agregar | O(n) | O(n) |
| Conjunto | Sacar | O(n) | O(n) |
| Conjunto | Pertenece | O(n) | O(n) |
| Conjunto | Elegir | O(1) | O(1) |
| DicSimple | Agregar | O(n) | — |
| DicSimple | Eliminar | O(n) | — |
| DicSimple | Recuperar | O(n) | — |
| DicSimple | Claves | O(n) | — |
| DicMultiple | Agregar | O(n+m) | — |
| DicMultiple | Recuperar | O(n+m) | — |
| DicMultiple | Claves | O(n) | — |
| Grafo MA | InicializarGrafo | **O(n²)** | — |
| Grafo MA | AgregarArista | O(V) | — |
| Grafo MA | ExisteArista | O(V) | — |
| Grafo LA | AgregarVertice | O(1) | — |
| Grafo LA | AgregarArista | O(1) | — |
| Grafo LA | ExisteArista | O(grado) | — |
| ABB | agregar | O(log n) prom | — |
| ABB | obtenerNivel | O(n) | — |

### TDAs de aplicación (GestorClientes)

| Método | Complejidad | Justificación |
|---|---|---|
| `agregarCliente` | O(1) | HashMap.put + GrafoLA.AgregarVertice |
| `buscarPorNombre` | O(1) | HashMap.get |
| `buscarPorScoring` | O(log n) | TreeMap.getOrDefault |
| `cantidadClientes` | O(1) | HashMap.size |
| `listarClientes` | O(n) | new ArrayList de todos los valores |
| `modificarCliente` | O(log n) | Actualiza TreeMap |
| `eliminarCliente` | O(V) | EliminarVertice en ambos grafos |
| `agregarSeguido` | O(grado) | GradoSalida + ExisteArista + AgregarArista |
| `quitarSeguido` | O(grado) | EliminarArista |
| `obtenerSeguidos` | O(grado) | ObtenerAdyacentes |
| `calcularDistanciaSeguimiento` | O(V+E) | BFS |
| `agregarAmistad` | O(grado) | 2× AgregarArista |
| `eliminarAmistad` | O(grado) | 2× EliminarArista |
| `calcularDistanciaAmistad` | O(V+E) | BFS |
| `consultarConexionesNivel4` | O(V+E+V·log V) | BFS + ABB |

---

## 8. Elecciones de diseño y por qué

### 8.1 ¿Por qué se implementaron TDAs propios si Java tiene ArrayList, LinkedList, etc.?

El objetivo académico del trabajo es **demostrar el manejo de estructuras de datos desde cero**. Se implementaron Pila, Cola, Conjunto y Diccionario sin usar ninguna colección de `java.util` en las implementaciones base.

Excepción legítima: `StaticClientesTDA` usa `HashMap` y `TreeMap` de Java para garantizar O(1) y O(log n) en búsquedas, ya que los TDAs propios son O(n) en esas operaciones. Esto es una decisión de eficiencia consciente y justificada.

---

### 8.2 ¿Por qué dos implementaciones (Static y Dynamic) para Pila/Cola/Conjunto?

| Aspecto | Static (arreglo) | Dynamic (lista enlazada) |
|---|---|---|
| Memoria | Fija, pre-alocada | Crece con los elementos |
| Velocidad acceso | O(1) por índice | O(1) por puntero |
| Desacolar (Cola) | **O(n)** — desplaza | O(1) — mueve puntero |
| Overhead | Bajo (array nativo) | Mayor (objeto Nodo por elemento) |
| Capacidad | Limitada (CAPACIDAD = 1.1M) | Ilimitada (hasta memoria disponible) |

La **implementación dinámica** se usa en producción (la aplicación instancia `DynamicColaTDA`, `DynamicPilaTDA`, `DynamicConjuntoTDA`) porque no tiene límite de capacidad y el `Desacolar` es O(1).

---

### 8.3 ¿Por qué GrafoLA y no GrafoMA para la red social?

| Criterio | GrafoMA | GrafoLA |
|---|---|---|
| Memoria | O(V²) | O(V + E) |
| `AgregarVertice` | O(V) | **O(1)** |
| `AgregarArista` | O(V) — `Vert2Indice` | **O(1)** — índice HashMap |
| `ExisteArista` | O(V) — `Vert2Indice` | O(grado) |
| Grafos dispersos | Desperdicia mucho espacio | Eficiente |
| `InicializarGrafo` | **O(n²)** — aloca matriz | O(1) |

Las redes sociales son grafos **muy dispersos**: con 1000 usuarios pero cada uno con máximo 2 seguidos y pocos amigos, la cantidad de aristas E << V². GrafoMA desperdiciaría O(V²) de memoria y su inicialización es O(n²).

**Optimización de GrafoLA:** Se agrega un `HashMap<Integer, NodoGrafo>` interno como índice. Sin él, encontrar un vértice sería O(V); con él es O(1). Esta es una mejora que va más allá del TDA básico.

---

### 8.4 ¿Por qué dos grafos separados (dirigido y no dirigido)?

- **Seguimiento** es una relación asimétrica: A puede seguir a B sin que B siga a A (como en Twitter). Se modela con un **grafo dirigido**.
- **Amistad** es una relación simétrica: si A es amigo de B, entonces B es amigo de A. Se modela con un **grafo no dirigido** (se agregan aristas en ambas direcciones: A→B y B→A).

Tener dos grafos separados también permite calcular distancias por separado y mantener las reglas de negocio independientes.

---

### 8.5 ¿Por qué Pila para el historial (deshacer)?

La funcionalidad de "deshacer" necesita acceder siempre a la **acción más reciente**. Esto es exactamente la propiedad LIFO de una pila: el último elemento apilado es el primero en recuperarse. Si usáramos una cola (FIFO), al deshacer obtendríamos la acción más antigua, que es incorrecto.

---

### 8.6 ¿Por qué Cola para las solicitudes de amistad?

Las solicitudes de amistad deben procesarse en el **orden en que se enviaron** (justicia, equidad). La primera solicitud enviada debe ser la primera en poderse aceptar o rechazar. Esto es exactamente FIFO (Cola). Si usáramos una pila, siempre se procesaría primero la última solicitud recibida.

---

### 8.7 ¿Por qué un ABB para `consultarConexionesNivel4`?

El requerimiento es encontrar los scorings de los contactos de la red extendida que están en el **nivel 4** de un árbol. Al insertar los scorings de los nodos visitados por BFS en un ABB, la estructura organiza los valores de forma jerárquica según su valor numérico. Extraer el nivel 4 permite recuperar los scorings que, dentro de la jerarquía del árbol, ocupan la cuarta generación.

---

### 8.8 ¿Por qué el modelo (Cliente) es inmutable?

`Cliente` usa campos `final`, no tiene setters, y devuelve copias inmutables de sus listas (`Collections.unmodifiableList`). Esto previene modificaciones accidentales al compartir referencias. Cuando se modifica un cliente (ej: cambio de scoring), se crea un nuevo objeto `Cliente` con los datos actualizados.

---

### 8.9 ¿Por qué se hace la carga en dos pasadas en CargadorClientesJson?

```java
// Primera pasada: cargar todos los clientes
for (ClienteJson cj : datos.clientes) gestor.agregarCliente(...);

// Segunda pasada: cargar relaciones
for (ClienteJson cj : datos.clientes) {
    for (String sig : cj.siguiendo) gestor.agregarSeguido(...);
    for (String con : cj.conexiones) gestor.agregarAmistad(...);
}
```

Si se cargaran las relaciones en la primera pasada, podría intentarse agregar una arista hacia un vértice que aún no existe en el grafo. La separación garantiza que todos los vértices existen antes de agregar las aristas.

---

## 9. Patrones de diseño aplicados

### 9.1 Builder (Construcción de Menús)

`MenuBuilder` construye instancias de `Menu` paso a paso con una API fluida (método encadenado). Separa la construcción del objeto de su representación.

```java
new MenuBuilder("RED SOCIAL")
    .agregarOpcion("1", "Buscar", scanner -> buscar())
    .setOpcionSalida("0")
    .build();
```

### 9.2 Facade (Fachadas de servicio)

`GestorClientes`, `HistorialAcciones` y `ColaSolicitudesSeguimiento` son fachadas que:
- Ocultan la complejidad interna de los TDAs.
- Exponen una API simple al `MenuRedSocial`.
- Dependen de abstracciones (interfaces TDA), no de implementaciones concretas.

### 9.3 Strategy / Inyección de dependencias implícita

Los TDAs de aplicación (ej: `HistorialAccionesTDA`) son interfaces. La implementación concreta (`StaticHistorialAccionesTDA`) se instancia dentro del servicio, pero podría cambiarse por otra implementación sin modificar el código del servicio.

### 9.4 Template Method (implícito en el menú)

`Menu.ejecutarUnaVez()` define el algoritmo general (mostrar menú → leer input → ejecutar opción), y las acciones concretas se inyectan como `Runnable` o `Consumer<Scanner>`.

---

## 10. Persistencia y entrada/salida

### clientes.json

- **Lectura:** `CargadorClientesJson.readFromFile(GestorClientes)` — O(n + e)
- **Escritura:** `GuardadorClientesJson.guardar(GestorClientes)` — O(n + e)
- Usa **Gson** para serialización/deserialización.
- Las relaciones (siguiendo, conexiones) se obtienen del grafo, no del objeto Cliente, evitando duplicación de datos.
- El guardado se activa automáticamente al salir del sistema.

### acciones.csv

- **Exportación:** `ExportadorAccionesCsv.exportar(List<Accion>, String)` — O(n)
- Formato: `tipo,detalle,fechaHora`
- Si el archivo ya existe, hace **append** (no sobrescribe).
- Usa escape de CSV para manejar comas y comillas en los campos.
- Se exporta automáticamente al salir del sistema.

---

## 11. Preguntas frecuentes del profesor

### Sobre TDAs

**¿Qué es un TDA?**  
Un Tipo de Dato Abstracto es una especificación de un conjunto de datos y las operaciones sobre ellos, independientemente de su implementación. Define el "qué" (interfaz), no el "cómo" (implementación). En este proyecto cada TDA es una interfaz Java con las operaciones y sus complejidades documentadas.

**¿Cuál es la diferencia entre implementación estática y dinámica?**  
La implementación estática usa arreglos de tamaño fijo pre-alocados. La dinámica usa nodos enlazados que se crean bajo demanda. La dinámica es más flexible (sin límite de capacidad) pero tiene overhead de memoria por nodo. La estática es más rápida en acceso por índice pero tiene capacidad fija.

**¿Por qué `StaticColaTDA.Desacolar()` es O(n) y `DynamicColaTDA.Desacolar()` es O(1)?**  
En la implementación estática, al eliminar el primero del arreglo hay que desplazar todos los elementos una posición hacia la izquierda. En la dinámica, simplemente se actualiza el puntero `primero = primero.sig`, operación de tiempo constante.

**¿Por qué `ConjuntoTDA.Agregar()` es O(n)?**  
Porque antes de insertar se llama a `Pertenece(x)` para verificar que el elemento no existe ya en el conjunto (invariante: no duplicados). `Pertenece` recorre la estructura linealmente → O(n).

**¿Cuál es el invariante de representación del Conjunto?**  
No hay elementos duplicados. Cada elemento aparece exactamente una vez.

---

### Sobre el Grafo

**¿Qué es un grafo dirigido vs. no dirigido?**  
En un grafo dirigido, las aristas tienen dirección (A→B no implica B→A). En uno no dirigido, son bidireccionales (A—B implica que existe conexión en ambos sentidos). En este proyecto, el grafo dirigido modela el seguimiento y el no dirigido las amistades.

**¿Por qué GrafoLA usa un HashMap interno si es un TDA hecho sin Java collections?**  
El HashMap interno es una optimización de implementación para mejorar la complejidad de `Vert2Nodo` de O(V) a O(1). El TDA define el "qué" (buscar un vértice), y la implementación decide el "cómo". El uso de HashMap aquí es una decisión de implementación válida que no viola la abstracción del TDA.

**¿Por qué `GrafoMA.InicializarGrafo()` es O(n²)?**  
Porque aloca una matriz bidimensional `MAdy[n][n]` donde `n = 1.1 millones`. Esto es extremadamente costoso e inviable en la práctica, lo que ilustra claramente la desventaja de la matriz de adyacencia para grafos grandes.

**¿Qué sucede al eliminar un vértice en GrafoLA?**  
Se remueve el vértice de la lista enlazada y del índice HashMap. Luego se recorren TODOS los demás vértices y se eliminan las aristas que apuntaban al vértice eliminado. Por eso la complejidad es O(V + E_v) donde E_v son las aristas que involucran al vértice.

**¿Cómo funciona el BFS?**  
Breadth-First Search (Búsqueda en Anchura) explora el grafo nivel por nivel usando una cola. Comienza desde el nodo origen, encola sus vecinos no visitados, y procesa nodo por nodo. La distancia hasta cada nodo visitado es la distancia del nodo actual más 1. Garantiza encontrar el camino más corto en grafos no ponderados. Complejidad: O(V + E).

---

### Sobre el ABB

**¿Cuándo el ABB degenera y por qué es un problema?**  
Si se insertan elementos en orden creciente o decreciente, el ABB degenera en una lista enlazada (todos los nodos en una sola rama). Las operaciones que son O(log n) en el caso promedio se vuelven O(n). Para evitar esto en producción se usan árboles balanceados (AVL, Red-Black).

**¿Qué significa "nivel 4" en el contexto del ABB?**  
La raíz está en el nivel 1. Sus hijos en el nivel 2. Los nietos en el nivel 3. Los bisnietos en el nivel 4. `obtenerNivel(4)` devuelve todos los nodos que están exactamente en la cuarta generación del árbol.

---

### Sobre la arquitectura

**¿Por qué se separaron los paquetes `basic_tdas` y `tda`?**  
`basic_tdas` contiene estructuras genéricas (Pila, Cola, Conjunto, Grafo) reutilizables en cualquier contexto. `tda` contiene las interfaces específicas del dominio de negocio (`ClientesTDA`, `HistorialAccionesTDA`, `SolicitudesSeguimientoTDA`). Esta separación sigue el principio de separación de responsabilidades y facilita la reutilización.

**¿Por qué `StaticClientesTDA` usa HashMap y TreeMap de Java si los otros TDAs son propios?**  
Es una decisión de eficiencia justificada. Los TDAs propios de búsqueda son O(n). Para el CRUD de clientes se necesita O(1) (por nombre) y O(log n) (por scoring), que solo lo proveen `HashMap` y `TreeMap` respectivamente. El objetivo académico es demostrar el conocimiento de TDAs; usar colecciones nativas para casos donde la eficiencia es crucial es una práctica correcta de ingeniería.

**¿Por qué el modelo `Cliente` es inmutable?**  
Para garantizar consistencia: si se comparte una referencia a un `Cliente` entre varias estructuras, nadie puede modificarlo accidentalmente. Si hay que cambiar datos (ej: scoring), se crea un nuevo objeto. Esto sigue el principio de diseño de objetos con estado controlado.

**¿Por qué se usan dos pasadas en CargadorClientesJson?**  
Para garantizar que todos los vértices del grafo existan antes de agregar aristas entre ellos. Si se intentara agregar una arista hacia un vértice inexistente, `GrafoLA.AgregarArista` lo ignoraría silenciosamente (verifica `if (n1 == null || n2 == null) return`). Las dos pasadas eliminan ese riesgo.

**¿Qué patrón de diseño usa MenuBuilder?**  
El patrón Builder. Permite construir objetos complejos paso a paso con una API fluida. Separa la lógica de construcción de la representación del objeto.

**¿Cómo funciona el "deshacer"?**  
Cada acción relevante registra en el historial (Pila) un objeto `Accion` con tipo y detalle. Al deshacer, se desapila la última acción y según el `tipo` se ejecuta la operación inversa:
- "Registrarse" → eliminar cliente
- "Seguir" → quitar seguido
- "Dejar de seguir" → volver a seguir
- "Enviar solicitud amistad" → quitar solicitud de la cola
- "Aceptar solicitud amistad" → eliminar la amistad del grafo
- "Rechazar solicitud amistad" → re-encolar la solicitud

**¿Qué estrategia de encapsulamiento usa la Cola de solicitudes al "listar sin modificar"?**  
El patrón de **cola auxiliar temporal**: vacía la cola principal en una cola temporal mientras procesa, luego reencola todo de vuelta. Esto preserva el orden FIFO original. Es O(n) en tiempo pero O(1) en espacio adicional relativo (solo una cola temporal).

---

### Preguntas trampa o de profundidad

**¿Qué pasa si se inserta un scoring negativo en el ABB?**  
El ABB acepta cualquier `Comparable`. Un scoring negativo se insertaría en el subárbol izquierdo si es menor que la raíz. No hay validación en el ABB; la validación debería hacerse en la capa de negocio.

**¿Puede `consultarConexionesNivel4` devolver duplicados de scoring?**  
Sí. Si dos clientes distintos tienen el mismo scoring y ambos están en el nivel 4 del ABB, el scoring aparecerá en el resultado. El ABB no impone unicidad (elemento igual va al subárbol derecho: `else agregarRec(nodo.der, elemento)`).

**¿Qué ocurre si el usuario intenta seguir a más de 2 personas?**  
`agregarSeguido` verifica `GradoSalida(idCliente) >= MAX_SEGUIDOS` y retorna `false`. El menú muestra el mensaje "alcanzaste el límite de seguidos".

**¿Qué ocurre si se elimina un cliente que tiene solicitudes de amistad pendientes?**  
`eliminarCliente` elimina el cliente del HashMap y de ambos grafos. Luego recorre todos los clientes restantes y elimina al cliente borrado de sus listas de `solicitudesPendientes`. Sin embargo, la cola `ColaSolicitudesSeguimiento` global no se limpia automáticamente de solicitudes que tenían ese cliente como origen o destino; esto podría considerarse una mejora pendiente.

**¿Cuál es la diferencia entre seguimiento y amistad en el modelo de grafos?**  
El seguimiento usa el `grafoDirigido`: A puede seguir a B sin que B siga a A. La amistad usa el `grafoNoDirigido`: al agregar amistad se crean dos aristas (A→B y B→A) en el mismo grafo, simulando la bidireccionalidad.

**¿Por qué `GrafoLA.EliminarVertice()` tiene complejidad O(Arist_v) y no O(V)?**  
Porque el índice HashMap permite encontrar el NodoGrafo en O(1). El costo dominante es recorrer las listas de adyacencia de TODOS los demás vértices para eliminar aristas entrantes al vértice eliminado. En el peor caso, si todos los nodos tienen aristas hacia el vértice eliminado, eso es O(V × grado_promedio) ≈ O(E). En el comentario del código se simplifica como O(Arist_v) refiriéndose a las aristas involucradas.

---

*Documento generado para preparación del examen oral — Programación II, UADE 2026*

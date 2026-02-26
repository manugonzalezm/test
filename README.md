# Resumen Teórico — Red Social Empresarial
### Programación II · UADE · 2026 - 1C · Grupo 1

---

## Índice

1. [Descripción del sistema](#1-descripción-del-sistema)
2. [Cobertura de las tres iteraciones](#2-cobertura-de-las-tres-iteraciones)
3. [Arquitectura en capas](#3-arquitectura-en-capas)
4. [TDAs básicos implementados](#4-tdas-básicos-implementados)
5. [ABB — Árbol Binario de Búsqueda](#5-abb--árbol-binario-de-búsqueda)
6. [TDAs de aplicación](#6-tdas-de-aplicación)
7. [StaticClientesTDA — la pieza central](#7-staticclientestda--la-pieza-central)
8. [Tabla de complejidades por método](#8-tabla-de-complejidades-por-método)
9. [Elecciones de diseño y por qué](#9-elecciones-de-diseño-y-por-qué)
10. [Patrones de diseño aplicados](#10-patrones-de-diseño-aplicados)
11. [Persistencia y entrada/salida](#11-persistencia-y-entradasalida)
12. [Preguntas frecuentes del profesor](#12-preguntas-frecuentes-del-profesor)

---

## 1. Descripción del sistema

Sistema de red social empresarial de consola. Los usuarios (clientes) pueden:

- Registrarse con nombre y puntuación (scoring).
- **Seguir** directamente a otros clientes (sin aprobación, máximo 2 seguidos).
- **Enviar solicitudes de amistad** almacenadas en la lista propia del receptor; el destinatario las acepta o rechaza por índice.
- Ver las **distancias** entre usuarios (saltos) por seguimiento o por amistad.
- Explorar la **red extendida** de un usuario con BFS + ABB (nivel 4).
- Deshacer la última acción (historial LIFO).
- Persistir el estado en `clientes.json` y exportar el historial a `acciones.csv`.

---

## 2. Cobertura de las tres iteraciones

### Iteración 1 — Gestión básica de clientes y acciones

| Requerimiento | Implementación | Complejidad |
|---|---|---|
| Almacenar y buscar clientes por nombre | `HashMap<String, Cliente>` en `StaticClientesTDA` | O(1) |
| Buscar clientes por scoring | `TreeMap<Integer, List<String>>` en `StaticClientesTDA` | O(log n) |
| Historial de acciones (registrar/deshacer) | `DynamicPilaTDA<Accion>` en `StaticHistorialAccionesTDA` | O(1) |
| Solicitudes de seguimiento en orden de envío | `DynamicColaTDA` (TDA implementado, estudiado en la iteración) | O(1) |
| Carga desde JSON | `CargadorClientesJson` con Gson | O(n + e) |
| Pruebas unitarias | `GestorClientesTest`, `HistorialAccionesTest`, etc. | — |

**Decisión clave — búsqueda eficiente:**  
Se usan `HashMap` y `TreeMap` de Java (en lugar de los TDAs propios O(n)) para garantizar O(1) y O(log n). Los TDAs propios se implementan y estudian, pero para las búsquedas de negocio se eligen las estructuras con mejor complejidad.

---

### Iteración 2 — Relaciones entre clientes (seguimiento + ABB)

| Requerimiento | Implementación | Complejidad |
|---|---|---|
| Cada cliente sigue hasta 2 clientes | `GrafoLA` dirigido + restricción `MAX_SEGUIDOS = 2` | O(grado) |
| Clientes ordenados por scoring | `TreeMap<Integer, List<String>>` (ya presente) | O(log n) |
| Consulta de conexiones con ABB | `ABB<Integer>` — BFS + inserción de scorings | O(V + E + V·log V) |
| Imprimir nivel 4 del ABB | `ABB.obtenerNivel(4)` | O(n) |
| Persistencia de seguimientos | Campo `siguiendo` en `clientes.json` | O(n + e) |
| Pruebas unitarias | `StaticClientesTDATest`, `GestorClientesTest` | — |

**Decisión clave — grafo dirigido:**  
El seguimiento es asimétrico (A sigue a B no implica que B siga a A). Se usa un `GrafoLA` dirigido. `GrafoLA` se elige sobre `GrafoMA` porque las redes son dispersas y `GrafoLA` usa O(V+E) de memoria vs O(V²).

---

### Iteración 3 — Relaciones generales (amistades + distancias)

Esta sección se desarrolla en detalle porque es la iteración más reciente.

#### 3.1 Relaciones generales — amistades/conexiones

**Requerimiento:** _"Implementar una estructura para gestionar relaciones generales entre clientes. Permitir agregar relaciones y obtener los vecinos de un cliente. Se valorará O(1) para obtener vecinos."_

**Implementación:**

Se usa un segundo `GrafoLA` **no dirigido** (`grafoNoDirigido` en `StaticClientesTDA`). La no dirección se implementa agregando la arista en ambos sentidos al aceptar una amistad:

```java
grafoNoDirigido.AgregarArista(idA, idB, 1);  // A → B
grafoNoDirigido.AgregarArista(idB, idA, 1);  // B → A
```

Los métodos expuestos:

| Método | Complejidad | Descripción |
|---|---|---|
| `agregarAmistad(a, b)` | O(grado) | Crea aristas A→B y B→A |
| `eliminarAmistad(a, b)` | O(grado) | Elimina aristas A→B y B→A |
| `obtenerVecinos(nombre)` | O(grado) | Devuelve el conjunto de amigos |

**¿Por qué O(grado) ≈ O(1) en la práctica?**  
En una red social dispersa, el grado promedio (cantidad de amigos por usuario) es una constante pequeña e independiente del total de usuarios V. Por eso, aunque formalmente es O(grado), en el contexto del sistema se comporta como O(1) para el usuario típico, que es lo que valora la consigna.

**¿Por qué un segundo grafo y no el mismo?**  
El seguimiento es una relación asimétrica y tiene la restricción de máximo 2. La amistad es simétrica y sin límite. Mezclarlas en el mismo grafo complicaría las validaciones y contaminaría las consultas de distancia.

#### 3.2 Distancia entre clientes

**Requerimiento:** _"Implementar un mecanismo para calcular la distancia (número de saltos) entre dos clientes en la red."_

**Implementación:** BFS (Breadth-First Search) implementado en `StaticClientesTDA.bfs()`, reutilizado para ambos grafos:

```
calcularDistanciaSeguimiento(origen, destino) → bfs(grafoDirigido,    idO, idD)
calcularDistanciaAmistad(origen, destino)     → bfs(grafoNoDirigido, idO, idD)
```

El BFS explora el grafo nivel por nivel usando una `Queue<Integer>` (de Java), manteniendo un `Map<Integer, Integer>` de distancias ya calculadas. Devuelve `-1` si no hay camino.

**Complejidad:** O(V + E) — visita cada vértice y arista exactamente una vez en el peor caso.

**¿Por qué BFS y no DFS?**  
BFS garantiza encontrar el **camino más corto** en grafos no ponderados (o con peso uniforme). DFS no garantiza el mínimo de saltos.

#### 3.3 Persistencia de relaciones generales

**Requerimiento:** _"Extender la carga desde JSON para incluir las relaciones generales."_

El archivo `clientes.json` tiene el campo `conexiones` que representa las amistades:

```json
{
  "nombre": "Alice",
  "scoring": 95,
  "siguiendo": ["Bob"],
  "conexiones": ["Bob", "Charlie"]
}
```

`CargadorClientesJson` carga en **dos pasadas**:
1. Primera pasada: crea todos los vértices (clientes) en los grafos.
2. Segunda pasada: agrega aristas (`siguiendo` → grafo dirigido; `conexiones` → grafo no dirigido).

La separación en dos pasadas garantiza que todos los vértices existan antes de intentar conectarlos.

`GuardadorClientesJson` serializa el estado actual consultando los grafos directamente (`obtenerVecinos`, `obtenerSeguidos`), no el objeto `Cliente`, evitando duplicación de datos.

#### 3.4 Solicitudes de amistad — diseño per-usuario

**Evolución de diseño:** Inicialmente las solicitudes se manejaban en una cola global (`ColaSolicitudesSeguimiento`). Tras el análisis, se detectaron dos problemas:

1. **Sin caso de uso global:** Ninguna operación del sistema necesita ver las solicitudes de todos los usuarios a la vez; siempre se filtra por usuario destino.
2. **Semántica incorrecta:** El usuario elige qué solicitud aceptar por índice (número en pantalla), no por orden de llegada FIFO.

**Diseño actual:** Cada `Cliente` tiene su propia `List<SolicitudSeguimiento> solicitudesRecibidas` (ArrayList).

| Operación | Estructura anterior (cola global) | Estructura actual (lista por usuario) |
|---|---|---|
| Enviar solicitud | O(1) encolar | O(n_sol_receptor) — verificar duplicado |
| Listar pendientes del usuario | O(n_global) — recorrer y reconstruir | O(1) — acceso directo a la lista |
| Aceptar por índice | O(n_global) — buscar en cola | O(n) — `ArrayList.remove(int)` |
| Rechazar por índice | O(n_global) | O(n) — `ArrayList.remove(int)` |
| Semántica | FIFO — no corresponde | Lista ordenada por llegada con acceso por posición ✅ |

donde `n_global` = total de solicitudes de todos los usuarios, y `n` = solicitudes del usuario receptor específico.

#### 3.5 Pruebas unitarias — Iteración 3

| Test | Qué valida |
|---|---|
| `amistadBidireccional` | `agregarAmistad` crea aristas en ambas direcciones |
| `eliminarAmistad` | `eliminarAmistad` quita ambas aristas |
| `calcularDistanciaSeguimiento` | BFS en grafo dirigido; devuelve -1 si no hay camino |
| `calcularDistanciaAmistad` | BFS en grafo no dirigido |
| `aceptarSolicitudAmistadCreaAmistad` | Flujo completo: enviar → aceptar → amistad creada |
| `rechazarSolicitudAmistad` | Rechazar elimina de la lista sin crear amistad |
| `deshacerAceptarSolicitudAmistad` | Deshacer elimina la amistad creada |
| `testCargaClientesDesdeJson` | Las conexiones del JSON se cargan correctamente |

---

## 3. Arquitectura en capas

```
┌─────────────────────────────────────────────────────────────┐
│  UI / Presentación                                          │
│  utils/Menu · MenuBuilder · MenuRedSocial                   │
├─────────────────────────────────────────────────────────────┤
│  Servicios (Fachadas)                                       │
│  GestorClientes · HistorialAcciones                         │
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
| `model` | POJOs del dominio (inmutables salvo lista de solicitudes) |
| `basic_tdas/tda` | Interfaces de estructuras elementales |
| `basic_tdas/implementation` | Implementaciones estáticas y dinámicas |
| `basic_tdas/entities` | Nodos internos (Nodo, NodoGrafo, NodoArista) |
| `tda` | Interfaces de TDAs de negocio |
| `implementation` | Implementaciones de TDAs de negocio |
| `EstructuraABB` | Árbol Binario de Búsqueda genérico |
| `services` | Fachadas de negocio + I/O |
| `utils` | Sistema de menús |

---

## 4. TDAs básicos implementados

### 4.1 PilaTDA — Pila (LIFO)

**Propósito en el sistema:** Historial de acciones (deshacer).  
**¿Por qué LIFO?** El deshacer siempre revierte la **última** acción realizada.

| Método | Static | Dynamic |
|---|---|---|
| `InicializarPila()` | O(1) | O(1) |
| `Apilar(x)` | O(1) | O(1) |
| `Desapilar()` | O(1) | O(1) |
| `Tope()` | O(1) | O(1) |
| `PilaVacia()` | O(1) | O(1) |

**Implementación dinámica:** lista enlazada con puntero al primer nodo. Inserción y eliminación siempre al frente.  
**Implementación estática:** arreglo fijo, tope en `a[indice-1]`.

---

### 4.2 ColaTDA — Cola (FIFO)

**Propósito en el sistema:** TDA implementado y estudiado; la cola dinámica se usa como base en `StaticSolicitudesSeguimientoTDA` (que se mantiene como TDA autónomo aunque el flujo de la aplicación ahora usa listas por usuario).  
**¿Por qué FIFO originalmente para solicitudes?** Las solicitudes deben procesarse en orden de llegada.

| Método | Static | Dynamic |
|---|---|---|
| `InicializarCola()` | O(1) | O(1) |
| `Acolar(x)` | O(1) | O(1) |
| `Desacolar()` | **O(n)** (desplaza) | O(1) |
| `Primero()` | O(1) | O(1) |
| `ColaVacia()` | O(1) | O(1) |

**Diferencia clave:** `StaticColaTDA.Desacolar()` es O(n) porque desplaza todo el arreglo. `DynamicColaTDA.Desacolar()` es O(1) porque solo avanza el puntero `primero`. La implementación dinámica mantiene además un puntero `ultimo` para `Acolar` en O(1).

---

### 4.3 ConjuntoTDA — Conjunto

**Invariante:** No hay elementos duplicados.

| Método | Complejidad |
|---|---|
| `InicializarConjunto()` | O(1) |
| `Agregar(x)` | O(n) — llama a `Pertenece` antes |
| `Sacar(x)` | O(n) |
| `Pertenece(x)` | O(n) |
| `Elegir()` | O(1) |
| `ConjuntoVacio()` | O(1) |

`Agregar` es O(n) porque debe verificar unicidad llamando a `Pertenece`.  
`Sacar` estático: swap del elemento encontrado con el último y decremento del contador — O(n) de búsqueda, O(1) de eliminación.

---

### 4.4 DiccionarioSimpleTDA

Mapeo 1 clave → 1 valor. Implementación: dos arreglos paralelos `claves[]` y `valores[]`.

| Método | Complejidad |
|---|---|
| `Agregar(clave, valor)` | O(n) |
| `Eliminar(clave)` | O(n) |
| `Recuperar(clave)` | O(n) |
| `Claves()` | O(n) |

Eliminación usa swap-and-decrement para no dejar huecos.

---

### 4.5 DiccionarioMultipleTDA

Mapeo 1 clave → N valores (sin duplicados). Implementación: arreglo de claves + matriz 2D de valores.

| Método | Complejidad |
|---|---|
| `Agregar(clave, valor)` | O(n+m) |
| `Recuperar(clave)` | O(n+m) |
| `Claves()` | O(n) |
| `Eliminar(clave)` | O(n) |
| `EliminarValor(clave, valor)` | O(n+m) |

`n` = cantidad de claves, `m` = cantidad de valores por clave.

---

### 4.6 GrafoTDA — Dos implementaciones

#### GrafoMA (Matriz de Adyacencia)

| Método | Complejidad |
|---|---|
| `InicializarGrafo()` | **O(n²)** — aloca la matriz n×n |
| `AgregarVertice(v)` | O(V) |
| `AgregarArista(v1,v2,p)` | O(V) — `Vert2Indice` es lineal |
| `ExisteArista(v1,v2)` | O(V) |
| `PesoArista(v1,v2)` | O(V) |
| `EliminarVertice(v)` | O(V) |

Memoria: O(V²). `n` hardcodeado en 1.100.000 → inviable en producción real.

#### GrafoLA (Lista de Adyacencia) — el usado en la aplicación

| Método | Complejidad |
|---|---|
| `InicializarGrafo()` | O(1) |
| `AgregarVertice(v)` | O(1) |
| `AgregarArista(v1,v2,p)` | O(1) — lookup O(1) por HashMap interno |
| `ExisteArista(v1,v2)` | O(grado) |
| `PesoArista(v1,v2)` | O(grado) |
| `EliminarVertice(v)` | O(Arist_v) |
| `ObtenerAdyacentes(v)` | O(grado) |
| `GradoSalida(v)` | O(grado) |

Memoria: O(V + E). Optimización: `HashMap<Integer, NodoGrafo>` interno para lookup de vértice en O(1).

---

## 5. ABB — Árbol Binario de Búsqueda

**Propósito:** `consultarConexionesNivel4` inserta los scorings de la red extendida en el ABB y recupera los del nivel 4.

**Propiedad:** subárbol izquierdo < nodo ≤ subárbol derecho.

| Método | Promedio | Peor caso |
|---|---|---|
| `agregar(x)` | O(log n) | O(n) — árbol degenerado |
| `obtenerNivel(k)` | O(n) | O(n) |
| `estaVacio()` | O(1) | O(1) |

**Caso degenerado:** insertar en orden creciente/decreciente genera una lista → O(n) en todo. En este sistema no es un problema práctico porque los scorings de una red son variados.

**¿Qué es el nivel 4?** Raíz = nivel 1, hijos = nivel 2, nietos = nivel 3, bisnietos = nivel 4. `obtenerNivel(4)` recorre el árbol recursivamente decrementando el nivel en cada llamada.

---

## 6. TDAs de aplicación

### 6.1 HistorialAccionesTDA → StaticHistorialAccionesTDA

Estructura interna: `DynamicPilaTDA<Accion>`.

| Método | Complejidad |
|---|---|
| `registrarAccion(a)` | O(1) — `pila.Apilar(a)` |
| `deshacerUltimaAccion()` | O(1) — `pila.Tope()` + `pila.Desapilar()` |
| `hayAcciones()` | O(1) |
| `listarUltimas(n)` | O(n) — pila auxiliar temporal |

`listarUltimas` usa una pila auxiliar: desapila `n` elementos al resultado, los reapila en `temp`, luego los vuelve a la pila original. Preserva el orden sin destruir el historial.

---

### 6.2 SolicitudesSeguimientoTDA → StaticSolicitudesSeguimientoTDA

TDA implementado (disponible y con tests). Usa `DynamicColaTDA<SolicitudSeguimiento>`.  
**Nota:** En el flujo actual de la aplicación este TDA no se usa directamente; las solicitudes viven en la lista propia de cada `Cliente`. Este TDA se conserva como implementación estudiada y testeada de la Cola aplicada a un dominio concreto.

---

## 7. StaticClientesTDA — la pieza central

Combina cuatro estructuras para distintos casos de uso:

```
StaticClientesTDA
│
├── HashMap<String, Cliente>          clientesPorNombre   → búsqueda O(1)
├── TreeMap<Integer, List<String>>    clientesPorScoring  → búsqueda O(log n)
├── GrafoLA  grafoDirigido            → seguimiento (A→B, max 2 por usuario)
└── GrafoLA  grafoNoDirigido          → amistades (A↔B, bidireccional)
```

Cada `Cliente` además tiene internamente:
```
Cliente
└── List<SolicitudSeguimiento> solicitudesRecibidas  → solicitudes de amistad por usuario
```

### Mapeo nombre ↔ ID

Los grafos usan IDs enteros. Dos HashMaps internos traducen:
- `nombreAId: HashMap<String, Integer>`
- `idANombre: HashMap<Integer, String>`

Un contador `nextId` asigna IDs únicos crecientes.

### Regla MAX_SEGUIDOS = 2

Se verifica con `GradoSalida(idCliente) >= MAX_SEGUIDOS` antes de agregar la arista en el grafo dirigido.

### BFS — algoritmo de distancias

```java
private int bfs(GrafoLA grafo, int idOrigen, int idDestino) // O(V + E)
```

- Usa `Queue<Integer>` y `Map<Integer, Integer>` (distancias).
- Retorna la distancia mínima en saltos, o -1 si no hay camino.
- Se reutiliza para el grafo dirigido (seguimiento) y no dirigido (amistad).

### consultarConexionesNivel4

```
1. BFS desde el nodo inicial sobre grafoDirigido    → O(V + E)
2. Insertar scoring de cada nodo visitado en ABB     → O(V · log V)
3. Devolver ABB.obtenerNivel(4)                     → O(V)
Total: O(V + E + V·log V)
```

---

## 8. Tabla de complejidades por método

### TDAs básicos

| TDA | Método | Static | Dynamic |
|---|---|---|---|
| Pila | Apilar / Desapilar / Tope | O(1) | O(1) |
| Cola | Acolar | O(1) | O(1) |
| Cola | Desacolar | **O(n)** | O(1) |
| Conjunto | Agregar / Sacar / Pertenece | O(n) | O(n) |
| Conjunto | Elegir / ConjuntoVacio | O(1) | O(1) |
| DicSimple | Agregar / Eliminar / Recuperar | O(n) | — |
| DicMultiple | Agregar / Recuperar | O(n+m) | — |
| Grafo MA | InicializarGrafo | **O(n²)** | — |
| Grafo MA | AgregarArista / ExisteArista | O(V) | — |
| Grafo LA | AgregarVertice / AgregarArista | O(1) | — |
| Grafo LA | ExisteArista / ObtenerAdyacentes | O(grado) | — |
| ABB | agregar | O(log n) prom. | — |
| ABB | obtenerNivel | O(n) | — |

### GestorClientes (operaciones de negocio)

| Método | Complejidad |
|---|---|
| `agregarCliente` | O(1) |
| `buscarPorNombre` | O(1) |
| `buscarPorScoring` | O(log n) |
| `cantidadClientes` | O(1) |
| `listarClientes` | O(n) |
| `modificarCliente` | O(log n) |
| `eliminarCliente` | O(V) |
| `agregarSeguido` | O(grado) |
| `quitarSeguido` | O(grado) |
| `obtenerSeguidos` | O(grado) |
| `calcularDistanciaSeguimiento` | O(V + E) |
| `agregarAmistad` | O(grado) |
| `eliminarAmistad` | O(grado) |
| `obtenerVecinos` | O(grado) |
| `calcularDistanciaAmistad` | O(V + E) |
| `enviarSolicitudAmistad` | O(n_sol_receptor) |
| `listarSolicitudesRecibidas` | O(1) |
| `aceptarSolicitudAmistad` | O(grado) |
| `rechazarSolicitudAmistad` | O(1) |
| `revocarSolicitudAmistad` | O(n_sol_receptor) |
| `consultarConexionesNivel4` | O(V + E + V·log V) |

---

## 9. Elecciones de diseño y por qué

### 9.1 ¿Por qué TDAs propios y no los de Java?

El objetivo académico es demostrar el manejo de estructuras de datos desde cero. Se implementan Pila, Cola, Conjunto, Diccionario y Grafo sin usar colecciones de `java.util` en las implementaciones base.

**Excepción justificada:** `StaticClientesTDA` usa `HashMap` y `TreeMap` de Java para garantizar O(1) y O(log n) en las búsquedas de negocio críticas. Los TDAs propios son O(n) en búsqueda, lo cual sería ineficiente para el caso de uso central.

---

### 9.2 ¿Por qué dos implementaciones (Static y Dynamic) para Pila/Cola/Conjunto?

| Aspecto | Static (arreglo) | Dynamic (lista enlazada) |
|---|---|---|
| Memoria | Fija, pre-alocada | Crece con los elementos |
| `Desacolar` (Cola) | **O(n)** — desplaza | O(1) — mueve puntero |
| Capacidad | Limitada (1.1M) | Sin límite práctico |
| Overhead | Bajo | Mayor (objeto `Nodo` por elemento) |

La **implementación dinámica** se usa en el sistema real (`DynamicColaTDA`, `DynamicPilaTDA`, `DynamicConjuntoTDA`).

---

### 9.3 ¿Por qué GrafoLA y no GrafoMA para la red social?

Las redes sociales son grafos **dispersos**: con 1000 usuarios y máximo 2 seguidos + pocos amigos, E << V².

- GrafoMA: O(V²) de memoria, `InicializarGrafo` O(n²), `AgregarArista` O(V).
- GrafoLA: O(V + E) de memoria, `InicializarGrafo` O(1), `AgregarArista` O(1).

**Optimización adicional:** GrafoLA usa un `HashMap<Integer, NodoGrafo>` interno. Sin él, encontrar un vértice sería O(V); con él es O(1).

---

### 9.4 ¿Por qué dos grafos separados?

- **Seguimiento** → relación asimétrica (como Twitter). `GrafoLA` **dirigido**.
- **Amistad** → relación simétrica. `GrafoLA` **no dirigido** (se agregan dos aristas: A→B y B→A).

Mantenerlos separados permite calcular distancias independientemente y aplicar reglas distintas (ej: límite de 2 seguidos solo aplica al dirigido).

---

### 9.5 ¿Por qué Lista por usuario para solicitudes y no cola global?

**Problemas de la cola global:**
1. Ningún caso de uso necesita ver las solicitudes de todos los usuarios a la vez.
2. El usuario elige qué solicitud aceptar **por índice**, no por orden de llegada FIFO.
3. `listarPendientesParaUsuario` requería O(n_global) recorriendo y reconstruyendo la cola entera.

**Solución: `List<SolicitudSeguimiento>` dentro de cada `Cliente`:**
- `listarSolicitudesRecibidas` → O(1), acceso directo.
- `aceptarSolicitudAmistad(receptor, indice)` → O(n) `ArrayList.remove(int)`.
- Escala con el volumen del usuario, no del sistema.

---

### 9.6 ¿Por qué Pila para el historial?

El deshacer necesita la acción más **reciente** primero → LIFO. Con una Cola (FIFO) se obtendría la más antigua, que es incorrecto.

---

### 9.7 ¿Por qué el modelo `Cliente` tiene la lista de solicitudes mutable?

Las solicitudes de amistad son datos **propios del receptor**; su ciclo de vida está atado al usuario. Es más cohesivo almacenarlas dentro del objeto `Cliente` con métodos controlados (`agregarSolicitudRecibida`, `eliminarSolicitudRecibidaPorIndice`) que en una estructura externa.

---

### 9.8 ¿Por qué dos pasadas en `CargadorClientesJson`?

Primera pasada: crear todos los clientes (vértices).  
Segunda pasada: crear relaciones (aristas).

Si se hiciera en una pasada, se intentaría agregar una arista a un vértice inexistente. `GrafoLA.AgregarArista` silencia esa situación (`if (n1 == null || n2 == null) return`), pero el dato se perdería.

---

### 9.9 ¿Por qué BFS para calcular distancias?

BFS garantiza el **camino más corto** en grafos con pesos uniformes. DFS no lo garantiza. La distancia en "saltos" es exactamente el nivel BFS al que se alcanza el destino.

---

## 10. Patrones de diseño aplicados

### Builder — construcción de menús

`MenuBuilder` construye `Menu` con API fluida (encadenamiento de métodos). Separa la construcción de la representación.

```java
new MenuBuilder("RED SOCIAL")
    .agregarOpcion("1", "Buscar", scanner -> buscar())
    .setOpcionSalida("0")
    .build();
```

### Facade — servicios de negocio

`GestorClientes` y `HistorialAcciones` ocultan la complejidad interna de los TDAs y exponen una API simple al `MenuRedSocial`. Dependen de la interfaz TDA, no de la implementación concreta.

### Strategy / Interfaces

`ClientesTDA`, `HistorialAccionesTDA` son interfaces. Se pueden intercambiar implementaciones sin modificar los servicios que las usan.

### Template Method — bucle del menú

`Menu.ejecutarUnaVez()` define el algoritmo general (mostrar → leer → ejecutar), y las acciones concretas se inyectan como `Runnable` o `Consumer<Scanner>`.

---

## 11. Persistencia y entrada/salida

### clientes.json

- **Lectura:** `CargadorClientesJson.readFromFile(GestorClientes)` — O(n + e)
- **Escritura:** `GuardadorClientesJson.guardar(GestorClientes)` — O(n + e)
- Usa **Gson** para serialización/deserialización.
- Las relaciones se obtienen de los grafos, no del objeto `Cliente`.
- Se guarda automáticamente al salir del sistema.

### acciones.csv

- **Exportación:** `ExportadorAccionesCsv.exportar(List<Accion>, String)` — O(n)
- Formato: `tipo,detalle,fechaHora`. Hace append si el archivo ya existe.
- Se exporta automáticamente al salir.

---

## 12. Preguntas frecuentes del profesor

### Sobre la consigna

**¿Cómo se implementaron las relaciones generales (Iteración 3)?**  
Con un `GrafoLA` no dirigido. Al agregar una amistad se crean dos aristas (A→B y B→A). `obtenerVecinos` llama a `ObtenerAdyacentes` que es O(grado). En redes dispersas el grado es una constante pequeña, lo que cumple el objetivo O(1) indicado en la consigna.

**¿Cómo se calcula la distancia entre clientes?**  
Con BFS sobre el grafo correspondiente. El BFS explora nivel por nivel usando una cola, garantizando el camino más corto. La distancia se devuelve en saltos; -1 si no hay camino. Complejidad O(V + E).

**¿Cómo se extiende la persistencia para las relaciones generales?**  
El archivo `clientes.json` tiene el campo `conexiones` (array de nombres). `CargadorClientesJson` las carga en la segunda pasada llamando a `gestor.agregarAmistad(nombre, conexion)`. `GuardadorClientesJson` las serializa consultando `gestor.obtenerVecinos(nombre)`.

**¿Qué TDA modela las relaciones generales?**  
`GrafoTDA`. La interface define las operaciones (AgregarVertice, AgregarArista, ObtenerAdyacentes, etc.) con complejidades documentadas. La implementación `GrafoLA` es la elegida para la aplicación.

---

### Sobre TDAs

**¿Qué es un TDA?**  
Especificación de datos + operaciones independiente de la implementación. Define el "qué" (interfaz), no el "cómo" (implementación).

**¿Diferencia entre Static y Dynamic en Cola?**  
`StaticColaTDA.Desacolar` = O(n) porque desplaza el arreglo. `DynamicColaTDA.Desacolar` = O(1) porque mueve el puntero `primero`.

**¿Por qué `Agregar` en Conjunto es O(n)?**  
Porque llama a `Pertenece(x)` antes de insertar para mantener el invariante de no duplicados. `Pertenece` recorre la estructura linealmente.

**¿Por qué `GrafoMA.InicializarGrafo` es O(n²)?**  
Aloca una matriz n×n donde n = 1.100.000. Completamente inviable en práctica — ilustra la desventaja de la MA para grafos grandes.

**¿Por qué GrafoLA usa HashMap interno si es un TDA propio?**  
El HashMap es una optimización de implementación válida. El TDA define el "qué"; la implementación decide el "cómo". Sin el HashMap, `Vert2Nodo` sería O(V); con él es O(1).

---

### Sobre arquitectura y decisiones

**¿Por qué hay cola de solicitudes (TDA) si la app usa listas por usuario?**  
El TDA `ColaSolicitudesSeguimiento` existe como implementación correcta y testeada de la Cola aplicada a un dominio. Se conserva como evidencia del trabajo en el TDA. En el flujo de la app se optó por lista per-usuario por las razones de semántica y eficiencia ya explicadas.

**¿Por qué `StaticClientesTDA` usa `HashMap` y `TreeMap` de Java?**  
Para O(1) y O(log n) en búsquedas críticas. Los TDAs propios son O(n). Usar colecciones Java para casos de eficiencia es una decisión de ingeniería correcta y justificada.

**¿Cómo funciona el deshacer?**  
Cada acción registra en la Pila un objeto `Accion` con tipo y detalle. Al deshacer se desapila y según el tipo se ejecuta la operación inversa: "Registrarse" → eliminar cliente, "Seguir" → quitar seguido, "Enviar solicitud" → revocar solicitud del receptor, "Aceptar solicitud" → eliminar amistad, "Rechazar solicitud" → re-enviar la solicitud.

---

### Preguntas trampa

**¿Puede `consultarConexionesNivel4` devolver duplicados?**  
Sí. Si dos clientes tienen el mismo scoring y ambos están en el nivel 4 del ABB, el scoring aparece dos veces. El ABB inserta iguales en el subárbol derecho, no deduplica.

**¿Qué pasa si se elimina un cliente con solicitudes pendientes enviadas a otros?**  
`eliminarCliente` recorre todos los clientes restantes y llama a `eliminarSolicitudRecibida` con el nombre del cliente borrado como origen. Esto limpia las solicitudes de la lista de los receptores.

**¿Qué pasa si dos clientes envían solicitud al mismo usuario?**  
Ambas se agregan a la lista del receptor en orden de llegada. Aparecen numeradas en pantalla; el receptor elige cuál aceptar/rechazar.

**¿Puede un cliente seguir a alguien que ya es su amigo?**  
Sí. Seguimiento y amistad son relaciones independientes en grafos distintos. Ser amigo no implica seguirse, y seguirse no implica ser amigos.

**¿Por qué `eliminarCliente` es O(V) y no O(1)?**  
Porque eliminar el vértice del grafo requiere recorrer las listas de adyacencia de todos los demás nodos para borrar las aristas entrantes. Además se limpian las solicitudes pendientes del cliente eliminado en otras listas.

---

*Documento generado para preparación del examen oral — Programación II, UADE 2026*

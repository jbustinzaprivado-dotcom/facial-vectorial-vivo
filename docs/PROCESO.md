# PROCESO.md — Registro vivo de decisiones

Este documento registra el proceso del proyecto: qué se decidió, por qué, y qué dudas se
resolvieron en el camino. Ninguna decisión se borra — si una decisión posterior corrige a una
anterior, se anota aquí cuál corrige a cuál y por qué. Las secciones `[Añadido]` marcan
decisiones que surgieron durante el desarrollo y no estaban en el pedido original.

---

## Fase 0 — Documentación base (antes de iniciar el desarrollo)

### Objetivo del proyecto

Desarrollar, para **BeatC**, un sistema que identifique automáticamente, en tiempo real y sin
intervención humana, a personas previamente registradas en una base de datos, mediante el
análisis vectorial de su rostro capturado por una cámara en vivo, registrando tanto su hora de
entrada como de salida a su instalación.

### D1 — Mecanismo de reconocimiento: análisis vectorial, no un sistema que aprende

Punto de diseño explícitamente aclarado por el equipo y central para todo lo que sigue: este
proyecto **no** es un sistema de aprendizaje automático que entrena o se ajusta con el uso.

Hay dos cosas distintas que conviven bajo el nombre "reconocimiento facial", y conviene no
confundirlas:

1. **Convertir un rostro en un vector (embedding).** Esta parte sí usa redes neuronales
   profundas (detección con YuNet, extracción de características con SFace) — pero son modelos
   **ya entrenados por terceros**, de fábrica. El proyecto no los entrena ni los reentrena: se
   usan como una caja cerrada y fija que siempre convierte "este rostro" en "este vector de
   números", igual hoy que en el futuro.

2. **Decidir si dos rostros son la misma persona.** Esto es álgebra lineal pura, no aprendizaje
   automático: se calcula la **similitud coseno** (un producto punto entre vectores
   normalizados) entre el vector capturado en vivo y el vector guardado de la persona
   registrada, y se compara ese único número contra un **umbral fijo**. Si lo supera, coincide;
   si no, no coincide. No hay clasificador entrenado, no hay probabilidad calculada por un
   modelo, no hay aprendizaje.

**Queda fuera de alcance, de forma deliberada:** cualquier componente que aprenda con el uso del
sistema — sin entrenamiento, sin calibración de probabilidad, sin un umbral que se ajuste solo
con los datos acumulados. El umbral es una constante que se define una vez (y se puede afinar
manualmente si hace falta), no algo que el sistema aprenda.

### Requisitos funcionales

**RF1 — Registro previo de personas**
El sistema debe contar con una base de datos de personas previamente registradas, cuyo rostro
fue procesado y almacenado como un vector biométrico (embedding) de referencia. Ese vector es la
base contra la cual se compara, en tiempo real, cualquier rostro capturado por la cámara.

**RF2 — Identificación automática en tiempo real**
El sistema debe analizar de forma continua el flujo de video de una cámara, detectar rostros y
compararlos —mediante operaciones vectoriales— contra los vectores almacenados. Junto a la
cámara debe existir una pantalla que muestre en vivo el rostro capturado y el resultado de la
verificación.

**RF3 — Procesamiento de una persona a la vez**
El sistema procesa exclusivamente un rostro a la vez. Es una limitación intencional: garantiza
un flujo completamente automático y secuencial, sin selección manual entre varias personas
frente a la cámara.

Flujo esperado:
1. La persona se ubica frente a la cámara.
2. Ve su propio rostro en el monitor contiguo a la cámara (retroalimentación visual en vivo).
3. El sistema analiza el rostro y lo compara contra la base de datos.
4. Si hay coincidencia, se muestra en pantalla: nombre, DNI, tipo de evento registrado (entrada
   o salida — ver RF6) y fecha/hora exacta del análisis.
5. Transcurridos 3 segundos desde que se muestra el resultado, el sistema vuelve
   automáticamente a su estado de espera inicial, listo para la siguiente persona.

**RF4 — Manejo de rostros no registrados**
Si el rostro detectado no coincide con ningún registro de la base de datos, el sistema lo marca
como "no registrado" y retorna automáticamente a su estado de espera inicial, sin intervención
manual.

**RF5 — Tiempo mínimo de enfoque (precisión)**
La persona debe mantener el rostro enfocado frente a la cámara durante al menos **5 segundos**
continuos antes de que el sistema emita un resultado (coincide / no registrado). Este umbral
reduce falsos negativos por movimiento, desenfoque o ángulos inadecuados durante la captura.

**RF6 — Registro de entrada y salida**
Además de identificar a la persona, el sistema debe registrar el tipo de evento correspondiente
a cada reconocimiento exitoso:
- El primer reconocimiento exitoso del día para una persona se registra como **entrada**.
- Un reconocimiento exitoso posterior, el mismo día, para esa misma persona, se registra como
  **salida**.
La pantalla debe reflejar claramente cuál de los dos eventos fue registrado, junto con nombre,
DNI y fecha/hora.

**RF7 — Cierre automático de jornada**
Si una persona registró su entrada pero no se logra capturar su salida durante el resto del día,
el sistema debe registrarla automáticamente a las **11:00 p.m. (23:00)**, usando esa hora como
marca de cierre de jornada.

**RF8 — Tipos de usuario / roles**
El sistema debe reconocer distintos tipos de persona, cada uno con sus propias características
de permanencia en la instalación:
- **Practicantes:** realizan una jornada diaria de 2 a 12 horas. Su información se gestiona y
  actualiza a través de un sistema externo ya existente, con el cual este proyecto se integra
  (ver D9).
- **Clientes del curso:** personas que pagaron un curso específico y permanecen entre 2 y 8
  horas dentro del local mientras lo cursan.

Ambos tipos de persona conviven dentro de la misma instalación de **BeatC**: los clientes de
curso asisten a aprender el curso que pagaron, mientras que los practicantes desarrollan
software, diseño u otras funciones de apoyo a la empresa.

> **Corrección posterior a RF8:** los rangos horarios se ampliaron de 4-8h a **2-12h**
> (practicantes) y de 4-6h a **2-8h** (clientes del curso), para cubrir casos de jornadas más
> cortas o más largas que las inicialmente previstas.

### D2 — Modelo de despliegue: kiosco local dedicado

El reconocimiento facial corre en un equipo físico instalado en el local (kiosco local
dedicado), conectado directamente a la cámara y al monitor — no en un backend remoto al que el
navegador le envía cada frame. Se descarta el modelo "todo en la nube vía navegador" (el que sí
tenía sentido en un proyecto anterior de captura puntual por clic) porque aquí la cámara analiza
video **continuo**: depender de la red para cada frame introduciría latencia y fragilidad
innecesarias en un proceso que debe ser automático y fluido.

### D3 — Stack tecnológico

**Motor de reconocimiento — Python + OpenCV (YuNet + SFace).** Se evaluaron alternativas antes
de decidir, no se reutiliza por comodidad:

| Opción | Evaluación |
|---|---|
| **Python + OpenCV (YuNet+SFace)** | Motor ya validado en un proyecto anterior del equipo, pesos ya disponibles, código de embeddings/similitud coseno ya probado. `cv2.VideoCapture` lee directo de una cámara local en loop continuo — encaja mejor aquí que en un escenario de navegador-a-nube, porque no hay que viajar por red en cada frame. |
| MediaPipe / InsightFace / DeepFace | Requerirían revalidar un motor nuevo sin ninguna ventaja concreta sobre el que ya funciona. Descartadas. |
| C++ nativo con OpenCV | Más rápido en teoría, pero sin base de código ni experiencia previa del equipo en este proyecto; el costo de desarrollo no se justifica para procesar una persona a la vez (no es un escenario de alta concurrencia). Descartado. |
| Node.js (face-api.js, onnxruntime-node) | Ecosistema de visión por computadora mucho más débil que Python. Descartado. |

Para la pantalla del kiosco: el mismo proceso Python dibuja el overlay (rostro + resultado)
directamente sobre el frame (rectángulos, texto) y lo muestra en una ventana a pantalla
completa — sin frameworks de UI aparte (PyQt, Tkinter) ni navegador. Menos piezas, mismo
resultado.

**Backend central — FastAPI + PostgreSQL.** Gestiona personas, roles, eventos de
entrada/salida, el cierre automático de las 23:00, y la integración con la API de practicantes.
Mismo stack ya dominado por el equipo en proyectos anteriores; nada en los requisitos lo pone en
duda.

### D4 — Arquitectura general

```
┌─────────────────────────┐         ┌──────────────────────────┐
│   KIOSCO LOCAL (local)   │         │   BACKEND CENTRAL (nube)  │
│                          │         │                          │
│  Cámara → loop Python    │  sync   │  FastAPI + PostgreSQL     │
│  (detección + embedding  │ ◄─────► │  - personas, roles        │
│   + comparación local)   │ periód. │  - eventos entrada/salida │
│                          │         │  - cierre automático 23h  │
│  Monitor: overlay en     │  POST   │                          │
│  vivo (rostro+resultado) │ evento  │                          │
└─────────────────────────┘         └───────────┬──────────────┘
                                                 │ API
                                                 ▼
                                   ┌──────────────────────────┐
                                   │ Sistema externo (practic.)│
                                   │  API disponible            │
                                   └──────────────────────────┘
```

**Decisión clave:** el kiosco **nunca envía video ni fotos** por red. Descarga periódicamente la
lista de vectores (embeddings) de personas registradas desde el backend central y la mantiene
en caché local — la comparación por coseno ocurre con latencia cero, sin depender de la red
frame a frame (mismo criterio de privacidad de proyectos anteriores: solo viajan vectores,
nunca imágenes). El kiosco solo llama a la red para **registrar un evento** (entrada o salida),
algo poco frecuente comparado con el análisis de video.

### D5 — Hardware recomendado

Mini-PC x86 (tipo Intel NUC o equivalente) con Linux, cámara USB 1080p, monitor contiguo. Se
descartan Raspberry Pi/placas ARM para esta primera versión: correr YuNet+SFace en ARM es
notablemente más lento y añade fricción de compilación de OpenCV — un mini-PC x86 es más simple
y predecible en rendimiento, y el costo adicional es bajo frente al riesgo de producción.

### D6 — Máquina de estados del kiosco

```
ESPERA (sin rostro detectado)
   │  aparece un rostro
   ▼
ENFOQUE (cronómetro de 5s — RF5)
   │  rostro desaparece antes de 5s → vuelve a ESPERA
   │  rostro sostenido 5s completos
   ▼
ANÁLISIS (comparación vectorial contra caché local)
   │
   ├─ coincide → ¿ya tiene entrada hoy sin salida?
   │      no  → registra ENTRADA
   │      sí  → registra SALIDA
   │      pantalla: nombre, DNI, tipo de evento, fecha/hora
   │
   └─ no coincide → pantalla: "no registrado"
   │
   ▼
RESULTADO EN PANTALLA (3 segundos — RF3)
   │
   ▼
vuelve a ESPERA
```

**D10 — Selección de rostro cuando hay más de uno en cámara (confirmado con el equipo):** si
aparecen varios rostros frente a la cámara al mismo tiempo, el sistema procesa el **más
prominente** (mayor tamaño / más cercano a la cámara) y lo analiza, ignorando los demás hasta
que vuelva a quedar uno solo. No se descarta esta situación como imposible por el diseño físico
del kiosco — el software debe manejarla explícitamente.

### D7 — Modelo de datos (esbozo, sujeto a diseño detallado en la siguiente fase)

- **personas**: id, nombre, dni, tipo (`practicante` | `cliente_curso`), rostro_embedding, activo
- **eventos_asistencia**: id, persona_id, tipo (`entrada`|`salida`), marca_tiempo,
  cierre_automatico (bool)
- **cierre automático**: tarea programada diaria a las 23:00 que busca entradas del día sin
  salida y las cierra con `cierre_automatico=true`

### D8 — Resiliencia sin conexión

Si el kiosco pierde conexión con el backend central, debe seguir reconociendo (usa su caché
local de embeddings) y encolar los eventos pendientes en disco local, reintentando el envío
cuando vuelva la conexión. Nunca debe detenerse por un corte de red momentáneo.

### D9 — Integración con el sistema externo de practicantes

El sistema externo donde se gestionan los practicantes **tiene API disponible** (confirmado por
el equipo). El backend central sincroniza periódicamente (o vía webhook, si el sistema externo
lo soporta) la lista de practicantes activos y sus datos. Autenticación de esa API y frecuencia
de sincronización quedan pendientes de definir en detalle cuando se tengan sus especificaciones
concretas (ver sección Pendiente).

### Cierre de la Fase 0

Ningún archivo de código fue escrito en esta fase — es documentación pura, previa al inicio del
desarrollo. El repositorio se creó vacío (`jbustinzaprivado-dotcom/facial-vectorial-vivo`) y este
documento consolida los requisitos funcionales y el análisis técnico acordados antes de empezar
a programar.

### Pendiente

- Especificaciones concretas de la API externa de practicantes (autenticación, endpoints,
  frecuencia de sincronización razonable).
- Selección final de hardware concreto (modelo exacto de mini-PC, modelo de cámara).
- Diseño detallado de base de datos (tablas, migraciones) — próxima fase.
- Plan de implementación por fases (scaffolding backend/kiosco, motor facial, integración,
  pruebas, despliegue).

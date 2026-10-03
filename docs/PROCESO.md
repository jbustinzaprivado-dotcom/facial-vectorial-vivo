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

**RF9 — Tiempo de espera entre registros (cooldown) [Añadido]**
Una vez que el sistema registra exitosamente a una persona (su entrada), debe esperar al menos
**30 minutos** desde ese registro antes de permitir un nuevo registro para esa misma persona —
ese segundo registro, cumplidos los 30 minutos, se guarda como su salida (RF6). Si la persona
vuelve a aparecer frente a la cámara **antes** de cumplirse los 30 minutos, el sistema sí la
reconoce (no la trata como desconocida), pero no genera un nuevo evento: en su lugar, muestra un
mensaje indicando que ya está registrada y la hora a partir de la cual puede volver a hacerlo
(confirmado con el equipo: no se usa el mensaje de "no registrado" para este caso, para no
confundir a la persona haciéndole creer que no está en el sistema).

**RF10 — Límite diario de registros [Añadido]**
Una persona no puede registrarse más de **3 veces** en el mismo día. Al alcanzar ese límite, el
sistema deja de reconocerla por el resto del día: cualquier intento posterior se trata y se
muestra igual que un rostro no registrado (RF4), sin importar que sí sea una persona conocida.
Solo cuentan para este límite los registros que efectivamente quedan guardados como evento
(confirmado con el equipo) — un intento bloqueado por el cooldown de RF9 no suma al conteo.

**RF11 — Pantalla publicitaria en reposo [Añadido]**
El segundo monitor está ubicado en la puerta del negocio: además de mostrar el reconocimiento,
funciona como señalización digital. En estado de espera (sin rostro detectado), la pantalla
muestra a pantalla completa videos publicitarios de BeatC en loop. En cuanto se detecta un
rostro, la pantalla cambia a pantalla completa de escaneo/resultado (RF2-RF3); al volver al
estado de espera, retoma la publicidad. Confirmado con el equipo: alternancia a pantalla
completa, no división fija de pantalla — mayor impacto visual en cada modo, sin dividir la
atención.

### D11 — RF9/RF10 se verifican en el backend central, no en el kiosco

Mismo criterio ya establecido para la decisión entrada/salida (sección de arquitectura del
análisis técnico): el kiosco solo reporta "persona X reconocida a las HH:MM:SS"; es el backend
central quien consulta el historial real de eventos de esa persona en el día y decide si el
registro se guarda, si corresponde mostrar el mensaje de cooldown (RF9), o si ya alcanzó el
límite diario (RF10). Esto evita inconsistencias si el kiosco estuvo offline y su estado local
quedó desactualizado.

### D12 — Verificar asistencia a una clase específica queda fuera de este sistema

Duda planteada por el equipo: el sistema registra entrada/salida de la instalación, pero no sabe
nada sobre cursos, horarios de clase ni matrícula — ¿hace falta otro sistema para determinar si
una persona asistió a la clase X?

**Respuesta: sí, es un concern distinto, y no es contradictorio dejarlo fuera de este sistema —
es la misma separación de responsabilidades que ya se aplica en D11.** Este sistema responde una
sola pregunta: "¿la persona P estuvo físicamente en la instalación entre tal hora y tal hora,
hoy?". Responde eso porque un kiosco de reconocimiento facial no tiene (ni debería tener) noción
de cursos, horarios ni matrícula — igual que ya se decidió que el kiosco no decide entrada/salida
(D11), ni aprende con el uso (D1). Determinar "¿asistió a la clase X?" exige cruzar estos eventos
de entrada/salida con datos que este sistema no tiene: el horario de esa clase, quién está
matriculado en ella, y una regla de negocio sobre cuánto tiempo dentro de esa ventana cuenta como
"asistió". Eso es trabajo de otro proceso — no necesariamente un sistema nuevo y separado en el
sentido de otro despliegue, pero sí una capa de interpretación distinta, construida sobre los
eventos que este sistema expone (vía su futura API de consulta, ver sección de pendientes del
análisis técnico), no dentro del propio motor de reconocimiento. Mantener a este sistema con un
rol único y acotado es justamente lo que permite que ese otro proceso (o sistema) se construya
después, sobre datos confiables, sin tener que reabrir ni expandir este.

### D13 — Determinar puntualidad (temprano/tarde) también queda fuera de este sistema

Pregunta del equipo, misma familia que D12: este sistema solo distingue dos resultados —
**registrado** o **no registrado** — y nunca dice si la persona llegó temprano o tarde. ¿Debería
resolverse eso en un sistema aparte, que tome la `marca_tiempo` del registro vectorial y la
compare contra el horario del cliente de curso/practicante?

**Respuesta: sí, viable, y es la extensión natural de D12.** Este sistema solo tiene un dato: el
instante exacto del evento (RF6/D7). No tiene, ni debería tener, la **hora esperada** contra la
cual comparar ese instante — y ese dato ni siquiera vive en un solo lugar: para practicantes está
en el sistema externo (RF8/D9); para clientes del curso, el horario de su curso no está modelado
en ningún lugar de este proyecto todavía. Calcular "temprano/tarde" exige cruzar ambas fuentes más
una regla de negocio (margen de tolerancia, por ejemplo), que es trabajo de interpretación, no de
reconocimiento. Generalizando D12: **cualquier lectura del evento entrada/salida contra un
calendario u horario externo (asistencia a una clase, puntualidad, horas trabajadas) es un
concern aparte, construido sobre los eventos que este sistema expone — nunca dentro del motor de
reconocimiento.**

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

---

## Fase 0b — Ampliación de contexto real: entorno, integración externa y restricciones de BeatC

Información que BeatC entregó después de cerrar la Fase 0, con implicancias reales de
arquitectura. Misma regla de siempre: no se borra nada de lo anterior — se anota qué decisión
corrige cada punto nuevo, y por qué.

### D14 — Entorno físico real: no es un kiosco dedicado — corrige D2

D2 asumía un equipo físico de un solo propósito. La realidad es distinta: la cámara y el segundo
monitor están conectados a la **PC de recepción**, una computadora de uso general que la
recepcionista ya usa para su propio trabajo en su monitor principal. Este sistema corre como
proceso secundario en esa misma PC, proyectando solo al segundo monitor (más alejado, de cara al
cliente escaneado).

Implicancias de diseño que esto agrega sobre D2:
- No puede asumir que tiene la PC para sí solo — debe correr con bajo consumo de CPU/memoria para
  no afectar el trabajo de la recepcionista en el monitor principal.
- No debe tomar el foco de teclado/mouse del sistema operativo ni interferir con otras ventanas —
  se limita a dibujar en su propia ventana, anclada al segundo monitor.
- Sigue siendo una sola máquina local (no cambia que el procesamiento es local, no en la nube),
  pero deja de ser una máquina *dedicada*.

### D15 — Arquitectura de verificación: sin caché local, el sistema externo decide — corrige D4

D4 asumía que este sistema cachea los vectores de referencia y compara localmente. BeatC planteó
un principio más estricto: **este sistema nunca accede a la base de datos del otro sistema** —
solo envía el vector recién capturado, y es el sistema externo quien compara y devuelve si
coincide o no.

| | Local (D4 original) | Remota (propuesta por BeatC) |
|---|---|---|
| Dónde compara | Este sistema, contra una caché propia | El sistema externo, contra su propia base |
| Copia de datos biométricos | Sí — este sistema guarda una copia | No — este sistema nunca la tiene |
| Funciona sin conexión | Sí (D8) | No — cada reconocimiento depende de la red |
| Requisito del sistema externo | Exponer sus vectores para descarga | Exponer un endpoint "verificar este vector" |

**Recomendación: la opción remota es la más consistente con el principio que BeatC planteó, y es
viable** — con dos condiciones pendientes de confirmar con quien mantiene los sistemas externos:
(1) que expongan un endpoint de verificación por vector (hoy no se sabe si ya existe), y (2)
aceptar que sin conexión a esas APIs, este sistema no puede reconocer a nadie — se pierde la
resiliencia offline de D8. **Pendiente de decisión final del equipo.**

### D16 — Dos sistemas externos, no uno — corrige D9 y RF8

D9 asumía un solo sistema externo (el de practicantes). BeatC confirmó que son **dos sistemas
separados, cada uno con su propia API**: uno para clientes del curso, otro para practicantes.
Ninguno de los dos es accesible directamente por este sistema (D15) — solo mediante su API.

> **Corrección a RF8:** donde decía que solo los practicantes se gestionan en un sistema externo,
> ahora: **ambos** tipos de persona (practicantes y clientes del curso) se gestionan en sistemas
> externos propios, cada uno con su propia API.

Consecuencia técnica directa de esto combinado con RF2/RF3 (sin filtro por DNI, reconocimiento
totalmente automático): como el sistema no sabe de antemano si el rostro capturado pertenece a un
cliente de curso o a un practicante, cada intento de reconocimiento necesitaría consultar **a las
dos APIs**, salvo que exista (o se construya) un punto de verificación único que cubra ambas
poblaciones. Pendiente de confirmar con quien mantiene esos dos sistemas.

### D17 — Lenguaje: PHP declarado como requisito; aislamiento vía API hace viable Python para el motor

BeatC indicó que el sistema debe estar desarrollado en PHP, por consistencia con el resto de su
ecosistema (también en PHP). Pregunta del equipo: ¿causaría problemas de compatibilidad construir
específicamente el motor de reconocimiento en otro lenguaje, dado que este sistema está aislado y
solo se comunica por API?

**Respuesta: no, ningún problema de compatibilidad — ese es precisamente el punto de un límite de
microservicio.** Si este sistema nunca comparte código, proceso ni base de datos con el resto del
ecosistema PHP, y solo intercambia HTTP, el lenguaje detrás de esa API es invisible para quien la
consume. El costo real no es técnico, es organizacional: un lenguaje adicional que el equipo debe
saber mantener, y un segundo pipeline de despliegue.

El costo real de forzar esto en PHP: la única vía para YuNet+SFace nativo en PHP es
`php-opencv`, una extensión nativa de nicho (confirmada que soporta estos modelos en una
investigación de otra conversación, pero con mucho menos mantenimiento y comunidad que el stack
Python ya validado en D3) — mayor riesgo de mantenimiento a largo plazo para un sistema
biométrico en producción.

**Recomendación:** motor de reconocimiento en Python (D3, sin cambios) detrás de una API HTTP;
el resto del ecosistema de BeatC permanece en PHP. **Pendiente de decisión final del equipo** —
si hay un mandato organizacional de PHP sin excepciones, `php-opencv` es viable pero con más
riesgo.

### Cierre de la Fase 0b

Documentación pura, igual que la Fase 0 — ningún archivo de código fue escrito.

### Pendiente

- Especificaciones concretas de las dos APIs externas (autenticación, endpoints, frecuencia de
  sincronización, y si exponen o no un endpoint de verificación por vector — D15).
- Decisión final: ¿verificación local con caché (D4) o remota sin caché (D15)? Recomendación
  dada, decisión pendiente del equipo.
- Decisión final: ¿motor en Python tras API, o todo en PHP con `php-opencv` (D17)? Recomendación
  dada, decisión pendiente del equipo.
- Si hace falta un punto de verificación único para las dos poblaciones (D16), o si este sistema
  debe consultar ambas APIs en cada intento.
- Selección final de hardware concreto — ahora en contexto de D14 (PC de recepción compartida,
  no un equipo dedicado).
- Fuente y formato de los videos publicitarios para RF11 (¿local, streaming, actualizable por
  BeatC?).
- Diseño detallado de base de datos (tablas, migraciones) — próxima fase.
- Plan de implementación por fases (scaffolding backend/kiosco, motor facial, integración,
  pruebas, despliegue).

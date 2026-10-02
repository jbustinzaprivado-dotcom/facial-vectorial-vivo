# CONTEXTO.md — Documento base de requisitos

Este documento reúne, en un solo lugar, los requisitos sobre los que se apoyará **todo** el
proyecto. Es la referencia a la que volver antes de tomar cualquier decisión técnica o de
implementación: si algo que se construye no sirve a uno de estos requisitos, o los contradice,
el documento manda sobre el código. El análisis técnico derivado de estos requisitos (stack,
arquitectura, hardware, modelo de datos) vive por separado en [`PROCESO.md`](./PROCESO.md).

---

## Planteamiento original

Una empresa busca un sistema de reconocimiento biométrico facial que analice el rostro de una
persona a tiempo real mediante una cámara, y que automáticamente registre quién es, cuando esa
persona ya fue registrada anteriormente en el sistema.

## Principio de diseño: análisis vectorial, no un sistema que aprende

Este es el punto central que define todo el proyecto, y no debe perderse de vista en ninguna
fase posterior:

- El sistema usa redes neuronales profundas **ya entrenadas por terceros** únicamente para
  convertir un rostro en un vector numérico (embedding). Esa parte es una caja cerrada y fija:
  el proyecto no la entrena ni la ajusta.
- La decisión de si dos rostros son la misma persona **no es aprendizaje automático**: es
  álgebra lineal pura — una comparación vectorial (similitud coseno / producto punto) contra un
  **umbral fijo**.
- **Queda fuera de alcance, de forma deliberada:** cualquier componente que aprenda o se calibre
  con el uso del sistema (sin entrenamiento, sin probabilidad calibrada por ML, sin un umbral
  que se ajuste solo con datos acumulados).

## Requisitos funcionales

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
  mediante API.
- **Clientes del curso:** personas que pagaron un curso específico y permanecen entre 2 y 8
  horas dentro del local mientras lo cursan.

Ambos tipos de persona conviven dentro de la misma instalación/empresa: los clientes de curso
asisten a aprender el curso que pagaron, mientras que los practicantes desarrollan software,
diseño u otras funciones de apoyo a la empresa.

## Resumen del ciclo operativo

```
Estado de espera (buscando rostro)
        ↓
Rostro detectado frente a cámara
        ↓
Enfoque sostenido ≥ 5 segundos (RF5)
        ↓
Análisis vectorial contra base de datos
        ↓
   ┌─────────────┴─────────────┐
Coincide                  No coincide
   ↓                           ↓
Registra entrada o        Muestra "no
salida según corresponda   registrado"
(RF6), muestra nombre,
DNI, fecha/hora
   └─────────────┬─────────────┘
                  ↓
         Espera 3 segundos (RF3)
                  ↓
      Vuelve al estado de espera
```

## Estado de este documento

Documento base de requisitos, cerrado antes de iniciar el desarrollo. Cualquier cambio futuro a
estos requisitos debe registrarse también en [`PROCESO.md`](./PROCESO.md), anotando qué requisito
cambia y por qué — nunca se sobreescribe en silencio.

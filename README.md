# Hermes Inventory

Un almacén pequeño de accesorios y equipos, llevado en la cabeza de una persona y en una hoja de cálculo. La idea era simple: que bastara una foto.

Le tomas una foto a la etiqueta, el modelo de visión propone el producto con su modelo, su serie y su cantidad, y tú confirmas o corriges. Lo que entra al inventario es lo que confirmaste, nunca lo que adivinó la máquina.

![Inicio](media/01-inicio.png)

## Cómo se captura, paso a paso

Son dos fotos y una confirmación. El teléfono no decide nada: propone, y quien captura manda.

**El escaneo.** El botón "Leer" abre el lector con su marco en pantalla y una instrucción poco poética: apunte al código dentro del marco. Lo que se lee se va al campo del producto sin teclear nada. El lector es local, con ML Kit, así que funciona igual de bien sin señal.

![Lector de código de barras](media/03-escaner.png)

**Los campos que se llenan solos.** De la etiqueta salen el nombre, la marca, el modelo, la categoría y la cantidad propuesta. Abajo, la app busca si ese producto ya tiene ficha y, si la encuentra, lo dice antes de que confirmes: la entrada se va a sumar a lo que ya hay en lugar de crear un duplicado. Siempre queda el botón de "Ficha nueva" para cuando de verdad es otro producto.

![Campos autocompletados y propuesta de sumar a la ficha existente](media/04-campos-autocompletados.png)

**Si el producto lleva serie, pide una por unidad.** Esta es la parte que más me gustó resolver. El modelo no decide por su cuenta que algo es serializado: la regla del prompt es estricta. Solo propone `SERIALIZED` cuando de verdad se leyeron números de serie legibles en la foto, y por cantidad en cualquier otro caso. Si detecta dos equipos con serie, aparecen dos campos, uno por unidad, cada uno con su botón "Leer" para escanear la etiqueta del equipo en vez de copiarla a mano.

![Un campo por serie, cada uno con su lector](media/05-serie-leida.png)

**Y antes de confirmar, ves a dónde va.** La suma a la ficha existente no queda escondida: aparece escrita en pantalla y el botón dice exactamente qué va a pasar.

![Confirmación con la ficha destino a la vista](media/06-series-prellenadas.png)

Hay un detalle que el modelo no decide: el modo entre por cantidad y por serie lo elige quien captura, no la ficha de destino. Eso costó una entrada perdida. Si la ficha existente era por cantidad, su modo pisaba el elegido y las series capturadas se descartaban en silencio. Ahora el modo viaja con la captura y el servidor lo respeta.

Y dos reglas más que parecen de sentido común y no lo son. La foto de la etiqueta nunca cuenta como unidad: si fotografías una etiqueta y cuatro cajas, son cuatro. Y si el empaque es un pack y el contenido no se puede confirmar, la cantidad vuelve vacía con un aviso, porque adivinar ahí es la forma más rápida de empezar a mentir.

## Por qué no es tan simple como suena

Porque una foto miente. La etiqueta está borrosa, la caja trae un pack de diez, el teléfono tiene dos IMEI, la serie ya existe, el modelo alucina una cantidad que nadie pidió. Y porque el almacén no tiene buena señal, así que capturar y confirmar tienen que funcionar sin internet y reconciliarse después sin duplicar nada.

La regla que sostiene todo el diseño: **el sistema nunca inventa un resultado**. Si la visión falla o el modelo devuelve basura, el trabajo queda como fallido y accionable, con las fotos guardadas y la razón escrita. Un inventario que miente es peor que una hoja de cálculo.

## Qué quedó funcionando

La app Android captura, revisa y confirma; el backend guarda productos, fotos, saldos y movimientos, y encola los análisis. El stock se mueve con estados (`AVAILABLE`, `ASSIGNED`, `DAMAGED`, `RMA`) y cada operación lleva una llave de idempotencia, así que reintentar por mala red no duplica entradas.

Los 18 casos de aceptación se cerraron con evidencia y con la procedencia declarada fila por fila: lo verificado de forma independiente, lo heredado de una corrida anterior y lo probado en el espejo. Tres pasadas de auditoría, dos de ellas pidiendo correcciones, hasta llegar a veredicto limpio. Entre los casos que más trabajo dieron:

- Serie o IMEI duplicado: rechazo `409` y ninguna fila nueva.
- Salida de 999 piezas sin saldo: `409 INSUFFICIENT_STOCK` y cero movimientos escritos.
- Foto capturada sin conexión, app cerrada a la fuerza y reabierta: el trabajo sobrevive en la cola y termina solo.
- Backend asesinado a media ejecución: el trabajo huérfano regresa a la cola, se reintenta y termina bien.
- Imagen con instrucciones maliciosas escritas encima: la orden no entra a los datos y el sistema avisa.

Y una cadena completa de estados probada de punta a punta: diez unidades que pasan a asignadas, dañadas, en RMA y de vuelta a disponibles, terminando donde empezaron.

![Historial](media/02-historial.png)

## Arquitectura

```mermaid
graph LR
  A["App Android<br/>Compose + Room + outbox"] -->|"HTTPS, red privada"| B["Inventory API<br/>FastAPI + SQLite"]
  B -->|"cola de trabajos"| C["Adaptador de visión"]
  C --> D["Modelo de visión principal"]
  C -.->|"respaldo"| E["Modelo local"]
  B --> F["Fotos y respaldo diario"]
```

## Decisiones que valen la pena

**La base de existencias es del inventario, no del agente.** El agente extrae datos y contesta consultas; no es la fuente de verdad del stock. Esa separación es la que permite caer el modelo sin perder el inventario.

**IDs creados en el cliente.** Los identificadores nacen en el teléfono, así que una captura hecha sin red tiene referencia estable desde el primer segundo y se reconcilia después sin adivinar a qué fila pertenece.

**"Pendiente" no es "existencia".** La app distingue explícitamente lo capturado de lo confirmado en servidor. Confundir los dos es cómo un inventario empieza a mentir sin que nadie lo note.

**Respaldo con restauración probada.** Ocho tablas, relaciones, saldos y fotos regresan idénticos, con `integrity_check: ok`. Respaldo que nunca se restauró es una carpeta con archivos, no un respaldo.

## Estado

Es un sistema de prueba, desarrollado y validado en laboratorio. No está desplegado de forma oficial en ningún lado, y así se documenta. Funciona de punta a punta en el teléfono y en el servidor de pruebas; las fases siguientes están en curso.

## Stack

Android con Kotlin, Compose, Room, WorkManager y CameraX; lectura local de códigos con ML Kit. Backend en Python 3.12 con FastAPI, Uvicorn y SQLite. Visión con un modelo en la nube y respaldo en un modelo local.

## Sobre el código

Repositorio de exhibición: arquitectura, criterios de aceptación y resultados medidos. No incluye APK, código de la app ni del backend, ni ningún dato real de inventario, número de serie o fotografía de etiquetas.

© 2026 Javier Cuevas. Todos los derechos reservados.

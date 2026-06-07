# 🤖 DeliveryBot — Sistema de Pedidos para Cafetería Institucional

**Automatización de pedidos vía Telegram + n8n + Google Sheets**

---

## ¿De qué se trata esto?

DeliveryBot nació como respuesta a un problema muy común en cafeterías de oficinas y universidades: las filas interminables, los pedidos mal tomados y la falta de trazabilidad. La idea fue simple — ¿qué tal si el usuario simplemente le escribe a un bot en Telegram y hace su pedido desde el celular?

El sistema usa **n8n** como motor de automatización, **Telegram** como interfaz de usuario y **Google Sheets** como base de datos. Nada de servidores costosos ni aplicaciones complicadas. Todo funciona con herramientas accesibles y gratuitas.

---

## ¿Qué puede hacer el bot?

- Mostrar el menú organizado por categorías (Bebidas, Comidas, Snacks)
- Agregar productos al carrito con la cantidad que quieras
- Calcular el total con IVA del 19% automáticamente
- Aplicar descuentos según los puntos de lealtad del usuario
- Confirmar el pedido y registrarlo en Google Sheets
- Notificar a la cocina cuando llega un pedido nuevo
- Permitir al administrador cambiar el estado del pedido desde Sheets
- Notificar al cliente cuando su pedido avanza de estado
- Mostrar el historial de pedidos del usuario
- Generar reportes diarios de ventas automáticamente cada mañana

---

## Tecnologías usadas

| Herramienta | Para qué |
|---|---|
| n8n Cloud | Motor de automatización y lógica de negocio |
| Telegram Bot API | Interfaz conversacional con el usuario |
| Google Sheets | Base de datos centralizada |
| ngrok | Túnel HTTPS para pruebas locales |
| JavaScript | Lógica dentro de los nodos Code de n8n |

---

## Estructura de la base de datos (DeliveryBot_DB)

La base de datos vive en Google Sheets y tiene 4 hojas:

### MENU
Aquí el administrador gestiona los productos disponibles.

![alt text](images/menu-googlesheets.png)

### PEDIDOS
Se llena automáticamente cuando alguien confirma un pedido.

![alt text](images/menu-googlesheets.png)

### USUARIOS
Registra automáticamente a cada usuario la primera vez que usa el bot.

![alt text](images/usuarios-googlesheets.png)

### SESSIONS
Guarda el estado de la conversación de cada usuario en tiempo real.

![alt text](images/sessions-googlesheets.png)

---

## Arquitectura del sistema

El sistema tiene 3 workflows en n8n:

```
DeliveryBot — Flujo Principal     →  Maneja toda la conversación y pedidos (38 nodos)
DeliveryBot — Monitor de Estados  →  Detecta cambios de estado en Sheets y notifica (4 nodos)
DeliveryBot — Reportes Diarios    →  Genera métricas de ventas cada mañana (4 nodos)
```

![alt text](<images/los 3 workflows.png>)

---

## Flujo Principal — Los 38 nodos explicados

![alt text](images/workflow-principal.png)

---

### NODO 1 — On message (Telegram Trigger)
El punto de entrada del sistema. Escucha todo lo que el usuario envía al bot, tanto mensajes de texto como clics en botones (callback_query).

```
Trigger On: Message + Callback Query
Credencial: Token del bot obtenido via @BotFather
```

![alt text](images/nodo-on-message.png)

---

### NODO 2 — obtener sesion (Google Sheets)
Busca en la hoja SESSIONS si el usuario tiene una sesión activa con carrito en curso, pantalla actual y producto pendiente de cantidad.

```
Operation: Get Rows
Sheet: SESSIONS
Filter: ID_TELEGRAM = {{ $json.message?.from?.id || $json.callback_query?.from?.id }}
```
![alt text](images/nodo-obtener-sesion.png)
---

### NODO 3 — contexto (Code)
El nodo más importante del flujo. Toma el mensaje crudo de Telegram y la sesión del usuario y construye un objeto limpio con todo lo que los demás nodos necesitan: telegram_id, nombre, texto, pantalla actual, carrito y ULTIMO_CAPITULO.

```javascript
const telegram_id = msg.message?.from?.id?.toString() 
  || msg.callback_query?.from?.id?.toString();
const texto = msg.message?.text || msg.callback_query?.data || '';
const carrito = sesion.CARRITO_TEMPORAL ? JSON.parse(sesion.CARRITO_TEMPORAL) : [];
return [{ json: { telegram_id, nombre, texto, pantalla_actual, carrito, ULTIMO_CAPITULO } }];
```
![alt text](images/nodo-contexto.png)
---

### NODO 4 — buscar usuario (Google Sheets)
Verifica si el usuario ya existe en la hoja USUARIOS antes de procesarlo.

```
Operation: Get Rows
Sheet: USUARIOS
Filter: ID_TELEGRAM = {{ $json.telegram_id }}
```
![alt text](images/nodo-buscar-usuario.png)
---

### NODO 5 — If
Decide si el usuario es nuevo o existente basándose en si tiene ID_TELEGRAM registrado.

```
Condition: String($json.ID_TELEGRAM) exists
True  → va directo a recuperar contexto
False → va a registrar usuario
```
![alt text](images/nodo-if.png)
---

### NODO 6 — registrar usuario (Google Sheets)
Registra automáticamente al usuario nuevo en la hoja USUARIOS con 0 puntos de lealtad y departamento "Sin asignar".

```
Operation: Append or Update Row
Sheet: USUARIOS
Column to match: ID_TELEGRAM
Campos: ID_TELEGRAM, NOMBRE_COMPLETO, DEPARTAMENTO = Sin asignar, PUNTOS_LEALTAD = 0
```

![alt text](images/nodo-registrar-usuario.png)

---

### NODO 7 — recuperar contexto (Code)
Une el contexto del usuario con sus puntos de lealtad actuales para que el nodo de procesar pedido pueda calcular descuentos correctamente.

```javascript
const ctx = $('contexto').first().json;
const usuario = $('buscar usuario').first().json;
return [{ json: { ...ctx, puntos_lealtad: parseInt(usuario.PUNTOS_LEALTAD) || 0 } }];
```
![alt text](images/nodo-recuperar-contexto.png)
---

### NODO 8 — Switch (Router Principal)
El cerebro del bot. Dependiendo de lo que el usuario escribió o en qué botón hizo clic, decide a qué rama del flujo ir. Tiene 10 reglas configuradas.

| Regla | Condición | Valor | Destino |
|---|---|---|---|
| 1 | equals | `/start` | msg inicio |
| 2 | equals | `ver_menu` | msg menu |
| 3 | starts with | `cat_` | obtener productos |
| 4 | starts with | `sel_` | preguntar cantidad |
| 5 | starts with | `add_` | obtener producto |
| 6 | equals | `ver_carrito` | mag carrito |
| 7 | equals | `confirmar_pedido` | verificar stock |
| 8 | equals | `mis_pedidos` | obtener pedidos |
| 9 | equals | `vaciar_carrito` | vaciar carrito |
| 10 | pantalla = | `esperando_cantidad` | procesar cantidad |

![alt text](images/nodo-switch.png)

---

### NODO 9 — msg inicio (Code)
Genera el mensaje de bienvenida con los 3 botones principales cuando el usuario escribe `/start`.

![alt text](images/msg-inicio.png)
![alt text](images/imagen-nodo-msginicio.jpeg)
---

### NODO 10 — msg menu (Code)
Muestra las 3 categorías del menú como botones inline: Bebidas, Comidas y Snacks.

![alt text](images/nodo-msgmenu.png)

---

### NODO 11 — obtener productos (Google Sheets)
Lee todos los productos de la categoría que el usuario eligió desde la hoja MENU.

```
Operation: Get Rows
Sheet: MENU
Filter: CATEGORIA = {{ $json.texto.replace('cat_', '') }}
```
![alt text](images/nodo-obtener-producto.png)
---

### NODO 12 — lista categoria (Code)
Toma los productos traídos por el nodo anterior y construye botones interactivos con el nombre, precio y stock de cada uno.

![alt text](images/nodo-lista-categoria.png)

---

### NODO 13 — obtener producto (Google Sheets)
Cuando el usuario va a agregar un producto al carrito, este nodo busca los datos completos de ese producto específico en la hoja MENU.

```
Operation: Get Rows
Sheet: MENU
Filter: ID_PRODUCTO = {{ $json.texto.split('_')[1] }}
```
![alt text](images/nodo-obtener-producto.png)
---

### NODO 14 — agregar carrito (Code)
Valida que haya stock disponible y agrega el producto al carrito con la cantidad seleccionada. Si el producto ya estaba en el carrito, incrementa la cantidad. Si es nuevo, lo agrega.

```javascript
if (parseInt(producto.STOCK) <= 0) {
  return [{ json: { mensaje: producto.NOMBRE + ' esta agotado.' } }];
}
const idx = carrito.findIndex(item => item.ID_PRODUCTO === producto.ID_PRODUCTO);
if (idx >= 0) {
  carrito[idx].cantidad += cantidad;
} else {
  carrito.push({ ID_PRODUCTO, NOMBRE, PRECIO, cantidad, subtotal });
}
```

![alt text](images/nodo-agregar-carrito.png)

---

### NODO 15 — mag carrito (Code)
Muestra el contenido actual del carrito con el detalle de cada producto, cantidad y subtotal. Si el carrito está vacío, avisa al usuario.

![alt text](images/msg-carrito.png)
---

### NODO 16 — verificar stock (Code)
Antes de confirmar el pedido, verifica que el carrito no esté vacío. Si está vacío, le avisa al usuario. Si tiene productos, deja pasar el flujo.

```javascript
const carrito = ctx.carrito || [];
if (carrito.length === 0) {
  return [{ json: { mensaje: 'Tu carrito esta vacio.', stock_ok: false } }];
}
return [{ json: { ...ctx, stock_ok: true } }];
```
![alt text](images/nodo-verificar-stock.png)

---

### NODO 16 — processar pedido (Code)
El nodo más complejo del flujo. Genera el ID único del pedido, calcula el subtotal, aplica el descuento por puntos de lealtad, calcula el IVA del 19% y suma los puntos ganados.

```javascript
const id_pedido = 'PED-' + Date.now();
const descuento = puntos >= 100 ? Math.floor(puntos / 100) * 0.10 : 0;
const descuento_valor = Math.round(subtotal * descuento);
const base = subtotal - descuento_valor;
const iva = Math.round(base * 0.19);
const total = base + iva;
const puntos_ganados = Math.floor(total / 1000);
```

![alt text](images/nodo-procesar-pedido.png)

---

### NODO 17 — guardar pedido (Google Sheets)
Registra el pedido confirmado en la hoja PEDIDOS con todos sus detalles y estado inicial "Recibido".

```
Operation: Append Row
Sheet: PEDIDOS
Campos: ID_PEDIDO, ID_TELEGRAM, DETALLES_PEDIDO, TOTAL_PAGO, ESTADO = Recibido, FECHA, HORA
```
![alt text](images/nodo-guardar-pedido.png)

---

### NODO 17 — notificar cocina (HTTP Request)
Envía una notificación automática al personal de cocina via Telegram con el ID del pedido y el total, para que sepan que hay un nuevo pedido entrante.

```json
POST https://api.telegram.org/bot{TOKEN}/sendMessage
{
  "chat_id": "ID_COCINA",
  "text": "Nuevo pedido: PED-xxx - Total: $9.500"
}
```

![alt text](images/nodo-notificar-cocina.png)
![alt text](images/imagen-notificar-cocina.jpeg)

---

### NODO 18 — actualizar puntos (Google Sheets)
Suma los puntos ganados en este pedido al total acumulado del usuario en la hoja USUARIOS. Por cada $1.000 pesos se gana 1 punto.

```
Operation: Append or Update Row
Sheet: USUARIOS
PUNTOS_LEALTAD = puntos_actuales + puntos_ganados
```
![alt text](images/nodo-actualizar-puntos.png)

---

### NODO 19 — extraer items pedidos (Code)
Desglosa el carrito en items individuales para poder procesar el descuento de stock de cada producto por separado.

```javascript
return carrito.map(item => ({
  json: { ID_PRODUCTO: item.ID_PRODUCTO, cantidad: item.cantidad, telegram_id }
}));
```
![alt text](images/nodo-extraer-items.png)
---

### NODO 20 — leer stock (Google Sheets)
Lee el stock actual de cada producto en la hoja MENU para poder calcular el nuevo valor después del descuento.

```
Operation: Get Rows
Sheet: MENU
Filter: ID_PRODUCTO = {{ $json.ID_PRODUCTO }}
Range: A:F
```
![alt text](images/nodo-leer-stock.png)
---

### NODO 21 — preparar stock (Code)
Calcula el nuevo valor de stock restando la cantidad comprada del stock actual leído en el nodo anterior.

```javascript
return items.map(item => ({
  json: {
    ID_PRODUCTO: item.json.ID_PRODUCTO,
    nuevo_stock: parseInt(item.json.STOCK) - item.json.cantidad
  }
}));
```
![alt text](images/nodo-preparar-stock.png)

---

### NODO 21 — recuperar datos (Code)
Recupera el mensaje y los datos de respuesta desde el nodo procesar pedido para enviárselos al usuario después de limpiar la sesión.

```javascript
const anterior = $('procesar pedido').first().json;
return [{ json: { telegram_id: anterior.telegram_id, mensaje: anterior.mensaje, reply_markup: anterior.reply_markup } }];
```
![alt text](images/nodo-recuperar-datos.png)

---

### NODO 22 — descontar stock (Google Sheets)
Actualiza el stock de cada producto en la hoja MENU restando la cantidad comprada.

```
Operation: Update Row
Sheet: MENU
Column to match: ID_PRODUCTO
STOCK = {{ $json.nuevo_stock }}
```

![alt text](images/nodo-descontar-stock.png)

---

### NODO 23 — limpiar sesion (Google Sheets)
Vacía el carrito en SESSIONS después de que el pedido fue confirmado y registrado.

```
Operation: Append or Update Row
Sheet: SESSIONS
CARRITO_TEMPORAL = []
PANTALLA_ACTUAL = inicio
```
![alt text](images/nodo-limpiar-sesion.png)

---

### NODO 24 — obtener pedidos (Google Sheets)
Lee todos los pedidos del usuario desde la hoja PEDIDOS filtrando por su ID de Telegram.

```
Operation: Get Rows
Sheet: PEDIDOS
Filter: ID_TELEGRAM = {{ $json.telegram_id }}
```
![alt text](images/nodo-obtener-pedidos.png)

---
### NODO 25 — preguntar cantidad (Code)
Cuando el usuario selecciona un producto, el bot le pregunta cuántas unidades quiere mediante un mensaje de texto. Guarda el ID del producto en la variable id_producto.

```javascript
const id_producto = ctx.texto.replace('sel_', '');
return [{ json: { telegram_id, id_producto, mensaje: 'Cuantas unidades quieres?\n\nEscribe un numero:' } }];
```

![alt text](images/nodo-preguntar-cantidad.png)

---

### NODO 26 — procesar cantidad (Code)
Recibe el número escrito por el usuario, recupera el ID del producto guardado en SESSIONS (ULTIMO_CAPITULO) y construye el callback `add_PRODUCTO_CANTIDAD` para que el Switch lo enrute correctamente.

```javascript
const cantidad = parseInt(ctx.texto);
const id_producto = ctx.ULTIMO_CAPITULO || '';
return [{ json: { ...ctx, texto: 'add_' + id_producto + '_' + cantidad } }];
```
![alt text](images/nodo-procesar-cantidad.png)

---

### NODO 27 — vaciar carrito (Code)
Permite al usuario limpiar su carrito manualmente antes de confirmar el pedido.

```javascript
return [{ json: { telegram_id, carrito_actualizado: [], mensaje: 'Carrito vaciado correctamente.' } }];
```
![alt text](images/nodo-vaciar-carrito.png)

---

### NODO 28 — guardar producto seleccionado (Google Sheets)
Cuando el bot pregunta la cantidad, guarda el ID del producto en la columna ULTIMO_CAPITULO de SESSIONS para no perderlo entre mensajes.
```
Operation: Append or Update Row
Sheet: SESSIONS
ULTIMO_CAPITULO = {{ $json.id_producto }}
PANTALLA_ACTUAL = esperando_cantidad
```
![alt text](images/nodo-guardar-producto.png)

---

### NODO 29 — guardar sesion vaciar (Google Sheets)
Persiste el carrito vacío en SESSIONS cuando el usuario elige vaciar el carrito manualmente.

```
Operation: Append or Update Row
Sheet: SESSIONS
CARRITO_TEMPORAL = []
PANTALLA_ACTUAL = inicio
```
![alt text](images/nodo-guardar-sesion-vaciar.png)

---

### NODO 30 — Code in JavaScript (recuperar datos vaciar)
Recupera el mensaje de confirmación desde el nodo vaciar carrito para enviárselo al usuario después de guardar la sesión.

```javascript
const anterior = $('vaciar carrito').first().json;
return [{ json: { telegram_id: anterior.telegram_id, mensaje: anterior.mensaje, reply_markup: anterior.reply_markup } }];
```
![alt text](images/nodo-code-javascript.png)

---

### NODO 31 — recuperar preguntar (Code)
Recupera el mensaje de preguntar cantidad para enviárselo al usuario después de guardar el producto seleccionado en SESSIONS.

```javascript
const anterior = $('preguntar cantidad').first().json;
return [{ json: { telegram_id: anterior.telegram_id, mensaje: anterior.mensaje, reply_markup: anterior.reply_markup } }];
```
![alt text](images/nodo-recuperar-preguntar.png)

---

### NODO 32 — guardar sesion (Google Sheets)
Persiste el carrito actualizado en SESSIONS cada vez que el usuario agrega un producto.

```
Operation: Append or Update Row
Sheet: SESSIONS
CARRITO_TEMPORAL = {{ JSON.stringify($json.carrito_actualizado) }}
```
![alt text](images/nodo-guardar-sesion.png)

---

### NODO 33 — unificar respuestas (Code)
Punto de convergencia de todos los flujos. Garantiza que el mensaje, el telegram_id y el reply_markup lleguen correctamente al nodo de envío sin importar por qué rama del flujo se llegó.

```javascript
const items = $input.all();
for (const item of items) {
  if (item.json.telegram_id && item.json.mensaje) {
    return [{ json: item.json }];
  }
}
```
![alt text](images/nodo-unificar-respuestas.png)

---

### NODO 34 — HTTP Request (Enviar mensaje)
Envía el mensaje final al usuario via la API oficial de Telegram. Se usa HTTP Request en lugar del nodo nativo de Telegram porque permite enviar el reply_markup como JSON completo.

```json
POST https://api.telegram.org/bot{TOKEN}/sendMessage
{
  "chat_id": "{{ $json.telegram_id }}",
  "text": "{{ $json.mensaje }}",
  "reply_markup": "{{ $json.reply_markup }}"
}
```

![alt text](images/nodo-http-request.png)

---


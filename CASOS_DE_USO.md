# Casos de Uso — Tienda Manorigen

> Documento vivo. Marcar con `[x]` los completados, `[ ]` los pendientes, `[~]` los en progreso.

---

## Módulo: Pedidos y Pagos

### CU-P01 — Crear pedido (guest)
**Actor:** Cliente sin cuenta (guest)
**Flujo:**
1. Cliente completa carrito y datos de envío en el front
2. Front envía `POST /pedidos` con items, dirección, email, método de pago
3. API valida precio de cada item contra Strapi
4. API valida stock de cada item contra Postgres
5. API calcula totales (subtotal, IVA, envío, total)
6. API crea pedido en estado `pendiente`
7. API retorna `numeroOrden` y `id`
**Estado:** [x] Implementado
**Notas:** Validación de precio contra Strapi, stock contra Postgres, recálculo de totales server-side

---

### CU-P02 — Crear pedido (usuario logueado)
**Actor:** Cliente con cuenta
**Flujo:**
1. Cliente inicia sesión (JWT)
2. Front envía `POST /pedidos` con `usuarioId` + items + dirección
3. API valida precio y stock (igual que guest)
4. API asocia pedido al usuario
5. API puede usar dirección guardada del usuario
**Estado:** [ ] Pendiente
**Dependencia:** Auth JWT (CU-U01)

---

### CU-P03 — Iniciar transacción Webpay
**Actor:** Cliente (guest o logueado)
**Flujo:**
1. Front envía `POST /transbank/init` con `numeroOrden`
2. API busca pedido por numeroOrden
3. API crea transacción en Transbank (Webpay Plus)
4. API guarda `token` y `url` en registro `pago_transbank`
5. API retorna `{ token, url }` al front
6. Front redirige a Webpay (`url` + `token_ws`)
**Estado:** [x] Implementado
**Notas:** Transbank SDK integration, guardado de token en DB

---

### CU-P04 — Confirmar pago (commit Webpay)
**Actor:** Transbank (callback) → Cliente (redirect)
**Flujo:**
1. Transbank hace `POST /transbank/commit` con `token_ws`
2. API confirma transacción con Transbank (`transaction.commit()`)
3. Si autorizada:
   a. Descuenta stock atómicamente en Postgres
   b. Actualiza pedido a estado `pagado`
   c. Guarda datos del pago (cardNumber, accountingDate, transactionDate)
   d. Redirige al front `/pago/resultado?status=autorizada`
4. Si rechazada:
   a. Actualiza pedido a estado `rechazado`
   b. Redirige al front `/pago/resultado?status=rechazada`
5. Si cancelada:
   a. Actualiza pedido a estado `cancelado`
   b. Redirige al front `/pago/resultado?status=cancelada`
**Estado:** [x] Implementado
**Notas:** Decremento atómico con QueryRunner, rollback si falla

---

### CU-P05 — Vista resultado de pago
**Actor:** Cliente
**Flujo:**
1. Cliente llega a `/pago/resultado?status=autorizada|rechazada|cancelada`
2. Front muestra mensaje según status
3. Si autorizada: muestra número de orden, detalles del pago
4. Botón para volver al inicio o seguir comprando
**Estado:** [x] Implementado

---

### CU-P06 — Tracking de pedido (guest)
**Actor:** Cliente sin cuenta
**Flujo:**
1. Cliente ingresa en `/seguimiento`
2. Ingresa `numeroOrden` + `emailContacto`
3. Front envía `GET /pedidos/guest/tracking?numeroOrden=...&emailContacto=...`
4. API valida que el email coincida con el del pedido
5. API retorna estado actual, items, total, fecha
6. Front muestra timeline de estados
**Estado:** [x] Implementado

---

### CU-P07 — Tracking de pedido (usuario logueado)
**Actor:** Cliente con cuenta
**Flujo:**
1. Cliente inicia sesión
2. Front envía `GET /pedidos/mis-pedidos` con JWT
3. API retorna lista de pedidos del usuario
4. Cliente puede ver detalle de cada pedido
**Estado:** [ ] Pendiente
**Dependencia:** Auth JWT (CU-U01)

---

### CU-P08 — Actualizar estado de pedido (admin)
**Actor:** Administrador
**Flujo:**
1. Admin envía `PATCH /pedidos/:id/estado` con nuevo estado
2. API valida transición de estado (máquina de estados)
3. Estados válidos: `pendiente → pagado → procesando → enviado → entregado`
4. Estados de error: `pendiente → rechazado`, `pendiente → cancelado`, `pagado → cancelado` (con restauración de stock)
5. API actualiza pedido y retorna nuevo estado
**Estado:** [x] Implementado
**Notas:** Máquina de estados con validación de transiciones

---

### CU-P09 — Cancelar pedido y restaurar stock
**Actor:** Administrador
**Flujo:**
1. Admin cambia estado a `cancelado`
2. API valida que el pedido esté en estado cancelable (`pendiente` o `pagado`)
3. Si estaba `pagado`:
   a. Restaura stock en Postgres (`restaurarStock`)
   b. Opcional: inicia reembolso en Transbank
4. Actualiza estado a `cancelado`
**Estado:** [x] Implementado (restauración de stock)
**Pendiente:** Reembolso Transbank (`transaction.refund()`)

---

### CU-P10 — Reembolso Transbank
**Actor:** Administrador
**Flujo:**
1. Admin solicita reembolso desde panel
2. API envía `transaction.refund(token, amount)` a Transbank
3. Transbank procesa reembolso a la tarjeta
4. API guarda registro del reembolso
5. Actualiza estado del pedido a `reembolsado`
**Estado:** [ ] Pendiente
**Dependencia:** Panel admin front

---

### CU-P11 — Email de confirmación de pago
**Actor:** Sistema (automático)
**Flujo:**
1. Pago confirmado (CU-P04 estado `pagado`)
2. API envía email al cliente con:
   - Número de orden
   - Detalle de productos
   - Total pagado
   - Método de pago (Webpay + últimos 4 dígitos)
   - Dirección de envío
   - Tiempo estimado de despacho
3. Email en formato HTML con branding
**Estado:** [ ] Pendiente
**Dependencia:** Configurar SMTP (`nodemailer` ya instalado)

---

### CU-P12 — Email de cambio de estado
**Actor:** Sistema (automático)
**Flujo:**
1. Admin cambia estado del pedido (CU-P08)
2. API envía email notificando al cliente:
   - Pedido `procesando`: "Tu pedido está siendo preparado"
   - Pedido `enviado`: "Tu pedido fue despachado" + link de seguimiento
   - Pedido `entregado`: "Tu pedido fue entregado"
   - Pedido `cancelado`: "Tu pedido fue cancelado" + motivo
**Estado:** [ ] Pendiente
**Dependencia:** CU-P11 (infraestructura de email)

---

### CU-P13 — Webhook de stock desde Strapi
**Actor:** Strapi CMS (automático)
**Flujo:**
1. Admin edita producto en Strapi (cambia stock, precio, etc.)
2. Strapi envía `POST /strapi/webhook` con evento + datos
3. API valida `Authorization: Bearer <secret>`
4. Según evento:
   - `entry.publish` / `entry.update` / `entry.create`: `upsertStock(id, stock)`
   - `entry.delete`: log informativo
5. Postgres actualizado
**Estado:** [x] Implementado
**Notas:** Requiere configurar webhook en Strapi Admin + `STRAPI_WEBHOOK_SECRET`

---

### CU-P14 — Sincronización manual de stock
**Actor:** Administrador
**Flujo:**
1. Admin envía `POST /stock/sincronizar`
2. API lee todos los productos de Strapi
3. Para cada producto: crea o actualiza registro en Postgres
4. Retorna resumen: `{ total, actualizados, creados }`
**Estado:** [x] Implementado

---

### CU-P15 — Consultar stock de un producto
**Actor:** Frontend / Admin
**Flujo:**
1. `GET /stock/:productoIdStrapi`
2. API retorna `{ productoIdStrapi, stock }`
**Estado:** [x] Implementado

---

### CU-P16 — Listar todo el stock
**Actor:** Frontend
**Flujo:**
1. `GET /stock`
2. API retorna array con todos los registros de stock
3. Front usa esto para mostrar stock real en vitrina
**Estado:** [x] Implementado

---

### CU-P17 — Actualizar stock manualmente
**Actor:** Administrador
**Flujo:**
1. Admin envía `POST /stock` con `{ productoIdStrapi, stock }`
2. API hace upsert (crea o actualiza)
3. Retorna registro actualizado
**Estado:** [x] Implementado

---

### CU-P18 — Manejo de stock insuficiente en checkout
**Actor:** Cliente
**Flujo:**
1. Cliente envía pedido con item cuyo stock es insuficiente
2. API valida stock en Postgres antes de crear el pedido
3. API retorna `400 Bad Request` con mensaje: "No hay stock suficiente para {producto}"
4. Front muestra error y permite ajustar cantidad
**Estado:** [x] Implementado

---

### CU-P19 — Manejo de cambio de precio en checkout
**Actor:** Cliente
**Flujo:**
1. Cliente envía pedido con precio desactualizado
2. API valida precio contra Strapi
3. Si el precio cambió: retorna `400 Bad Request` con mensaje del producto y precios
4. Front muestra error y permite recargar producto
**Estado:** [x] Implementado

---

### CU-P20 — Transacción fallida / timeout Webpay
**Actor:** Transbank / Cliente
**Flujo:**
1. Transbank no responde o hay timeout de red
2. API retorna error al front
3. Pedido queda en estado `pendiente`
4. Front muestra mensaje de error y opción de reintentar
5. Si el cliente no reintenta: pedido expira (TBD: cron job que cancele pedidos pendientes después de X minutos)
**Estado:** [x] Implementado (manejo de error)
**Pendiente:** Expiración automática de pedidos pendientes

---

### CU-P21 — Expiración automática de pedidos pendientes
**Actor:** Sistema (cron job)
**Flujo:**
1. Cron job ejecuta cada 15 minutos
2. Busca pedidos en estado `pendiente` con más de 30 minutos de antigüedad
3. Cambia estado a `cancelado` con motivo "expirado"
4. No restaura stock (no se descontó porque no se pagó)
**Estado:** [ ] Pendiente

---

## Módulo: Usuarios y Autenticación

### CU-U01 — Registro de usuario cliente
**Actor:** Visitante
**Flujo:**
1. Visitante completa formulario (nombre, apellido, rut, email, password, teléfono)
2. Front envía `POST /auth/register` con datos
3. API valida email y RUT únicos
4. API hashea password con bcrypt
5. Crea usuario en Postgres con `tipo: 'cliente'`
6. Retorna JWT + datos del usuario (sin password)
7. Front guarda token en localStorage
**Estado:** [x] Implementado
**Notas:** Columna `password` agregada a entidad Usuario (select: false). bcrypt hash en AuthService. POST /auth/register operativo

---

### CU-U02 — Login de usuario
**Actor:** Usuario registrado (cliente o admin)
**Flujo:**
1. Usuario ingresa email + password en `/login`
2. Front envía `POST /auth/login` con `{ email, password }`
3. API busca usuario por email en Postgres
4. API compara password con bcrypt
5. Si es válido: retorna `{ access_token, usuario: { id, nombre, email, tipo } }`
6. Front guarda token en localStorage
7. Front redirige según `tipo`: `cliente` → `/`, `admin` → `/admin/pedidos`
**Estado:** [x] Implementado
**Dependencias:** `@nestjs/jwt`, `bcrypt`, `passport-jwt`, `@nestjs/passport` instalados. JwtStrategy + JwtAuthGuard + AdminGuard creados

---

### CU-U02b — Logout de usuario
**Actor:** Usuario logueado
**Flujo:**
1. Usuario click en "Cerrar sesión"
2. Front elimina token de localStorage
3. Front redirige a `/`
**Estado:** [x] Implementado
**Notas:** JWT sin estado, logout limpia token y user de localStorage + store. Opcional: blacklist de tokens en Redis

---

### CU-U03 — Gestión de direcciones guardadas
**Actor:** Usuario logueado
**Flujo:**
1. Usuario puede guardar múltiples direcciones
2. `GET /direcciones` — listar direcciones del usuario
3. `POST /direcciones` — agregar dirección
4. `PATCH /direcciones/:id` — editar
5. `DELETE /direcciones/:id` — eliminar
6. En checkout, usuario puede seleccionar dirección guardada o ingresar nueva
**Estado:** [ ] Pendiente
**Dependencia:** Auth JWT (CU-U01)
**Notas:** Entidad `DireccionGuardada` ya existe en la API

---

### CU-U04 — Perfil de usuario
**Actor:** Usuario logueado
**Flujo:**
1. `GET /usuarios/perfil` con JWT
2. Retorna datos del usuario (nombre, email, teléfono)
3. `PATCH /usuarios/perfil` — actualizar datos
4. `PATCH /usuarios/password` — cambiar password
**Estado:** [ ] Pendiente

---

## Módulo: Panel Administrativo

### CU-A01 — Login admin
**Actor:** Administrador
**Flujo:**
1. Admin navega a `/admin/login`
2. Ingresa email + password
3. Front envía `POST /auth/login`
4. API valida credenciales y verifica `tipo === 'admin'`
5. Si no es admin: retorna `403 Forbidden`
6. Si es admin: retorna JWT + redirige a `/admin/pedidos`
**Estado:** [x] Implementado
**Dependencia:** CU-U02 (login genérico). AdminGuard verifica tipo === 'admin'. Frontend redirige a /admin/pedidos

---

### CU-A02 — Listar y filtrar pedidos
**Actor:** Administrador (con JWT)
**Flujo:**
1. Admin entra a `/admin/pedidos`
2. Front envía `GET /pedidos` con JWT en header `Authorization: Bearer <token>`
3. API valida JWT y que `tipo === 'admin'`
4. API retorna lista de pedidos con relaciones (items, direccion, pago)
5. Front muestra tabla con columnas: número, fecha, cliente, total, estado
6. Filtros: por estado (dropdown), búsqueda por número o email
7. Paginación (20 por página)
**Estado:** [x] Implementado
**Notas:** `findAll()` en PedidosService con relaciones. `GET /pedidos` protegido con JwtAuthGuard + AdminGuard. Frontend AdminPedidosView con tabla, filtros y detalle modal

---

### CU-A03 — Ver detalle de pedido
**Actor:** Administrador (con JWT)
**Flujo:**
1. Admin click en pedido de la tabla
2. Front envía `GET /pedidos/:id` con JWT
3. API retorna pedido con items, dirección, pago transbank
4. Front muestra vista detalle:
   - Datos del cliente (nombre, email, teléfono)
   - Items (producto, cantidad, precio, subtotal)
   - Dirección de envío completa
   - Datos del pago (monto, estado, código autorización, fecha)
   - Timeline de estados
5. Botones de acción según estado actual
**Estado:** [x] Implementado
**Notas:** Endpoint `GET /pedidos/:id` ya existía. Frontend muestra modal con items, dirección, pago y timeline de estados

---

### CU-A04 — Cambiar estado de pedido
**Actor:** Administrador (con JWT)
**Flujo:**
1. Admin en vista detalle de pedido
2. Click en botón de cambio de estado (ej: "Marcar como procesando")
3. Front envía `PATCH /pedidos/:id/estado` con `{ estado }` + JWT
4. API valida transición (máquina de estados)
5. API actualiza estado en Postgres
6. (Futuro) API envía email al cliente notificando cambio
7. Front actualiza vista con nuevo estado
**Estados y transiciones:**
- `pendiente` → `cancelado`
- `pagado` → `procesando` | `cancelado` (restaura stock)
- `procesando` → `enviado` | `cancelado`
- `enviado` → `entregado`
**Estado:** [x] Implementado
**Notas:** Endpoint cambiado a `PUT /pedidos/:id/estado` (PATCH causaba problemas CORS cacheados en navegador). Frontend con botones de transición según estado actual. `pagado` y `rechazado` solo los setea el flujo de Transbank, no manual

---

### CU-A05 — Dashboard de ventas
**Actor:** Administrador (con JWT)
**Flujo:**
1. Admin entra a `/admin/dashboard`
2. Front envía `GET /admin/dashboard` con JWT
3. API retorna métricas:
   - Ventas del día (total, cantidad)
   - Pedidos por estado (pendiente, pagado, procesando, enviado, entregado)
   - Ingresos del mes
   - Productos más vendidos (top 5)
4. Front muestra tarjetas con métricas + gráfico simple
**Estado:** [ ] Pendiente
**Dependencia:** CU-A02 (panel admin funcional)

---

### CU-A06 — Gestión de stock desde panel
**Actor:** Administrador (con JWT)
**Flujo:**
1. Admin entra a `/admin/stock`
2. Front envía `GET /stock` con JWT
3. Front muestra tabla: producto (nombre desde Strapi), stock actual, última actualización
4. Admin puede editar stock inline → `POST /stock` con `{ productoIdStrapi, stock }`
5. Botón "Sincronizar desde Strapi" → `POST /stock/sincronizar`
6. Alertas visuales: rojo si stock = 0, amarillo si stock < 5
**Estado:** [~] En progreso
**Notas:** Todos los endpoints ya existen (GET /stock, POST /stock, POST /stock/sincronizar). Webhook de Strapi configurado para sincronización automática. Falta front admin de stock + guard

---

## Módulo: Logística y Envíos

### CU-L01 — Cálculo de envío por región
**Actor:** Cliente
**Flujo:**
1. Cliente selecciona región/comuna en checkout
2. Front consulta `GET /envio/costo?region=...&comuna=...`
3. API retorna costo de envío según tabla
4. Front actualiza total del carrito
**Estado:** [ ] Pendiente
**Notas:** Hoy el costo de envío es fijo (5000)

---

### CU-L02 — Integración Chilexpress / Starken
**Actor:** Sistema
**Flujo:**
1. Pedido pasa a `enviado`
2. API genera etiqueta de envío con transportista
3. API obtiene número de seguimiento
4. Guarda tracking en el pedido
5. Email al cliente con link de seguimiento
**Estado:** [ ] Pendiente

---

## Módulo: Marketing y Promociones

### CU-M01 — Cupones de descuento
**Actor:** Cliente / Admin
**Flujo:**
1. Admin crea cupón (código, % descuento, vigencia, uso máximo)
2. Cliente ingresa cupón en checkout
3. API valida: vigencia, usos restantes, monto mínimo
4. Aplica descuento al total
5. Registra uso del cupón
**Estado:** [ ] Pendiente

---

### CU-M02 — Productos destacados / banners
**Actor:** Admin Strapi
**Flujo:**
1. Admin marca productos como destacados en Strapi
2. Front lee campo `destacado` de Strapi
3. Muestra carrusel de destacados en home
**Estado:** [ ] Pendiente

---

## Resumen

| Estado | Cantidad |
|--------|----------|
| [x] Implementado | 21 |
| [~] En progreso | 1 |
| [ ] Pendiente | 12 |
| **Total** | **34** |

### Fix aplicado
- `transbank.service.ts`: `Math.round(Number(pedido.total))` para convertir decimal string a int (Transbank requiere pesos sin decimales)

### Próximos recomendados (orden sugerido)

### Orden de implementación actual

**Fase 1 — Auth (fundación para panel admin)** ✅
1. CU-U01 — Registro de usuario cliente
2. CU-U02 — Login (cliente + admin)
3. CU-U02b — Logout

**Fase 2 — Panel admin** ✅
4. CU-A01 — Login admin
5. CU-A02 — Listar y filtrar pedidos
6. CU-A03 — Ver detalle de pedido
7. CU-A04 — Cambiar estado de pedido
8. CU-A06 — Gestión de stock desde panel (en progreso: backend listo, falta front)

**Fase 3 — Seguimiento cliente**
9. Vista `/seguimiento` — Formulario + consumo de `GET /pedidos/guest/tracking`

**Fase 4 — Mejoras**
10. CU-P11 — Email de confirmación de pago
11. CU-A05 — Dashboard de ventas
12. CU-P21 — Expiración automática de pedidos pendientes
13. CU-P10 — Reembolso Transbank
14. CU-L01 — Cálculo de envío por región

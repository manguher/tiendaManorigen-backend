# Flujo Técnico — Tienda Manorigen

> Documentación técnica de la interacción entre los tres artefactos del sistema.

---

## Arquitectura General

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Strapi CMS │     │  NestJS API      │     │  Vue.js Frontend │
│  (Catálogo)  │     │  (Backend)       │     │  (Tienda Lira)   │
│  Puerto 1337 │     │  Puerto 3000     │     │  Puerto 5173     │
│  PostgreSQL  │     │  PostgreSQL      │     │  Vite Dev Server │
│  (strapi DB) │     │  (tienda_pedidos)│     │                  │
└──────────────┘     └──────────────────┘     └──────────────────┘
       │                     │                        │
       │   Webhooks ────────▶│                        │
       │   (producto CRUD)   │                        │
       │                     │◀─── REST API ──────────│
       │◀─── HTTP GET ───────│                        │
       │   (validar precio)  │   (JWT Auth)           │
       │                     │                        │
```

### Responsabilidades por artefacto

| Artefacto | Responsabilidad | Puerto |
|-----------|----------------|--------|
| **Strapi CMS** | Catálogo de productos (CRUD admin), gestión de contenido, imágenes | 1337 |
| **NestJS API** | Pedidos, pagos (Transbank), stock, autenticación (JWT), usuarios | 3000 |
| **Vue.js Frontend** | UI tienda, carrito, checkout, login, panel admin, seguimiento | 5173 |

### Bases de datos

| Base de datos | Tablas principales | Uso |
|---------------|-------------------|-----|
| `tienda_manorigen` (Strapi) | productos, imagenes, categorias | Catálogo CMS |
| `tienda_pedidos` (NestJS) | usuarios, pedidos, pedidos_items, producto_stock, pagos_transbank, direcciones_guardadas, pedido_direccion_envio | Transaccional |

---

## Módulo: Autenticación (JWT)

### Flujo de autenticación

```
Frontend (Vue)                    NestJS API                    PostgreSQL
     │                               │                              │
     │  POST /auth/register          │                              │
     │  {nombre, email, password}   │                              │
     │──────────────────────────────▶│                              │
     │                               │  bcrypt.hash(password)       │
     │                               │  INSERT INTO usuarios        │
     │                               │─────────────────────────────▶│
     │                               │  JWT.sign({sub, email, tipo})│
     │  {access_token, usuario}      │                              │
     │◀──────────────────────────────│                              │
     │                               │                              │
     │  POST /auth/login             │                              │
     │  {email, password}            │                              │
     │──────────────────────────────▶│                              │
     │                               │  SELECT * FROM usuarios      │
     │                               │  WHERE email = ?             │
     │                               │─────────────────────────────▶│
     │                               │  bcrypt.compare(password)    │
     │                               │  JWT.sign({sub, email, tipo})│
     │  {access_token, usuario}      │                              │
     │◀──────────────────────────────│                              │
     │                               │                              │
     │  Guardar token en localStorage│                              │
     │  Axios interceptor:           │                              │
     │  Authorization: Bearer <token>│                              │
     │                               │                              │
     │  GET /pedidos (con token)     │                              │
     │──────────────────────────────▶│                              │
     │                               │  JwtAuthGuard: verify token  │
     │                               │  AdminGuard: tipo === 'admin'│
     │                               │  SELECT * FROM pedidos       │
     │                               │─────────────────────────────▶│
     │  [lista de pedidos]           │                              │
     │◀──────────────────────────────│                              │
```

### Componentes del módulo Auth

| Componente | Archivo | Función |
|------------|---------|---------|
| AuthService | `src/auth/auth.service.ts` | register() y login() con bcrypt |
| AuthController | `src/auth/auth.controller.ts` | POST /auth/register, POST /auth/login |
| JwtStrategy | `src/auth/strategies/jwt.strategy.ts` | Extrae y valida JWT del Bearer token |
| JwtAuthGuard | `src/auth/guards/jwt-auth.guard.ts` | Verifica que el request tenga JWT válido |
| AdminGuard | `src/auth/guards/admin.guard.ts` | Verifica que `user.tipo === 'admin'` |
| RegisterDTO | `src/auth/dto/register.dto.ts` | Validación de campos de registro |
| LoginDTO | `src/auth/dto/login.dto.ts` | Validación de email y password |

### Endpoints de autenticación

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| POST | `/auth/register` | Público | Registra usuario cliente, retorna JWT |
| POST | `/auth/login` | Público | Valida credenciales, retorna JWT |
| GET | `/pedidos` | JWT + Admin | Lista todos los pedidos (panel admin) |
| GET | `/pedidos/:id` | Público | Detalle de pedido por ID |
| PUT | `/pedidos/:id/estado` | JWT + Admin | Cambia estado de pedido |
| GET | `/pedidos/guest/tracking` | Público | Seguimiento por número + email |

### Frontend — Componentes de autenticación

| Componente | Archivo | Función |
|------------|---------|---------|
| auth API | `src/api/auth.js` | Cliente HTTP para /auth/login y /auth/register |
| auth store | `src/stores/auth.js` | Estado global (user, token, isAdmin), login(), register(), logout() |
| backendClient | `src/api/backendClient.js` | Axios con interceptor que adjunta JWT en headers |
| authGuard | `src/router/guards.js` | Protege rutas con `requiresAuth` y `requiresAdmin` |
| guestGuard | `src/router/guards.js` | Redirige usuarios autenticados fuera de rutas `guestOnly` |
| LoginView | `src/views/auth/LoginView.vue` | Formulario de login |
| RegisterView | `src/views/auth/RegisterView.vue` | Formulario de registro |
| AdminPedidosView | `src/views/admin/AdminPedidosView.vue` | Panel admin: listar, filtrar, ver detalle, cambiar estado |

### Roles de usuario

| Rol | tipo en DB | Acceso |
|-----|-----------|--------|
| Cliente | `cliente` | Login, registro, compra, seguimiento |
| Admin | `admin` | Todo lo anterior + panel admin (GET /pedidos, PUT /pedidos/:id/estado) |

### Variables de entorno (JWT)

```env
JWT_SECRET=manorigen-jwt-secret-dev-2026
JWT_EXPIRES=24h
```

---

## Módulo: Productos y Stock

### Flujo de sincronización de stock

```
Strapi Admin                NestJS API                    PostgreSQL
     │                          │                              │
     │  Admin crea/edita        │                              │
     │  producto con stock       │                              │
     │                          │                              │
     │  Webhook POST            │                              │
     │  /strapi/webhook         │                              │
     │  {event, model, entry}   │                              │
     │─────────────────────────▶│                              │
     │  Authorization: Bearer   │                              │
     │  <STRAPI_WEBHOOK_SECRET> │                              │
     │                          │  upsertStock(entry.id, stock)│
     │                          │─────────────────────────────▶│
     │                          │  INSERT/UPDATE producto_stock│
     │                          │                              │
     │  {received: true}        │                              │
     │◀──────────────────────────│                              │
```

### Sincronización manual

```
POST /stock/sincronizar
→ Lee todos los productos desde Strapi (GET /api/productos)
→ Para cada producto, crea o actualiza registro en producto_stock
→ Retorna {total, actualizados, creados}
```

### Endpoints de stock

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| GET | `/stock` | Público | Lista todo el stock |
| GET | `/stock/:productoIdStrapi` | Público | Stock de un producto |
| POST | `/stock` | Público | Upsert manual de stock |
| POST | `/stock/sincronizar` | Público | Sincroniza desde Strapi |
| POST | `/strapi/webhook` | Webhook secret | Recibe eventos de Strapi |

### Frontend — Carga de productos

```
1. productoStore.fetchProducts()
2. GET Strapi /api/productos (catálogo con imágenes)
3. GET NestJS /stock (stock real desde Postgres)
4. Merge: reemplaza stock de Strapi con stock de Postgres
5. Render: ProductosHome.vue con imágenes y stock actualizado
```

### Manejo de imágenes nulas

- Strapi puede retornar `imagenes.data: null` para productos sin imágenes
- El store usa optional chaining: `imagenes?.data?.map(...)` con fallback `|| []`
- El componente muestra "Sin imagen" cuando no hay imágenes disponibles

---

## Módulo: Pedidos y Pagos (Transbank)

### Flujo de compra completo

```
Frontend (Vue)              NestJS API              Strapi          Transbank
     │                         │                       │               │
     │  1. Carrito + checkout  │                       │               │
     │  POST /pedidos          │                       │               │
     │  {items, direccion,     │                       │               │
     │   email, metodoPago}    │                       │               │
     │────────────────────────▶│                       │               │
     │                         │  2. Validar precios   │               │
     │                         │  GET /api/productos/:id│              │
     │                         │──────────────────────▶│               │
     │                         │  3. Validar stock     │               │
     │                         │  SELECT stock FROM    │               │
     │                         │  producto_stock       │               │
     │                         │  4. Crear pedido      │               │
     │                         │  (estado: pendiente)  │               │
     │  {id, numeroOrden}      │                       │               │
     │◀────────────────────────│                       │               │
     │                         │                       │               │
     │  5. POST /transbank/init│                       │               │
     │  {orden, monto, session}│                       │               │
     │────────────────────────▶│                       │               │
     │                         │  WebpayPlus.Transaction.create()      │
     │                         │──────────────────────────────────────▶│
     │                         │  {token, url}         │               │
     │  {token_ws, url}        │                       │               │
     │◀────────────────────────│                       │               │
     │                         │                       │               │
     │  6. Redirect a Webpay   │                       │               │
     │  (usuario paga)         │                       │               │
     │───────────────────────────────────────────────────────────────▶│
     │                         │                       │               │
     │                         │  7. Webpay redirect   │               │
     │                         │  POST /transbank/commit│              │
     │                         │  {token_ws}           │               │
     │◀───────────────────────────────────────────────────────────────│
     │                         │                       │               │
     │                         │  8. WebpayPlus.Transaction.commit()  │
     │                         │──────────────────────────────────────▶│
     │                         │  {status, card, vci}  │               │
     │                         │  9. Actualizar pedido │               │
     │                         │  (estado: pagado)     │               │
     │                         │  10. Descontar stock  │               │
     │                         │  UPDATE producto_stock│               │
     │                         │  SET stock = stock - n│               │
     │                         │  11. Guardar pago     │               │
     │                         │  INSERT pagos_transbank│             │
     │                         │  12. Redirect al front│               │
     │  13. GET /transbank/commit?token_ws=...         │               │
     │  → Redirect /pago-resultado?status=approved     │               │
     │◀────────────────────────│                       │               │
```

### Máquina de estados de pedidos

```
pendiente → procesando → enviado → entregado
    │
    └─→ cancelado (stock restaurado)
    └─→ pagado → procesando → enviado → entregado
```

| Transición | Trigger | Stock |
|------------|---------|-------|
| pendiente → pagado | Transbank commit OK | Descontar |
| pendiente → cancelado | Admin o timeout | Restaurar |
| pagado → procesando | Admin panel | — |
| procesando → enviado | Admin panel | — |
| enviado → entregado | Admin panel | — |

---

## Módulo: Panel Admin

### Rutas protegidas del frontend

| Ruta | Componente | Meta | Guard |
|------|-----------|------|-------|
| `/login` | LoginView | `guestOnly` | guestGuard |
| `/register` | RegisterView | `guestOnly` | guestGuard |
| `/admin/pedidos` | AdminPedidosView | `requiresAuth`, `requiresAdmin` | authGuard |

### Funcionalidades del panel admin

1. **Listar pedidos** — GET /pedidos con JWT admin
2. **Filtrar por estado** — Filtrado client-side
3. **Ver detalle** — GET /pedidos/:id con items, dirección, pago
4. **Cambiar estado** — PUT /pedidos/:id/estado con máquina de transiciones
5. **Cerrar sesión** — Limpia token y user del store + localStorage

### Usuario admin de prueba

```
Email: admin@manorigen.cl
Password: admin123
Tipo: admin
```

---

## Configuración de Webhooks (Strapi → NestJS)

### Configuración en Strapi Admin

1. Settings → Webhooks → Create new webhook
2. **URL:** `http://localhost:3000/strapi/webhook`
3. **Headers:** `Authorization: Bearer <STRAPI_WEBHOOK_SECRET>`
4. **Events:** `entry.publish`, `entry.update` del modelo `producto`

### Eventos manejados

| Evento | Modelo | Acción |
|--------|--------|--------|
| `entry.publish` | producto | upsertStock(entry.id, entry.stock) |
| `entry.update` | producto | upsertStock(entry.id, entry.stock) |
| `entry.delete` | producto | Log (no afecta stock local) |

---

## Configuración CORS

```typescript
// main.ts
app.enableCors({
  origin: ['http://localhost:5173'], // Vite dev server
  methods: ['GET', 'HEAD', 'PUT', 'PATCH', 'POST', 'DELETE', 'OPTIONS'],
  credentials: true,
});
```

### Variables de entorno relevantes

```env
CORS_ORIGINS=http://localhost:5173
STRAPI_URL=http://localhost:1337
STRAPI_API_TOKEN=<token generado en Strapi Admin>
STRAPI_WEBHOOK_SECRET=manorigen-webhook-secret-2026
JWT_SECRET=manorigen-jwt-secret-dev-2026
JWT_EXPIRES=24h
```

---

## Stack Tecnológico

| Capa | Tecnología | Versión |
|------|-----------|---------|
| CMS | Strapi | v4 |
| Backend | NestJS | ^10.0.0 |
| ORM | TypeORM | ^0.3.x |
| DB | PostgreSQL | 15+ |
| Auth | Passport + JWT | @nestjs/jwt, passport-jwt |
| Hash | bcrypt | ^5.x |
| Pagos | Transbank SDK | webpay-plus |
| Frontend | Vue 3 | ^3.x |
| Build | Vite | ^5.x |
| Estado | Pinia | ^2.x |
| Router | Vue Router | ^4.x |
| HTTP | Axios | ^1.x |
| CSS | Tailwind CSS | ^3.x |

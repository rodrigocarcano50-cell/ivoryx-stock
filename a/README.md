# Ivoryx — Gestión de Stock e Inventario

MVP funcional listo para usar. Sólo abrí `index.html` en el navegador (doble click) y ya tenés:

- **Dashboard** con totales, ventas del día, ingresos, ganancia y alertas.
- **7 categorías** (Fundas / Templados / Cargadores / Auriculares / Bolsas / Tarjetas / Otros) con tabla, búsqueda, filtro de stock bajo, alta/edición/borrado de productos.
- **Nuevo ingreso** y **Nueva venta** con cálculo automático de ganancia, descuento de stock y registro de movimiento.
- **Historial** filtrable por fecha, tipo, producto y categoría.
- **Estadísticas**: top productos, ganancia total, ventas por mes, valor de inventario.
- **Alertas** de stock bajo / crítico / agotado (umbrales configurables).
- **Modo oscuro**, búsqueda global, diseño responsivo PC/celular.
- **Backup/restauración** JSON, **import/export CSV** de productos y movimientos.

Los datos se guardan en `localStorage`. Hacé backup periódico desde *Configuración → Exportar backup*.

---

## Sincronización en la nube con Supabase

La app ya soporta Supabase como backend. Pasos:

### 1. Crear el proyecto
1. Entrá a https://supabase.com → New project.
2. Anotá la región y la **Database password**.
3. Una vez creado, vas a **Settings → API** y copiás:
   - **Project URL** (`https://xxxx.supabase.co`)
   - **anon public key** (empieza con `eyJ...`)

### 2. Crear las tablas
En el proyecto, abrí **SQL Editor → New query**, pegá y corré:

```sql
create table products (
  id          text primary key,
  name        text not null,
  category    text not null,
  model       text,
  color       text,
  sku         text,
  stock       integer not null default 0,
  buy_price   numeric not null default 0,
  sell_price  numeric not null default 0,
  battery     integer,
  created_at  timestamptz not null default now()
);

create table customers (
  id          text primary key,
  first_name  text,
  last_name   text,
  phone       text,
  created_at  timestamptz not null default now()
);
create index idx_customer_phone on customers(phone);

create table movements (
  id          text primary key,
  product_id  text not null references products(id) on delete cascade,
  customer_id text references customers(id) on delete set null,
  type        text not null check (type in ('in','out','adj')),
  qty         integer not null,
  unit_cost   numeric,
  unit_price  numeric,
  date        timestamptz not null default now(),
  note        text
);
create index idx_mov_date on movements(date desc);
create index idx_mov_product on movements(product_id);
create index idx_mov_customer on movements(customer_id);

-- Permitir acceso desde el frontend con la anon key
alter table products  enable row level security;
alter table movements enable row level security;
alter table customers enable row level security;

create policy "anon read products"  on products  for select using (true);
create policy "anon write products" on products  for insert with check (true);
create policy "anon update products" on products for update using (true);
create policy "anon delete products" on products for delete using (true);

create policy "anon read mov"  on movements for select using (true);
create policy "anon write mov" on movements for insert with check (true);
create policy "anon update mov" on movements for update using (true);
create policy "anon delete mov" on movements for delete using (true);

create policy "anon read cust"  on customers for select using (true);
create policy "anon write cust" on customers for insert with check (true);
create policy "anon update cust" on customers for update using (true);
create policy "anon delete cust" on customers for delete using (true);
```

> **Si ya creaste las tablas antes** (sin `customers` ni `color`), corré este migrador:
> ```sql
> alter table products add column if not exists color text;
> alter table products add column if not exists battery integer;
> create table if not exists customers (
>   id text primary key, first_name text, last_name text, phone text,
>   created_at timestamptz not null default now()
> );
> alter table movements add column if not exists customer_id text references customers(id) on delete set null;
> alter table customers enable row level security;
> create policy "anon read cust"  on customers for select using (true);
> create policy "anon write cust" on customers for insert with check (true);
> create policy "anon update cust" on customers for update using (true);
> create policy "anon delete cust" on customers for delete using (true);
> ```

> ⚠️ Esas políticas dan acceso público a quien tenga la URL del sitio. Está bien para uso interno; si querés restringir, agregá Supabase Auth y reemplazá `using (true)` por `using (auth.uid() is not null)`.

### 3. Conectar la app
Abrí la app → **Configuración → Sincronización en la nube**, pegá la Project URL y la anon key, y tocá **Conectar y sincronizar**. Si la nube está vacía, te ofrece subir los datos locales.

### 4. Deploy en Vercel
- Subí la carpeta a GitHub.
- En vercel.com → Add New Project → importá el repo.
- Como es HTML puro, no necesita framework ni build command. Deploy.
- Cada dispositivo donde abras la URL y conectes con la misma URL+key verá los mismos datos.

---

## Alternativa: backend propio (Node + Express + SQLite)

Cuando quieras pasar a multi-dispositivo / multi-usuario, la arquitectura recomendada:

```
ivoryx/
├─ apps/
│  ├─ web/                 # Frontend React + Vite + Tailwind (migrar este index.html)
│  └─ api/                 # Express
│     ├─ src/
│     │  ├─ index.ts
│     │  ├─ routes/        # products, movements, stats, auth
│     │  ├─ db/            # better-sqlite3 + migraciones
│     │  └─ services/
│     └─ data/ivoryx.db
└─ packages/shared/        # tipos compartidos (Product, Movement, etc.)
```

### Esquema SQL sugerido

```sql
CREATE TABLE products (
  id          TEXT PRIMARY KEY,
  name        TEXT NOT NULL,
  category    TEXT NOT NULL,
  model       TEXT,
  sku         TEXT,
  stock       INTEGER NOT NULL DEFAULT 0,
  buy_price   REAL NOT NULL DEFAULT 0,
  sell_price  REAL NOT NULL DEFAULT 0,
  created_at  TEXT NOT NULL
);

CREATE TABLE movements (
  id          TEXT PRIMARY KEY,
  product_id  TEXT NOT NULL REFERENCES products(id),
  type        TEXT NOT NULL CHECK (type IN ('in','out','adj')),
  qty         INTEGER NOT NULL,
  unit_cost   REAL,
  unit_price  REAL,
  date        TEXT NOT NULL,
  note        TEXT
);
CREATE INDEX idx_mov_date ON movements(date);
CREATE INDEX idx_mov_product ON movements(product_id);

CREATE TABLE settings (key TEXT PRIMARY KEY, value TEXT);
```

### Endpoints REST

```
GET    /api/products?category=Fundas&lowOnly=true&q=iphone
POST   /api/products
PATCH  /api/products/:id
DELETE /api/products/:id

POST   /api/movements              # body: {type,productId,qty,unitPrice|unitCost,note}
GET    /api/movements?from=&to=&type=&productId=&category=

GET    /api/stats/dashboard
GET    /api/stats/top-products
GET    /api/stats/monthly-sales

POST   /api/backup/export          # devuelve JSON
POST   /api/backup/import
POST   /api/import/csv
```

La capa de UI ya está estructurada para que pegar `fetch('/api/...')` en lugar de los helpers locales sea casi reemplazo 1:1.

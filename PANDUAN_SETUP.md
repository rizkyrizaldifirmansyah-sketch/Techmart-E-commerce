# Panduan Setup RUNCTRL di VSCode

## Prasyarat

Pastikan sudah install:
- [Node.js](https://nodejs.org) versi 18 ke atas (pilih LTS)
- [VSCode](https://code.visualstudio.com)
- [Git](https://git-scm.com) (opsional)

---

## Langkah 1 — Download Kode

1. Di Enter, klik tombol **Export / Download ZIP**
2. Extract file ZIP ke folder pilihan Anda (contoh: `C:/Projects/runctrl`)
3. Buka VSCode → **File → Open Folder** → pilih folder tersebut

---

## Langkah 2 — Install pnpm

Buka terminal di VSCode (`Ctrl + backtick`) lalu ketik:

```bash
npm install -g pnpm
```

---

## Langkah 3 — Install Dependencies Proyek

```bash
pnpm install
```

Tunggu hingga selesai (biasanya 1-2 menit).

---

## Langkah 4 — Setup Supabase (Database)

Website ini menggunakan Supabase untuk database dan autentikasi.

### 4a. Buat akun & project Supabase

1. Daftar gratis di [supabase.com](https://supabase.com)
2. Klik **"New Project"**
3. Isi nama project: `runctrl`
4. Pilih region terdekat (Singapore untuk Indonesia)
5. Buat password database (simpan baik-baik)
6. Tunggu project selesai dibuat (~2 menit)

### 4b. Jalankan SQL Migration (Buat tabel database)

1. Di dashboard Supabase, klik **SQL Editor**
2. Klik **"New Query"**
3. Copy-paste seluruh SQL di bawah ini, lalu klik **Run**:

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Profiles table
CREATE TABLE IF NOT EXISTS profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name TEXT,
  email TEXT,
  phone TEXT,
  address TEXT,
  city TEXT,
  postal_code TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view own profile" ON profiles FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Users can update own profile" ON profiles FOR UPDATE USING (auth.uid() = id);
CREATE POLICY "Users can insert own profile" ON profiles FOR INSERT WITH CHECK (auth.uid() = id);

-- Products table
CREATE TABLE IF NOT EXISTS products (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  category TEXT NOT NULL CHECK (category IN ('men', 'women', 'accessories')),
  subcategory TEXT,
  price NUMERIC(12,2) NOT NULL,
  original_price NUMERIC(12,2),
  description TEXT,
  images TEXT[] DEFAULT '{}',
  sizes TEXT[] DEFAULT '{}',
  colors TEXT[] DEFAULT '{}',
  stock INTEGER DEFAULT 0,
  rating NUMERIC(3,2) DEFAULT 0,
  review_count INTEGER DEFAULT 0,
  is_featured BOOLEAN DEFAULT false,
  is_bestseller BOOLEAN DEFAULT false,
  badge TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Products are publicly readable" ON products FOR SELECT USING (true);

-- Cart items table
CREATE TABLE IF NOT EXISTS cart_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  product_id UUID REFERENCES products(id) ON DELETE CASCADE NOT NULL,
  size TEXT NOT NULL,
  color TEXT NOT NULL,
  quantity INTEGER DEFAULT 1 CHECK (quantity > 0),
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, product_id, size, color)
);
ALTER TABLE cart_items ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can manage own cart" ON cart_items FOR ALL USING (auth.uid() = user_id);

-- Wishlist items table
CREATE TABLE IF NOT EXISTS wishlist_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  product_id UUID REFERENCES products(id) ON DELETE CASCADE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(user_id, product_id)
);
ALTER TABLE wishlist_items ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can manage own wishlist" ON wishlist_items FOR ALL USING (auth.uid() = user_id);

-- Orders table
CREATE TABLE IF NOT EXISTS orders (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled')),
  payment_method TEXT NOT NULL CHECK (payment_method IN ('transfer_bank', 'qris', 'cod')),
  payment_status TEXT DEFAULT 'unpaid' CHECK (payment_status IN ('unpaid', 'paid', 'failed')),
  subtotal NUMERIC(12,2) NOT NULL,
  shipping_cost NUMERIC(12,2) DEFAULT 0,
  total NUMERIC(12,2) NOT NULL,
  shipping_name TEXT NOT NULL,
  shipping_phone TEXT NOT NULL,
  shipping_address TEXT NOT NULL,
  shipping_city TEXT NOT NULL,
  shipping_postal_code TEXT NOT NULL,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view own orders" ON orders FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert own orders" ON orders FOR INSERT WITH CHECK (auth.uid() = user_id);

-- Order items table
CREATE TABLE IF NOT EXISTS order_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  order_id UUID REFERENCES orders(id) ON DELETE CASCADE NOT NULL,
  product_id UUID REFERENCES products(id) ON DELETE SET NULL,
  product_name TEXT NOT NULL,
  product_image TEXT,
  size TEXT NOT NULL,
  color TEXT NOT NULL,
  quantity INTEGER NOT NULL,
  price NUMERIC(12,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);
ALTER TABLE order_items ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can view own order items" ON order_items
  FOR SELECT USING (EXISTS (SELECT 1 FROM orders WHERE orders.id = order_items.order_id AND orders.user_id = auth.uid()));
CREATE POLICY "Users can insert own order items" ON order_items
  FOR INSERT WITH CHECK (EXISTS (SELECT 1 FROM orders WHERE orders.id = order_items.order_id AND orders.user_id = auth.uid()));

-- Reviews table
CREATE TABLE IF NOT EXISTS reviews (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  product_id UUID REFERENCES products(id) ON DELETE CASCADE NOT NULL,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
  comment TEXT,
  reviewer_name TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(product_id, user_id)
);
ALTER TABLE reviews ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Reviews are publicly readable" ON reviews FOR SELECT USING (true);
CREATE POLICY "Authenticated users can insert reviews" ON reviews FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Users can update own reviews" ON reviews FOR UPDATE USING (auth.uid() = user_id);

-- Trigger: auto-create profile saat user daftar
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER LANGUAGE plpgsql SECURITY DEFINER AS $$
BEGIN
  INSERT INTO profiles (id, full_name, email)
  VALUES (NEW.id, NEW.raw_user_meta_data->>'full_name', NEW.email)
  ON CONFLICT (id) DO NOTHING;
  RETURN NEW;
END;
$$;
DROP TRIGGER IF EXISTS on_auth_user_created ON auth.users;
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

### 4c. Aktifkan Auto Confirm Email

1. Di Supabase dashboard → **Authentication → Settings**
2. Scroll ke **"Email confirmations"**
3. **Matikan** toggle "Enable email confirmations" (agar bisa langsung login tanpa verifikasi email saat testing)

### 4d. Ambil URL dan API Key

1. Di Supabase dashboard → **Settings → API**
2. Copy nilai:
   - **Project URL** (contoh: `https://abcdefgh.supabase.co`)
   - **anon / public key** (string panjang dimulai `eyJ...`)

---

## Langkah 5 — Hubungkan Kode ke Supabase Baru

Buka file: `src/integrations/supabase/client.ts`

Ganti nilai berikut dengan milik Anda:

```typescript
const SUPABASE_URL = "https://GANTI_DENGAN_URL_ANDA.supabase.co";
const SUPABASE_PUBLISHABLE_KEY = "GANTI_DENGAN_ANON_KEY_ANDA";
```

---

## Langkah 6 — Jalankan Website

```bash
pnpm run dev
```

Buka browser → **http://localhost:5173**

Website RUNCTRL sudah berjalan di komputer Anda!

---

## Langkah 7 — Tambah Data Produk

Setelah berhasil jalan, tambah produk lewat Supabase:

1. Supabase dashboard → **Table Editor → products**
2. Klik **"Insert row"**
3. Isi kolom:
   - `name`: nama produk
   - `slug`: nama-url-friendly (contoh: `kaos-lari-hitam`)
   - `category`: `men`, `women`, atau `accessories`
   - `price`: harga dalam rupiah (contoh: `299000`)
   - `images`: URL foto produk (contoh: `["https://link-foto.com/foto.jpg"]`)
   - `sizes`: ukuran tersedia (contoh: `["S","M","L","XL"]`)
   - `colors`: warna tersedia (contoh: `["Hitam","Putih"]`)
   - `stock`: jumlah stok
   - `is_featured`: `true` untuk tampil di homepage

---

## Struktur Folder Penting

```
src/
├── pages/              ← Halaman-halaman website
│   ├── Index.tsx       ← Homepage
│   ├── Products.tsx    ← Daftar produk
│   ├── ProductDetail.tsx ← Detail produk
│   ├── Auth.tsx        ← Login/Register
│   ├── Checkout.tsx    ← Checkout
│   ├── Account.tsx     ← Akun pengguna
│   ├── About.tsx       ← Halaman About
│   ├── Contact.tsx     ← Halaman Contact
│   └── FAQ.tsx         ← Halaman FAQ
│
├── components/
│   ├── home/           ← Komponen homepage
│   │   ├── HeroSection.tsx     ← Banner utama
│   │   ├── CategorySection.tsx ← Kategori
│   │   ├── FeaturedProducts.tsx
│   │   └── BestSellers.tsx
│   ├── layout/         ← Navbar, Footer
│   └── cart/           ← Cart drawer
│
├── contexts/           ← State global (auth, cart)
├── hooks/              ← Data fetching (produk, wishlist)
└── index.css           ← Warna & tema utama
```

---

## Tips Mengedit Tampilan

| Yang ingin diubah | File yang diedit |
|---|---|
| Warna/tema website | `src/index.css` |
| Teks & foto hero banner | `src/components/home/HeroSection.tsx` |
| Foto kategori Men/Women | `src/components/home/CategorySection.tsx` |
| Data produk | Supabase → tabel `products` |
| Navbar & menu | `src/components/layout/Navbar.tsx` |
| Footer | `src/components/layout/Footer.tsx` |

---

## Masalah Umum

**Error: "pnpm not found"**
→ Jalankan `npm install -g pnpm` terlebih dahulu

**Produk tidak muncul**
→ Pastikan sudah menjalankan SQL migration di Supabase dan menambah data produk

**Tidak bisa login/daftar**
→ Pastikan URL dan API key Supabase sudah benar di `client.ts`

**Port sudah digunakan**
→ Ganti port dengan `pnpm run dev -- --port 3000`

---

*Dibuat dengan Enter — enterapp.pro*

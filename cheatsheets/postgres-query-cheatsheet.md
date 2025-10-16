🧠 PostgreSQL Query Cheat Sheet

Kumpulan query PostgreSQL yang sering digunakan dalam — dikelompokkan berdasarkan tingkat dan fungsi (dasar, lanjut, JSON, agregasi, validasi, dll).

🟩 DASAR

🔹 SELECT & WHERE

SELECT * FROM users WHERE status = 'ACTIVE';
SELECT id, name FROM customers WHERE city = 'Jakarta';

🔹 ORDER, LIMIT, OFFSET

SELECT * FROM orders ORDER BY created_at DESC LIMIT 10 OFFSET 20;

🔹 DISTINCT & COUNT

-- Ambil semua nilai unik dari satu kolom
SELECT DISTINCT email FROM users;

-- Hitung total baris unik berdasarkan kolom tertentu
SELECT COUNT(DISTINCT user_id) AS total_user FROM orders;

-- Ambil kombinasi unik dari dua kolom
SELECT DISTINCT user_id, product_id
FROM order_items;

-- Ambil satu record unik berdasarkan kolom prioritas (DISTINCT ON)
SELECT DISTINCT ON (customer_id) customer_id, order_id, created_at
FROM orders
ORDER BY customer_id, created_at DESC;
```sql
SELECT DISTINCT email FROM users;
SELECT COUNT(*) AS total_user FROM users WHERE active = true;

🔹 BETWEEN, IN, LIKE

SELECT * FROM payments WHERE amount BETWEEN 10000 AND 50000;
SELECT * FROM products WHERE category IN ('Shoes', 'Bags');
SELECT * FROM logs WHERE message ILIKE '%error%';

🟨 JOIN & RELASI

-- Inner join
SELECT o.id, u.name
FROM orders o
JOIN users u ON u.id = o.user_id;

-- Left join (ambil semua order, walau user-nya null)
SELECT o.id, u.name
FROM orders o
LEFT JOIN users u ON u.id = o.user_id;

-- Cek data tanpa pasangan (orphan)
SELECT o.id FROM orders o
LEFT JOIN payments p ON p.order_id = o.id
WHERE p.id IS NULL;

🟦 AGGREGATION & GROUPING

🔹 COUNT, SUM, AVG, MAX, MIN

SELECT user_id, COUNT(*) AS total_order, SUM(amount) AS total_spent
FROM orders
GROUP BY user_id;

🔹 HAVING (filter setelah GROUP BY)

SELECT category, COUNT(*) AS total_item
FROM products
GROUP BY category
HAVING COUNT(*) > 10;

🔹 ARRAY_AGG dan STRING_AGG

-- Gabungkan beberapa nilai menjadi array
SELECT customer_id, ARRAY_AGG(product_name ORDER BY product_name) AS products
FROM orders
GROUP BY customer_id;

-- Gabungkan nilai menjadi string dengan pemisah
SELECT customer_id, STRING_AGG(product_name, ', ') AS products
FROM orders
GROUP BY customer_id;

-- Gabungkan JSON field
SELECT order_id, JSON_AGG(item_detail) AS items
FROM order_items
GROUP BY order_id;

💡 Tips:

ARRAY_AGG() cocok untuk hasil yang ingin di-unpack kembali pakai UNNEST()

STRING_AGG() cocok untuk laporan

JSON_AGG() berguna untuk hasil API

🟧 JSON FIELD HANDLING

-- Ambil field dalam kolom JSON
SELECT json_data ->> 'msg' AS message FROM api_logs;

Catatan (Gotcha) :
- msg harus dalam tanda petik dua ('')
- jika msg tidak dalam tanda petik dua ('') dan ada kolom msg dalam tabel, maka NULL (tidak error)

-- Filter berdasarkan isi JSON
SELECT * FROM api_logs WHERE (json_data ->> 'code') != '1';
SELECT * FROM api_logs WHERE (json_data ->> 'msg') != 'success';

-- Akses nested key
SELECT json_data -> 'data' ->> 'zone' AS zone FROM api_logs;

🟪 VALIDATION & DUPLICATE CHECK

-- Cek reference_id yang punya lebih dari satu record unik
SELECT reference_id, COUNT(DISTINCT record_id)
FROM trxhistory
GROUP BY reference_id
HAVING COUNT(DISTINCT record_id) > 1;

-- Cek data duplikat berdasarkan email
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Cek user yang punya lebih dari satu aktivitas
SELECT user_id, COUNT(*) FROM user_activity GROUP BY user_id HAVING COUNT(*) > 1;

🟥 DATE & TIME HANDLING

-- Hari ini saja
SELECT * FROM logs WHERE created_at::date = CURRENT_DATE;

-- 7 hari terakhir
SELECT * FROM transactions WHERE created_at >= NOW() - INTERVAL '7 days';

-- Ambil bulan & tahun
SELECT EXTRACT(MONTH FROM created_at) AS month, EXTRACT(YEAR FROM created_at) AS year FROM orders;

🟫 CONDITIONAL & NULL HANDLING

-- Ganti null jadi default
SELECT COALESCE(discount, 0) AS discount_applied FROM orders;

-- CASE/IF ELSE
SELECT CASE
         WHEN status = 'PENDING' THEN 'In Progress'
         WHEN status = 'SUCCESS' THEN 'Completed'
         ELSE 'Unknown'
       END AS status_label
FROM transactions;

🟦 UPDATE & DELETE PRAKTIS

-- Update status gagal
UPDATE orders SET status = 'CANCELLED' WHERE payment_status = 'FAILED';

-- Hapus semua newline dan spasi berlebih, jadikan satu spasi
UPDATE nama_tabel
SET nama_kolom = trim(regexp_replace(nama_kolom, E'[\\n\\r\\s]+', ' ', 'g'));

-- Hapus log lama
DELETE FROM logs WHERE created_at < NOW() - INTERVAL '90 days';

🟪 TOOLS & DEBUGGING

-- Buat tabel sementara
CREATE TEMP TABLE temp_users AS SELECT * FROM users WHERE active = false;

-- Lihat struktur tabel
\d table_name

-- Lihat tipe data kolom (desc)
SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'users';

🟦 CTE (Common Table Expression)

Memecah query kompleks jadi beberapa tahap yang lebih mudah dibaca & reusable.

-- Total transaksi per user
WITH user_total AS (
  SELECT user_id, SUM(amount) AS total_spent
  FROM transactions
  GROUP BY user_id
)
SELECT u.name, t.total_spent
FROM users u
JOIN user_total t ON t.user_id = u.id
WHERE t.total_spent > 100000;

-- CTE bertingkat / multi-step
WITH recent_orders AS (
  SELECT * FROM orders WHERE created_at >= NOW() - INTERVAL '7 days'
),
order_stats AS (
  SELECT customer_id, COUNT(*) AS total_recent
  FROM recent_orders
  GROUP BY customer_id
)
SELECT c.name, s.total_recent
FROM customers c
JOIN order_stats s ON c.id = s.customer_id
ORDER BY s.total_recent DESC;

📘 Gunakan CTE ketika:

Subquery terlalu panjang dan ingin lebih terbaca

Perlu reuse hasil query sebelumnya di query utama

Analisis bertingkat (aggregate di atas aggregate)

🟨 OPTIMISASI

-- EXISTS vs IN (lebih efisien untuk cek keberadaan data)
SELECT * FROM orders o WHERE EXISTS (SELECT 1 FROM payments p WHERE p.order_id = o.id);

-- Gunakan EXPLAIN untuk profiling
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'PENDING';

-- Index sederhana
CREATE INDEX idx_orders_status ON orders(status);

🧾 Catatan:

Gunakan alias singkat (o, u, p) untuk efisiensi baca.

Hindari SELECT * di query produksi kecuali untuk debugging.

Selalu tambahkan LIMIT saat eksplorasi data di database besar.


💡 TIPS & TRICKS

🔤 Manipulasi String

-- Hapus semua newline & spasi berlebih
regexp_replace(text_column, E'[\\n\\r\\s]+', ' ', 'g')

-- Hapus newline tanpa spasi tambahan
regexp_replace(text_column, E'[\\n\\r]+', '', 'g')

-- Cari teks yang mengandung tanda kutip '
WHERE text_column LIKE '%''%'

-- Ganti karakter tertentu (contoh backslash)
replace(json_request::text, E'\\', '')

🧩 Regex & Array Functions

-- Cari semua match regex di dalam string
SELECT regexp_matches('abc123def456', '[0-9]+', 'g');
-- → {123, 456}

-- Pecah string berdasarkan delimiter (hasilnya array)
SELECT regexp_split_to_array('a,b,c,d', ',');
-- → {a,b,c,d}

-- Alternatif tanpa regex
SELECT string_to_array('x|y|z', '|');
-- → {x,y,z}

-- Gabungkan kembali array menjadi string
SELECT array_to_string(ARRAY['a','b','c'], ', ');
-- → 'a, b, c'

-- Kombinasi cepat split-join
SELECT array_to_string(string_to_array('a|b|c', '|'), ', ');
-- → 'a, b, c'

🧠 JSON & JSONB Tricks

-- Tampilkan semua key dari kolom JSON
SELECT jsonb_object_keys(payload)
FROM webhook_events
WHERE awb = '123456';

-- Ambil nilai tertentu
SELECT payload ->> 'status' AS status
FROM webhook_events;

-- Iterasi key-value JSON
SELECT key, value
FROM jsonb_each(payload);








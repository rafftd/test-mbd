# test-mbd

Analisis

## Analisis Perbandingan Query Plan

| Aspek                  | Sebelum Tuning                                                                 | Setelah Tuning                                                          |
|-------------------------|--------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| **Jenis Join**          | Hash Join antara `orders` dan `customers`                                       | Nested Loop dengan Memoize untuk efisiensi lookup                      |
| **Scan**                | Sequential Scan (`Seq Scan`) di `orders` dan `customers`                      | Index Scan (`index_order_pk` di `orders`, `pk_customers` di `customers`) |
| **Sort**                | External Merge Sort (menggunakan disk 45872kB)                                 | Incremental Sort (menggunakan memory, lebih ringan)                    |
| **Penghapusan Duplikat**| DISTINCT + Unique operator (biaya tinggi)                                       | GROUP BY (lebih efisien untuk grup data)                                |
| **Resource**            | High I/O, High Disk Usage, Memory Pressure                                     | Lebih hemat memory dan disk usage                                       |
| **Planning Time**       | 0.252 ms                                                                       | 0.221 ms                                                               |
| **Execution Time**      | 2600.570 ms (~2.6 detik)                                                       | 751.308 ms (~0.75 detik)                                                |
| **Jumlah Rows**         | ~900.000 rows diproses                                                         | ~900.000 rows diproses (lebih cepat karena index & cache)              |
| **Optimisasi**          | Tidak optimal, heavy resource usage                                            | Optimal dengan index, memoize, dan sort lebih ringan                  |

## Ringkasan
- Query setelah tuning **3.5x lebih cepat** dari sebelumnya.
- Penggunaan **Index Scan**, **Memoize**, dan **Incremental Sort** sangat mengurangi beban sistem.
- **GROUP BY** lebih disukai daripada **DISTINCT** dalam kasus ini karena lebih predictable untuk PostgreSQL planner.
- Pemakaian **Memoize** efektif karena banyak lookup berulang ke `customers`.

## Saran Optimasi Tambahan
- Buat **Partial Index** di tabel `orders` untuk kolom `order_date` (`WHERE order_date >= '1997-01-01'`).
- Aktifkan **Parallel Execution** di PostgreSQL untuk query lebih besar.
- Pertimbangkan buat **Materialized View** kalau query sering dipanggil.

Kesimpulan

## Kesimpulan

| Aspek        | Sebelum Tuning | Setelah Tuning | Perbedaan |
|--------------|----------------|----------------|-----------|
| Eksekusi     | 2600 ms         | 751 ms          | ~3.5x lebih cepat |
| Join Type    | Hash Join       | Nested Loop + Memoize | Nested Loop lebih hemat untuk lookup kecil |
| Scan Type    | Seq Scan        | Index Scan      | Index Scan lebih targeted dan cepat |
| Sort         | External Merge Sort (pakai disk) | Incremental Sort (pakai memory) | Hemat resource, lebih cepat |
| Distinct     | DISTINCT + Unique | GROUP BY       | GROUP BY lebih efisien untuk eliminasi duplikat |

## Catatan
- Setelah tuning, query lebih optimal karena memanfaatkan index dan caching (`Memoize`).
- Execution Time berkurang drastis dari **2.6 detik** menjadi **0.75 detik**.
- Potensi optimasi tambahan: 
  - Buat partial index di `orders (order_date)` untuk kondisi `>= '1997-01-01'`
  - Pakai parallel query processing.
  - Materialized view jika query sering digunakan.

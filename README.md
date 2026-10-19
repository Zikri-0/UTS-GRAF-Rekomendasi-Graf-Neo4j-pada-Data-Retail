# UTS-GRAF-Rekomendasi-Graf-Neo4j-pada-Data-Retail
Sistem rekomendasi berbasis graf Neo4j pada data e-commerce RetailRocket: proyeksi user-item, cosine similarity, dan user-based KNN dengan evaluasi Hit@5 dan NDCG@5. UTS Graf Sains Data, Unesa.

Project ini membahas penerapan **graf database Neo4j** untuk membangun sistem rekomendasi dari data interaksi e-commerce **RetailRocket**. Data interaksi (view, add to cart, transaction) dimodelkan sebagai graf antara *Visitor* dan *Item*, kemudian diproyeksikan menjadi graf antar-user untuk menghitung kemiripan dan menghasilkan rekomendasi dengan **K-Nearest Neighbors (KNN)**.

## Dataset

Dataset yang digunakan adalah **RetailRocket E-commerce Dataset** dari Kaggle (sumber: Roman Zykov), khususnya file `events.csv`.

| Kolom | Keterangan |
| ----- | ---------- |
| timestamp | Waktu kejadian dalam unix epoch (milidetik) |
| visitorid | ID unik pengunjung |
| event | Jenis interaksi: `view`, `addtocart`, `transaction` |
| itemid | ID unik produk |
| transactionid | ID transaksi, hanya terisi saat event `transaction` |

Data dibagi menjadi dua periode waktu:

| Periode | Rentang Waktu | Fungsi | Jumlah Event |
| ------- | ------------- | ------ | ------------ |
| P1 | 18 - 31 Mei 2015 | Training (dimasukkan ke graf) | 294.333 (177.741 setelah filter) |
| P2 | 16 - 30 Juni 2015 | Testing (evaluasi rekomendasi) | 309.830 |

Karena Neo4j Aura Free dibatasi **200.000 node**, visitor dengan kurang dari 2 event pada P1 dibuang (`MIN_EVENT = 2`). Setelah filter, P1 berisi **87.308 node** dan **177.741 relasi**.

## Alur Pengerjaan

```text
events.csv
    ↓
Filter Periode (P1 dan P2)
    ↓
Impor Batch ke Neo4j Aura
    ↓
Query Komposisi Subgraph
    ↓
Vector Space Modelling (User dan Item)
    ↓
Perbandingan Proyeksi
    ↓
Cosine Similarity antar User
    ↓
Relasi SIMILAR_TO pada Graf
    ↓
Rekomendasi User-based KNN (k = 5)
    ↓
Evaluasi Hit@5 dan NDCG@5
```

## Model Graf

```text
(:Visitor)-[:VIEWED]->(:Item)
(:Visitor)-[:ADDED_TO_CART]->(:Item)
(:Visitor)-[:BOUGHT]->(:Item)
(:Visitor)-[:SIMILAR_TO {score}]->(:Visitor)
```

Komposisi graf setelah impor P1:

| Simpul / Relasi | Jumlah |
| --------------- | ------ |
| Item | 44.556 |
| Visitor | 42.752 |
| VIEWED | 168.861 |
| ADDED_TO_CART | 6.675 |
| BOUGHT | 2.205 |

## Vector Space Modelling

Interaksi direpresentasikan sebagai matriks biner (bernilai 1 jika ada interaksi) dalam format sparse.

| | Proyeksi User | Proyeksi Item |
| - | ------------- | ------------- |
| Shape | 42.752 x 44.556 | 44.556 x 42.752 |
| Entri tak nol | 115.327 | 115.327 |
| Density | 0,000061 | 0,000061 |
| Rata-rata interaksi per baris | 2,70 | 2,59 |
| Median / maksimum | 2 / 729 | 1 / 144 |
| Jumlah operasi proyeksi | 1.045.057 | 2.926.253 |

Proyeksi **user** dipilih karena lebih efisien (sekitar 2,8 kali lebih sedikit operasi) dan vektornya sedikit lebih kaya informasi.

## Similarity

Kemiripan antar user dihitung dengan **cosine similarity**, kemudian diverifikasi dengan query Cypher yang menghitung hal yang sama langsung di dalam graf. Hasil untuk user acuan `286616` (user dengan item terbanyak):

| Peringkat | User | Similarity |
| --------- | ---- | ---------- |
| 1 | 327437 | 0,1444 |
| 2 | 138131 | 0,1094 |
| 3 | 57905 | 0,1068 |
| 4 | 163561 | 0,1059 |
| 5 | 389532 | 0,1028 |

Kelima pasangan ini ditulis kembali ke graf sebagai relasi `SIMILAR_TO`.

## Rekomendasi dan Evaluasi

User dipilih secara acak dengan `numpy.random.default_rng(seed=42)` sehingga hasilnya dapat direproduksi. Rekomendasi dibuat dengan **user-based KNN (k = 5)**: item dari 5 tetangga terdekat yang belum pernah dilihat user diberi skor berdasarkan jumlah similarity tetangga.

| Keterangan | Hasil |
| ---------- | ----- |
| User terpilih | 121532 (7 item pada P1) |
| Tetangga (k = 5) | 699350, 232952, 219749, 805732, 163561 |
| Top-5 rekomendasi | 20709, 46285, 165629, 196739, 218239 |
| 3 item yang diiklankan | 20709, 46285, 165629 |
| Hit@5 | 0 |
| NDCG@5 | 0,0000 |

Item relevan didefinisikan sebagai item baru yang diakses user pada P2. Hasil 0 pada user ini wajar karena profil user tipis, hanya ada satu item relevan di antara 44.556 item, dan matriks interaksi sangat jarang. Evaluasi pada satu user juga memiliki variansi yang besar.

## Tech Stack

* Python
* Neo4j Aura dan Cypher
* Neo4j Python Driver
* Pandas
* NumPy
* SciPy
* Scikit-learn
* NetworkX
* Matplotlib
* python-dotenv
* tqdm
* Jupyter Notebook

## Cara Menjalankan

1. Clone repository dan buat virtual environment.

```bash
uv venv .venv --python 3.10
.venv\Scripts\activate
uv pip install -r requirements.txt
uv pip install pandas numpy scipy scikit-learn networkx matplotlib tqdm python-dotenv ipykernel
```

2. Unduh dataset dari Kaggle (PowerShell).

```bash
curl.exe -L -o ecommerce-dataset.zip https://www.kaggle.com/api/v1/datasets/download/retailrocket/ecommerce-dataset
```

3. Buat instance Neo4j Aura dan simpan kredensialnya dalam file `.env`.

```text
NEO4J_URI=neo4j+s://<id-instance>.databases.neo4j.io
NEO4J_USERNAME=<username>
NEO4J_PASSWORD=<password>
NEO4J_DATABASE=<nama-database>
```

4. Sesuaikan path pada `load_dotenv(...)` di cell pertama notebook, lalu jalankan notebook secara berurutan.

> Catatan: pada jaringan tertentu koneksi `neo4j+s://` gagal karena verifikasi sertifikat, sehingga notebook memakai `neo4j+ssc://`. File `.env` dan file credentials Aura tidak boleh di-commit ke repository.

## Project Structure

```text
.
├── UTS_088.ipynb
├── events_p1.csv
├── events_p2.csv
├── requirements.txt
├── .gitignore
└── README.md
```

## Fokus Project

Project ini menggabungkan beberapa konsep dalam satu analisis:

**Impor Data ke Graf → Komposisi Subgraph → Vector Space Modelling → Perbandingan Proyeksi → Cosine Similarity → Relasi SIMILAR_TO → User-based KNN → Hit@5 dan NDCG@5**

Project ini juga menjadi latihan untuk memahami bagaimana data interaksi dapat dimodelkan sebagai graf dan dimanfaatkan untuk membangun sistem rekomendasi.

## Author

**Muhammad Zikri Widiandra**
S1 Sains Data, Universitas Negeri Surabaya

# Analisis Big Data Dokumen Epstein Files

> **Proyek Akhir Mata Kuliah Kecerdasan Web dan Big Data**
> **Departemen Teknik Komputer, Institut Teknologi Sepuluh Nopember (ITS)**

---

## Deskripsi Proyek

Proyek ini membangun ekosistem *data engineering* berskala besar yang berfungsi untuk melakukan *crawling* otomatis, analisis sentimen, ekstraksi poin kunci, dan deteksi emosi secara *real-time* dari **Epstein Files** — kumpulan dokumen intelijen sensitif — menggunakan kata kunci spesifik **"Iran"**.

Sistem ini mengintegrasikan pipa data otomatis (*automated data pipeline*) dengan arsitektur **RAG (Retrieval-Augmented Generation)** menggunakan LLM kelas berat untuk melayani pertanyaan analitis pengguna melalui **Bot Telegram**. Seluruh hasil analisis divisualisasikan melalui **Dashboard HTML interaktif** yang terhubung langsung ke **Supabase** sebagai database utama.

---

## Arsitektur Sistem & Alur Data

Ekosistem ini terbagi menjadi dua sub-sistem utama yang bekerja secara independen namun terhubung pada database Supabase yang sama:

### 1. Hulu: Automated Ingestion Pipeline (Jalur Penambangan & Analisis Data)

```
[Schedule Trigger] ➡️ [Edit Fields (Keyword: Iran)] ➡️ [Apify Actor — Epstein Files Scraper]
[Get Dataset Items] ➡️ [Loop Over Items (Batch: 20)] ➡️ [Groq AI (Llama 3.3 70B — Analisis)]
[Create a Row → Supabase] ➡️ [Wait]
```

### 2. Hilir: Serving & RAG Chatbot Pipeline (Jalur Interaksi Bot Telegram)

```
[Telegram Trigger] ➡️ [Groq AI (Keyword Extraction)] ➡️ [Supabase — Get Many Rows (ilike filter)]
[Code in JavaScript (Context Flattening)] ➡️ [Groq AI (Llama 3.3 70B — Answer)] ➡️ [Send to Telegram]
```

### 3. Visualisasi: Dashboard HTML Interaktif

```
[Browser] ➡️ [Dashboard.html] ➡️ [Supabase JS Client] ➡️ [Render Chart.js & Tabel Rekaman]
```

---

## Fitur Utama

1. **Automated Keyword Crawler:** Menambang dokumen Epstein Files secara berkala menggunakan Apify Actor (`lofomachines/epstein-files-scraper-api`) dengan parameter kata kunci dinamis, mampu mengambil hingga 1.000 item per eksekusi dengan batas dataset 1.500 item.

2. **AI Intelligence Analysis:** Setiap dokumen yang di-crawl dianalisis otomatis menggunakan LLM ke dalam 3 dimensi data:
   - **Sentimen:** Positif, Negatif, atau Netral.
   - **Key Points:** Ekstraksi poin-poin intelijen terpenting dari teks dokumen.
   - **Emosi:** Deteksi emosi dominan yang terkandung dalam narasi teks.

3. **Business Intelligence (BI) Dashboard:** Visualisasi data *real-time* berbasis HTML murni dengan koneksi langsung ke Supabase, menampilkan:
   - Ringkasan statistik (total dokumen, distribusi sentimen, total key points).
   - Grafik distribusi sentimen dengan progress bar.
   - Grafik frekuensi emosi yang terdeteksi.
   - Grafik popularitas keyword.
   - Keyword cloud interaktif.
   - Tabel rekaman lengkap dengan fitur ekspansi detail, pencarian, filter sentimen, dan paginasi.

4. **Zero-Hallucination RAG Chatbot:** Bot Telegram analitis yang menjawab pertanyaan menggunakan konteks data langsung dari Supabase. Menggunakan teknik *separation of concerns*: ekstraksi keyword dilakukan deterministik oleh LLM ringan, lalu hasil query database diratakan oleh JavaScript, dan nalar bahasa diproses oleh LLM 70B.

---

## Spesifikasi Teknologi (Tech Stack)

| Komponen | Teknologi |
|---|---|
| **Orkestrasi & Otomatisasi** | n8n v1.x (Workflow-driven automation) |
| **Web Scraping / Crawling** | Apify — `epstein-files-scraper-api` |
| **Database** | Supabase (PostgreSQL-based, tabel: `epstein_analysis`) |
| **Dashboard Visualisasi** | HTML5 + Chart.js v4.4.0 + Supabase JS Client v2 |
| **Mesin Inferensi AI** | Groq Cloud API |
| **Model Crawl & Analisis** | `llama-3.3-70b-versatile` (Analisis intelijen & ekstraksi JSON) |
| **Model Chatbot** | `llama-3.3-70b-versatile` (Penalaran bahasa tingkat tinggi) |
| **Bot Messaging** | Telegram Bot API |
| **Scripting** | JavaScript (ES6+) untuk data flattening & context building di n8n |

---

## Skema Database (Supabase)

Tabel: **`epstein_analysis`**

| Kolom | Tipe | Keterangan |
|---|---|---|
| `id` | UUID / Serial | Primary key auto-generated |
| `keyword` | text | Kata kunci pencarian (contoh: `Iran`) |
| `extracted_text` | text | Teks mentah hasil ekstraksi dokumen dari Apify |
| `key_points` | text[] / jsonb | Array poin-poin kunci hasil analisis AI |
| `sentiment` | text | Sentimen dokumen: `Positif`, `Negatif`, atau `Netral` |
| `emotions` | text[] / jsonb | Array emosi yang terdeteksi dalam dokumen |

---

## Struktur Repositori

```
.
├── README.md                   → Laporan proyek akhir (file ini)
├── Epstein_Crawl.json          → Backup workflow n8n untuk automated crawler & analisis
├── Chatbot_AI.json             → Backup workflow n8n untuk RAG Chatbot Telegram
└── Dashboard.html              → Dashboard visualisasi interaktif (standalone HTML)
```

---

## Rekayasa Pipa Data & Implementasi Kode (Engineering Deep Dive)

### 1. Konfigurasi Apify Actor (Epstein Crawl Workflow)

Crawling menggunakan Apify Actor `lofomachines/epstein-files-scraper-api` dengan mode `fetch-details` untuk mendapatkan teks penuh dokumen. Keyword diinjeksi secara dinamis dari node **Edit Fields**:

```json
{
  "keywords": ["Iran"],
  "maxItems": 1000,
  "mode": "fetch-details",
  "proxyConfiguration": {
    "useApifyProxy": true,
    "apifyProxyGroups": []
  }
}
```

Dataset hasil crawling kemudian diambil melalui node **Get Dataset Items** dengan batas 1.500 item, lalu diproses secara *batch* menggunakan **Loop Over Items** (20 item per batch) untuk menghindari rate limiting pada Groq API.

### 2. Analisis AI dengan Structured JSON Output (HTTP Request Node)

Agar hasil analisis LLM dapat langsung di-*parse* dan disimpan ke Supabase, model diperintahkan untuk menghasilkan output JSON murni dengan skema yang sudah ditentukan:

```json
{
  "model": "llama-3.3-70b-versatile",
  "response_format": { "type": "json_object" },
  "messages": [
    {
      "role": "system",
      "content": "Kamu adalah analis intelijen. Ekstrak poin penting dari dokumen, tentukan sentimen (Positif/Negatif/Netral), dan deteksi emosi dari teks. Output WAJIB dalam format JSON murni dengan struktur persis seperti ini: {\"key_points\": [\"poin1\", \"poin2\"], \"sentiment\": \"Sentimen\", \"emotions\": [\"emosi1\", \"emosi2\"]}"
    },
    {
      "role": "user",
      "content": "Analisis teks ini terkait Iran: {{ $json.extractedText }}"
    }
  ]
}
```

Hasil JSON kemudian di-*parse* dan disimpan ke Supabase melalui node **Create a row**:

```javascript
// Contoh mapping field ke Supabase
keyword:        "={{ $('Edit Fields').item.json.keyword }}"
extracted_text: "={{ $('Loop Over Items').item.json.extractedText }}"
key_points:     "={{ JSON.parse($json.choices[0].message.content).key_points }}"
sentiment:      "={{ JSON.parse($json.choices[0].message.content).sentiment }}"
emotions:       "={{ JSON.parse($json.choices[0].message.content).emotions }}"
```

### 3. Sistem RAG Chatbot Telegram: Ekstraksi Keyword Cerdas

Untuk mencegah pencarian database menggunakan subjek utama yang terlalu luas, chatbot menggunakan dua tahap pemrosesan LLM. Tahap pertama mengekstrak kata kunci spesifik dari pertanyaan user dan menerjemahkannya ke Bahasa Inggris (karena database berbahasa Inggris):

```json
{
  "role": "system",
  "content": "Kamu adalah mesin ekstraksi dan penerjemah kata kunci. Database kami berisi dokumen berbahasa Inggris tentang Iran/Epstein.\n\nTugasmu: Ambil 1 atau maksimal 2 kata spesifik yang ditanyakan user, lalu TERJEMAHKAN ke Bahasa Inggris.\n\nATURAN MUTLAK:\n1. Abaikan subjek utama (Jangan pernah output kata 'Iran' atau 'Epstein').\n2. Output HARUS dalam Bahasa Inggris.\n3. Hanya output kata spesifik.\n4. Output murni kata tanpa tanda baca."
}
```

### 4. Context Flattening dengan JavaScript (Code Node)

Agar LLM tidak menerima data terstruktur JSON yang kompleks, node JavaScript meratakan 20 baris hasil Supabase menjadi satu blok teks narasi yang siap dikonsumsi LLM:

```javascript
let gabunganKonteks = "";

// Menggabungkan 20 data menjadi 1 teks panjang
for (let item of $input.all()) {
  gabunganKonteks += `[Sentimen: ${item.json.sentiment}] Teks: ${item.json.extracted_text}\n\n`;
}

// Meneruskan hasil gabungan ke node selanjutnya
return [{ json: { referensi_data: gabunganKonteks } }];
```

### 5. Generasi Jawaban Akhir dengan Bilingual Support

LLM akhir diperintahkan untuk menjawab menggunakan bahasa yang sama persis dengan bahasa yang digunakan user (Bahasa Indonesia atau Inggris), meskipun referensi data berbahasa Inggris:

```json
{
  "role": "system",
  "content": "Kamu adalah asisten intelijen. Jawab pertanyaan user HANYA berdasarkan REFERENSI DATA berikut.\n\nATURAN MUTLAK:\n1. Jawab menggunakan BAHASA YANG SAMA persis dengan bahasa yang digunakan user saat bertanya.\n2. Jika referensi data berbahasa Inggris dan user bertanya dalam bahasa Indonesia, terjemahkan informasi dari referensi tersebut ke bahasa Indonesia saat menjawab.\n\nREFERENSI DATA:\n[data dari Supabase]"
}
```

---

## Panduan Instalasi & Pengoperasian

### Prasyarat

- n8n (self-hosted atau cloud) dengan node Apify dan Supabase tersedia.
- Akun Apify dengan akses ke actor `lofomachines/epstein-files-scraper-api`.
- Project Supabase dengan tabel `epstein_analysis` sesuai skema di atas.
- Akun Groq Cloud dengan API Key aktif.
- Telegram Bot Token (buat via @BotFather).

### Langkah Import Workflow

1. Login ke dashboard n8n.
2. Buat workflow baru, klik menu **⋯ → Import from File**.
3. Import `Epstein_Crawl.json` untuk pipeline crawler.
4. Import `Chatbot_AI.json` untuk pipeline chatbot.
5. Pada masing-masing workflow, perbarui **Credentials**:
   - `apifyApi`: masukkan Apify API Key.
   - `supabaseApi`: masukkan Supabase URL dan anon key.
   - `telegramApi`: masukkan Telegram Bot Token.
   - Header `Authorization` pada HTTP Request nodes: ganti dengan Groq API Key aktif (`Bearer gsk_...`).
6. Aktifkan kedua workflow menggunakan toggle **Active**.

### Menjalankan Dashboard

Buka file `Dashboard.html` di browser modern mana pun. Dashboard langsung terhubung ke Supabase dan menampilkan data secara *real-time* tanpa memerlukan server tambahan.

> **Catatan:** Pastikan Supabase URL dan anon key pada bagian `<script>` di dalam `Dashboard.html` sudah dikonfigurasi sesuai project Supabase Anda sebelum digunakan di lingkungan lain.

---
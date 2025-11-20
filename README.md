# ETL Data Transaksi Pelanggan

## Gambaran Umum
DAG Airflow ini melakukan operasi ETL (Extract, Transform, Load) harian untuk mentransfer data transaksi pelanggan dari Oracle Database ke PostgreSQL.

## Detail DAG
- **ID DAG**: `etl_data_transaksi_pelanggan`
- **Jadwal**: Setiap hari pukul 00:00 (`0 0 * * *`)
- **Deskripsi**: ETL harian dari Oracle ke PostgreSQL untuk data transaksi pelanggan
- **Tags**: `etl`, `oracle`, `postgres`, `transaksi`

## Prasyarat

### Oracle Instant Client
DAG membutuhkan Oracle Instant Client yang terinstall di:
```
/opt/oracle/instantclient
```

File library yang diperlukan:
- `libclntsh.so`
- `libocci.so`

### Koneksi Airflow
DAG membutuhkan dua koneksi Airflow:

1. **Koneksi Oracle** (`oracle_conn_id`)
   - Tipe Koneksi: Oracle
   - Host: Host database Oracle
   - Port: Port database Oracle
   - Schema: Nama service Oracle
   - Login: Username Oracle
   - Password: Password Oracle

2. **Koneksi PostgreSQL** (`postgres_conn_id`)
   - Tipe Koneksi: PostgreSQL
   - Host: Host database PostgreSQL
   - Port: Port database PostgreSQL
   - Schema: Nama database PostgreSQL
   - Login: Username PostgreSQL
   - Password: Password PostgreSQL

## Data Sumber

### Tabel Sumber Oracle
- **Tabel**: `APPS.DATA_TRANSAKSI_PELANGGAN_V` (View)
- **Lokasi**: Database Oracle

### Tabel Target
- **Schema**: `master_data`
- **Tabel**: `data_transaksi_pelanggan`
- **Lokasi**: Database PostgreSQL

## Skema Tabel

Tabel target mencakup kolom-kolom berikut:

| Nama Kolom | Tipe Data | Deskripsi |
|------------|-----------|-----------|
| REQUEST_ID | NUMERIC | Identifikasi permintaan |
| REQUEST_DATE | DATE | Tanggal permintaan |
| CUSTOMER_TRX_LINE_ID | NUMERIC | ID baris transaksi pelanggan |
| CREATION_DATE | DATE | Tanggal pembuatan |
| INVOICE_NUMBER | VARCHAR(20) | Nomor invoice |
| SO_NUMBER | NUMERIC | Nomor sales order |
| ORDER_TYPE | VARCHAR(50) | Tipe pesanan |
| ORDER_TYPE_CODE | VARCHAR(20) | Kode tipe pesanan |
| ORDER_TYPE_DESC | VARCHAR(30) | Deskripsi tipe pesanan |
| ORG_ID | NUMERIC | ID organisasi |
| ORG_NAME | VARCHAR(240) | Nama organisasi |
| ORG_CODE | VARCHAR(10) | Kode organisasi |
| CUST_ACCOUNT_ID | NUMERIC | ID akun pelanggan |
| CUSTOMER_NAME | VARCHAR(360) | Nama pelanggan |
| CUSTOMER_CITY | VARCHAR(60) | Kota pelanggan |
| CUSTOMER_PROVINCE | VARCHAR(80) | Provinsi pelanggan |
| LINE_NUMBER | NUMERIC | Nomor baris |
| ITEM_ID | NUMERIC | ID item |
| ITEM_CODE | VARCHAR(40) | Kode item |
| ITEM_DESCRIPTION | VARCHAR(240) | Deskripsi item |
| ITEM_TYPE | VARCHAR(30) | Tipe item |
| QUANTITY | NUMERIC | Kuantitas |
| PRICE | NUMERIC | Harga |
| TOTAL_PRICE | NUMERIC | Total harga |
| KOTA | VARCHAR(50) | Kota |
| PROVINSI | VARCHAR(50) | Provinsi |
| LOCATION_ID | NUMERIC | ID lokasi |
| A_CUST_NAME | VARCHAR(500) | Nama pelanggan alternatif |
| CITY | VARCHAR(100) | Kota |
| PROVINCE | VARCHAR(100) | Provinsi |
| LATITUDE | NUMERIC | Koordinat latitude |
| LONGITUDE | NUMERIC | Koordinat longitude |

## Proses ETL

### 1. Inisialisasi Klien Oracle
- Memvalidasi instalasi Oracle Instant Client
- Mengatur variabel environment yang diperlukan
- Menginisialisasi klien cx_Oracle

### 2. Koneksi Database
- Membangun koneksi ke Oracle menggunakan DSN
- Membangun koneksi ke PostgreSQL
- Memvalidasi kedua koneksi

### 3. Ekstraksi Data
- Menghitung total record dalam tabel sumber
- Streaming data dari Oracle menggunakan `stream_results=True`
- Memproses data dalam chunk untuk efisiensi memori

### 4. Loading Data
- Membuat tabel target jika belum ada (dengan skema yang sesuai)
- Mengosongkan data yang ada di tabel target
- Memasukkan data dalam batch 10.000 record
- Menggunakan manajemen transaksi untuk konsistensi data

### 5. Penanganan Error
- Penanganan exception yang komprehensif untuk kedua database
- Rollback transaksi jika terjadi error
- Pembersihan koneksi yang proper di blok finally

## Fitur Performa

- **Streaming Results**: Mencegah masalah memori dengan dataset besar
- **Batch Processing**: Memproses data dalam chunk 10.000 record
- **Manajemen Koneksi**: Pembukaan dan penutupan koneksi yang proper
- **Keamanan Transaksi**: Menggunakan transaksi PostgreSQL dengan kemampuan rollback

## Pemantauan

DAG menyediakan logging detail termasuk:
- Status koneksi untuk kedua database
- Jumlah record sebelum transfer
- Update progres selama transfer
- Statistik transfer akhir
- Detail error dengan stack trace

## Pemulihan Error

- **Percobaan Ulang**: 1 retry jika gagal
- **Jeda Retry**: 5 menit antara retry
- **Dependensi**: Tidak bergantung pada run sebelumnya

## Deployment

1. Pastikan Oracle Instant Client terinstall di `/opt/oracle/instantclient`
2. Buat koneksi Airflow yang diperlukan di Airflow UI
3. Deploy DAG ke instance Airflow Anda
4. Aktifkan DAG di Airflow UI

## Pemeliharaan

- Pantau run DAG melalui Airflow UI
- Periksa logs untuk masalah koneksi atau data
- Pastikan ruang disk cukup untuk kedua database
- Pantau performa untuk volume data besar

## Troubleshooting

Masalah umum dan solusinya:

1. **Error Klien Oracle**: Verifikasi instalasi Instant Client dan permission file
2. **Masalah Koneksi**: Periksa konfigurasi koneksi Airflow
3. **Masalah Memori**: Kurangi ukuran chunk untuk dataset yang sangat besar
4. **Error Permission**: Verifikasi permission user database untuk sumber dan target

## Catatan Keamanan

- Kredensial database dikelola melalui koneksi Airflow
- Tidak ada password yang hardcoded dalam kode DAG
- Pastikan akses jaringan yang proper antara Airflow dan kedua database

## Konfigurasi Kode

Perubahan yang diperlukan dalam kode:

```python
# Ganti nama tabel sumber
result = oracle_connection.execution_options(stream_results=True).execute(
    text("SELECT * FROM APPS.DATA_TRANSAKSI_PELANGGAN_V")
)

# Ganti nama tabel target
pg_connection.execute(text("""
    CREATE TABLE IF NOT EXISTS data_transaksi_pelanggan (
        ...
    )
"""))

# Ganti schema target
pg_connection.execute(text("SET search_path TO master_data"))
```

DAG ini dirancang untuk mentransfer data transaksi pelanggan secara efisien dan aman antara sistem Oracle dan PostgreSQL dengan jadwal harian.

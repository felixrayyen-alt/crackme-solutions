# 📓 Learning Journal : Reverse Engineering - Kriptografi Kustom "findlicensekey"

**Disclaimer:** Dokumen ini merupakan catatan jurnal pembelajaran saya dalam menganalisis algoritma pembuatan lisensi (*keygen*) pada tantangan *crackme*. Fokus analisis ini adalah memetakan kerentanan kriptografi kustom menggunakan metode analisis statis (*Static Analysis*).

---

##  1. Fase Pengintaian (Static Analysis & String Extraction)

Prosedur analisis diawali dengan melakukan pemeriksaan permukaan terhadap komponen internal *binary* menggunakan utilitas perintah `strings`. Langkah ini bertujuan untuk mengidentifikasi keberadaan konstanta teks terenkripsi ataupun tabel substitusi (*lookup table*).

### A. Identifikasi Kakas Teks (Strings Extraction)

Saat mengeksekusi perintah ekstraksi teks:
bash
strings ./findlicensekey
Saya menemukan sebuah rangkaian karakter alfanumerik acak yang sangat spesifik dan mencurigakan di dalam segmen .rodata:QAZPLWSXOKMEYDCIJNRFVUHBTGqpalzmwoeirutyskdjfhgxncbv1750284369Karakteristik rangkaian ini terdiri atas huruf besar, huruf kecil, dan angka tanpa pola sekuensial standar. Berdasarkan analisis awal, saya menduga kuat bahwa rangkaian ini berfungsi sebagai Custom Lookup Table untuk algoritma substitusi chiper.

### B. Dekompilasi Fungsi main via Ghidra

Setelah memuat binary ke dalam Ghidra, saya menganalisis fungsi main untuk memahami bagaimana program berinteraksi dengan pengguna. Alur kontrol program dapat dipetakan sebagai berikut:

<img width="886" height="997" alt="image" src="https://github.com/user-attachments/assets/2dcc5417-d638-4098-a683-8517e2160477" />

- Program mewajibkan pengguna untuk menyertakan sebuah parameter argumen berbasis teks saat eksekusi awal, yang bertindak sebagai <username>.

- Program kemudian meminta input sekunder melalui standard input (stdin) berupa sebuah kode lisensi (License Key).

- Terakhir, program melakukan validasi dengan membandingkan nilai kode lisensi yang dimasukkan pengguna dengan kode lisensi hasil kalkulasi internal menggunakan fungsi strcmp.

## 2. Membedah Fungsi Generator Utama (FUN_00101189)

Kunci lisensi yang valid tidak disimpan secara statis, melainkan dihasilkan secara dinamis di dalam memori melalui subrutin khusus, yaitu fungsi FUN_00101189.Berikut adalah rekonstruksi kode asal berdasarkan jendela Decompiler Ghidra:Cvoid FUN_00101189(long param_1, long param_2)

<img width="626" height="287" alt="image" src="https://github.com/user-attachments/assets/529e1b8c-40dc-46ce-9a0f-6d4f5555963e" />

### Analisis Logika dan Aljabar Kode:

Berdasarkan pembacaan parameter register dan struktur perulangan di atas, saya merumuskan beberapa batasan teknis sebagai berikut:

- Batasan Panjang Kunci (0x18): Nilai heksadesimal 0x18 setara dengan 24 dalam desimal. Hal ini mengindikasikan bahwa batas iterasi looping adalah 24 kali, sehingga panjang karakter License Key yang dihasilkan akan selalu konstan sepanjang 24 karakter.

- Kerentanan Buffer Overflow Semantik: Mengingat perulangan membaca nilai karakter dari variabel param_1 (alamat memori username) hingga indeks ke-23, maka variabel username yang diinputkan pengguna wajib memiliki panjang minimal 24 karakter. Jika kurang, program akan mengalami out-of-bounds read (membaca sampah memori/alamat memori tetangga), yang mengakibatkan hasil kalkulasi kunci menjadi tidak konsisten.
  
- Operasi Modulo Berbasis Tabel (0x3e): Nilai heksadesimal 0x3e setara dengan 62 dalam desimal. Angka ini tepat mewakili jumlah total karakter alfanumerik yang ada pada Custom Lookup Table. Operasi modulo berguna untuk mengunci nilai indeks agar tidak keluar dari batas array string substitusi.Secara matematis, rumus pembentukan setiap karakter lisensi pada posisi indeks $i$ dapat dinyatakan dengan persamaan berikut:
  
Indeks_Substitusi_i = (i + ASCII_Username_i) mod{62}

## 3. Pembuatan Skrip Keygen Interaktif (Python)

Untuk mereplikasi perilaku kriptografi internal program tanpa harus melakukan patching atau manipulasi register memori secara dinamis, saya mengimplementasikan algoritma di atas ke dalam skrip otomatisasi berbasis Python.Python#!/usr/bin/env python3
# keygen.py - Otomatisator Pembuat Lisensi untuk findlicensekey

def generate_key(username: str) -> str:
    lookup_table = "QAZPLWSXOKMEYDCIJNRFVUHBTGqpalzmwoeirutyskdjfhgxncbv1750284369"
    license_key = ""
    
    # Memastikan panjang input memenuhi syarat keamanan pembacaan memori
    if len(username) < 24:
        raise ValueError("Panjang username harus minimal 24 karakter!")
        
    for i in range(24):
        ascii_val = ord(username[i])
        index = (i + ascii_val) % 62
        license_key += lookup_table[index]
        
    return license_key

if __name__ == "__main__":
    # Menggunakan sampel username valid sepanjang 24 karakter
    sample_user = "felixrayyenzomeklopoiuyt"
    try:
        generated_key = generate_key(sample_user)
        print(f"[+] Username   : {sample_user}")
        print(f"[+] License Key: {generated_key}")
    except Exception as e:
        print(f"[-] Kesalahan: {e}")
        
## 4. Langkah Prosedural Eksekusi Akhir (Terminal Commands)

Berikut adalah urutan instruksi di terminal lingkungan kerja untuk mengeksploitasi fungsi validasi lisensi secara tepat:

- Langkah 1: Pembuatan Kode Kunci Menggunakan SkripMengeksekusi skrip generator yang telah dibuat sebelumnya guna mendapatkan pasangan data lisensi yang valid.Bashpython3 keygen.py
Output Sistem:Plaintext[+] Username   : felixrayyenzomeklopoiuyt
[+] License Key: ssngQ8kLWn4K9676QLSSACFI

- Langkah 2: Inisialisasi Program Target dengan Parameter ArgumenMenjalankan berkas crackme dengan menyematkan parameter username 24 karakter yang sama persis saat proses pembuatan kunci.Bash./findlicensekey felixrayyenzomeklopoiuyt

- Langkah 3: Inputasi Lisensi dan Kelulusan Gerbang ValidasiKetika program memicu interupsi input, masukkan kode lisensi hasil kalkulasi skrip Python:Plaintext
Enter license key to continue: 
ssngQ8kLWn4K9676QLSSACFI

Hasil Akhir Konfirmasi:PlaintextKey validated
## 5. Kesimpulan Analisis

Eksperimen ini membuktikan bahwa metode analisis statis (Static Analysis) dan pendekatan matematika terbalik (Reverse Cryptography) jauh lebih efisien untuk menembus proteksi berbasis validasi logis konstan.Dengan memetakan cara kerja fungsi pengubah indeks FUN_00101189, pembuatan program pembobol (keygen) independen dapat dilakukan secara bersih tanpa perlu melakukan intervensi memori yang berisiko memicu proteksi anti-debugging aktif.

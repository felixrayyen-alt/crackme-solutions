# 📓 Learning Journal: Reverse Engineering - Crackme "James"
https://crackmes.one/crackme/6a0c8b608fab7bbca2730221 

**Disclaimer:** 
Hari ini gua nyoba ngebedah sebuah file tantangan (*crackme*) yang namanya **james**. Tujuannya simpel: kita harus nyari *password* yang bener biar programnya ngeluarin pesan sukses, bukan malah ngatain kita salah *password*. Sebelumnya, file ini sebenernya ada 2 setelah di kompress. 

<img width="512" height="395" alt="image" src="https://github.com/user-attachments/assets/189782c0-252d-4204-8eef-d76087610e41" />

Jadi, konteks crackme si james ini ada di asset semua ya. 

Berikut adalah perjalanan gua nge-crack si James dari awal sampai tembus!

---


##  Tahap 1: Pemanasan Pake Perintah `file`
Langkah pertama yang selalu diajarin dosen/suhu RE adalah ngecek jenis file dari program ini. Kenapa? Biar kita tau ntar mau dianalisa pake *tools* apa.

Gua jalanin perintah ini di terminal:
bash
file james
<img width="764" height="95" alt="image" src="https://github.com/user-attachments/assets/c844e73e-7ba4-4487-bb72-76ab79091055" />

Hasilnya:
Biasanya output-nya bakal ngasih tau kalau file ini adalah ELF 64-bit LSB pie executable.
Artinya apa bang? Gampangnya, ini adalah file aplikasi (kayak .exe di Windows) tapi buat jalan di sistem Linux. Terus arsitekturnya 64-bit. Jadi gua tau nih, oh ini file Linux, berarti aman buat di-Ghidra-in.

## Tahap 2: Ngintip Pake Perintah strings

Karena gua mager langsung baca kode, gua pake cara kedua yaitu strings. Perintah ini fungsinya buat nampilin semua teks utuh (yang bisa dibaca manusia) yang nyelip di dalem file tersebut. Siapa tau pembuatnya males securenya dan ninggalin password aslinya di situ.

Gua jalanin:

Bash
strings james

<img width="764" height="478" alt="image" src="https://github.com/user-attachments/assets/e37b1e1f-d7f7-4362-9edd-ec7856b7e0a8" />

(_ga bisa masukin semua karena panjang banget pas di cek_)
Hasilnya:
Gua dapet banyak banget teks sampah, tapi mata gua langsung tertuju ke beberapa kalimat menarik:

Enter password: (Ini pasti tempat kita ngisi input).

Teks-teks aneh yang berpotensi jadi password.

Pesan gagal (kayak Wrong password!).

Pesan sukses yang bikin lega.

Dari sini gua udah dapet bayangan: "Oh, program ini bakal minta input, terus input gua bakal dicocokin sama string/teks rahasia yang ada di dalem memori."

## Tahap 3: Membedah Otak James di Ghidra (Analisis Statis)
Karena strings doang belum cukup buat ngasih kepastian (kadang teksnya diacak), akhirnya gua mutusin buat masukin file james ini ke dalem Ghidra.

Ini langkah-langkah gua di Ghidra:

### Nyari Fungsi main
Gua langsung meluncur ke tab Symbol Tree, cari folder Functions, terus klik fungsi main. Ini ibarat nyari pintu masuk utama rumah si James.
<img width="1180" height="796" alt="image" src="https://github.com/user-attachments/assets/0b14cd48-c82a-46c0-abe7-9c2ae57f26f0" />

### Menganalisa Alur Program (Decompile)
Di layar sebelah kanan (layar Decompile), Ghidra langsung nerjemahin bahasa alien (Assembly) jadi bahasa C++ yang lumayan bisa dipahami.

Gua baca perlahan logika dari programnya:

Pengecekan Argumen/Input: Program ngecek apakah kita masukin argumen (password) pas jalanin program.

Pengecekan Logika Kunci: Nah, di sini biasanya letak keseruannya. Gua merhatiin fungsi perbandingan (kayak strcmp atau fungsi custom yang ngecek huruf per huruf).

Kondisi Sukses/Gagal: Ada if-else yang ngecek: "Kalau perbandingan tadi hasilnya sama dengan 0 (alias match), maka cetak pesan Sukses. Kalau nggak, cetak pesan Gagal."

### Menemukan Titik Terang
Setelah gua perhatiin kode decompile dari Ghidra di atas, keliatan banget kalau fungsi ini adalah fungsi yang kepanggil pas kita neken tombol "Login" atau "Submit". Berikut adalah alur logika yang terjadi di dalam fungsinya:
<img width="512" height="337" alt="image" src="https://github.com/user-attachments/assets/f2c5133b-bf91-45b3-b39f-d898e5c9fa05" />

Di baris awal, program ngambil teks yang kita ketik di kotak input.
Kodenya: QLineEdit::text() dan QString::toStdString.
Artinya: Program ngebaca tulisan dari antarmuka kotak teks (biasanya buat masukin username atau password), lalu diubah jadi format string yang bisa diproses sama bahasa C++.

2. Pengecekan Input Kosong (Null Check)
Program gak mau dikibulin pake input kosong.
Kodenya: if ((local_a0 == 0) || (local_80 == 0))
Artinya: Kalau panjang inputnya nol (kita gak ngetik apa-apa tapi nekat pencet Submit), program bakal nyetak pesan error: "Error: Input arguments cannot be NULL" ke layar (QLabel::setText), terus fungsi langsung berhenti.

3. Fungsi Misterius & Perbandingan Kunci (The Core Logic)
Ini dia jantung pertahanannya!
Kodenya: FUN_00107840(&local_68,&local_a8); dilanjutin sama memcmp(local_88,local_68,local_80);
Artinya: Program manggil fungsi rahasia (FUN_00107840) buat ngeracik kunci jawaban asli. Terus, fungsi memcmp (Memory Compare) dipakai buat men-coba mencocokkan hasil inputan kita dengan racikan kunci asli tadi.
Kalau hasilnya gak sama (nilai iVar6 != 0), program bakal langsung jump (lompat) ke LAB_0010582e.

4. Jalur Zonk (Authentication Failed)
Kalau memcmp tadi gagal, kita dilempar ke label LAB_0010582e.
Artinya: Program bakal nampilin pesan "Authentication failed. Invalid client credentials." ke layar, dan jahatnya kotak input kita bakal dikosongin otomatis pake QLineEdit::clear().

5. Jalur Kemenangan & Rahasia Admin (The Easter Egg)
Nah, kalau memcmp tadi cocok, program bakal masuk ke jalur sukses. Dia bakal nyatet waktu login (QDateTime), ngasih pesan Session Established, dan ganti halaman layar. Tapi ada yang epik di sini!
Kodenya: if ((local_a0 == 8) && (*local_a8 == 0x656f645f6e686f6a))
Artinya: Kalau input kita panjangnya tepat 8 huruf, DAN nilainya sama dengan nilai Hex 0x656f645f6e686f6a, kita dapet jackpot!
Fun Fact: Di dunia komputer (yang pakai sistem Little Endian / bacanya dari belakang), kalau kita terjemahin hex 65 6f 64 5f 6e 68 6f 6a ke karakter ASCII, hasilnya adalah john_doe!
Kalau kita masukin john_doe, program bakal ngeluarin ASCII Art raksasa bertuliskan YOU CRACKED IT ditambah pesan sakti: "Admin Authorization bypass code successfully captured."

### Kesimpulan & Solusi
Dari hasil nge-reverse pake Ghidra, ketahuan lah rahasia terdalam si James. Ternyata cara kerja programnya sesimpel nyocokin input kita dengan kunci rahasianya.

Sebelum gua coba jalanin programnya di terminal dan masukin password yang gua dapet dari analisis Ghidra tadi, Ternyata dia menggunakan platform GUI yaitu Qt6 yang dimana platform itu membutuhkan library tambahan pada Ubuntu kita. Jadi, Setelah gua udah coba _Static analysis_ , gua langsung lakuin _Dynamic Analysis_ di Qt6 tersebut dengan menggunakan breakpoint karena memang dari awal gua sendiri ga menemukan password aslinya jadi kita pake teknik _Dynamic bypassing_

### Eksekusi (Step-by-Step GDB Commands)

#### Langkah 1: Masuk ke Dimensi GDB (Catatan: Pastikan urusan LD_LIBRARY_PATH buat Qt6 udah aman).
Bash
gdb ./target_bin

#### Langkah 2: Buka Restoran Biarkan pelayan Qt6 nyiapin meja dan jendela GUI sampai selesai.
Code snippet
r

#### Langkah 3: Siapkan Tumbal Masakan Di jendela aplikasi yang udah kebuka, masukin Username: john_doe dan kasih Password bohongan: 12345678. Jangan klik Login dulu.

#### Langkah 4: Hentikan Waktu Balik ke terminal GDB, lalu tekan tombol ini di keyboard buat nge-pause paksa: Ctrl + C

#### Langkah 5: Pasang Jebakan Khusus Juri Karena restorannya udah selesai loading, sekarang aman buat ngehadang Juri memcmp.
Code snippet
b memcmp

#### Langkah 6: Lanjutkan Waktu dan Hidangkan! Suruh waktu berjalan normal lagi, lalu lu pindah ke jendela aplikasi dan Klik Login.
Code snippet
c

#### Langkah 7: Intip Piring Si Juri (Ekstraksi Password) Aplikasi bakal langsung freeze pas Juri mau nyicip. Di momen ini, kita bongkar isi piringnya untuk ngintip wujud aslinya dalam format teks (x/s = examine string).
Code snippet
x/s $rdi
x/s $rsi
Hasil Bongkaran Memori:
Piring kanan ($rdi) isinya masakan kita: "12345678"
Piring rahasia ($rsi) isinya resep asli: "MTAwMTA="
<img width="1280" height="960" alt="WhatsApp Image 2026-07-04 at 17 08 10" src="https://github.com/user-attachments/assets/8b122c56-980d-4b30-87b9-0b960994621d" />
<img width="1280" height="960" alt="WhatsApp Image 2026-07-04 at 17 08 11" src="https://github.com/user-attachments/assets/3421ab9a-d8e3-4cc2-b462-c9ba0b7f0a85" />




# 📓 Learning Journal: Reverse Engineering - Crackme "halftwin"

**Disclaimer:** Dokumen ini merupakan catatan jurnal pembelajaran saya dalam menganalisis dan menyelesaikan tantangan *Reverse Engineering* (RE) pada *crackme* bernama **halftwin**. Analisis dilakukan menggunakan pendekatan analisis statis untuk memahami logika validasi internal program.

---

## 🔍 1. Analisis Awal (Basic Reconnaissance)

Langkah pertama dalam menganalisis file *binary* adalah mengumpulkan informasi dasar menggunakan *command-line tools* standar di lingkungan Linux. 

### A. Memeriksa Informasi File
Meskipun tahap ini terkesan mendasar, mengeksekusi perintah `file` sangat penting untuk mengidentifikasi arsitektur program:
bash
file halftwin

<img width="645" height="111" alt="image" src="https://github.com/user-attachments/assets/0bbb7cb1-fffb-4426-b7c1-3d478b64d12c" />

(Umumnya program seperti ini adalah ELF 64-bit LSB executable, yang menandakan bahwa ini adalah aplikasi berbasis Linux 64-bit).



B. Ekstraksi String
Selanjutnya, saya menggunakan perintah strings untuk mengekstrak teks (ASCII) yang dapat dibaca oleh manusia dari dalam file binary. Langkah ini krusial untuk memetakan pesan-pesan error dan menemukan petunjuk tersembunyi.

Bash
strings ./halftwin

<img width="637" height="398" alt="image" src="https://github.com/user-attachments/assets/55776e96-f1bf-4336-b1f3-e0ec7c2cde0c" />


Dari hasil perintah tersebut, saya menemukan beberapa strings berupa dialog karakter fiksi yang memberikan korelasi langsung dengan syarat password:

- hmm... i'm not sure you know what the word "twins" mean :/
(Indikasi bahwa program mengharapkan dua buah input argumen).

- Abby: i'm older than that :( dan Gabby: i'm older than that :(
(Indikasi adanya batasan minimum panjang/karakter untuk masing-masing input).

- Abby & Gabby: for god's sake we are TWINS! we were born the same night!!
(Indikasi bahwa panjang kedua string harus sama).

- Abby & Gabby: we are not "odd" years old :(
(Indikasi bahwa panjang input tidak boleh ganjil/odd, melainkan harus bernilai genap).

- Abby & Gabby: we're half twins you know...
(Petunjuk utama tentang adanya logika perbandingan setengah-setengah).

- Abby & Gabby: yaayy!! nice job! :D
(Pesan target yang menandakan keberhasilan).

## Fase Pengintaian (Static Analysis via Ghidra)
Berdasarkan kepingan petunjuk dari tahap strings, analisis dilanjutkan secara statis menggunakan decompiler Ghidra. Setelah melakukan dekompilasi pada fungsi main, ditemukan bahwa program ini tidak menggunakan pencocokan password statis atau fungsi konversi angka murni. Program membedah karakter teks secara manual menggunakan pengkondisian (if-else) yang bertingkat.

Berikut adalah potongan kode penting dari jendela Decompiler Ghidra:

<img width="620" height="846" alt="image" src="https://github.com/user-attachments/assets/e58e83c6-68ee-4b5b-9b15-1ed2088431db" />


## Membedah Logika Sensor Si Kembar (The Rules)
Berdasarkan hasil sintesis dari kode C di atas dan korelasi dengan strings sebelumnya, pembuat crackme menerapkan 4 aturan QC (Quality Control) ketat yang wajib dipenuhi oleh dua input string kita (__s dan __s_00):

- Aturan 1 (Jumlah Argumen): if (param_1 == 3)
Program membutuhkan tepat 2 argumen tambahan di terminal setelah nama program. Jika kurang, program akan mengeluarkan peringatan tentang arti kata "twins".

- Aturan 2 (Batas Minimal Karakter): if (sVar1 < 7)
Panjang dari masing-masing string tidak boleh kurang dari 7 karakter.

- Aturan 3 (Syarat Angka Genap): if ((sVar1 & 1) == 0)
Operasi bitwise AND ini digunakan untuk memastikan bahwa panjang karakter harus Genap (Least Significant Bit bernilai 0). Mengingat batas minimumnya adalah 7, maka panjang string valid terkecil yang bisa diinputkan adalah 8 karakter.

- Aturan 4 (Gerbang "Half Twins"):
Program membagi pengecekan karakter melalui dua buah perulangan (for loop):

Setengah Kiri (Indeks 0 s.d Length/2): Menggunakan pengecekan if (__s[i] != __s_00[i]). Ini berarti, paruh pertama karakter dari kedua input harus sama persis.

Setengah Kanan (Indeks Length/2 s.d Selesai): Menggunakan pengecekan if (__s[i] == __s_00[i]). Ini berarti, paruh kedua karakter dari kedua input harus berbeda di setiap posisinya.

## Pembuatan Payload & Langkah Eksekusi (Terminal Commands)
Untuk memenuhi semua gerbang pengkondisian logika di atas, saya merancang sebuah strategi payload eksperimental dengan panjang 8 karakter:

Argumen 1 (__s): AAAABBBB

Argumen 2 (__s_00): AAAACCCC

Validasi Logika: Panjang total adalah 8 (Memenuhi kriteria Genap & > 7). Empat huruf pertama identik (AAAA), dan empat huruf terakhir sengaja dibedakan secara posisi (BBBB vs CCCC).

##  Eksekusi Akhir
Oleh karena seluruh validasi murni bersifat logika statis, pendekatan dinamis menggunakan debugger tidak diperlukan. Payload yang telah dirancang dapat langsung disuapkan ke dalam program melalui terminal.

Bash
./halftwin AAAABBBB AAAACCCC

<img width="668" height="55" alt="image" src="https://github.com/user-attachments/assets/64777ba3-548f-4bf1-8b32-1e01dd5241b0" />

Output Kemenangan (Success):
Apabila parameter tepat, program akan melewati seluruh instruksi penghentian prematur dan mencetak teks berikut:

Plaintext
Abby & Gabby: yaayy!! nice job! :D

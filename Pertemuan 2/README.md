Nama: Wendy Virtus  
NIM: H1H024048  
Shift Awal: C  
Shift Akhir: C

---

# Pertanyaan Praktikum - Percobaan 2A
## 1. Gambarkan diagram alur (flowchart) proses koneksi ESP32 ke jaringan WiFi pada program di atas!
<img width="512" height="422" alt="image" src="https://github.com/user-attachments/assets/ecc4fbd9-bc33-43ef-8390-25462aeed73e" />

---

## 2. Apa fungsi dari perintah WiFi.mode(WIFI_STA) pada program tersebut?
Perintah WiFi.mode(WIFI_STA) digunakan untuk mengatur mode operasi modul WiFi pada ESP32 agar berperan sebagai Station (klien), yaitu perangkat yang akan terhubung ke jaringan WiFi yang sudah tersedia (seperti router atau hotspot), bukan sebagai penyedia jaringan. Penentuan mode ini penting karena ESP32 mendukung beberapa mode WiFi (STA, AP, AP+STA) dan setiap mode memiliki perilaku serta fungsi API yang berbeda, sehingga mode harus ditentukan secara eksplisit di awal program agar fungsi WiFi.begin() bekerja sesuai yang diharapkan.

---

## 3. Jelaskan apa yang terjadi apabila SSID atau password yang dimasukkan salah!
Apabila SSID yang dimasukkan salah (jaringan tidak ditemukan), ESP32 tidak akan pernah mencapai status WL_CONNECTED sehingga program akan terus berada pada perulangan while (WiFi.status() != WL_CONNECTED), mencetak tanda titik tanpa henti karena access point yang dituju tidak dapat ditemukan sama sekali. Apabila SSID benar namun password yang dimasukkan salah, ESP32 akan mencoba melakukan autentikasi ke access point namun ditolak, sehingga status koneksi juga tidak pernah berubah menjadi WL_CONNECTED dan program tetap terjebak pada perulangan yang sama. Pada kedua kasus tersebut, program tidak memberikan pesan kesalahan spesifik yang membedakan sebab kegagalan, sehingga diperlukan mekanisme timeout tambahan agar program dapat memberi tahu pengguna secara eksplisit dan tidak menunggu tanpa batas.

---

## 4. Modifikasi program agar ESP32 mencoba menghubungkan ulang (reconnect) secara otomatis apabila koneksi WiFi terputus, dan berikan penjelasan di setiap baris kode yang ditambahkan dalam bentuk README.md!
```cpp
#include <WiFi.h>
 
const char* ssid     = "NAMA_WIFI_ANDA";
const char* password = "PASSWORD_WIFI_ANDA";
const int ledPin = 2;
 
// --- Kode baru: variabel untuk mekanisme reconnect ---
unsigned long waktuTerakhirCek = 0;              // menyimpan waktu (millis) pemeriksaan status terakhir
const unsigned long intervalCek = 5000;          // interval pemeriksaan status koneksi (5 detik)
 
void hubungkanWiFi() {
  // --- Kode baru: fungsi terpisah agar proses koneksi dapat dipanggil ulang ---
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  Serial.print("Menghubungkan ke WiFi");
 
  // --- Kode baru: batas waktu (timeout) agar tidak menunggu tanpa henti ---
  unsigned long waktuMulai = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - waktuMulai < 10000) {
    delay(500);
    Serial.print(".");
  }
 
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi berhasil terhubung!");
    digitalWrite(ledPin, HIGH);
  } else {
    Serial.println("\nGagal terhubung, akan dicoba lagi nanti.");
    digitalWrite(ledPin, LOW);
  }
}
 
void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, LOW);
  hubungkanWiFi();   // panggil fungsi koneksi pertama kali
}
 
void loop() {
  // --- Kode baru: pemeriksaan berkala tanpa memblokir program dengan delay panjang ---
  if (millis() - waktuTerakhirCek >= intervalCek) {
    waktuTerakhirCek = millis();
 
    if (WiFi.status() == WL_CONNECTED) {
      Serial.println("Status: Terhubung");
    } else {
      // --- Kode baru: jika koneksi terputus, coba hubungkan ulang secara otomatis ---
      Serial.println("Status: Terputus, mencoba menghubungkan ulang...");
      digitalWrite(ledPin, LOW);
      hubungkanWiFi();
    }
  }
}
```

---

# Pertanyaan Praktikum - Percobaan 2B
## 1. Mengapa alamat IP default Access Point pada ESP32 umumnya bernilai 192.168.4.1?
Nilai 192.168.4.1 merupakan alamat IP default yang telah ditentukan oleh pustaka WiFi/Espressif SDK untuk mode Access Point pada ESP32, berada dalam rentang alamat IP privat (192.168.0.0/16) yang memang dialokasikan untuk jaringan lokal dan tidak dirutekan di internet publik. Penggunaan alamat default yang konsisten ini memudahkan developer maupun pengguna karena selalu dapat diakses melalui alamat yang sama (misalnya untuk mengakses halaman konfigurasi web) tanpa perlu mencari tahu alamat IP terlebih dahulu, kecuali dikonfigurasi ulang secara khusus menggunakan WiFi.softAPConfig().

---

## 2. Apa perbedaan mendasar antara mode Station dan mode Access Point pada ESP32?
Pada mode Station (STA), ESP32 berperan sebagai klien yang bergantung pada jaringan WiFi lain (router/hotspot) untuk mendapatkan akses jaringan, sehingga ESP32 harus terlebih dahulu mengetahui SSID dan password dari jaringan tersebut serta menerima alamat IP dari perangkat lain (DHCP server pada router). Sebaliknya, pada mode Access Point (AP), ESP32 justru berperan sebagai penyedia jaringan itu sendiri; ESP32 membuat SSID dan password sendiri, bertindak sebagai DHCP server yang memberikan alamat IP kepada perangkat lain yang terhubung, serta tidak memerlukan jaringan WiFi eksternal untuk dapat diakses.

---

## 3. Jelaskan risiko keamanan apabila password Access Point tidak diberikan atau terlalu sederhana!
Apabila Access Point tidak diberikan password (open network) atau menggunakan password yang terlalu sederhana (misalnya angka berurutan atau kata umum), siapa pun dalam jangkauan sinyal dapat dengan mudah terhubung ke jaringan tersebut tanpa izin. Hal ini membuka risiko seperti penyadapan data yang dikirim melalui jaringan (karena tidak terenkripsi dengan baik), akses tidak sah ke perangkat ESP32 dan sistem IoT yang terhubung dengannya (misalnya mengubah konfigurasi, mengirim perintah palsu ke aktuator), pembebanan berlebih pada Access Point oleh perangkat yang tidak berkepentingan, hingga serangan brute-force yang lebih mudah berhasil pada password yang lemah. Oleh karena itu, password Access Point sebaiknya cukup panjang, kompleks, dan tidak mudah ditebak.

---

## 4. Modifikasi program agar ESP32 berjalan pada mode AP+STA (terhubung ke WiFi rumah sekaligus menyediakan Access Point), dan berikan penjelasan di setiap baris kode nya dalam bentuk README.md!
```cpp
#include <WiFi.h>
 
// --- Kode baru: kredensial untuk mode Station (menghubungkan ke WiFi rumah) ---
const char* sta_ssid     = "NAMA_WIFI_RUMAH";
const char* sta_password = "PASSWORD_WIFI_RUMAH";
 
// Kredensial untuk mode Access Point (tetap disediakan ESP32)
const char* ap_ssid     = "ESP32_AccessPoint";
const char* ap_password = "12345678";
 
void setup() {
  Serial.begin(115200);
 
  // --- Kode baru: mengaktifkan mode gabungan AP+STA ---
  WiFi.mode(WIFI_AP_STA);
 
  // Bagian Access Point: ESP32 tetap menyediakan hotspot sendiri
  WiFi.softAP(ap_ssid, ap_password);
  Serial.print("AP IP Address   : ");
  Serial.println(WiFi.softAPIP());
 
  // --- Kode baru: bagian Station, ESP32 sekaligus terhubung ke WiFi rumah ---
  WiFi.begin(sta_ssid, sta_password);
  Serial.print("Menghubungkan ke WiFi rumah");
  unsigned long waktuMulai = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - waktuMulai < 10000) {
    delay(500);
    Serial.print(".");
  }
 
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println();
    Serial.print("STA IP Address  : ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("\nGagal terhubung ke WiFi rumah (mode AP tetap aktif).");
  }
}
 
void loop() {
  // Menampilkan jumlah perangkat yang terhubung ke AP setiap 5 detik
  int jumlahClient = WiFi.softAPgetStationNum();
  Serial.print("Jumlah client AP: ");
  Serial.println(jumlahClient);
  delay(5000);
}
```

---

# Pertanyaan Analisis
## 1. Uraikan hasil tugas pada praktikum yang telah dilakukan pada setiap percobaan!
Pada Percobaan 2A, ESP32 berhasil dikonfigurasi pada mode Station dan terhubung ke jaringan WiFi yang telah ditentukan, ditandai dengan tampilnya IP Address, MAC Address, dan nilai RSSI yang stabil pada Serial Monitor, serta LED indikator yang menyala sesuai spesifikasi. Pengujian dengan kredensial salah (password salah maupun SSID salah) menunjukkan bahwa ESP32 tidak pernah mencapai status terhubung dan terus mencoba tanpa henti, yang mengonfirmasi pentingnya validasi kredensial dan mekanisme timeout. Pada Percobaan 2B, ESP32 berhasil dikonfigurasi sebagai Access Point mandiri dengan SSID ESP32_AccessPoint dan alamat IP default 192.168.4.1, serta mampu memantau jumlah perangkat yang terhubung secara real-time melalui Serial Monitor, sesuai dengan seluruh spesifikasi yang diharapkan pada modul.

---

## 2. Bagaimana pengaruh kekuatan sinyal (RSSI) terhadap kestabilan koneksi WiFi pada perangkat IoT?
Nilai RSSI (Received Signal Strength Indicator) menggambarkan kekuatan sinyal WiFi yang diterima oleh ESP32, biasanya bernilai negatif dalam satuan dBm, di mana nilai yang semakin mendekati nol menunjukkan sinyal yang semakin kuat. Semakin lemah RSSI (misalnya di bawah -80 dBm), semakin besar pula kemungkinan terjadinya gangguan koneksi seperti packet loss, penurunan kecepatan transfer data, atau bahkan terputusnya koneksi secara tiba-tiba, karena sinyal yang lemah lebih rentan terhadap noise dan interferensi. Oleh karena itu, pada aplikasi IoT yang membutuhkan pengiriman data secara kontinu dan andal, penempatan perangkat pada area dengan RSSI yang cukup kuat (umumnya di atas -70 dBm) sangat penting untuk menjaga kestabilan koneksi.

---

## 3. Bagaimana cara kerja ESP32 dalam membedakan peran sebagai klien (Station) dan sebagai penyedia jaringan (Access Point)?
Perbedaan peran ditentukan oleh mode yang diatur melalui fungsi WiFi.mode(), yaitu WIFI_STA untuk berperan sebagai klien atau WIFI_AP untuk berperan sebagai penyedia jaringan (dan WIFI_AP_STA untuk keduanya sekaligus). Pada mode Station, modul WiFi ESP32 dikonfigurasi untuk melakukan proses scanning dan asosiasi ke access point eksternal menggunakan WiFi.begin(), lalu bertindak sebagai node dalam jaringan yang sudah ada dan meminta alamat IP dari DHCP server pada router. Pada mode Access Point, modul WiFi ESP32 justru dikonfigurasi untuk memancarkan SSID-nya sendiri melalui WiFi.softAP(), bertindak sebagai DHCP server yang mengalokasikan alamat IP kepada perangkat lain yang terhubung, serta menangani proses autentikasi client secara mandiri.

---

## 4. Bagaimana kombinasi mode Station dan Access Point (AP+STA) dapat dimanfaatkan dalam skenario nyata sistem IoT, misalnya pada proses konfigurasi awal perangkat (provisioning)?
Kombinasi mode AP+STA sangat bermanfaat pada skenario provisioning, yaitu proses awal ketika perangkat IoT belum mengetahui kredensial WiFi rumah/kantor yang akan digunakan. Pada tahap awal, ESP32 dapat mengaktifkan Access Point sementara agar pengguna dapat terhubung langsung ke perangkat menggunakan smartphone, lalu memasukkan SSID dan password jaringan rumah melalui halaman web sederhana yang dihosting oleh ESP32. Setelah kredensial tersimpan, ESP32 dapat langsung mencoba terhubung ke jaringan rumah tersebut melalui mode Station tanpa perlu memutus Access Point-nya, sehingga proses konfigurasi dapat dilakukan tanpa perlu meng-hardcode SSID/password ke dalam kode program maupun menghubungkan ESP32 ke komputer secara langsung. Pendekatan ini banyak digunakan pada perangkat IoT komersial (seperti smart plug atau smart bulb) agar proses setup menjadi lebih mudah bagi pengguna akhir.

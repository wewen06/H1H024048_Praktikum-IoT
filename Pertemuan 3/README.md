Nama: Wendy Virtus  
NIM: H1H024048  
Shift Awal: C  
Shift Akhir: C  

---

# Pertanyaan Praktikum - Percobaan 3A
## 1. Gambarkan diagram alur (flowchart) proses pengiriman data melalui HTTP POST pada program di atas!
<img width="2720" height="4136" alt="flowchart_http_post_rapi" src="https://github.com/user-attachments/assets/952b1523-4aee-423b-8790-2ab76f10aca4" />

---

## 2. Apa fungsi dari perintah http.addHeader("Content-Type", "application/json") pada program tersebut?
Perintah tersebut menambahkan header HTTP bernama "Content-Type" dengan nilai "application/json" pada request yang akan dikirimkan. Header ini memberi tahu server bahwa isi (body) dari request yang dikirim ESP8266 berformat JSON, sehingga server dapat mem-parsing dan menafsirkan data tersebut dengan benar sesuai strukturnya. Tanpa header ini, server dapat memperlakukan body request sebagai teks biasa (plain text) dan gagal mengurai data JSON yang dikirimkan.

---

## 3. Jelaskan arti dari kode response HTTP 200 dan sebutkan salah satu contoh kode response HTTP lain beserta artinya!
Kode response HTTP 200 (OK) menandakan bahwa request yang dikirimkan oleh klien (ESP32) berhasil diproses oleh server tanpa ada kesalahan, dan server mengembalikan data/response sesuai yang diharapkan; pada percobaan ini, kode 200 menunjukkan bahwa data JSON berhasil diterima oleh httpbin.org dan di-echo kembali sebagai response body. Contoh kode response HTTP lain adalah kode 404 (Not Found), yang berarti server tidak dapat menemukan resource atau endpoint yang diminta oleh klien, misalnya karena URL yang salah atau path endpoint tidak tersedia pada server.

---

## 4. Modifikasi program agar ESP8266 dapat mengirimkan data tambahan berupa waktu (dalam milidetik sejak dinyalakan menggunakan millis()) ke dalam JSON yang dikirim, dan berikan penjelasan di setiap baris kode yang ditambahkan dalam bentuk README.md
```cpp
#include <ESP8266WiFi.h>        
#include <ESP8266HTTPClient.h>  
#include <WiFiClientSecure.h>   
#include <ArduinoJson.h>

const char* ssid = "eduroamer";
const char* password = "wendygantengpoll";
const char* serverUrl = "https://httpbin.org/post"; 

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500); Serial.print(".");
  }
  Serial.println("WiFi berhasil terhubung!");
}
 
void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(serverUrl);
    http.addHeader("Content-Type", "application/json");
 
    JsonDocument doc;
    doc["suhu"] = 28.5;
    doc["kelembaban"] = 65.0;
    // --- Kode baru: menambahkan data waktu sejak ESP32 dinyalakan ---
    doc["waktu_ms"] = millis();
 
    String requestBody;
    serializeJson(doc, requestBody);
 
    int httpResponseCode = http.POST(requestBody);
    if (httpResponseCode > 0) {
      Serial.println(httpResponseCode);
      Serial.println(http.getString());
    }
    http.end();
  }
  delay(10000);
}
```

---

# Pertanyaan Praktikum - Percobaan 3B
## 1. Apa fungsi dari topic pada protokol MQTT, dan mengapa topic yang digunakan perlu dibuat unik?
Topic pada protokol MQTT berfungsi sebagai alamat atau label pengelompokan pesan, yang digunakan broker untuk menentukan ke mana suatu pesan yang dipublikasikan (publish) harus diteruskan, yaitu kepada seluruh client yang telah subscribe ke topic yang sama. Dengan adanya topic, satu broker dapat melayani banyak jenis data dan banyak perangkat sekaligus tanpa saling tercampur. Topic perlu dibuat unik (misalnya menyertakan nama kelompok atau identitas perangkat) terutama pada broker publik seperti broker.hivemq.com, karena broker tersebut dapat diakses oleh siapa saja; apabila topic yang digunakan sama dengan topic milik pengguna atau kelompok lain, data yang dipublikasikan dapat tercampur, terbaca oleh pihak yang tidak berkepentingan, atau bahkan menimbulkan konflik/gangguan pada pengujian masing-masing pihak.

---

## 2. Jelaskan fungsi dari perintah client.loop() yang dipanggil pada setiap iterasi loop()!
Perintah client.loop() pada pustaka PubSubClient wajib dipanggil secara berkala agar client MQTT dapat memproses berbagai aktivitas latar belakang yang diperlukan untuk menjaga koneksi ke broker tetap berjalan dengan baik, seperti mengirimkan paket keep-alive (PINGREQ) ke broker agar koneksi tidak dianggap terputus oleh server, memeriksa dan memproses pesan masuk apabila ESP8266 juga melakukan subscribe ke suatu topic, serta menangani proses internal lain terkait status koneksi. Apabila client.loop() tidak dipanggil secara rutin, koneksi MQTT dapat terputus meskipun secara fisik jaringan WiFi masih tersambung, karena broker tidak menerima sinyal keep-alive dari client dalam batas waktu yang ditentukan.

---

## 3. Apa yang akan terjadi apabila koneksi ke broker MQTT terputus di tengah program berjalan?
Apabila koneksi ke broker MQTT terputus, fungsi client.connected() akan mengembalikan nilai false pada iterasi loop() berikutnya. Pada program di atas, kondisi if (!client.connected()) akan terpenuhi sehingga program akan memanggil kembali fungsi hubungkanMQTT(), yang berisi perulangan while (!client.connected()) untuk terus mencoba menyambungkan ulang (reconnect) ke broker menggunakan client ID baru, dengan jeda 2 detik pada setiap percobaan yang gagal. Selama proses reconnect ini berlangsung, data sensor tidak dapat dipublikasikan (publish) karena koneksi belum tersedia, sehingga akan terjadi jeda pada pengiriman data hingga koneksi berhasil dipulihkan kembali.

---

# Pertanyaan Analisis
## 1. Uraikan hasil tugas pada praktikum yang telah dilakukan pada setiap percobaan!
Pada Percobaan 3A, ESP8266 berhasil terhubung ke WiFi dan mengirimkan data suhu-kelembaban dalam format JSON ke httpbin.org melalui HTTP POST, ditandai dengan kode response HTTP 200 dan isi response yang meng-echo data yang dikirim. Pada Percobaan 3B, ESP8266 berhasil terhubung ke broker broker.hivemq.com dan mempublikasikan data JSON secara berkala ke topic yang telah ditentukan, yang dapat diverifikasi melalui aplikasi client MQTT yang subscribe ke topic yang sama.

---

## 2. Bandingkan besar overhead data dan pola komunikasi antara protokol HTTP dan MQTT berdasarkan hasil percobaan yang telah dilakukan!
Berdasarkan hasil percobaan, protokol HTTP memiliki overhead yang jauh lebih besar dibandingkan MQTT karena setiap request HTTP POST menyertakan header lengkap (method, path, Host, Content-Type, Content-Length, dan header lain) serta harus membuka koneksi TCP/TLS baru pada setiap pengiriman data (bersifat request-response, stateless), sehingga proses pengiriman data suhu dan kelembaban yang berukuran kecil tetap membutuhkan pertukaran data tambahan yang relatif besar. Sebaliknya, protokol MQTT hanya memerlukan satu kali koneksi (dan autentikasi client) di awal, kemudian setiap publish hanya mengirimkan payload data beserta header MQTT yang berukuran sangat kecil (dapat kurang dari 10 byte) melalui koneksi yang sudah terjaga (persistent connection), sehingga pola komunikasinya lebih efisien untuk pengiriman data secara berkala/kontinu dibandingkan HTTP yang harus membangun ulang koneksi pada setiap pengiriman.

---

## 3. Untuk skenario pengiriman data sensor secara terus-menerus setiap beberapa detik dalam jangka waktu lama, protokol manakah (HTTP atau MQTT) yang lebih sesuai digunakan? Jelaskan alasannya!
Untuk skenario pengiriman data sensor secara terus-menerus setiap beberapa detik dalam jangka waktu lama, protokol MQTT lebih sesuai digunakan dibandingkan HTTP. Hal ini karena MQTT menggunakan koneksi persisten (persistent connection) yang hanya perlu dibangun satu kali, sehingga setiap pengiriman data berikutnya tidak perlu mengulang proses handshake koneksi seperti pada HTTP, yang secara signifikan mengurangi konsumsi daya dan bandwidth pada perangkat IoT dengan sumber daya terbatas seperti ESP8266. Selain itu, pola publish-subscribe pada MQTT memungkinkan satu data yang dipublikasikan diterima oleh banyak subscriber sekaligus tanpa perlu mengirim data berulang kali ke masing-masing tujuan, yang membuatnya lebih skalabel untuk pengiriman data kontinu dalam jangka waktu lama dibandingkan HTTP yang lebih cocok untuk komunikasi periodik atau tidak sering.

---

## 4. Bagaimana peran format JSON dalam mendukung interoperabilitas data antara perangkat IoT dan berbagai platform/aplikasi yang berbeda?
Format JSON berperan penting dalam mendukung interoperabilitas karena JSON merupakan format pertukaran data berbasis teks yang bersifat ringan, terstruktur, dan tidak bergantung pada bahasa pemrograman maupun platform tertentu (language-agnostic), sehingga data yang dihasilkan oleh ESP8266 (menggunakan bahasa C/C++ melalui pustaka ArduinoJson) dapat dengan mudah dibaca dan diproses oleh berbagai platform lain seperti aplikasi web (JavaScript), aplikasi mobile, dashboard IoT, maupun layanan cloud, tanpa perlu melakukan konversi format data yang rumit. Struktur data JSON berbasis pasangan key-value (misalnya suhu, kelembaban) juga memudahkan proses pemetaan (mapping) data ke berbagai jenis basis data maupun antarmuka pengguna, sehingga format ini menjadi standar de facto yang mendukung komunikasi antar perangkat dan aplikasi yang berbeda dalam ekosistem IoT.

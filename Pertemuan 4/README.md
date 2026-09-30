Nama: Wendy Virtus  
NIM: H1H024048  
Shift Awal: C  
Shift Akhir: C  

---

# Pertanyaan Praktikum - Percobaan 4A
## 1. Gambarkan diagram alur (flowchart) proses penerimaan dan pemrosesan pesan pada fungsi callback di atas! 
<img width="1933" height="2676" alt="flowchart_callback" src="https://github.com/user-attachments/assets/b4dabb95-dcf2-4a88-a240-291f5aeba45f" />

---

## 2. Apa yang akan terjadi apabila pesan yang dipublikasikan bukan merupakan format JSON yang valid?
Fungsi deserializeJson() akan mengembalikan kode error (misalnya InvalidInput untuk teks seperti perintah ON, atau IncompleteInput jika JSON terpotong), sehingga kondisi if (error) terpenuhi. Program mencetak "Gagal parsing JSON: ..." ke Serial Monitor lalu keluar dari callback melalui return. Akibatnya perintah tidak diproses dan LED tetap pada kondisi terakhirnya. Program tidak berhenti atau restart; ESP8266 tetap terhubung ke broker dan siap menerima pesan berikutnya. Tanpa pengecekan ini, program akan membaca nilai dari objek kosong dan perilaku aktuator menjadi tidak terjamin.

---

## 3. Jelaskan arti dari kode response HTTP 200 dan sebutkan salah satu contoh kode response HTTP lain beserta artinya!
Ada dua alasan. Pertama, client.subscribe() hanya berhasil bila client sudah terhubung ke broker, sedangkan di setup() koneksi MQTT belum dibuat (hanya setServer() yang dipanggil), sehingga perintah subscribe di sana akan gagal. Kedua, langganan melekat pada sesi koneksi. Pada PubSubClient, sesi dibuat dengan clean session sehingga saat koneksi terputus broker melupakan langganan client. Jika subscribe() hanya ditulis di setup() (yang hanya berjalan sekali), setelah reconnect ESP8266 tersambung kembali tetapi tidak lagi menerima pesan dari topic perintah. Dengan menaruhnya di hubungkanMQTT(), langganan didaftarkan ulang setiap kali koneksi berhasil dibuat atau dipulihkan.

---

## 4. Modifikasi program agar data JSON yang diterima juga memuat nilai intensitas (misalnya {"perintah": "ON", "intensitas": 200}) yang digunakan untuk mengatur kecerahan LED menggunakan PWM, dan berikan penjelasan di setiap baris kode yang ditambahkan dalam bentuk README.md
```cpp
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
 
const char* ssid = "eduroamer";
const char* password = "wendygantengpoll";
const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;
const char* topicPerintah =
  "unsoed/tk245004/kelompokwewenroland/perintah";
 
const int ledPin = D1;  // D1 mendukung PWM pada ESP8266
 
WiFiClient espClient;
PubSubClient client(espClient);
 
void callback(char* topic, byte* payload, unsigned int length) {
  String pesan;
  for (unsigned int i = 0; i < length; i++) pesan += (char)payload[i];
 
  Serial.print("Pesan diterima [");
  Serial.print(topic);
  Serial.print("]: ");
  Serial.println(pesan);
 
  JsonDocument doc;
  DeserializationError error = deserializeJson(doc, pesan);
  if (error) {
    Serial.print("Gagal parsing JSON: ");
    Serial.println(error.c_str());
    return;
  }
 
  const char* perintah = doc["perintah"] | "";          // [BARU] default ""
  int intensitas = doc["intensitas"] | 255;             // [BARU] default 255
  intensitas = constrain(intensitas, 0, 255);           // [BARU] batasi 0-255
 
  if (String(perintah) == "ON") {
    int duty = map(intensitas, 0, 255, 0, 1023);        // [BARU] skala PWM
    analogWrite(ledPin, duty);                          // [BARU] atur kecerahan
    Serial.print("Aktuator: ON, intensitas = ");        // [BARU]
    Serial.print(intensitas);                           // [BARU]
    Serial.print(" (duty PWM = ");                      // [BARU]
    Serial.print(duty);                                 // [BARU]
    Serial.println(")");                                // [BARU]
  } else if (String(perintah) == "OFF") {
    analogWrite(ledPin, 0);                             // [BARU] LED mati
    Serial.println("Aktuator: OFF");
  }
}
 
void hubungkanWiFi() {
  WiFi.begin(ssid, password);
  Serial.print("Menghubungkan ke WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi berhasil terhubung!");
}
 
void hubungkanMQTT() {
  while (!client.connected()) {
    String clientId = "ESP8266Client-" + String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      client.subscribe(topicPerintah);
      Serial.println("Terhubung & subscribe topic perintah");
    } else {
      delay(2000);
    }
  }
}
 
void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);
  analogWrite(ledPin, 0);                               // [BARU] awal mati
  hubungkanWiFi();
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback);
}
 
void loop() {
  if (!client.connected()) hubungkanMQTT();
  client.loop();
}
```

---

# Pertanyaan Praktikum - Percobaan 4B
## 1. Mengapa penggunaan delay() yang lama sebaiknya dihindari pada program yang menggabungkan proses publish dan subscribe secara bersamaan?
delay() bersifat blocking: selama jeda berlangsung, mikrokontroler tidak menjalankan baris kode lain, termasuk client.loop(). Padahal client.loop() yang membaca pesan masuk dari broker, memanggil callback, dan mengirim paket keep-alive. Jika delay() lama dipakai (misalnya 5–10 detik) di antara proses publish, perintah yang dikirim dari aplikasi baru diproses setelah delay selesai sehingga aktuator terlambat merespons. Bila jeda melebihi batas keep-alive (default PubSubClient 15 detik), broker menganggap client mati dan memutus koneksi. Karena itu sistem full duplex perlu dijalankan secara non-blocking.

---

## 2. Jelaskan cara kerja mekanisme non-blocking menggunakan fungsi millis() pada program di atas!
millis() mengembalikan jumlah milidetik sejak board menyala tanpa menghentikan program. Variabel waktuTerakhirPublish menyimpan waktu publish terakhir. Pada setiap putaran loop(), program menghitung millis() - waktuTerakhirPublish dan membandingkannya dengan intervalPublish (5000 ms). Jika belum tercapai, blok publish dilewati dan loop() langsung berputar lagi (ribuan kali per detik), sehingga client.loop() terus dipanggil untuk memeriksa pesan masuk. Jika sudah lebih dari 5 detik, waktuTerakhirPublish diperbarui, suhu dibaca, lalu data dipublish. Dengan pola ini ESP8266 dapat "menunggu" tanpa berhenti. Pengurangan pada tipe unsigned long juga tetap benar meskipun millis() mengalami overflow setelah ± 49 hari.

---

## 3. Apa yang akan terjadi apabila fungsi client.loop() jarang dipanggil (misalnya hanya sekali setiap 10 detik)?
Pesan perintah baru diproses saat client.loop() dipanggil, sehingga LED bisa terlambat hingga 10 detik setelah perintah dikirim dan terasa tidak real-time. Paket keep-alive juga terkirim tidak teratur; dengan keep-alive 15 detik pada PubSubClient, jarak panggilan 10 detik sudah mendekati batas dan sedikit keterlambatan saja dapat membuat broker memutus koneksi. Setelah terputus, ESP8266 harus reconnect dan pesan perintah yang dikirim saat itu dapat hilang karena QoS 0 tidak menyimpan pesan untuk client yang tidak terhubung. Oleh sebab itu client.loop() harus dipanggil di setiap iterasi loop().

---

## 4. Modifikasi program agar menambahkan satu topic perintah baru untuk mengendalikan aktuator kedua (misalnya buzzer), dengan fungsi callback yang dapat membedakan topic mana yang menerima pesan, dan berikan penjelasan di setiap baris kode yang ditambahkan dalam bentuk README.md!
```cpp
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <DHT.h>
 
const char* ssid = "eduroamer";
const char* password = "wendygantengpoll";
const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;
const char* topicData =
  "unsoed/tk245004/kelompokwewenroland/data";
const char* topicPerintah =
  "unsoed/tk245004/kelompokwewenroland/perintah";
const char* topicBuzzer =                               // [BARU]
  "unsoed/tk245004/kelompokwewenroland/buzzer";         // [BARU]
 
#define DHTPIN D2
#define DHTTYPE DHT11
const int ledPin = D1;
const int buzzerPin = D5;                               // [BARU] buzzer (GPIO14)
 
DHT dht(DHTPIN, DHTTYPE);
WiFiClient espClient;
PubSubClient client(espClient);
 
unsigned long waktuTerakhirPublish = 0;
const long intervalPublish = 5000;
 
void callback(char* topic, byte* payload, unsigned int length) {
  String pesan;
  for (unsigned int i = 0; i < length; i++) pesan += (char)payload[i];
 
  JsonDocument doc;
  if (deserializeJson(doc, pesan)) return;
 
  const char* perintah = doc["perintah"] | "";
  bool aktif = (String(perintah) == "ON");              // [BARU]
 
  if (strcmp(topic, topicPerintah) == 0) {              // [BARU] pesan untuk LED
    digitalWrite(ledPin, aktif ? HIGH : LOW);           // [BARU]
    Serial.print("LED -> ");                            // [BARU]
  } else if (strcmp(topic, topicBuzzer) == 0) {         // [BARU] pesan untuk buzzer
    digitalWrite(buzzerPin, aktif ? HIGH : LOW);        // [BARU]
    Serial.print("Buzzer -> ");                         // [BARU]
  }
  Serial.println(perintah);
}
 
void hubungkanWiFi() {
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  Serial.println("WiFi berhasil terhubung!");
}
 
void hubungkanMQTT() {
  while (!client.connected()) {
    String clientId = "ESP8266Client-" + String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      client.subscribe(topicPerintah);
      client.subscribe(topicBuzzer);                    // [BARU] subscribe topic buzzer
      Serial.println("Terhubung & subscribe topic perintah + buzzer");
    } else {
      delay(2000);
    }
  }
}
 
void setup() {
  Serial.begin(115200);
  pinMode(ledPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);                           // [BARU]
  digitalWrite(buzzerPin, LOW);                         // [BARU] buzzer awal mati
  dht.begin();
  hubungkanWiFi();
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback);
}
 
void loop() {
  if (!client.connected()) hubungkanMQTT();
  client.loop();
 
  if (millis() - waktuTerakhirPublish > intervalPublish) {
    waktuTerakhirPublish = millis();
    float suhu = dht.readTemperature();
    if (!isnan(suhu)) {
      JsonDocument doc;
      doc["suhu"] = suhu;
      char buffer[128];
      serializeJson(doc, buffer);
      client.publish(topicData, buffer);
      Serial.print("Data terkirim: ");
      Serial.println(buffer);
    }
  }
}
```
---

# Pertanyaan Analisis
## 1. Uraikan hasil tugas pada praktikum yang telah dilakukan pada setiap percobaan!
Pada Percobaan 4A, ESP8266 berhasil terhubung ke WiFi dan broker broker.hivemq.com, lalu melakukan subscribe ke topic perintah. Pesan {"perintah": "ON"} dan {"perintah": "OFF"} dari MQTT Explorer diterima, ditampilkan sebagai pesan mentah di Serial Monitor, dideserialisasi, dan menyalakan/mematikan LED. Pesan berformat tidak valid ditolak dengan pesan error tanpa mengganggu program. Pada tugas tambahan, LED dapat diatur kecerahannya melalui nilai intensitas menggunakan PWM. Pada Percobaan 4B, ESP8266 mempublikasikan suhu dari DHT11 ke topic data setiap 5 detik sekaligus menerima perintah LED lewat topic terpisah. LED merespons perintah tanpa terganggu proses publish, koneksi tetap stabil, dan pada tugas tambahan buzzer dapat dikendalikan melalui topic tersendiri.

---

## 2. Bandingkan mekanisme komunikasi satu arah (publish saja, seperti pada Modul Praktikum 3) dengan komunikasi dua arah (publish dan subscribe) yang diimplementasikan pada modul ini!
Pada komunikasi satu arah (Modul 3), ESP8266 hanya menjadi publisher: data dikirim ke broker dan perangkat tidak dapat dikendalikan dari jauh. Program relatif sederhana (serialisasi JSON dan publish()), dan delay() masih dapat dipakai karena tidak ada pesan masuk yang perlu ditunggu. Pada komunikasi dua arah (Modul 4), ESP8266 juga menjadi subscriber sehingga dapat menerima perintah dan menggerakkan aktuator secara real-time. Konsekuensinya, program membutuhkan callback, deserialisasi JSON, subscribe ulang saat reconnect, pemanggilan client.loop() tanpa hambatan, serta pengaturan waktu non-blocking dengan millis(). Sistem dua arah membentuk loop monitoring dan kontrol, sedangkan sistem satu arah hanya monitoring.

---

## 3. Mengapa pendekatan non-blocking (menggunakan millis()) lebih sesuai dibandingkan pendekatan blocking (menggunakan delay()) pada sistem IoT yang memerlukan komunikasi dua arah secara real-time?
Karena pada komunikasi dua arah, ESP8266 harus selalu siap menerima pesan kapan pun sambil tetap mengirim data secara berkala. Pendekatan blocking menghentikan seluruh program selama jeda, sehingga client.loop() tidak terpanggil: perintah terlambat diproses, keep-alive tidak terkirim, dan koneksi dapat terputus. Pendekatan non-blocking dengan millis() hanya memeriksa "apakah sudah waktunya publish?" pada setiap putaran loop() sehingga tugas lain (menerima pesan, memelihara koneksi, membaca tombol atau sensor tambahan) tetap berjalan. Hasilnya respons aktuator cepat, koneksi stabil, dan sistem mudah dikembangkan untuk menambah tugas lain.

---

## 4. Berikan contoh penerapan komunikasi dan pertukaran data dua arah pada sistem IoT nyata (sesuaikan dengan bidang peminatan), dan jelaskan manfaatnya dibandingkan sistem yang hanya satu arah!
Contoh: sistem smart home atau smart greenhouse. Node ESP8266 mempublikasikan suhu, kelembaban udara, dan kelembaban tanah ke broker (arah perangkat ke aplikasi), sementara aplikasi dashboard atau logika otomatis mengirim perintah ke topic kendali untuk menyalakan kipas, pompa penyiram, atau lampu (arah aplikasi ke perangkat). Manfaat dibandingkan sistem satu arah: (1) pengguna tidak hanya memantau tetapi juga dapat bertindak dari jarak jauh, misalnya menyalakan pompa saat tanah kering; (2) kontrol dapat otomatis berbasis data (misalnya kipas aktif jika suhu melebihi ambang) dan dapat dikoreksi manual; (3) respons terhadap kondisi darurat lebih cepat sehingga menghemat waktu, energi, dan air; dan (4) satu infrastruktur MQTT dapat melayani banyak sensor dan aktuator sekaligus. Pada sistem satu arah, pengguna hanya melihat data dan harus datang langsung ke lokasi untuk mengoperasikan perangkat.

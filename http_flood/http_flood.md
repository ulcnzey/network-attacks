# HTTP Flood

## 1. Genel Bakış

HTTP Flood, uygulama katmanında (Layer 7) gerçekleştirilen bir Denial-of-Service (DoS) saldırısıdır. Saldırının temel amacı, hedef web sunucusuna kısa süre içerisinde çok sayıda HTTP isteği göndererek sunucunun kaynaklarını tüketmek ve normal kullanıcı isteklerinin işlenmesini zorlaştırmaktır.

Bu çalışmada HTTP Flood saldırısı, yalnızca izole edilmiş VirtualBox laboratuvar ortamında gerçekleştirilmiştir. Saldırı trafiği Wireshark kullanılarak yakalanmış ve PCAPNG formatında kaydedilmiştir.

Bu çalışma CyberTrace projesinde kullanılmak üzere saldırı veri setinin oluşturulması amacıyla gerçekleştirilmiştir.

---

## 2. HTTP Flood Nasıl Çalışır?

HTTP Flood saldırısında saldırgan sistem, hedef web sunucusuna çok sayıda HTTP request gönderir.

Laboratuvar ortamındaki trafik akışı şu şekildedir:

```text
Windows Attacker
192.168.56.1
      |
      |  HTTP GET Requests
      |  TCP/8000
      v
Kali Linux Web Server
192.168.56.101
```

Saldırı sırasında aynı kaynak IP adresinden aynı hedef IP adresine ve aynı HTTP servisine çok sayıda istek gönderilmiştir.

Örneğin:

```http
GET / HTTP/1.1
Host: 192.168.56.101:8000
```

Bu isteğin çok kısa zaman aralıklarında tekrar tekrar gönderilmesi HTTP Flood davranışını oluşturur.

---

## 3. TCP Connection Flood ile Farkı

HTTP Flood ile TCP Connection Flood birbirinden farklı saldırı davranışlarına sahiptir.

### TCP Connection Flood

Temel olarak çok sayıda TCP bağlantısı oluşturmayı hedefler.

```text
Attacker
   |
   |--- TCP Connection ---> Server
   |--- TCP Connection ---> Server
   |--- TCP Connection ---> Server
   |--- TCP Connection ---> Server
```

### HTTP Flood

TCP bağlantısı üzerinden uygulama katmanında çok sayıda HTTP request gönderilir.

```text
Attacker
   |
   |--- GET / ----------> Web Server
   |--- GET / ----------> Web Server
   |--- GET / ----------> Web Server
   |--- GET / ----------> Web Server
```

Bu nedenle HTTP Flood tespitinde yalnızca TCP bağlantı sayısına değil, HTTP request rate gibi uygulama katmanı özelliklerine de bakılması gerekir.

---

## 4. Laboratuvar Ortamı

| Bileşen        | Bilgi                |
| -------------- | -------------------- |
| Saldırgan      | Windows Host         |
| Saldırgan IP   | `192.168.56.1`       |
| Hedef          | Kali Linux           |
| Hedef IP       | `192.168.56.101`     |
| Hedef Port     | `8000`               |
| Protokol       | HTTP / TCP           |
| Web Server     | Python HTTP Server   |
| Ağ             | VirtualBox Host-Only |
| Trafik Analizi | Wireshark            |
| Kayıt Formatı  | PCAPNG               |

---

## 5. HTTP Test Sunucusunun Oluşturulması

Kali Linux üzerinde HTTP Flood trafiğinin oluşturulabilmesi için Python'ın yerleşik HTTP server modülü kullanılmıştır.

Çalışma klasörü oluşturulmuştur:

```bash
cd ~/Desktop
mkdir -p http-flood-lab
cd http-flood-lab
```

Test amacıyla basit bir HTML sayfası oluşturulmuştur:

```bash
echo "<html><body><h1>CyberTrace HTTP Flood Lab</h1></body></html>" > index.html
```

Ardından HTTP server başlatılmıştır:

```bash
python3 -m http.server 8000 --bind 192.168.56.101
```

Server aşağıdaki adres üzerinden çalıştırılmıştır:

```text
http://192.168.56.101:8000/
```

---

## 6. Normal HTTP Trafiğinin Test Edilmesi

Saldırı gerçekleştirilmeden önce hedef web sunucusunun erişilebilir olduğu kontrol edilmiştir.

Windows PowerShell üzerinden:

```powershell
Test-NetConnection 192.168.56.101 -Port 8000
```

Port erişilebilirliği başarılı olduktan sonra HTTP request gönderilmiştir:

```powershell
curl.exe http://192.168.56.101:8000/
```

Normal HTTP request sonucunda Python HTTP server üzerinde aşağıdakine benzer bir kayıt oluşmuştur:

```text
192.168.56.1 - - [date] "GET / HTTP/1.1" 200 -
```

Bu aşama saldırı trafiğinden önce normal HTTP davranışının gözlemlenmesi açısından önemlidir.

---

## 7. Wireshark ile Trafik Yakalama

HTTP Flood trafiğini yakalamak için Kali Linux üzerinde Wireshark kullanılmıştır.

Yakalama yapılan ağ arayüzü:

```text
eth1
```

IP adresi:

```text
192.168.56.101
```

Capture sırasında herhangi bir display filter kullanılmamıştır.

Bunun amacı saldırı trafiğinin ham haliyle PCAPNG dosyasına kaydedilmesidir.

> **Not:** Wireshark display filter'ları trafik kaydedildikten sonra analiz amacıyla kullanılmıştır.

---

## 8. HTTP Flood Saldırısının Gerçekleştirilmesi

Kontrollü HTTP Flood simülasyonu Windows sisteminden gerçekleştirilmiştir.

500 adet HTTP GET request gönderilmiştir:

```powershell
1..500 | ForEach-Object {
    curl.exe -s http://192.168.56.101:8000/ > $null
}
```

Bu komut sonucunda hedef web sunucusuna kısa süre içerisinde çok sayıda HTTP request gönderilmiştir.

Her request aynı hedefe yönelmiştir:

```text
Source:      192.168.56.1
Destination: 192.168.56.101
Protocol:    HTTP
Port:        8000
Method:      GET
Path:        /
```

Bu çalışma yalnızca kontrollü ve izole edilmiş laboratuvar ortamında gerçekleştirilmiştir.

---

## 9. Wireshark Analizi

PCAP içerisinde HTTP Flood trafiğinin incelenmesi için aşağıdaki display filter'lar kullanılmıştır.

### HTTP trafiği

```text
http
```

### HTTP request'leri

```text
http.request
```

### GET request'leri

```text
http.request.method == "GET"
```

### Hedef HTTP servisi

```text
tcp.port == 8000
```

### Saldırgan → Hedef trafiği

```text
ip.src == 192.168.56.1 &&
ip.dst == 192.168.56.101 &&
tcp.port == 8000
```

---

## 10. Gözlemlenen Trafik Davranışı

Wireshark analizi sırasında aynı kaynak sistemden hedef web sunucusuna çok sayıda HTTP GET request gönderildiği gözlemlenmiştir.

Temel trafik özellikleri:

* Kaynak IP adresi sabittir.
* Hedef IP adresi sabittir.
* Hedef port `8000` olarak sabittir.
* HTTP GET request sayısı kısa süre içerisinde artmaktadır.
* Aynı `/` endpoint'ine tekrar tekrar istek gönderilmektedir.
* HTTP request rate normal kullanıcı trafiğine göre belirgin şekilde yüksektir.

Örnek trafik modeli:

```text
192.168.56.1
      |
      | GET /
      | GET /
      | GET /
      | GET /
      | GET /
      | GET /
      v
192.168.56.101:8000
```

Bu davranış HTTP Flood saldırısının temel karakteristiğini oluşturmaktadır.

---

## 11. Saldırının Ağ Üzerindeki Etkisi

HTTP Flood saldırılarında çok sayıda uygulama katmanı request'i hedef sunucu tarafından işlenmek zorunda kalır.

Saldırı yoğunluğu arttığında aşağıdaki durumlar ortaya çıkabilir:

* Web server CPU kullanımının artması
* Web server kaynaklarının tüketilmesi
* HTTP request queue'sunun büyümesi
* Normal kullanıcı request'lerinin gecikmesi
* Sunucu yanıt süresinin artması
* Servisin erişilebilirliğinin azalması

Bu laboratuvar çalışmasında saldırının temel amacı gerçek bir sistemi devre dışı bırakmak değil, saldırı davranışının ağ seviyesinde gözlemlenmesi ve PCAP verisinin oluşturulmasıdır.

---

## 12. HTTP Flood Tespitinde Kullanılabilecek Özellikler

CyberTrace projesinde HTTP Flood saldırısının makine öğrenmesi tabanlı olarak tespit edilebilmesi için aşağıdaki özelliklerden yararlanılabilir.

| Feature              | Açıklama                                  |
| -------------------- | ----------------------------------------- |
| Source IP            | İsteği gönderen sistem                    |
| Destination IP       | Hedef sistem                              |
| Destination Port     | HTTP servis portu                         |
| Protocol             | TCP / HTTP                                |
| HTTP Method          | GET, POST vb.                             |
| Request Count        | Belirli zaman aralığındaki request sayısı |
| Request Rate         | Saniyedeki HTTP request sayısı            |
| Packet Count         | Toplam paket sayısı                       |
| Byte Count           | Toplam veri miktarı                       |
| Flow Duration        | Akış süresi                               |
| Average Packet Size  | Ortalama paket boyutu                     |
| TCP Connection Count | TCP bağlantı sayısı                       |
| HTTP Response Rate   | Sunucunun response oranı                  |

Özellikle `Request Count`, `Request Rate` ve `Flow Duration` gibi zaman tabanlı özellikler HTTP Flood davranışının belirlenmesinde önemlidir.

---

## 13. Saldırı İçin Veri Seti Etiketi

CyberTrace veri setinde bu trafik aşağıdaki etiket ile sınıflandırılabilir:

```text
HTTP_Flood
```

Örnek veri mantığı:

```text
Source IP: 192.168.56.1
Destination IP: 192.168.56.101
Destination Port: 8000
Protocol: HTTP
Label: HTTP_Flood
```

Bu etiket daha sonraki veri ön işleme ve makine öğrenmesi aşamalarında kullanılabilir.

---

## 14. PCAP Dosyası

Oluşturulan saldırı trafiği PCAPNG formatında kaydedilmiştir.

Dosya:

```text
http_flood.pcapng
```

Dosyanın doğrulanması için:

```bash
ls -lh ~/Desktop/http-flood-lab/http_flood.pcapng
```

PCAP hakkında genel bilgi almak için:

```bash
capinfos ~/Desktop/http-flood-lab/http_flood.pcapng
```

### PCAP Bilgileri

> Bu bölüm gerçek `capinfos` çıktısına göre doldurulacaktır.

```text
File:
Capture duration:
Packet count:
Data size:
First packet:
Last packet:
```

---

## 15. Ekran Görüntüleri

### 15.1 HTTP Server

Python HTTP server'ın çalıştığını gösteren ekran görüntüsü.



### 15.2 Normal HTTP Request

Normal HTTP GET request'in Wireshark üzerinde görüntülenmesi.


### 15.3 HTTP Flood Traffic

Çok sayıda HTTP request'in Wireshark üzerinde görüntülenmesi.

<img width="1365" height="627" alt="image" src="https://github.com/user-attachments/assets/e0e9cfd4-b7b2-400d-9b41-6018a80ac2b2" />


### 15.4 HTTP GET Filter

Aşağıdaki filtre kullanılarak elde edilen trafik:

```text
http.request.method == "GET"
```



### 15.5 PCAP Statistics

`
---

## 16. Laboratuvar Sonucu

Bu çalışmada kontrollü bir HTTP Flood saldırısı oluşturulmuş ve saldırı trafiği Wireshark kullanılarak analiz edilmiştir.

Saldırı sırasında kısa süre içerisinde çok sayıda HTTP GET request gönderildiği gözlemlenmiştir. Trafikte kaynak ve hedef IP adreslerinin sabit olması, aynı hedef portunun kullanılması ve yüksek HTTP request rate oluşması saldırının belirgin özellikleri olarak değerlendirilmiştir.

Oluşturulan `http_flood.pcapng` dosyası CyberTrace projesinin saldırı veri setinde kullanılmak üzere saklanmıştır.

Bu çalışma ile HTTP Flood saldırısının ağ üzerindeki davranışı incelenmiş ve ilerleyen aşamalarda makine öğrenmesi modeli tarafından kullanılabilecek trafik özellikleri belirlenmiştir.

---

## 17. Klasör Yapısı

GitHub repository içerisinde saldırı aşağıdaki yapıda saklanacaktır:

```text
network-attacks/
└── http-flood/
    ├── http-flood.md
    ├── http_flood.pcapng
    └── screenshots/
        ├── http-server.png
        ├── normal-http.png
        ├── http-flood.png
        ├── http-get-filter.png
        └── pcap-statistics.png
```

---

## 18. Sonraki Aşama

HTTP Flood çalışmasının tamamlanmasının ardından CyberTrace saldırı veri setine bir sonraki saldırı türü eklenebilir.

Planlanan saldırı sırasına göre sonraki çalışma:

```text
11. Slow HTTP Attack
```

Slow HTTP Attack, HTTP Flood'dan farklı olarak çok sayıda request göndermek yerine HTTP bağlantılarını uzun süre açık tutarak sunucu kaynaklarını tüketmeye odaklanır.

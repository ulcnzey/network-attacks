# Network Service Scanning

## 1. Çalışmanın Amacı

Bu çalışmanın amacı, bir hedef sistemde açık olan bir ağ portunun arkasında hangi servis veya uygulamanın çalıştığını tespit etmektir.

Port Scanning çalışmasında temel olarak:

> "Hangi portlar açık?"

sorusuna cevap aranırken, Network Service Scanning çalışmasında:

> "Açık olan portta hangi servis çalışıyor ve mümkünse hangi sürüm kullanılıyor?"

sorusuna cevap aranır.

Bu çalışma kapsamında Nmap kullanılarak Kali Linux üzerindeki TCP/8000 portu incelenmiş, çalışan HTTP servisinin tespit edilmesi ve Wireshark üzerinden gerçekleştirilen iletişimin paket seviyesinde analiz edilmesi amaçlanmıştır.

---

## 2. Network Service Scanning Nedir?

Network Service Scanning, bir ağ üzerindeki açık portlarda çalışan servislerin belirlenmesine yönelik gerçekleştirilen bilgi toplama işlemidir.

Bir portun açık olması tek başına yeterli bilgi sağlamaz.

Örneğin:

```text
8000/tcp open
```

sonucunu elde ettiğimizde yalnızca 8000 numaralı portun bağlantı kabul ettiğini biliriz.

Service scanning ile bunun yerine:

```text
8000/tcp open http
```

ve daha ayrıntılı olarak:

```text
8000/tcp open http SimpleHTTPServer 0.6 (Python 3.13.7)
```

gibi bilgiler elde edilebilir.

Bu nedenle Network Service Scanning, hedef sistem hakkında daha ayrıntılı bilgi toplamak amacıyla kullanılan bir reconnaissance tekniğidir.

---

## 3. Laboratuvar Ortamı

Çalışma izole edilmiş VirtualBox laboratuvar ortamında gerçekleştirilmiştir.

### Ağ Yapısı

| Sistem     | IP Adresi      | Rol                  |
| ---------- | -------------- | -------------------- |
| Windows    | 192.168.56.1   | Tarama yapan istemci |
| Kali Linux | 192.168.56.101 | Hedef sistem         |

Ağ bağlantısı VirtualBox Host-Only Network üzerinden gerçekleştirilmiştir.

Kali Linux üzerinde HTTP servisi TCP/8000 portunda çalıştırılmıştır.

---

## 4. Hedef Sistem Üzerindeki Servisin Hazırlanması

Service scanning işleminin anlamlı şekilde gerçekleştirilebilmesi için hedef sistem üzerinde erişilebilir bir HTTP servisi oluşturulmuştur.

Kali Linux üzerinde çalışma dizini oluşturulmuştur:

```bash
cd ~/Desktop
mkdir -p service-scan-lab
cd service-scan-lab
```

Test amacıyla basit bir HTML dosyası oluşturulmuştur:

```bash
echo "NetServiceScan Lab" > index.html
```

Daha sonra Python'un yerleşik HTTP sunucusu 8000 numaralı port üzerinde çalıştırılmıştır:

```bash
python3 -m http.server 8000 --bind 0.0.0.0
```

Bu komut sonucunda:

```text
Serving HTTP on 0.0.0.0 port 8000 ...
```

benzeri bir çıktı elde edilmiştir.

---

## 5. Servisin Dinlediğinin Kontrol Edilmesi

Servisin gerçekten TCP/8000 portunda dinleme yaptığını kontrol etmek için aşağıdaki komut kullanılmıştır:

```bash
sudo ss -tulpn
```

Çıktıda aşağıdaki satır görülmüştür:

```text
tcp    LISTEN    0    5    0.0.0.0:8000    0.0.0.0:*    users:(("python3",pid=42137,fd=3))
```

Bu çıktı şu şekilde yorumlanabilir:

* `tcp`: TCP protokolü kullanılmaktadır.
* `LISTEN`: Sistem bağlantı beklemektedir.
* `0.0.0.0:8000`: 8000 numaralı TCP portu tüm ağ arayüzlerinden bağlantı kabul etmektedir.
* `python3`: Servis Python tarafından çalıştırılmaktadır.


---

## 6. Normal HTTP Trafiğinin Kontrol Edilmesi

Tarama işleminden önce Windows üzerinden aşağıdaki adres ziyaret edilmiştir:

```text
http://192.168.56.101:8000
```

Bu işlem ile Kali üzerindeki HTTP servisine normal bir istemci bağlantısı gerçekleştirilmiştir.

Wireshark üzerinde TCP/8000 trafiği incelenmiştir.

Normal TCP bağlantısının temel akışı:

```text
Windows → Kali     SYN
Kali → Windows     SYN, ACK
Windows → Kali     ACK
```

şeklindedir.

TCP bağlantısı kurulduktan sonra HTTP iletişimi başlamıştır.

---

## 7. HTTP GET İsteğinin İncelenmesi

Wireshark üzerinde gerçekleştirilen incelemede aşağıdaki HTTP paketi görülmüştür:

```text
192.168.56.1 → 192.168.56.101
HTTP
GET / HTTP/1.1
```

Bu paket Windows tarafının Kali üzerindeki HTTP sunucusundan `/` kaynağını istediğini göstermektedir.

Buradaki:

```text
GET /
```

ifadesi HTTP istemcisinin sunucudan kök kaynağı istediğini belirtmektedir.


---

## 8. HTTP Response Paketinin İncelenmesi

HTTP isteğine karşılık Kali tarafından Windows'a aşağıdaki cevap gönderilmiştir:

```text
192.168.56.101 → 192.168.56.1
HTTP
HTTP/1.0 304 Not Modified
```

`304 Not Modified` HTTP durum kodu, istemcinin sahip olduğu kaynağın güncel olduğunu ve sunucunun kaynağı tekrar göndermesine gerek olmadığını belirtir.

Bu paket, hedef sistemde HTTP protokolünün aktif olarak çalıştığını gösteren önemli bir gözlemdir.



---

## 9. Nmap ile Network Service Scanning

Servis tespiti gerçekleştirmek için Windows üzerinde Nmap kullanılmıştır.

Kullanılan komut:

```powershell
nmap -sV -p 8000 192.168.56.101
```

Buradaki:

```text
-sV
```

parametresi Nmap'in servis ve sürüm tespiti yapmasını sağlar.

```text
-p 8000
```

parametresi ise yalnızca TCP/8000 portunun taranmasını sağlar.

---

## 10. Nmap Tarama Sonucu

Nmap tarafından aşağıdaki sonuç elde edilmiştir:

```text
PORT     STATE SERVICE VERSION
8000/tcp open  http    SimpleHTTPServer 0.6 (Python 3.13.7)
```

Bu sonuç şu şekilde yorumlanabilir:

```text
8000/tcp
```

TCP protokolünün 8000 numaralı portudur.

```text
open
```

Portun bağlantı kabul ettiğini göstermektedir.

```text
http
```

Port üzerinde HTTP servisinin tespit edildiğini göstermektedir.

```text
SimpleHTTPServer 0.6
```

Servisin türünü belirtmektedir.

```text
Python 3.13.7
```

Servisin çalıştığı Python sürümünü göstermektedir.

![Nmap Service Scanning sonucu](screenshots/nmap-service-scan.png)

---

## 11. Nmap Service Detection Mantığının İncelenmesi

Nmap'in service detection işlemi yalnızca portun açık olup olmadığını kontrol etmekle sınırlı değildir.

Çalışma sırasında aşağıdaki iletişim gözlemlenmiştir:

```text
1. TCP SYN
2. TCP SYN/ACK
3. TCP ACK
4. HTTP GET isteği
5. HTTP Response
```

Wireshark üzerinde gözlemlenen HTTP isteği:

```text
GET / HTTP/1.1
```

şeklindedir.

Buna karşılık hedef sistem:

```text
HTTP/1.0 304 Not Modified
```

cevabını göndermiştir.

Bu iletişim, hedef port üzerinde HTTP protokolünün aktif olduğunu göstermektedir.

Dolayısıyla Nmap'in:

```text
8000/tcp open http
```

sonucunu üretmesinin arkasındaki temel mantık, hedef servisin protokol davranışının incelenmesidir.

---

## 12. TCP Bağlantısının İncelenmesi

Network Service Scanning sırasında TCP bağlantısının kurulması Wireshark üzerinde incelenmiştir.

Temel bağlantı:

```text
Windows → Kali
SYN
```

```text
Kali → Windows
SYN, ACK
```

```text
Windows → Kali
ACK
```

şeklinde gerçekleşmiştir.

Sonrasında uygulama katmanında HTTP iletişimi başlamıştır.



---

## 13. Port Scanning ile Network Service Scanning Arasındaki Fark

Port Scanning ile Network Service Scanning birbirine yakın olsa da farklı amaçlara sahiptir.

### Port Scanning

Temel soru:

> Hangi portlar açık?

Örneğin:

```text
8000/tcp open
```

### Network Service Scanning

Temel soru:

> Açık portta hangi servis çalışıyor?

Örneğin:

```text
8000/tcp open http
```

Daha ayrıntılı sonuç:

```text
8000/tcp open http SimpleHTTPServer 0.6 (Python 3.13.7)
```

Bu nedenle service scanning, port scanning sonrasında hedef hakkında daha fazla bilgi elde edilmesini sağlar.

---

## 14. Wireshark Analizi

Network Service Scanning sırasında Wireshark üzerinde TCP/8000 trafiği filtrelenerek incelenmiştir.

Kullanılan temel filtre:

```text
tcp.port == 8000
```

Belirli iki sistem arasındaki trafik için:

```text
ip.addr == 192.168.56.1 && ip.addr == 192.168.56.101 && tcp.port == 8000
```

filtresi kullanılabilir.

İncelenen trafik içerisinde:

* TCP bağlantı kurulumu
* HTTP GET isteği
* HTTP Response

paketleri gözlemlenmiştir.

---

## 15. Network Service Scanning Trafiğinin Özellikleri

Bu çalışma sonucunda service scanning davranışını belirlemede kullanılabilecek bazı özellikler belirlenmiştir.

### IP Bilgileri

```text
source_ip
destination_ip
```

### Port Bilgileri

```text
destination_port
source_port
```

### Protokol Bilgileri

```text
TCP
HTTP
```

### Servis Bilgileri

```text
service
service_version
```

### Trafik Davranışı

```text
connection_count
unique_destination_ports
scan_duration
```

Bu özellikler ileride CyberTrace içerisinde servis tarama davranışlarının analiz edilmesinde kullanılabilir.

---

## 16. CyberTrace Açısından Değerlendirme

CyberTrace yalnızca tek bir pakete bakarak Network Service Scanning tespiti yapmamalıdır.

Bunun yerine trafik davranışının bir bütün olarak değerlendirilmesi daha doğru olacaktır.

Örneğin:

```text
source_ip
      ↓
destination_ip
      ↓
destination_port
      ↓
TCP connection
      ↓
application protocol
      ↓
service response
```

şeklinde bir analiz gerçekleştirilebilir.

Service scanning tespitinde aşağıdaki bilgiler değerlendirilebilir:

* Aynı kaynak IP'nin birden fazla servise erişmesi
* Kısa zaman içerisinde farklı portlara bağlantı kurulması
* TCP bağlantılarının ardından servis/protokol sorgularının yapılması
* HTTP, SSH, FTP gibi protokollere yönelik probe davranışları
* Servis ve sürüm tespiti amacıyla gönderilen istekler

---

## 17. IOC ve Davranışsal Göstergeler

Network Service Scanning sırasında tek başına bir IP adresini IOC olarak değerlendirmek doğru olmayabilir.

Sistem yöneticileri veya güvenlik araçları da servis taraması gerçekleştirebilir.

Bu nedenle davranışsal göstergeler daha değerlidir.

Örneğin:

```text
Aynı kaynak IP
        ↓
Birden fazla hedef port
        ↓
Kısa zaman aralığı
        ↓
TCP bağlantıları
        ↓
Servis/protokol sorguları
```

şeklindeki davranışlar birlikte değerlendirilebilir.

---

## 18. Güvenlik Açısından Önemi

Network Service Scanning, saldırganların hedef sistem hakkında bilgi toplamak için kullanabileceği önemli bir reconnaissance tekniğidir.

Bir saldırgan:

1. Açık portları belirleyebilir.
2. Çalışan servisleri belirleyebilir.
3. Servis sürümlerini öğrenebilir.
4. Eski veya savunmasız servisleri araştırabilir.
5. Daha sonraki saldırılar için hedef belirleyebilir.

Bu nedenle service scanning, saldırı zincirinin erken aşamalarında görülebilecek davranışlardan biridir.

---

## 19. Savunma Yaklaşımı

Network Service Scanning'e karşı aşağıdaki önlemler uygulanabilir:

* Kullanılmayan servislerin kapatılması
* Gereksiz portların erişime kapatılması
* Güvenlik duvarı kurallarının uygulanması
* Servislerin güncel tutulması
* Ağ segmentasyonu
* IDS/IPS sistemlerinin kullanılması
* Anormal tarama davranışlarının izlenmesi
* Kaynak IP ve bağlantı davranışlarının loglanması

Özellikle kısa zaman içerisinde çok sayıda servis veya porta erişim gerçekleştiren sistemlerin izlenmesi faydalıdır.

---

## 20. Kullanılan Araçlar

Bu çalışma kapsamında:

* Kali Linux
* Windows
* VirtualBox
* Nmap
* Wireshark
* Python HTTP Server

kullanılmıştır.

---

## 21. Üretilen Veri

Çalışma sonucunda Network Service Scanning davranışını içeren aşağıdaki PCAP dosyası oluşturulmuştur:

```text
service_scan.pcapng
```

Bu dosya daha sonra CyberTrace veri setinin oluşturulması sırasında kullanılabilecek örneklerden biridir.

PCAP içerisinde:

* TCP bağlantı kurulumu
* HTTP GET isteği
* HTTP Response
* Servis iletişimi

gibi trafikler bulunmaktadır.

---

## 22. Sonuç

Bu çalışmada Network Service Scanning kavramı uygulamalı olarak incelenmiştir.

Öncelikle Kali Linux üzerinde TCP/8000 portunda Python tabanlı bir HTTP servisi çalıştırılmıştır. Daha sonra Windows sistemi üzerinden Nmap kullanılarak:

```bash
nmap -sV -p 8000 192.168.56.101
```

komutu ile servis ve sürüm tespiti gerçekleştirilmiştir.

Nmap sonucunda:

```text
8000/tcp open http SimpleHTTPServer 0.6 (Python 3.13.7)
```

sonucu elde edilmiştir.

Wireshark üzerinde ise TCP bağlantısının kurulması ve HTTP iletişimi incelenmiştir. Özellikle:

```text
GET / HTTP/1.1
```

isteği ve:

```text
HTTP/1.0 304 Not Modified
```

cevabı gözlemlenmiştir.

Bu çalışma sonucunda yalnızca bir portun açık olduğunu belirlemek yerine, açık port üzerinde çalışan servisin ve servis davranışının nasıl tespit edilebildiği anlaşılmıştır.

Elde edilen `service_scan.pcapng` dosyası, ilerleyen aşamalarda CyberTrace için oluşturulacak ağ saldırısı/veri seti çalışmalarında kullanılacaktır.

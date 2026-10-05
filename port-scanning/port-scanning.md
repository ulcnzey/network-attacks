# Port Scanning

## 1. Çalışmanın Amacı

Bu çalışmanın amacı, bir sistemde bulunan TCP portlarının nasıl tarandığını, portların açık veya kapalı olduğunun nasıl anlaşılabildiğini ve port tarama işleminin ağ trafiğinde nasıl göründüğünü öğrenmektir.

Çalışma kapsamında Windows sistemi üzerinden Kali Linux sistemine port taraması gerçekleştirilmiş ve oluşan ağ trafiği Wireshark ile incelenmiştir.

Elde edilen bilgiler ilerleyen aşamalarda **CyberTrace** projesinde kullanılmak üzere oluşturulacak ağ saldırı veri setine dahil edilecektir.

> **Not:** Tüm çalışmalar yalnızca tarafıma ait ve izole edilmiş laboratuvar ortamında gerçekleştirilmiştir.

---

## 2. Port Nedir?

Bilgisayarlar ağ üzerinden farklı servislerle iletişim kurarken port numaralarını kullanır.

IP adresini bir binanın adresi olarak düşünürsek, portları bu binadaki farklı kapılar gibi düşünebiliriz.

Bazı yaygın portlar:

| Port | Servis |
| ---: | ------ |
|   21 | FTP    |
|   22 | SSH    |
|   23 | Telnet |
|   25 | SMTP   |
|   53 | DNS    |
|   80 | HTTP   |
|  443 | HTTPS  |
|  445 | SMB    |

Bir sistemde hangi portların açık olduğu, sistem üzerinde hangi servislerin çalıştığı hakkında bilgi verebilir.

---

## 3. Port Scanning Nedir?

Port scanning, bir hedef sistemde hangi portların açık, kapalı veya filtrelenmiş olduğunu belirlemek amacıyla yapılan tarama işlemidir.

Port taraması sırasında istemci, hedef sistemdeki farklı portlara bağlantı istekleri gönderir.

TCP bağlantılarında özellikle **SYN** paketi önemlidir.

Genel TCP davranışı şu şekilde düşünülebilir:

```text
İstemci → Hedef
SYN

Hedef → İstemci
SYN/ACK  → Port açık olabilir
RST/ACK  → Port kapalı
Yanıt yok → Filtrelenmiş olabilir
```

Bu nedenle port taraması sırasında yalnızca port numaralarına değil, hedef sistemin verdiği TCP yanıtlarına da bakılır.

---

## 4. Laboratuvar Ortamı

Çalışmada Windows sistemi tarama yapan sistem, Kali Linux ise hedef sistem olarak kullanılmıştır.

| Sistem     | Rol                 | IP Adresi        |
| ---------- | ------------------- | ---------------- |
| Windows    | Tarama yapan sistem | `192.168.56.1`   |
| Kali Linux | Hedef sistem        | `192.168.56.101` |

İki sistem VirtualBox Host-Only Network üzerinden aynı izole ağda bulunmaktadır.

Wireshark, Kali Linux üzerinde `eth1` arayüzünde çalıştırılmıştır.

### Ağ Yapısı

```text
Windows
192.168.56.1
     |
     |  Port Scanning
     ↓
Kali Linux
192.168.56.101
```

---

## 5. Normal Trafiğin İncelenmesi

Port taramasına başlamadan önce iki sistem arasındaki normal ağ iletişimi incelenmiştir.

Windows tarafından Kali Linux'a ping gönderildiğinde:

```text
192.168.56.1 → 192.168.56.101
ICMP Echo Request
```

paketi gönderilmiştir.

Kali Linux ise:

```text
192.168.56.101 → 192.168.56.1
ICMP Echo Reply
```

ile cevap vermiştir.

Bu işlem saldırı trafiğinden önce normal ağ iletişiminin nasıl göründüğünü anlamak için gerçekleştirilmiştir.


---

# 6. İlk Port Tarama Deneyi

İlk olarak yaygın olarak kullanılan bazı TCP portları taranmıştır.

Kullanılan Nmap komutu:

```bash
nmap -p 21,22,23,25,53,80,110,139,443,445 192.168.56.101
```

Taranan portlar:

```text
21    FTP
22    SSH
23    Telnet
25    SMTP
53    DNS
80    HTTP
110   POP3
139   NetBIOS
443   HTTPS
445   Microsoft-DS
```

Tarama sonucunda bu portların tamamı **closed** olarak tespit edilmiştir.

Örnek:

```text
21/tcp    closed    ftp
22/tcp    closed    ssh
23/tcp    closed    telnet
25/tcp    closed    smtp
...
443/tcp   closed    https
445/tcp   closed    microsoft-ds
```

### Ekran Görüntüsü

<img width="587" height="273" alt="image" src="https://github.com/user-attachments/assets/3db649d0-a93f-4fd4-ae9e-c4b1e87224f6" />


---

# 7. TCP SYN Paketi

Tarama sırasında Wireshark üzerinde port 21 için gönderilen TCP SYN paketi incelenmiştir.

Paket:

```text
192.168.56.1 → 192.168.56.101
TCP
57492 → 21
[SYN]
```

Burada:

* `192.168.56.1` → Windows sisteminin IP adresidir.
* `192.168.56.101` → Kali Linux hedef sisteminin IP adresidir.
* `57492` → Windows tarafından kullanılan geçici kaynak portudur.
* `21` → hedef porttur.
* `SYN` → TCP bağlantısının başlatılmak istendiğini gösterir.

Yani Windows sistemi temelde hedef sisteme:

> "21 numaralı TCP portunda bir servis var mı?"

şeklinde bir bağlantı isteği göndermektedir.

### Ekran Görüntüsü

<img width="778" height="614" alt="image" src="https://github.com/user-attachments/assets/6f0caf08-abed-4020-b54e-5560f7243231" />


---

# 8. RST/ACK Paketi ve Kapalı Port

Port 21 için gönderilen SYN paketinden sonra Kali Linux tarafından:

```text
192.168.56.101 → 192.168.56.1
TCP
21 → 57492
[RST, ACK]
```

paketi gönderilmiştir.

Buradaki `RST` (Reset), TCP bağlantısının sıfırlandığını veya reddedildiğini gösterir.

Bu çalışmada hedef sistemde 21 numaralı portta çalışan bir servis olmadığı için port kapalı olarak değerlendirilmiştir.

Basitleştirilmiş iletişim:

```text
Windows                         Kali
   |                              |
   |-------- SYN :21 ------------>|
   |                              |
   |<------- RST/ACK -------------|
   |                              |
   Port kapalı
```

Bu nedenle TCP SYN isteğine verilen `RST/ACK` yanıtı, bu deneyde kapalı portun önemli göstergesi olmuştur.

### Ekran Görüntüsü

<img width="782" height="615" alt="image" src="https://github.com/user-attachments/assets/7a29803f-be08-4fde-bb80-98aaaccb0de9" />


---

# 9. Açık ve Kapalı Portların Karşılaştırılması

TCP bağlantısının başlangıcında portun durumunu anlamak için paketlerin davranışı incelenebilir.

### Açık port

Genel TCP bağlantı başlangıcı:

```text
İstemci                    Hedef
   |                         |
   |-------- SYN ----------->|
   |<------- SYN/ACK --------|
   |-------- ACK ----------->|
```

Hedef portta bir servis dinliyorsa `SYN/ACK` cevabı alınabilir.

### Kapalı port

```text
İstemci                    Hedef
   |                         |
   |-------- SYN ----------->|
   |<------- RST/ACK --------|
```

Bu çalışmada kapalı portlarda ikinci davranış gözlemlenmiştir.

> **Not:** `RST` her TCP senaryosunda otomatik olarak "port kapalı" anlamına gelmez. Bu çalışmada klasik TCP SYN taramasına verilen `RST/ACK` yanıtı kapalı port göstergesi olarak değerlendirilmiştir.

---

# 10. 1-1000 Port Tarama Deneyi

Daha geniş bir tarama gerçekleştirmek amacıyla hedef sistemin ilk 1000 TCP portu taranmıştır.

Kullanılan komut:

```bash
nmap -p 1-1000 192.168.56.101
```

Nmap sonucunda:

```text
All 1000 scanned ports on 192.168.56.101 are in ignored states.
Not shown: 1000 closed tcp ports (reset)
```

sonucu alınmıştır.

Bu sonuç, taranan 1000 TCP portunun tamamının kapalı olduğunu göstermektedir.

Tarama yaklaşık:

```text
0.54 saniye
```

içerisinde tamamlanmıştır.

Yaklaşık tarama hızı:

```text
1000 / 0.54 ≈ 1852 port/saniye
```

olarak hesaplanabilir.

Bu değer doğrudan bir saldırı eşiği olarak kullanılmamalıdır. Ancak çok sayıda portun kısa sürede taranmasının tespit açısından önemli bir davranışsal özellik olduğunu göstermektedir.

### Ekran Görüntüsü

<img width="770" height="617" alt="image" src="https://github.com/user-attachments/assets/01eaa14a-0476-4aeb-9f3b-08efe36bd68e" />


---

# 11. Wireshark ile SYN Paketlerinin İncelenmesi

1000 portluk tarama sırasında Windows sisteminden Kali Linux'a çok sayıda TCP SYN paketi gönderilmiştir.

Bu paketleri incelemek için aşağıdaki Wireshark filtresi kullanılmıştır:

```text
tcp.flags.syn == 1 && ip.src == 192.168.56.1
```

Bu filtre ile Windows tarafından gönderilen TCP SYN paketleri görüntülenmiştir.

İnceleme sırasında özellikle aşağıdaki alanlara dikkat edilmiştir:

* Kaynak IP adresi
* Hedef IP adresi
* Kaynak port
* Hedef port
* TCP SYN flag
* Paket zamanı

Port taramasında önemli noktalardan biri, kısa bir zaman aralığında çok sayıda farklı hedef portuna bağlantı isteği gönderilmesidir.



---

# 12. RST Paketlerinin İncelenmesi

Hedef sistem tarafından gönderilen TCP Reset paketlerini incelemek için aşağıdaki filtre kullanılabilir:

```text
tcp.flags.reset == 1 && ip.src == 192.168.56.101
```

Bu filtre Kali Linux tarafından gönderilen RST paketlerini göstermektedir.

Genel trafik davranışı:

```text
Windows
192.168.56.1
     |
     | SYN → Port 1
     | SYN → Port 2
     | SYN → Port 3
     | SYN → Port 4
     | ...
     |
     ↓
Kali Linux
192.168.56.101

Kali:
RST → Port 1
RST → Port 2
RST → Port 3
RST → Port 4
...
```

şeklindedir.

Bu yapı port taramasının ağ trafiğinde belirgin bir şekilde gözlemlenmesini sağlamaktadır.

---

# 13. Port Scanning Trafiğinin Özellikleri

Bu çalışmada port taramasını normal ağ trafiğinden ayırabilecek çeşitli davranışsal özellikler gözlemlenmiştir.

## 13.1 Çok Sayıda SYN Paketi

Tarama yapan sistem kısa sürede çok sayıda TCP SYN paketi gönderir.

```text
SYN
SYN
SYN
SYN
SYN
...
```

## 13.2 Çok Sayıda Farklı Hedef Port

Port taramasında aynı hedef sistem üzerinde birçok farklı port denenmektedir.

Örneğin:

```text
21
22
23
24
25
26
...
1000
```

## 13.3 Kısa Tarama Süresi

1000 port yaklaşık 0.54 saniyede taranmıştır.

Bu nedenle kısa zaman içerisinde yüksek sayıda bağlantı denemesi yapılması önemli bir davranışsal göstergedir.

## 13.4 RST Yanıtları

Kapalı TCP portlarına gönderilen SYN isteklerine hedef sistem tarafından RST/ACK yanıtları verilmiştir.

Genel olarak:

```text
SYN sayısı ↑
+
Farklı hedef port sayısı ↑
+
Kısa tarama süresi
+
RST sayısı ↑
```

port scanning davranışının göstergeleri arasında değerlendirilebilir.

---

# 14. CyberTrace Açısından Değerlendirme

Bu çalışma CyberTrace projesinin saldırı tespit mekanizması açısından önemlidir.

Bir NIDS yalnızca kaynak IP adresine bakarak:

> "Bu IP saldırgandır."

şeklinde karar vermemelidir.

Bunun yerine ağ davranışının incelenmesi gerekir.

Port scanning için kullanılabilecek özellikler:

| Özellik                    | Açıklama                         |
| -------------------------- | -------------------------------- |
| `source_ip`                | Trafiği başlatan sistem          |
| `destination_ip`           | Hedef sistem                     |
| `unique_destination_ports` | Denenen farklı hedef port sayısı |
| `tcp_syn_count`            | SYN paketlerinin sayısı          |
| `tcp_rst_count`            | RST paketlerinin sayısı          |
| `scan_duration`            | Taramanın süresi                 |
| `packets_per_second`       | Saniyedeki paket sayısı          |
| `destination_port_range`   | Denenen port aralığı             |

Örneğin sistem şu davranışı tespit ederse:

```text
Source IP:
192.168.56.1

Destination IP:
192.168.56.101

TCP SYN:
yüksek

Unique Destination Ports:
yüksek

Scan Duration:
kısa
```

port scanning şüphesi oluşturulabilir.

Bu değerler henüz sabit saldırı eşikleri olarak belirlenmemiştir. CyberTrace kapsamında daha fazla normal trafik ve saldırı verisi toplandıkça uygun eşikler veya makine öğrenmesi özellikleri belirlenebilir.

---

# 15. IOC ve Davranışsal Göstergeler

Port scanning için yalnızca IP adresini IOC olarak değerlendirmek doğru değildir.

Örneğin:

```text
192.168.56.1
```

adresinin tarama yapması bu laboratuvar ortamında saldırı trafiğini göstermektedir. Ancak gerçek bir ağ ortamında sistem yöneticileri veya güvenlik araçları da port taraması gerçekleştirebilir.

Bu nedenle port scanning tespitinde davranışsal göstergeler daha değerlidir:

* Çok sayıda farklı hedef port
* Çok sayıda SYN paketi
* Kısa zaman aralığında yoğun trafik
* Çok sayıda RST yanıtı
* Aynı hedefe yönelen seri bağlantı denemeleri

Bu yaklaşım yanlış pozitiflerin azaltılmasına yardımcı olabilir.

---

# 16. Port Scanning'in Güvenlik Açısından Önemi

Port scanning genellikle saldırıdan önce gerçekleştirilen keşif aşamalarından biridir.

Bir saldırgan hedef sistem hakkında bilgi toplamaya çalışabilir.

Örneğin:

```text
Port 22 açık
      ↓
SSH servisi olabilir

Port 80 açık
      ↓
HTTP servisi olabilir

Port 445 açık
      ↓
SMB servisi olabilir
```

Bu bilgiler daha sonraki saldırıların planlanmasında kullanılabilir.

Bu nedenle port scanning tek başına sisteme zarar vermese bile saldırı zincirinin erken aşamalarından biri olarak değerlendirilebilir.

---

# 17. Savunma Yaklaşımı

Port taramalarını tespit etmek ve etkilerini azaltmak için:

* Kısa sürede çok sayıda farklı porta yapılan bağlantı denemeleri izlenebilir.
* TCP SYN trafiği analiz edilebilir.
* Kaynak ve hedef IP ilişkileri takip edilebilir.
* IDS/IPS sistemleri kullanılabilir.
* Gereksiz servisler kapatılabilir.
* Gereksiz portlar firewall ile sınırlandırılabilir.
* Ağ segmentasyonu uygulanabilir.
* Şüpheli tarama davranışları için alarm oluşturulabilir.

NIDS açısından tek bir paketten karar vermek yerine belirli bir zaman aralığındaki trafik davranışını birlikte değerlendirmek daha sağlıklı bir yaklaşımdır.

---

# 18. Kullanılan Araçlar

Bu çalışma sırasında aşağıdaki araçlar kullanılmıştır:

* **Windows** → Tarama yapan sistem
* **Kali Linux** → Hedef sistem
* **VirtualBox** → Sanallaştırma ortamı
* **Nmap 7.99** → Port tarama
* **Wireshark** → Paket yakalama ve analiz

---

# 19. Üretilen Veri

Bu çalışma sonucunda port tarama trafiğini içeren bir PCAPNG dosyası oluşturulmuştur.

Dosya adı:

```text
port_scan_1000.pcapng
```

Bu dosya ilerleyen aşamalarda CyberTrace için oluşturulacak saldırı veri setinde **Port Scanning** sınıfının örneklerinden biri olarak kullanılacaktır.

Planlanan klasör yapısı:

```text
port-scanning/
│
├── port-scanning.md
├── port_scan_1000.pcapng
│
└── screenshots/
    ├── normal-ping.png
    ├── nmap-common-ports.png
    ├── tcp-syn-port-21.png
    ├── tcp-rst-port-21.png
    ├── nmap-1000-ports.png
    └── wireshark-syn-scan.png
```

---

# 20. Sonuç

Bu çalışmada port scanning işleminin temel çalışma mantığı incelenmiş ve izole bir sanal ağ ortamında uygulamalı olarak gerçekleştirilmiştir.

İlk olarak normal ICMP trafiği incelenmiş, ardından Windows sisteminden Kali Linux sistemine TCP port taraması gerçekleştirilmiştir.

İlk aşamada yaygın olarak kullanılan TCP portları taranmış ve tamamının kapalı olduğu görülmüştür.

Daha sonra hedef sistemin ilk 1000 TCP portu taranmıştır. Bu taramada tüm portların kapalı olduğu ve taramanın yaklaşık 0.54 saniyede tamamlandığı görülmüştür.

Wireshark incelemesinde Windows sisteminin farklı hedef portlarına TCP SYN paketleri gönderdiği ve kapalı portlardan RST/ACK yanıtları aldığı gözlemlenmiştir.

Çalışma sonucunda port scanning davranışının tek bir paket üzerinden değil;

```text
Çok sayıda farklı hedef port
            +
Çok sayıda SYN
            +
Kısa zaman aralığı
            +
RST yanıtları
            ↓
    Port Scanning davranışı
```

şeklindeki davranışsal özelliklerin birlikte değerlendirilmesiyle daha doğru şekilde tespit edilebileceği anlaşılmıştır.

Elde edilen PCAP verisi, CyberTrace projesinin saldırı veri setinde kullanılmak üzere saklanmıştır.

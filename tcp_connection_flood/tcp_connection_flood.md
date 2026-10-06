# TCP Connection Flood

## 1. Genel Bakış

TCP Connection Flood, hedef sisteme kısa bir süre içerisinde çok sayıda TCP bağlantısı oluşturularak sistemin bağlantı yönetimiyle ilgili kaynaklarının tüketilmeye çalışıldığı bir Hizmet Engelleme (DoS - Denial of Service) saldırısıdır.

TCP SYN Flood saldırısından farklı olarak bu saldırıda TCP üçlü el sıkışması (Three-Way Handshake) tamamlanır ve gerçek TCP bağlantıları oluşturulur.

Bu çalışmada saldırı, izole edilmiş VirtualBox ağ ortamında gerçekleştirilmiştir. Windows saldırı makinesi, Kali Linux ise hedef sistem olarak kullanılmıştır.

Bu deneyin temel amacı gerçek bir sistemi etkisiz hale getirmek değil, **CyberTrace projesinde kullanılmak üzere kontrollü TCP Connection Flood saldırı trafiği oluşturmak ve PCAP/PCAPNG verisi elde etmektir.**

---

## 2. Saldırının Çalışma Mantığı

TCP protokolünde bir bağlantı oluşturulmadan önce üç aşamalı bir el sıkışma gerçekleştirilir.

Bu işlem aşağıdaki şekilde gerçekleşir:

```text
Windows                         Kali Linux
(Saldırgan)                     (Hedef)
   |                                |
   | -------- SYN ----------------> |
   | <------- SYN/ACK ------------- |
   | -------- ACK ----------------> |
   |                                |
   |       TCP bağlantısı           |
```

Bu işlemden sonra TCP bağlantısı kurulmuş olur.

TCP Connection Flood saldırısında ise bu işlem çok yüksek sayıda bağlantı için tekrar tekrar gerçekleştirilir.

Örneğin:

```text
SYN
SYN/ACK
ACK
Bağlantı
Bağlantının kapatılması

SYN
SYN/ACK
ACK
Bağlantı
Bağlantının kapatılması

SYN
SYN/ACK
ACK
...
```

Bu işlemin kısa süre içerisinde çok sayıda kez gerçekleştirilmesi hedef sistemin TCP bağlantılarını yönetmek için daha fazla kaynak kullanmasına neden olabilir.

---

## 3. Saldırının Yapılma Amacı

TCP Connection Flood saldırısının temel amacı hedef sistemin TCP bağlantılarını yönetmek için kullandığı kaynakları tüketmektir.

Saldırı sırasında aşağıdaki kaynaklar etkilenebilir:

* TCP socket'leri
* Kernel belleği
* File descriptor'lar
* Connection tracking tabloları
* CPU kaynakları
* Sunucu process/thread kaynakları
* Uygulamanın bağlantı havuzu (connection pool)

Saldırı yoğunluğu yeterince yüksek olduğunda hedef sistemde:

* CPU kullanımında artış,
* bellek kullanımında artış,
* bağlantı sayısında artış,
* servis yanıt süresinde artış,
* yeni bağlantıların kurulmasında gecikme,
* servis erişilebilirliğinde azalma

gibi sonuçlar ortaya çıkabilir.

---

## 4. TCP Connection Flood ve TCP SYN Flood Farkı

TCP Connection Flood ile TCP SYN Flood birbirine benzese de aynı saldırı değildir.

| Özellik               | TCP SYN Flood                    | TCP Connection Flood                            |
| --------------------- | -------------------------------- | ----------------------------------------------- |
| SYN paketleri         | Çok fazla                        | Çok fazla                                       |
| SYN/ACK               | Gönderilir                       | Gönderilir                                      |
| Son ACK               | Genellikle gönderilmez           | Gönderilir                                      |
| Three-Way Handshake   | Tamamlanmaz                      | **Tamamlanır**                                  |
| Gerçek TCP bağlantısı | Genellikle oluşmaz               | **Oluşur**                                      |
| Temel amaç            | Yarım-açık bağlantıları artırmak | Çok sayıda bağlantı oluşturarak kaynak tüketmek |
| Wireshark görünümü    | SYN ağırlıklı trafik             | SYN → SYN/ACK → ACK tekrarları                  |

Bu ayrım CyberTrace projesi açısından önemlidir.

Çünkü ileride makine öğrenmesi modelinde:

```text
TCP_SYN_FLOOD
```

ve

```text
TCP_Connection_Flood
```

iki farklı saldırı sınıfı olarak kullanılacaktır.

---

## 5. Laboratuvar Ortamı

Saldırı izole edilmiş VirtualBox Host-Only ağında gerçekleştirilmiştir.

### Saldırgan Sistem

```text
İşletim Sistemi: Windows
IP Adresi: 192.168.56.1
Rol: Saldırgan
```

### Hedef Sistem

```text
İşletim Sistemi: Kali Linux
Ağ Arayüzü: eth1
IP Adresi: 192.168.56.101
Rol: Hedef
```

### Hedef TCP Servisi

Saldırının kontrollü şekilde gerçekleştirilebilmesi için Kali Linux üzerinde basit bir Python TCP sunucusu çalıştırılmıştır.

```text
Protokol: TCP
Hedef IP: 192.168.56.101
Hedef Port: 9000
```

Bu servis yalnızca laboratuvar ortamında saldırı trafiği oluşturmak ve Wireshark üzerinden inceleme yapmak amacıyla kullanılmıştır.

---

## 6. Saldırı Yapılandırması

Kali Linux üzerinde `192.168.56.101:9000` adresinde TCP bağlantılarını kabul eden basit bir sunucu çalıştırılmıştır.

Windows sistemi üzerinden hedef sunucuya çok sayıda TCP bağlantısı oluşturulmuştur.

Deneyde kullanılan temel yapı:

```text
Saldırgan:
192.168.56.1

Hedef:
192.168.56.101

Hedef Port:
9000

Protokol:
TCP
```

Kontrollü bir deney oluşturmak amacıyla belirli sayıda TCP bağlantısı oluşturulmuştur.

---

## 7. Saldırının Gerçekleştirilmesi

Saldırı sırasında Windows sistemi tarafından Kali Linux üzerindeki TCP/9000 servisine art arda bağlantılar oluşturulmuştur.

Her bağlantı için TCP Three-Way Handshake gerçekleştirilmiştir:

```text
Windows → Kali       SYN
Kali → Windows       SYN/ACK
Windows → Kali       ACK
```

Bağlantı kurulduktan sonra bağlantı kapatılmış ve yeni bir TCP bağlantısı oluşturulmuştur.

Bu işlem çok sayıda kez tekrarlandığında hedef sistem kısa süre içerisinde çok sayıda TCP bağlantısı ile karşılaşmıştır.

Bu nedenle oluşan ağ trafiği TCP Connection Flood saldırısının karakteristik özelliklerini taşımaktadır.

---

## 8. Wireshark ile Trafik Analizi

Saldırı trafiği Kali Linux üzerinde Wireshark kullanılarak `eth1` ağ arayüzünden yakalanmıştır.

Saldırı trafiğini filtrelemek için aşağıdaki Wireshark filtresi kullanılabilir:

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && tcp.port == 9000
```

Bu filtre Windows ile Kali arasındaki TCP/9000 trafiğini göstermektedir.

### SYN Paketlerini İnceleme

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && tcp.flags.syn == 1 && tcp.flags.ack == 0
```

Bu filtre TCP bağlantılarının başlatılması için gönderilen SYN paketlerini gösterir.

### SYN/ACK Paketlerini İnceleme

```text
ip.src == 192.168.56.101 && ip.dst == 192.168.56.1 && tcp.flags.syn == 1 && tcp.flags.ack == 1
```

Bu filtre hedef sistem tarafından gönderilen SYN/ACK cevaplarını gösterir.

### ACK Paketlerini İnceleme

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && tcp.flags.ack == 1
```

Bu paketler TCP Three-Way Handshake'in tamamlanmasında kullanılır.

---

## 9. Wireshark'ta Gözlemlenen Trafik Özellikleri

TCP Connection Flood trafiğinde aynı bağlantı oluşturma sürecinin çok sayıda kez tekrarlandığı görülür.

Tipik bir bağlantı sırası:

```text
Windows → Kali       SYN
Kali → Windows       SYN/ACK
Windows → Kali       ACK
                     ↓
               TCP bağlantısı
                     ↓
               Bağlantı kapatılır
                     ↓
                 Yeni SYN
```

Saldırı sırasında aşağıdaki trafik özellikleri gözlemlenebilir:

* Çok sayıda TCP paketi
* Kısa zaman içerisinde yüksek bağlantı sayısı
* Çok sayıda SYN paketi
* Çok sayıda SYN/ACK paketi
* Çok sayıda ACK paketi
* Tamamlanmış TCP Three-Way Handshake'ler
* Çok sayıda kısa süreli TCP bağlantısı
* Yüksek bağlantı oluşturma oranı
* Aynı kaynak ve hedef IP adreslerinin tekrarlanması
* Aynı hedef portun yoğun şekilde kullanılması

Bu özellikler ileride CyberTrace içerisinde saldırının tespit edilmesinde kullanılabilecek önemli göstergelerdir.

---

## 10. TCP Bağlantı Durumlarının İncelenmesi

Kali Linux üzerinde TCP bağlantılarının durumları `ss` komutu ile incelenebilir.

```bash
ss -ant
```

Yalnızca 9000 numaralı portla ilgili bağlantıları görmek için:

```bash
ss -ant | grep ':9000'
```

Saldırı sırasında bağlantıların hızlı şekilde oluşturulup kapatılması nedeniyle aşağıdaki TCP durumları gözlemlenebilir:

```text
ESTAB
TIME-WAIT
CLOSE-WAIT
```

Özellikle `TIME-WAIT` durumundaki bağlantıların artması, çok sayıda kısa süreli TCP bağlantısının oluşturulduğunu gösterebilir.

---

## 11. Hedef Sistem Üzerindeki Etkileri

TCP Connection Flood saldırısının etkisi saldırı hızına, hedef sistemin kaynaklarına ve çalışan uygulamanın yapısına bağlıdır.

Yüksek saldırı yoğunluklarında aşağıdaki etkiler ortaya çıkabilir:

1. CPU kullanımının artması
2. Bellek tüketiminin artması
3. TCP socket sayısının artması
4. Connection tracking yükünün artması
5. Sunucu uygulamasının daha fazla bağlantı işlemesi
6. Servis yanıt süresinin artması
7. Yeni bağlantıların kurulmasında gecikme
8. Servis kullanılabilirliğinin azalması

Bu çalışmada saldırı kontrollü bir laboratuvar ortamında gerçekleştirildiği için amaç hedef sistemi tamamen kullanılamaz hale getirmek değil, saldırıya ait karakteristik ağ trafiğini oluşturmaktır.

---

## 12. Saldırının Tespit Edilebilecek Göstergeleri

Bir Network Intrusion Detection System (NIDS), TCP Connection Flood saldırısını tespit etmek için çeşitli trafik özelliklerini inceleyebilir.

Önemli göstergeler:

* Çok yüksek TCP connection rate
* Aynı kaynak IP'den çok sayıda bağlantı
* Aynı hedef IP'ye çok sayıda bağlantı
* Aynı hedef portuna yoğun bağlantı
* Tekrarlanan SYN → SYN/ACK → ACK dizileri
* Yüksek packet rate
* Çok sayıda kısa süreli TCP bağlantısı
* Çok sayıda `TIME-WAIT` bağlantısı
* Kısa zaman aralığında yüksek bağlantı sayısı

Basitleştirilmiş bir tespit mantığı:

```text
Yüksek bağlantı oranı
        +
Tekrarlanan TCP Three-Way Handshake
        +
Aynı kaynak/hedef
        +
Kısa süreli bağlantılar
        ↓
TCP Connection Flood şüphesi
```

---

## 13. CyberTrace Veri Seti İçin Kullanılabilecek Özellikler

Oluşturulan PCAP dosyası ilerleyen aşamada ağ akışlarına (network flows) dönüştürülerek makine öğrenmesi için kullanılabilir.

Kullanılabilecek bazı özellikler:

| Özellik              | Açıklama                 |
| -------------------- | ------------------------ |
| `src_ip`             | Kaynak IP adresi         |
| `dst_ip`             | Hedef IP adresi          |
| `src_port`           | Kaynak TCP portu         |
| `dst_port`           | Hedef TCP portu          |
| `protocol`           | Kullanılan protokol      |
| `packet_count`       | Paket sayısı             |
| `byte_count`         | Toplam veri miktarı      |
| `flow_duration`      | Akış süresi              |
| `packets_per_second` | Saniyedeki paket sayısı  |
| `bytes_per_second`   | Saniyedeki byte miktarı  |
| `syn_count`          | SYN paket sayısı         |
| `syn_ack_count`      | SYN/ACK paket sayısı     |
| `ack_count`          | ACK paket sayısı         |
| `fin_count`          | FIN paket sayısı         |
| `rst_count`          | RST paket sayısı         |
| `connection_rate`    | Bağlantı oluşturma oranı |
| `label`              | Saldırı etiketi          |

Özellikle `connection_rate`, `packet_count`, TCP flag sayıları ve kısa süreli bağlantıların sayısı TCP Connection Flood tespiti açısından önemli olabilir.

---

## 14. CyberTrace Veri Seti Etiketi

Bu saldırı için veri setinde kullanılacak etiket:

```text
TCP_Connection_Flood
```

Örnek kayıt:

```text
Source IP:        192.168.56.1
Destination IP:   192.168.56.101
Protocol:         TCP
Destination Port: 9000
Attack Type:      TCP_Connection_Flood
```

Bu etiket, saldırı PCAP'ının daha sonra diğer saldırı türleri ve normal ağ trafiği ile birleştirilerek CyberTrace veri setine dahil edilmesini sağlayacaktır.

---

## 15. PCAP Dosyası

Saldırı sırasında Wireshark ile yakalanan trafik aşağıdaki dosya içerisinde saklanmıştır:

```text
tcp_connection_flood.pcapng
```

Önerilen GitHub konumu:

```text
tcp-connection-flood/
└── tcp_connection_flood.pcapng
```

PCAP dosyası saldırı sırasında oluşturulan TCP bağlantı trafiğini içermektedir.

---

## 16. Ekran Görüntüleri

### Wireshark – Genel Saldırı Trafiği

```text
[EKRAN GÖRÜNTÜSÜ EKLENECEK]
```

Bu görüntüde Windows ve Kali arasındaki yoğun TCP bağlantı trafiği gösterilecektir.

### TCP Three-Way Handshake

```text
[EKRAN GÖRÜNTÜSÜ EKLENECEK]
```

Bu görüntüde:

```text
SYN
SYN/ACK
ACK
```

paketleri gösterilerek TCP bağlantısının tamamlandığı açıklanacaktır.

### TCP Bağlantı Yoğunluğu

<img width="1352" height="624" alt="image" src="https://github.com/user-attachments/assets/3da13cee-0004-47f3-8dd6-654293bd4ab9" />


Bu görüntüde kısa süre içerisinde oluşturulan yüksek sayıdaki TCP bağlantısı gösterilecektir.

---

## 17. Sonuç

Bu çalışmada TCP Connection Flood saldırısının çalışma mantığı ve ağ üzerindeki davranışı kontrollü bir laboratuvar ortamında incelenmiştir.

Saldırı sırasında Windows sistemi tarafından Kali Linux üzerindeki TCP/9000 servisine çok sayıda TCP bağlantısı oluşturulmuştur.

TCP SYN Flood saldırısından farklı olarak TCP Three-Way Handshake tamamlanmış ve gerçek TCP bağlantıları oluşturulmuştur.

Wireshark üzerinde çok sayıda:

```text
SYN
SYN/ACK
ACK
```

paket dizisi ve kısa süreli TCP bağlantıları gözlemlenmiştir.

Elde edilen PCAP dosyası, CyberTrace projesinde kullanılmak üzere **`TCP_Connection_Flood`** etiketiyle veri setine dahil edilecektir.

İlerleyen aşamada bu trafik normal ağ trafiği ve diğer saldırı türlerinden elde edilen PCAP dosyalarıyla birleştirilerek makine öğrenmesi tabanlı Network Intrusion Detection System (NIDS) modelinin eğitiminde kullanılacaktır.

---

## 18. Deney Özeti

| Parametre          | Değer                              |
| ------------------ | ---------------------------------- |
| Saldırı            | TCP Connection Flood               |
| Saldırı Kategorisi | Denial of Service (DoS)            |
| Protokol           | TCP                                |
| Saldırgan          | Windows                            |
| Saldırgan IP       | 192.168.56.1                       |
| Hedef              | Kali Linux                         |
| Hedef IP           | 192.168.56.101                     |
| Hedef Port         | 9000                               |
| Yakalama Arayüzü   | eth1                               |
| Dosya Formatı      | PCAPNG                             |
| Veri Seti Etiketi  | `TCP_Connection_Flood`             |
| Laboratuvar Ortamı | İzole VirtualBox Host-Only Network |
| Amaç               | CyberTrace Veri Seti Oluşturma     |

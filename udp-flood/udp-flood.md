# UDP Flood

## 1. Saldırının Amacı

Bu çalışmanın amacı, UDP tabanlı yoğun trafik oluşturularak UDP Flood saldırısının temel çalışma mantığını gözlemlemek ve saldırı trafiğinin Wireshark üzerinden nasıl analiz edilebileceğini öğrenmektir.

Çalışma kapsamında normal UDP trafiği ile yoğun UDP trafiği karşılaştırılmış, oluşturulan paketlerin kaynak ve hedef bilgileri incelenmiş ve saldırının CyberTrace projesinde tespit edilebilmesi için kullanılabilecek ağ özellikleri belirlenmiştir.

---

## 2. UDP Nedir?

UDP (User Datagram Protocol), OSI modelinin taşıma katmanında (Layer 4) çalışan, bağlantısız bir iletişim protokolüdür.

TCP'den farklı olarak UDP iletişim başlamadan önce bağlantı kurulmasını sağlayan bir üçlü el sıkışma mekanizması kullanmaz.

UDP'nin temel özellikleri:

* Bağlantısız çalışır.
* TCP gibi üçlü el sıkışma gerçekleştirmez.
* Paketlerin ulaşacağı garanti edilmez.
* Paketlerin sırası garanti edilmez.
* Yeniden iletim mekanizması bulunmaz.
* TCP'ye göre daha düşük protokol yüküne sahiptir.
* DNS, DHCP, VoIP, çevrim içi oyunlar ve bazı gerçek zamanlı uygulamalarda kullanılabilir.

UDP paketleri küçük bir başlık ve uygulama verisinden oluşur.

---

## 3. UDP Flood Nedir?

UDP Flood, hedef sisteme kısa bir zaman aralığında çok sayıda UDP paketi gönderilmesine dayanan bir hizmet engelleme (DoS) saldırısı türüdür.

Saldırının temel mantığı:

```text
Saldırgan
192.168.56.1
       │
       │ Çok sayıda UDP paketi
       │
       ▼
Hedef
192.168.56.101
```

Hedef sistem gelen UDP paketlerini işlemeye çalışabilir. Hedef portta çalışan bir servis bulunmuyorsa sistem ICMP "Destination Unreachable / Port Unreachable" cevapları da oluşturabilir.

Yüksek miktarda UDP trafiği; ağ bant genişliği, işlemci kullanımı ve hedef uygulamanın kaynakları üzerinde yük oluşturabilir.

Bu çalışmada gerçek bir sisteme yönelik saldırı yerine, yalnızca izole edilmiş VirtualBox laboratuvar ortamında kontrollü ve sınırlı sayıda paket kullanılmıştır.

---

## 4. Laboratuvar Ortamı

Çalışma aşağıdaki izole ağ ortamında gerçekleştirilmiştir:

| Cihaz      | IP Adresi         | Rol                |
| ---------- | ----------------- | ------------------ |
| Windows    | `192.168.56.1`    | UDP trafik kaynağı |
| Kali Linux | `192.168.56.101`  | Hedef sistem       |
| Ağ         | `192.168.56.0/24` | Host-Only Network  |

Wireshark, Kali Linux üzerinde `eth1` arayüzünde çalıştırılmıştır.

Bu yapı sayesinde çalışma gerçek internete veya üçüncü taraf sistemlere yönelmeden kontrollü bir laboratuvar ortamında gerçekleştirilmiştir.

---

## 5. Normal UDP Trafiğinin Gözlemlenmesi

Öncelikle normal UDP iletişiminin nasıl göründüğünü incelemek için Kali Linux üzerinde UDP/9999 portu dinlemeye açılmıştır.

Kullanılan komut:

```bash
nc -u -l -p 9999
```

Windows tarafından Kali'ye küçük bir UDP mesajı gönderilmiştir.

Örnek paket:

```text
192.168.56.1 → 192.168.56.101
UDP
54587 → 9999
Len=3
```

Daha önce yapılan normal UDP testinde `Hello UDP` mesajı da gözlemlenmiştir.

Bu aşamada UDP trafiğinin TCP'den farklı olarak herhangi bir TCP handshake gerçekleştirmeden doğrudan veri taşıdığı gözlemlenmiştir.

---

## 6. UDP Flood Testi

Kontrollü saldırı trafiğini oluşturmak amacıyla Windows üzerinden Kali'nin UDP/9999 portuna çok sayıda UDP paketi gönderilmiştir.

Test sırasında kullanılan veri:

```text
UDP-FLOOD-TEST
```

Bu verinin uzunluğu:

```text
14 byte
```

Wireshark üzerinde paketler şu şekilde gözlemlenmiştir:

```text
192.168.56.1 → 192.168.56.101
UDP
62020 → 9999
Len=14
```

ve devam eden paketlerde kaynak portların değiştiği görülmüştür:

```text
62020 → 9999
62021 → 9999
62022 → 9999
...
62119 → 9999
```

Bu durum Windows tarafından oluşturulan UDP paketlerinin hedef UDP/9999 portuna yoğun şekilde gönderildiğini göstermektedir.

---

## 7. Wireshark Analizi

UDP trafiğini incelemek için aşağıdaki Wireshark filtresi kullanılmıştır:

```text
udp.dstport == 9999 && ip.src == 192.168.56.1
```

Daha sonra yalnızca kontrollü UDP Flood paketlerini ayırmak için:

```text
ip.src == 192.168.56.1 &&
ip.dst == 192.168.56.101 &&
udp.dstport == 9999 &&
udp.length == 22
```

filtresi kullanılmıştır.

Buradaki `udp.length == 22` değeri:

```text
8 byte  → UDP başlığı
14 byte → UDP payload
---------------------
22 byte → Toplam UDP uzunluğu
```

şeklinde açıklanabilir.

Wireshark üzerinde saldırı trafiğinde aynı hedef IP ve hedef port için kısa zaman aralığında çok sayıda UDP paketi gözlemlenmiştir.

---

## 8. UDP Flood Paketlerinin İncelenmesi

Yakalanan saldırı trafiğinin başlangıcındaki paketlerden biri:

```text
192.168.56.1 → 192.168.56.101
UDP
62020 → 9999
Len=14
```

Saldırı trafiğinin sonlarına doğru ise:

```text
192.168.56.1 → 192.168.56.101
UDP
62119 → 9999
Len=14
```

paketi görülmüştür.

Kaynak portların sıralı olarak değişmesi, Windows tarafında oluşturulan her UDP gönderiminde yeni bir geçici kaynak port kullanılmasından kaynaklanmaktadır.

---

## 9. Saldırı Trafiğinin Zaman Analizi

Kontrollü testte 100 UDP paketi gözlemlenmiştir.

İlk ve son saldırı paketlerinin zaman değerleri yaklaşık olarak:

```text
İlk paket : 1084.193710214
Son paket : 1084.319218393
```

şeklindedir.

Aradaki süre yaklaşık:

```text
0.1255 saniye
```

olmaktadır.

Yaklaşık paket gönderim hızı:

```text
100 / 0.1255 ≈ 797 paket/saniye
```

olarak hesaplanmıştır.

Bu değer, kontrollü laboratuvar testinde UDP paketlerinin kısa bir zaman aralığında yoğun şekilde gönderildiğini göstermektedir.

---

## 10. ICMP Destination Unreachable Paketleri

Çalışma sırasında bazı UDP paketlerinin ardından Kali tarafından ICMP:

```text
Destination unreachable (Port unreachable)
```

mesajları oluşturulduğu gözlemlenmiştir.

Örneğin:

```text
192.168.56.101 → 192.168.56.1
ICMP
Destination unreachable (Port unreachable)
```

Bu mesaj, hedef sistemde ilgili UDP portunda uygun bir uygulama bulunmadığında işletim sisteminin göndericiye portun kullanılamadığını bildirmesiyle oluşabilir.

Bu nedenle ICMP Port Unreachable paketleri tek başına UDP Flood göstergesi olarak değerlendirilmemelidir. Ancak yoğun UDP trafiği ile birlikte incelendiğinde saldırı davranışının anlaşılmasına yardımcı olabilir.

---

## 11. Normal UDP ve UDP Flood Karşılaştırması

| Özellik         | Normal UDP               | UDP Flood                          |
| --------------- | ------------------------ | ---------------------------------- |
| Paket yoğunluğu | Düşük                    | Yüksek                             |
| Gönderim süresi | Normal                   | Çok kısa zaman aralığında yoğun    |
| Hedef port      | Uygulama ihtiyacına göre | Belirli bir porta yoğunlaşabilir   |
| Paket sayısı    | Az                       | Çok fazla                          |
| Paket hızı      | Düşük/normal             | Yüksek                             |
| Payload         | Uygulamaya bağlı         | Tekrarlı veya farklı olabilir      |
| Kaynak port     | Değişebilir              | Çok sayıda geçici port görülebilir |
| Tespit          | Normal trafik profili    | Ani trafik artışı                  |

Bu karşılaştırmada saldırıyı belirleyen tek bir özellik bulunmadığı görülmektedir. Özellikle paket sayısı, paket hızı, zaman aralığı ve hedef port gibi birden fazla özelliğin birlikte değerlendirilmesi daha doğru sonuç verebilir.

---

## 12. CyberTrace İçin Kullanılabilecek Özellikler

UDP Flood tespitinde CyberTrace içerisinde aşağıdaki ağ özellikleri kullanılabilir:

```text
source_ip
destination_ip
source_port
destination_port
protocol
udp_packet_count
udp_packet_rate
total_bytes
average_packet_size
unique_source_ports
flow_duration
```

Özellikle aşağıdaki özellikler saldırı tespitinde önemlidir:

### UDP Paket Sayısı

Belirli bir zaman aralığında aynı hedefe gönderilen UDP paketlerinin sayısı incelenebilir.

### UDP Paket Hızı

```text
packet_count / time_window
```

formülü ile saniyedeki UDP paket sayısı hesaplanabilir.

### Hedef Port

Belirli bir hedef porta kısa sürede yoğun UDP trafiği gönderilmesi davranışsal bir gösterge olabilir.

### Kaynak Port Çeşitliliği

Kısa süre içerisinde çok sayıda farklı kaynak portundan aynı hedef porta UDP paketleri gönderilmesi incelenebilir.

### Trafik Yoğunluğu

Normal trafik profiline göre ani UDP trafik artışı tespit edilebilir.

---

## 13. Tespit İçin Davranışsal Göstergeler

CyberTrace açısından UDP Flood için değerlendirilebilecek davranışsal göstergeler:

* Kısa zaman aralığında yüksek UDP paket sayısı
* Yüksek UDP paket/saniye oranı
* Aynı hedef IP'ye yoğun trafik
* Aynı hedef porta yoğunlaşma
* Çok sayıda farklı kaynak portu
* Normal trafik seviyesine göre ani UDP trafik artışı
* UDP paketlerinin tekrarlı yapısı
* Yoğun UDP trafiğinin ardından ICMP Port Unreachable mesajlarının oluşması

Ancak bu göstergelerin hiçbiri tek başına saldırı anlamına gelmemelidir. DNS, oyun, medya veya gerçek zamanlı iletişim gibi normal uygulamalar da yüksek UDP trafiği oluşturabilir.

Bu nedenle CyberTrace içerisinde birden fazla özelliğin birlikte değerlendirilmesi planlanmaktadır.

---

## 14. CyberTrace Açısından Önemi

UDP Flood çalışması CyberTrace projesi açısından önemlidir çünkü sistemin yalnızca tek tek paketleri değil, trafik davranışını da analiz etmesi gerekmektedir.

Örneğin tek bir UDP paketi normal olabilir:

```text
UDP → 9999
```

Ancak kısa bir zaman aralığında yüzlerce veya binlerce benzer paketin aynı hedefe gönderilmesi farklı bir davranış profili oluşturur.

Bu nedenle CyberTrace'in ilerleyen aşamalarında zaman penceresi tabanlı özelliklerin kullanılması planlanmaktadır.

Örneğin:

```text
5 saniyelik zaman penceresi
        ↓
UDP paketlerini say
        ↓
Paket hızını hesapla
        ↓
Hedef IP/port dağılımını incele
        ↓
Normal trafik ile karşılaştır
        ↓
Risk / saldırı sınıfı üret
```

Bu yaklaşım, daha sonra oluşturulacak makine öğrenmesi veri seti için de kullanılabilecek özelliklerin belirlenmesine yardımcı olacaktır.

---

## 15. PCAP Verisi

Bu çalışma sonucunda UDP Flood trafiğini içeren PCAP dosyası oluşturulmuştur.

Dosya:

```text
udp_flood.pcapng
```

PCAP içerisinde kontrollü UDP Flood testine ait trafik bulunmaktadır.

Saldırı trafiğinin temel özellikleri:

```text
Source IP      : 192.168.56.1
Destination IP : 192.168.56.101
Protocol       : UDP
Destination    : 9999
Packet Count   : 100
Payload        : UDP-FLOOD-TEST
Payload Size   : 14 bytes
Approx. Rate   : 797 packets/sec
```

---

## 16. Kullanılan Araçlar

* Kali Linux
* Windows
* VirtualBox
* Wireshark
* Netcat (`nc`)
* PowerShell
* UDP
* PCAP/PCAPNG

---

## 17. Oluşturulan Veri

Bu çalışma kapsamında:

* Normal UDP trafiği gözlemlendi.
* UDP/9999 portuna test mesajı gönderildi.
* Kontrollü UDP yoğun trafiği oluşturuldu.
* Wireshark ile UDP paketleri analiz edildi.
* UDP Flood trafiğine ait PCAP verisi oluşturuldu.
* ICMP Port Unreachable cevapları incelendi.
* CyberTrace için kullanılabilecek ağ özellikleri belirlendi.

---

## 18. Sonuç

Bu çalışmada UDP Flood saldırısının temel çalışma mantığı kontrollü bir laboratuvar ortamında incelenmiştir.

Normal UDP iletişiminde az sayıda UDP paketinin hedef porta gönderildiği gözlemlenirken, UDP Flood testinde kısa bir zaman aralığında aynı hedef IP ve hedef porta çok sayıda UDP paketi gönderildiği görülmüştür.

Wireshark analizi sonucunda 100 adet kontrollü UDP Flood paketi ve yaklaşık 0.1255 saniyelik bir trafik aralığı gözlemlenmiş, buna göre yaklaşık 797 paket/saniye seviyesinde trafik oluştuğu hesaplanmıştır.

Bu çalışma sonucunda UDP Flood'un yalnızca paket içeriğine bakılarak değil; paket sayısı, paket hızı, hedef port, kaynak port çeşitliliği ve zaman aralığı gibi davranışsal özelliklerin birlikte değerlendirilmesiyle daha doğru şekilde tespit edilebileceği anlaşılmıştır.

Oluşturulan `udp_flood.pcapng` verisi ilerleyen aşamalarda CyberTrace projesinde analiz ve makine öğrenmesi çalışmalarında kullanılmak üzere veri setine dahil edilecektir.

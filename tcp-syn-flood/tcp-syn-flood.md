# TCP SYN Flood

## 1. Saldırının Amacı

TCP SYN Flood, TCP bağlantısının kurulma aşamasını hedefleyen bir hizmet engelleme (DoS) saldırısıdır. Saldırının temel amacı, hedef sisteme çok sayıda TCP SYN paketi göndererek hedefin bağlantı kaynaklarını meşgul etmek ve normal istemcilerin bağlantı kurmasını zorlaştırmaktır.

Bu çalışmada TCP SYN Flood davranışı izole edilmiş bir laboratuvar ortamında oluşturulmuş, Wireshark kullanılarak TCP paketleri incelenmiş ve elde edilen trafik PCAP formatında kaydedilmiştir.

Çalışmanın temel amacı sistemi kullanılamaz hâle getirmek değil, saldırının ağ üzerindeki davranışını gözlemlemek ve CyberTrace projesinde kullanılabilecek ağ trafiği özelliklerini belirlemektir.

---

## 2. TCP Nedir?

TCP (Transmission Control Protocol), ağ üzerindeki iki cihaz arasında güvenilir ve bağlantı tabanlı iletişim sağlayan bir taşıma katmanı protokolüdür.

TCP iletişiminde veri aktarımından önce istemci ve sunucu arasında bir bağlantı kurulması gerekir.

Bu bağlantı genellikle **Three-Way Handshake** olarak adlandırılan üç aşamalı işlem ile gerçekleştirilir:

```text
İstemci                         Sunucu
   |                              |
   | -------- SYN -------------> |
   | <------- SYN/ACK ---------- |
   | -------- ACK -------------> |
   |                              |
   |      TCP bağlantısı          |
```

Bu üç paket:

* **SYN:** Bağlantının başlatılmasını ister.
* **SYN/ACK:** Sunucu bağlantı isteğini kabul ettiğini bildirir.
* **ACK:** İstemci sunucunun cevabını aldığını onaylar.

Bu işlem tamamlandıktan sonra TCP bağlantısı üzerinden veri aktarımı başlayabilir.

---

## 3. TCP SYN Flood Nedir?

TCP SYN Flood, TCP Three-Way Handshake mekanizmasının ilk aşaması olan SYN paketlerini hedef alan bir saldırıdır.

Normal bir bağlantıda:

```text
SYN → SYN/ACK → ACK
```

şeklinde bağlantı tamamlanırken SYN Flood davranışında çok sayıda SYN isteği oluşturulur.

Basitleştirilmiş saldırı davranışı:

```text
Saldırgan                         Hedef
    |                               |
    | -------- SYN ---------------> |
    | -------- SYN ---------------> |
    | -------- SYN ---------------> |
    | -------- SYN ---------------> |
    | -------- SYN ---------------> |
    |                               |
```

Hedef sistem aldığı SYN paketlerine SYN/ACK ile cevap vermeye çalışır. Bağlantının tamamlanmaması durumunda hedef üzerinde çok sayıda yarım bağlantı durumu oluşabilir.

Yoğun trafik altında sistemin bağlantı kaynakları zorlanabilir ve meşru istemcilerin bağlantı kurması etkilenebilir.

---

## 4. Laboratuvar Ortamı

Çalışma yalnızca kontrol edilen VirtualBox Host-Only ağı üzerinde gerçekleştirilmiştir.

| Cihaz      | IP Adresi        | Rol                      |
| ---------- | ---------------- | ------------------------ |
| Windows    | `192.168.56.1`   | Trafik oluşturan istemci |
| Kali Linux | `192.168.56.101` | Hedef sistem             |

Kali Linux üzerinde TCP 8000 portunda Python HTTP servisi çalıştırılmıştır.

Servisin kontrolü için:

```bash
sudo ss -tulpn
```

komutu kullanılmıştır.

Çıktıda:

```text
0.0.0.0:8000
python3
LISTEN
```

bilgisi görülmüştür.

Bu nedenle TCP SYN trafiği için hedef olarak:

```text
192.168.56.101:8000
```

kullanılmıştır.

---

## 5. Normal TCP Trafiğinin Gözlemlenmesi

Saldırı gerçekleştirilmeden önce normal TCP bağlantısı gözlemlenmiştir.

Windows üzerinden Kali'nin 8000 numaralı HTTP servisine erişildiğinde Wireshark üzerinde Three-Way Handshake görülmüştür.

### Paket 1 – SYN

```text
192.168.56.1 → 192.168.56.101
19251 → 8000 [SYN]
```

Bu paket Windows istemcisinin Kali üzerindeki 8000 numaralı TCP portuna bağlantı başlatmak istediğini gösterir.

### Paket 2 – SYN/ACK

```text
192.168.56.101 → 192.168.56.1
8000 → 19251 [SYN, ACK]
```

Kali, bağlantı isteğine SYN/ACK paketi ile cevap vermiştir.

### Paket 3 – ACK

```text
192.168.56.1 → 192.168.56.101
19251 → 8000 [ACK]
```

Windows son ACK paketini göndererek TCP bağlantısının kurulmasını tamamlamıştır.

Bu nedenle normal TCP bağlantısında:

```text
SYN
 ↓
SYN/ACK
 ↓
ACK
```

şeklinde tamamlanan bir handshake gözlemlenmiştir.

### Ekran Görüntüsü

<img width="1366" height="457" alt="1" src="https://github.com/user-attachments/assets/e844773e-c5a5-4031-8ee4-9fb91f8e08e9" />


---

## 6. Kontrollü SYN Trafiğinin Oluşturulması

SYN trafiği oluşturmak için Windows üzerinde Nping kullanılmıştır.

İlk olarak küçük bir test gerçekleştirilmiştir:

```powershell
nping --tcp -p 8000 --flags syn -c 10 192.168.56.101
```

Bu test ile 8000 numaralı porta SYN paketlerinin gönderildiği ve Wireshark üzerinde gözlemlenebildiği doğrulanmıştır.

Daha sonra kontrollü miktarda SYN trafiği oluşturmak amacıyla:

```powershell
nping --tcp -p 8000 --flags syn -c 100 --rate 20 192.168.56.101
```

komutu kullanılmıştır.

Komuttaki parametrelerin anlamları:

* `--tcp` → TCP paketleri oluşturur.
* `-p 8000` → Hedef TCP portunu 8000 olarak belirler.
* `--flags syn` → SYN bayrağı bulunan TCP paketleri oluşturur.
* `-c 100` → 100 paket gönderilmesini ister.
* `--rate 20` → Paketlerin kontrollü bir hızda gönderilmesini sağlar.
* `192.168.56.101` → Kali laboratuvar makinesinin IP adresidir.

Bu işlem yalnızca izole edilmiş Host-Only laboratuvar ağı içerisinde gerçekleştirilmiştir.

---

## 7. Wireshark Analizi

SYN paketlerini incelemek için aşağıdaki Wireshark filtresi kullanılmıştır:

```text
tcp.flags.syn == 1 && tcp.flags.ack == 0 && ip.src == 192.168.56.1 && ip.dst == 192.168.56.101
```

Bu filtre ile Windows'tan Kali'ye gönderilen SYN paketleri incelenmiştir.

Gözlemlenen trafik:

```text
192.168.56.1 → 192.168.56.101
TCP
Destination Port: 8000
Flags: SYN
```

Capture içerisinde filtre uygulandığında 110 SYN paketi görülmüştür.

Bu sayı doğrudan Nping tarafından gönderilen paket sayısı olarak değerlendirilmemelidir. Capture içerisinde önceki kontrollü testlerden kalan ve aynı filtre koşullarını sağlayan paketler de bulunmaktadır.

### Ekran Görüntüsü

<img width="1366" height="336" alt="image" src="https://github.com/user-attachments/assets/dc340feb-38b2-4437-a69e-6cfb4b28b2ce" />


---

## 8. SYN/ACK Paketlerinin İncelenmesi

Hedef sistemin SYN paketlerine verdiği cevapları incelemek için:

```text
tcp.srcport == 8000 && tcp.flags.syn == 1 && tcp.flags.ack == 1 && ip.src == 192.168.56.101
```

filtresi kullanılmıştır.

Bu filtre Kali'nin 8000 numaralı servisi tarafından gönderilen SYN/ACK paketlerini göstermektedir.

SYN/ACK paketleri, hedef sistemin gelen TCP bağlantı isteklerine cevap verdiğini gösterir.

Ancak capture içerisinde birden fazla test ve normal TCP trafiği bulunduğu için gözlemlenen toplam SYN/ACK sayısı doğrudan saldırı sırasında oluşturulan bağlantı sayısı olarak değerlendirilmemiştir.

Bu nedenle analiz sırasında yalnızca tek bir paket sayısına değil, trafik davranışına ve paketlerin zaman içerisindeki dağılımına bakılması gerektiği değerlendirilmiştir.

---

## 9. TCP Paketlerinin Normal ve SYN Flood Davranışı Açısından Karşılaştırılması

| Özellik          | Normal TCP                 | SYN Flood Davranışı                  |
| ---------------- | -------------------------- | ------------------------------------ |
| SYN sayısı       | Düşük                      | Kısa sürede yüksek olabilir          |
| SYN/ACK          | SYN'e cevap olarak oluşur  | Çok sayıda oluşabilir                |
| ACK              | Handshake'i tamamlar       | Bazı bağlantılarda tamamlanmayabilir |
| Trafik yoğunluğu | Düşük/normal               | Ani ve yüksek olabilir               |
| Hedef port       | Belirli servis             | Genellikle belirli bir TCP servisi   |
| Zaman aralığı    | Normal kullanıcı davranışı | Kısa sürede yoğunlaşabilir           |

Burada önemli olan yalnızca SYN paketlerinin sayısını kontrol etmek değildir.

Meşru bir istemci de kısa süre içerisinde çok sayıda bağlantı oluşturabilir. Bu nedenle CyberTrace gibi bir sistemin saldırı tespitinde paket sayısı, zaman aralığı, bağlantıların tamamlanma oranı ve kaynak/hedef bilgilerini birlikte değerlendirmesi daha doğru olacaktır.

---

## 10. CyberTrace İçin Kullanılabilecek Özellikler

TCP SYN Flood tespiti için PCAP verilerinden aşağıdaki özellikler çıkarılabilir:

```text
source_ip
destination_ip
source_port
destination_port
syn_count
syn_ack_count
ack_count
connection_attempt_count
completed_connection_count
incomplete_connection_count
syn_rate
connection_completion_rate
scan_duration
packets_per_second
```

Örneğin:

```text
SYN Rate = SYN Paket Sayısı / Zaman Aralığı
```

ve:

```text
Connection Completion Rate =
Tamamlanan Bağlantılar / Toplam Bağlantı Denemeleri
```

gibi değerler oluşturulabilir.

Bu özellikler ileride makine öğrenmesi modeline giriş verisi olarak kullanılabilir.

---

## 11. Tespit İçin Davranışsal Göstergeler

CyberTrace açısından TCP SYN Flood için değerlendirilebilecek göstergeler:

* Kısa süre içerisinde yüksek SYN yoğunluğu
* Aynı hedef IP ve porta çok sayıda SYN gönderilmesi
* SYN paketlerinin normal kullanıcı davranışından belirgin şekilde fazla olması
* SYN paketleri ile SYN/ACK paketleri arasındaki oranın değişmesi
* Çok sayıda tamamlanmamış TCP bağlantısı
* Aynı kaynak IP'den kısa sürede çok sayıda bağlantı denemesi
* Birden fazla kaynak IP'den aynı hedef porta yoğun SYN trafiği
* Paketlerin zaman içerisinde ani şekilde artması

Tek başına IP adresi saldırı göstergesi olarak kabul edilmemelidir. Meşru istemciler, güvenlik tarayıcıları veya sistem yönetim araçları da çok sayıda TCP bağlantı isteği oluşturabilir.

Bu nedenle davranışsal özelliklerin birlikte değerlendirilmesi daha güvenilir bir yaklaşım sağlar.

---

## 12. CyberTrace Açısından Önemi

CyberTrace'in amacı yalnızca paketleri listelemek değil, ağ trafiğindeki şüpheli davranışları anlamlandırabilmektir.

TCP SYN Flood bu açıdan önemli bir örnektir çünkü saldırı doğrudan belirli bir TCP mekanizmasının davranışından anlaşılabilir.

Örneğin CyberTrace aşağıdaki akışı analiz edebilir:

```text
SYN yoğunluğu
      ↓
Hedef IP/Port analizi
      ↓
SYN/ACK oranı
      ↓
ACK ile tamamlanan bağlantılar
      ↓
Tamamlanmamış bağlantılar
      ↓
Risk değerlendirmesi
      ↓
TCP SYN Flood şüphesi
```

Bu yaklaşım, yalnızca sabit eşiklere bağlı bir sistem yerine trafik davranışını değerlendiren bir saldırı tespit mekanizmasının temelini oluşturabilir.

---

## 13. PCAP Verisi

Bu çalışmada oluşturulan ağ trafiği aşağıdaki PCAP dosyasında saklanmıştır:

```text
tcp_syn_flood.pcapng
```

Dosya, TCP SYN trafiğinin Wireshark üzerinde daha sonra tekrar incelenebilmesi ve CyberTrace veri setinin oluşturulmasında kullanılabilmesi amacıyla kaydedilmiştir.

---

## 14. Kullanılan Araçlar

* Kali Linux
* Windows
* VirtualBox
* Wireshark
* Nping
* Python HTTP Server

---

## 15. Oluşturulan Veri

Bu çalışma sonucunda:

```text
tcp_syn_flood.pcapng
```

PCAP dosyası oluşturulmuştur.

Ayrıca normal TCP bağlantısının incelenmesi için ekran görüntüsü alınmıştır:

```text
screenshots/
├── normal-tcp-handshake.png
└── tcp-syn-test.png
```

---

## 16. Sonuç

Bu çalışmada TCP SYN Flood saldırısının temel çalışma mantığı incelenmiştir.

Öncelikle normal TCP Three-Way Handshake gözlemlenmiş ve SYN, SYN/ACK ve ACK paketlerinin bağlantı kurulmasındaki görevleri incelenmiştir.

Daha sonra izole edilmiş laboratuvar ortamında Nping kullanılarak kontrollü SYN trafiği oluşturulmuş ve Wireshark ile bu trafik analiz edilmiştir.

Çalışma sonucunda TCP SYN Flood tespitinde yalnızca SYN paketlerinin sayısına bakmanın yeterli olmadığı; paket yoğunluğu, zaman aralığı, SYN/ACK davranışı ve bağlantıların tamamlanma durumunun birlikte değerlendirilmesinin daha anlamlı olduğu görülmüştür.

Elde edilen PCAP verisi, CyberTrace projesinde TCP SYN Flood saldırılarını temsil eden veri olarak kullanılabilecek ve ilerleyen aşamalarda normal trafik ile birlikte makine öğrenmesi tabanlı saldırı tespiti çalışmalarına dahil edilebilecektir.

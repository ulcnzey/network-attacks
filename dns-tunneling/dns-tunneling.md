# DNS Tunneling

## 1. Saldırı Hakkında

DNS Tunneling, DNS protokolünün veri taşımak amacıyla kötüye kullanılmasıdır. Normal DNS iletişiminde istemci, bir alan adının IP adresini öğrenmek amacıyla DNS sunucusuna sorgu gönderir.

DNS Tunneling saldırısında ise veri, DNS sorgularındaki alan adı veya alt alan adı bölümlerine kodlanarak taşınabilir. Bu nedenle saldırı trafiği ilk bakışta normal DNS sorgularına benzeyebilir.

Bu çalışmada gerçek bir sisteme yönelik saldırı gerçekleştirilmemiştir. DNS Tunneling davranışı, tamamen izole edilmiş Kali Linux ve Windows sanal makine ortamında kontrollü olarak simüle edilmiştir.

---

## 2. Laboratuvar Ortamı

Saldırı, VirtualBox üzerinde oluşturulan izole ağ ortamında gerçekleştirilmiştir.

| Sistem     | Rol                                | IP Adresi        |
| ---------- | ---------------------------------- | ---------------- |
| Kali Linux | Saldırı trafiğini oluşturan sistem | `192.168.56.101` |
| Windows    | Hedef sistem                       | `192.168.56.1`   |

Ağ:

```text
192.168.56.0/24
```

Kali Linux ve Windows arasında bağlantı kontrolü `ping` komutu kullanılarak doğrulanmıştır.

---

## 3. Saldırının Çalışma Mantığı

DNS Tunneling sırasında aktarılmak istenen veri doğrudan DNS paketinin içerisine normal metin olarak yerleştirilmek yerine kodlanarak DNS sorgusunun alan adı bölümüne eklenebilir.

Bu çalışmada aşağıdaki işlem gerçekleştirilmiştir:

```text
Veri
  ↓
Base32 ile kodlama
  ↓
Küçük parçalara ayırma
  ↓
DNS sorgusu oluşturma
  ↓
Subdomain içerisine yerleştirme
  ↓
UDP/53 üzerinden gönderme
```

Oluşturulan DNS sorguları aşağıdaki yapıya benzer şekilde oluşturulmuştur:

```text
000-<encoded-data>.tunnel.local
001-<encoded-data>.tunnel.local
002-<encoded-data>.tunnel.local
003-<encoded-data>.tunnel.local
```

Buradaki numaralandırma, verinin farklı parçalara ayrılarak gönderildiğini göstermek amacıyla kullanılmıştır.

---

## 4. Kullanılan Protokoller

Çalışmada temel olarak aşağıdaki protokoller kullanılmıştır:

* DNS
* UDP
* IP

DNS sorguları UDP üzerinden **53 numaralı porta** gönderilmiştir.

Temel iletişim:

```text
Kali Linux
192.168.56.101
      |
      | UDP / 53
      | DNS Query
      ↓
Windows
192.168.56.1
```

---

## 5. Saldırı Trafiğinin Oluşturulması

DNS Tunneling davranışını oluşturmak için Kali Linux üzerinde Python ve Scapy kullanılmıştır.

Simülasyonda örnek bir metin veri olarak alınmış, Base32 ile kodlanmış ve küçük parçalara ayrılmıştır.

Her veri parçası DNS sorgusunun subdomain bölümüne yerleştirilmiştir.

Örnek trafik:

```text
000-in4wezlskrzgcy3febce4uzao.tunnel.local
001-r2w43tfnruw4zzanrqwe33smf.tunnel.local
002-2g64tzeb2heylgmzuwgidhmvx.tunnel.local
003-gk4tborswiidjnzzwszdfebqw.tunnel.local
004-4idjonxwyylumvsca5tjoj2hk.tunnel.local
```

Bu sorgular yaklaşık 0,5 saniyelik aralıklarla oluşturulmuştur.

---

## 6. Wireshark Analizi

Oluşturulan trafik Wireshark kullanılarak yakalanmıştır.

DNS paketlerini görüntülemek için aşağıdaki display filter kullanılmıştır:

```text
dns
```

Sadece oluşturulan tunneling sorgularını filtrelemek için:

```text
dns.qry.name contains "tunnel.local"
```

kullanılmıştır.

Yakalanan trafik içerisindeki örnek bir paket:

```text
Source:
192.168.56.101

Destination:
192.168.56.1

Protocol:
DNS

Query:
000-in4wezlskrzgcy3febce4uzao.tunnel.local
```

---

## 7. Wireshark'ta Gözlemlenen Özellikler

Analiz sırasında DNS Tunneling davranışını destekleyen çeşitli trafik özellikleri gözlemlenmiştir.

### 7.1. Uzun DNS Sorguları

Normal DNS sorgularına kıyasla daha uzun alan adları oluşturulmuştur.

Örneğin:

```text
google.com
```

gibi kısa bir alan adı yerine:

```text
000-in4wezlskrzgcy3febce4uzao.tunnel.local
```

gibi daha uzun bir sorgu oluşturulmuştur.

---

### 7.2. Değişen Subdomain Değerleri

Ana domain:

```text
tunnel.local
```

sabit kalırken subdomain bölümü her sorguda değiştirilmiştir.

Örneğin:

```text
000-...............tunnel.local
001-...............tunnel.local
002-...............tunnel.local
003-...............tunnel.local
```

Bu yapı veri parçalarının DNS sorguları üzerinden taşınmasıyla uyumludur.

---

### 7.3. Kodlanmış Veri Görünümü

Subdomain bölümlerinde:

```text
in4wezlskrzgcy3febce4uzao
r2w43tfnruw4zzanrqwe33smf
2g64tzeb2heylgmzuwgidhmvx
```

gibi normal kelimelere benzemeyen karakter dizileri görülmüştür.

Bu tür yüksek çeşitlilik gösteren ve kodlanmış veriye benzeyen subdomainler DNS Tunneling tespitinde değerlendirilebilecek özellikler arasındadır.

---

### 7.4. Sıralı Sorgular

Sorguların başında:

```text
000
001
002
003
...
010
```

şeklinde sıralı parça numaraları bulunmaktadır.

Bu yapı, verinin parçalara bölünerek aktarılması davranışını simüle etmektedir.

---

### 7.5. Düzenli Sorgu Aralıkları

DNS sorguları yaklaşık 0,5 saniyelik aralıklarla oluşturulmuştur.

Bu nedenle zaman çizelgesinde düzenli aralıklarla gerçekleşen DNS sorguları gözlemlenmiştir.

---

## 8. Tespit Açısından Önemli Özellikler

DNS Tunneling'in tespit edilmesinde tek bir özelliğe güvenmek yerine birden fazla trafik özelliğinin birlikte değerlendirilmesi gerekir.

Bu çalışma kapsamında CyberTrace'ın ilerleyen aşamalarında kullanılabilecek özellikler:

| Özellik                              | Gözlem          |
| ------------------------------------ | --------------- |
| DNS paket sayısı                     | Artabilir       |
| Benzersiz subdomain sayısı           | Yüksek          |
| Ortalama domain uzunluğu             | Yüksek          |
| Maksimum domain uzunluğu             | Yüksek          |
| Subdomain uzunluğu                   | Yüksek          |
| Subdomain entropy                    | Yüksek olabilir |
| Sorgu sıklığı                        | Artabilir       |
| Aynı domain altında farklı subdomain | Belirgin        |
| Kodlanmış karakter paterni           | Belirgin        |
| UDP/53 trafiği                       | Mevcut          |

Bu özellikler ilerleyen aşamada makine öğrenmesi tabanlı saldırı tespit modelinde kullanılabilecek aday özellikler olarak değerlendirilecektir.

---

## 9. PCAP Dosyası

DNS Tunneling sırasında oluşturulan ağ trafiği Wireshark kullanılarak kaydedilmiştir.

Dosya:

```text
dns_tunneling.pcapng
```

PCAP dosyası, saldırı trafiğinin daha sonra tekrar incelenebilmesi ve CyberTrace veri setine dahil edilebilmesi amacıyla saklanmıştır.

---

## 10. Ekran Görüntüleri

### DNS Tunneling Trafiği

> Buraya Wireshark'ta DNS Tunneling paketlerinin göründüğü ekran görüntüsü eklenecektir.

Örnek:

```text
screenshots/dns-tunneling-traffic.png
```

### DNS Query Detayları

> Buraya seçilen DNS paketinin Packet Details bölümünü gösteren ekran görüntüsü eklenecektir.

Örnek:

```text
screenshots/dns-tunneling-query-details.png
```

---

## 11. Sonuç

Bu çalışmada izole sanal makine ortamında kontrollü bir DNS Tunneling trafik simülasyonu gerçekleştirilmiştir.

Kali Linux üzerinden oluşturulan DNS sorgularında veri parçaları kodlanarak subdomain bölümlerine yerleştirilmiş ve UDP/53 üzerinden DNS sorguları oluşturulmuştur.

Wireshark analizi sonucunda;

* uzun DNS sorguları,
* sürekli değişen subdomainler,
* kodlanmış karakter dizileri,
* sıralı veri parçaları,
* düzenli DNS sorgu trafiği

gibi DNS Tunneling davranışını destekleyen özellikler gözlemlenmiştir.

Oluşturulan `dns_tunneling.pcapng` dosyası, CyberTrace projesinde kullanılacak özel ağ saldırısı veri setinin bir parçası olarak saklanacaktır.

Bu trafik ilerleyen aşamada normal ağ trafiğiyle birlikte değerlendirilerek makine öğrenmesi tabanlı saldırı tespit modelinin geliştirilmesinde kullanılacaktır.

---

## 12. Kullanılan Araçlar

* Kali Linux
* Windows
* VirtualBox
* Wireshark
* Python
* Scapy

---

## 13. Çalışma Ortamı

```text
Attacker:
Kali Linux
192.168.56.101

Target:
Windows
192.168.56.1

Protocol:
DNS / UDP

Destination Port:
53

Network:
192.168.56.0/24
```

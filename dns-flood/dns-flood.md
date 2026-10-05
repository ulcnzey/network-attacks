# DNS Flood

## 1. Amaç

Bu çalışmanın amacı, kontrollü ve izole bir laboratuvar ortamında DNS Flood saldırısının gerçekleştirilmesi, ağ trafiğinin Wireshark ile yakalanması ve saldırı trafiğinin normal DNS trafiğinden ayırt edilebilecek özelliklerinin incelenmesidir.

Elde edilen PCAP dosyası, CyberTrace projesinde saldırı trafiğinin analiz edilmesi ve ilerleyen aşamalarda makine öğrenmesi tabanlı saldırı tespit sisteminin veri setinin oluşturulması amacıyla kullanılacaktır.

---

## 2. DNS Nedir?

DNS (Domain Name System), alan adlarını IP adreslerine dönüştüren bir sistemdir.

Örneğin:

```text
lab.test → 192.168.56.101
```

Bir istemci bir alan adının IP adresini öğrenmek istediğinde DNS sunucusuna sorgu gönderir. DNS sunucusu uygun bir kayıt varsa istemciye IP adresini içeren bir yanıt gönderir.

Normal DNS iletişiminde temel olarak:

```text
İstemci → DNS Sunucusu
DNS Sorgusu

DNS Sunucusu → İstemci
DNS Yanıtı
```

şeklinde bir iletişim gerçekleşir.

---

## 3. DNS Flood Nedir?

DNS Flood, DNS sunucusuna kısa bir zaman aralığında çok sayıda DNS sorgusu gönderilmesine dayanan bir saldırı türüdür.

Amaç, DNS sunucusunun yoğun miktarda sorguyu işlemek zorunda kalmasına neden olmaktır.

Normal bir DNS iletişiminde sorgu trafiği sınırlıyken DNS Flood sırasında:

* Çok sayıda DNS sorgusu oluşur.
* Sorgular kısa bir zaman aralığında yoğunlaşır.
* Paket/saniye veya sorgu/saniye oranı belirgin şekilde yükselir.
* DNS sunucusunun işlem yükü artırılabilir.

Bu çalışmada saldırı yalnızca izole laboratuvar ağı içerisinde gerçekleştirilmiştir.

---

## 4. Laboratuvar Ortamı

Çalışmada aşağıdaki ağ yapısı kullanılmıştır:

| Cihaz      | IP Adresi      | Rol                             |
| ---------- | -------------- | ------------------------------- |
| Windows    | 192.168.56.1   | DNS istemcisi / saldırı kaynağı |
| Kali Linux | 192.168.56.101 | DNS sunucusu / hedef            |

Ağ:

```text
VirtualBox Host-Only Network
        |
        +--- Windows
        |    192.168.56.1
        |
        +--- Kali Linux
             192.168.56.101
             dnsmasq
```

Kali Linux üzerinde `dnsmasq` kullanılarak laboratuvar DNS sunucusu oluşturulmuştur.

---

## 5. Normal DNS Trafiğinin İncelenmesi

Saldırı gerçekleştirilmeden önce normal DNS iletişimi test edilmiştir.

Windows üzerinden Kali DNS sunucusuna:

```text
192.168.56.1 → 192.168.56.101
```

yönünde DNS sorgusu gönderilmiştir.

Örnek alan adı:

```text
lab.test
```

DNS sunucusu tarafından:

```text
lab.test → 192.168.56.101
```

şeklinde yanıt verilmiştir.

Wireshark üzerinde normal DNS trafiğinde sorgu ve yanıt paketlerinin karşılıklı olarak oluştuğu gözlemlenmiştir.

---

## 6. DNS Flood Saldırısının Gerçekleştirilmesi

DNS Flood testi sırasında Windows sistemi üzerinden Kali Linux üzerindeki DNS sunucusuna çok sayıda DNS sorgusu gönderilmiştir.

Hedef:

```text
192.168.56.101:53
```

Sorgularda birbirinden farklı alan adları kullanılmıştır:

```text
flood1.lab.test
flood2.lab.test
flood3.lab.test
...
flood200.lab.test
```

Toplam:

```text
200 DNS sorgusu
```

gönderilmiştir.

Her sorgu için DNS sunucusundan yanıt alınmıştır.

---

## 7. Wireshark Analizi

Yakalanan trafik Wireshark üzerinde incelenmiştir.

DNS trafiğini filtrelemek için:

```text
dns
```

filtresi kullanılabilir.

Saldırı trafiğini kaynak ve hedef IP adreslerine göre incelemek için:

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && dns
```

filtresi kullanılabilir.

Belirli sorguları incelemek için:

```text
dns.qry.name
```

alanı kullanılabilir.

---

## 8. Elde Edilen Sonuçlar

Yakalanan PCAP verisinin incelenmesi sonucunda saldırı sırasında:

* **200 DNS sorgusu**
* **200 DNS yanıtı**
* **Toplam 400 DNS paketi**

tespit edilmiştir.

İlk saldırı sorgusu:

```text
flood1.lab.test
```

Son saldırı sorgusu:

```text
flood200.lab.test
```

şeklindedir.

İlk DNS sorgusu ile son DNS sorgusu arasındaki süre yaklaşık:

```text
0.296160 saniye
```

olarak ölçülmüştür.

Buna göre yaklaşık DNS sorgu hızı:

```text
675.31 sorgu/saniye
```

olarak hesaplanmıştır.

Sorgu ve yanıtların tamamı birlikte değerlendirildiğinde toplam 400 DNS paketinin yaklaşık:

```text
1349.79 paket/saniye
```

oranında gerçekleştiği görülmüştür.

---

## 9. DNS Flood Trafiğinin Belirgin Özellikleri

Wireshark analizi sonucunda DNS Flood trafiğinde aşağıdaki özellikler gözlemlenmiştir:

1. Çok kısa bir zaman aralığında yüksek sayıda DNS sorgusu oluşmuştur.
2. DNS sorgularında farklı alan adları kullanılmıştır.
3. `flood1.lab.test` ile `flood200.lab.test` arasında ardışık sorgular görülmüştür.
4. Sorguların tamamı aynı DNS sunucusuna yönlendirilmiştir.
5. Sorgulara karşılık çok sayıda DNS yanıtı oluşmuştur.
6. Normal DNS trafiğine kıyasla sorgu yoğunluğu belirgin şekilde artmıştır.

Bu özellikler, ilerleyen aşamada CyberTrace içerisinde DNS tabanlı anormal trafik tespitinde kullanılabilecek trafik göstergeleri olarak değerlendirilebilir.

---

## 10. PCAP Dosyası

Yakalanan saldırı trafiği:

```text
dns_flood.pcapng
```

dosyasında saklanmıştır.

Bu dosya, oluşturulan saldırı veri setinin bir parçası olarak kullanılacaktır.

---

## 11. Kullanılan Araçlar

* Kali Linux
* Windows
* VirtualBox
* dnsmasq
* Wireshark
* PowerShell
* UDP/DNS trafik oluşturma aracı

---

## 12. Sonuç

Bu çalışmada izole bir VirtualBox Host-Only ağında DNS Flood saldırısı kontrollü şekilde gerçekleştirilmiştir.

Wireshark ile yapılan incelemede 200 DNS sorgusu ve bunlara karşılık gelen 200 DNS yanıtı olmak üzere toplam 400 DNS paketi tespit edilmiştir.

Yaklaşık 0.296 saniyelik süre içerisinde 200 DNS sorgusunun gönderilmesi sonucunda yaklaşık 675 sorgu/saniye seviyesinde bir DNS trafik yoğunluğu oluşturulmuştur.

Elde edilen PCAP dosyası, CyberTrace projesinde saldırı trafiğinin incelenmesi, özellik çıkarımı ve makine öğrenmesi tabanlı saldırı tespit çalışmalarında kullanılmak üzere veri setine dahil edilecektir.

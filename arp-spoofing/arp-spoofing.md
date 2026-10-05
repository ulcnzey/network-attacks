# ARP Spoofing / ARP Poisoning

## 1. Saldırının Amacı

ARP Spoofing (ARP Poisoning), yerel ağdaki cihazların ARP tablolarına sahte IP-MAC eşleşmeleri gönderilmesiyle gerçekleştirilen bir ağ saldırısıdır.

Bu çalışmada amaç, ARP protokolünün normal çalışma şeklini gözlemlemek, sahte ARP cevapları üretmek, bu paketleri Wireshark üzerinden incelemek ve saldırının ağ üzerindeki etkisini gözlemlemektir.

Çalışma yalnızca izole edilmiş VirtualBox Host-Only laboratuvar ortamında gerçekleştirilmiştir.

---

## 2. Laboratuvar Ortamı

| Cihaz      | IP Adresi        | MAC Adresi          |
| ---------- | ---------------- | ------------------- |
| Windows    | `192.168.56.1`   | `0a:00:27:00:00:17` |
| Kali Linux | `192.168.56.101` | `08:00:27:7d:bb:61` |
| Diğer VM   | `192.168.56.100` | `08:00:27:11:ef:4c` |

Ağ:

```text
192.168.56.0/24
```

Saldırı Kali Linux üzerinden gerçekleştirilmiştir.

<img width="378" height="177" alt="image" src="https://github.com/user-attachments/assets/26b3eb97-bfdb-4551-9e1d-045452ac51c0" />


---

## 3. ARP Protokolünün Normal Çalışması

ARP (Address Resolution Protocol), yerel ağlarda IP adreslerinin MAC adresleriyle eşleştirilmesini sağlar.

Saldırı öncesinde `192.168.56.100` cihazının gerçek MAC adresi:

```text
192.168.56.100 → 08:00:27:11:ef:4c
```

şeklindeydi.

Wireshark üzerinde normal ARP iletişimi gözlemlendi.

Örnek ARP isteği:

```text
Who has 192.168.56.100? Tell 192.168.56.101
```

Buna karşılık cihaz:

```text
192.168.56.100 is at 08:00:27:11:ef:4c
```

şeklinde cevap verdi.

Bu paketler `normal-arp-request-reply.png` ekran görüntüsünde gösterilmiştir.

<img width="1366" height="478" alt="image" src="https://github.com/user-attachments/assets/43c5b25c-cfcf-41d5-8a1e-be232bfa69a9" />

---

## 4. ARP Spoofing Saldırısının Gerçekleştirilmesi

Saldırıda Kali Linux'un `eth1` arayüzü kullanılmıştır.

Kullanılan komut:

```bash
sudo arpspoof -i eth1 -t 192.168.56.1 192.168.56.100
```

Komutun anlamı:

* `-i eth1`: Saldırının gerçekleştirileceği ağ arayüzü.
* `-t 192.168.56.1`: Hedef cihaz olan Windows.
* `192.168.56.100`: Taklit edilen IP adresi.

Bu işlem sonucunda Kali, Windows'a `192.168.56.100` adresinin kendi MAC adresine ait olduğunu belirten sahte ARP cevapları göndermiştir.

---

## 5. Wireshark Analizi

Saldırı sırasında Wireshark üzerinde ARP paketleri filtrelenmiştir:

```text
arp
```

Yakalanan önemli paketlerden biri:

```text
192.168.56.100 is at 08:00:27:7d:bb:61
```

Buradaki önemli nokta, `08:00:27:7d:bb:61` MAC adresinin Kali Linux'a ait olmasıdır.

Normal durumda:

```text
192.168.56.100 → 08:00:27:11:ef:4c
```

olması gerekirken saldırı sırasında:

```text
192.168.56.100 → 08:00:27:7d:bb:61
```

eşleşmesi oluşturulmuştur.

Bu durum ARP Poisoning saldırısının temel göstergesidir.

`arp-spoof-reply.png` ekran görüntüsünde saldırıya ait ARP cevabı gösterilmiştir.

---

## 6. Windows ARP Tablosunun Değişmesi

Saldırı öncesinde Windows'un ARP tablosunda:

```text
192.168.56.100 → 08-00-27-11-ef-4c
```

eşleşmesi bulunmaktaydı.

Saldırı sırasında ise:

```text
192.168.56.100 → 08-00-27-7d-bb-61
```

eşleşmesi görülmüştür.

Buradaki `08-00-27-7d-bb-61`, Kali Linux'un MAC adresidir.

Bu değişiklik, Windows'un `.100` IP adresinin gerçek cihaz yerine Kali'ye ait olduğunu düşünmesine neden olmuştur.

`arp-poisoned-table.png` ekran görüntüsü bu durumu göstermektedir.

<img width="382" height="184" alt="image" src="https://github.com/user-attachments/assets/493b18fd-4b8e-456b-af36-76620e80e0f1" />


---

## 7. Saldırının Wireshark Üzerinden Tespit Edilmesi

ARP Spoofing tespit edilirken IP-MAC eşleşmelerinin tutarlılığı incelenebilir.

Örneğin:

```text
192.168.56.100 → 08:00:27:11:ef:4c
```

normal eşleşmedir.

Aynı IP adresi için daha sonra:

```text
192.168.56.100 → 08:00:27:7d:bb:61
```

gibi farklı bir MAC adresinin görülmesi şüpheli bir durum oluşturur.

Özellikle aynı IP adresinin farklı MAC adresleriyle ilişkilendirilmesi ARP spoofing açısından önemli bir davranışsal göstergedir.

<img width="1340" height="538" alt="image" src="https://github.com/user-attachments/assets/b71f66ca-dd12-4d26-91dc-40d030ecac6d" />

---

## 8. Saldırı Göstergeleri

Bu çalışmada gözlemlenen göstergeler:

* Aynı IP adresi için farklı MAC adreslerinin görülmesi
* Gerçek cihazın MAC adresinin değişmiş gibi görünmesi
* Tekrarlanan sahte ARP Reply paketleri
* ARP Reply paketlerinin normal bir ARP Request olmadan gönderilmesi
* Saldırganın MAC adresinin başka bir IP ile ilişkilendirilmesi

---

## 9. CyberTrace İçin Kullanılabilecek Özellikler

Bu saldırının daha sonra CyberTrace içerisinde makine öğrenmesi ve davranış analizi için kullanılabilecek bazı özellikleri şunlardır:

```text
source_ip
source_mac
target_ip
target_mac
arp_reply_count
unique_mac_per_ip
ip_mac_change_count
arp_reply_frequency
```

Özellikle `unique_mac_per_ip` ve `ip_mac_change_count` değerleri ARP spoofing davranışının tespit edilmesinde kullanılabilir.

---

## 10. Saldırının Sonlandırılması

Saldırı tamamlandıktan sonra `arpspoof` işlemi durdurulmuştur.

Windows üzerindeki ARP kaydı temizlenmiş ve hedef cihaza tekrar ping gönderilerek doğru IP-MAC eşleşmesinin oluşması sağlanmıştır.

Beklenen normal eşleşme:

```text
192.168.56.100 → 08-00-27-11-ef-4c
```

şeklindedir.

---

## 11. Kullanılan Araçlar

* Kali Linux
* VirtualBox
* Wireshark
* arpspoof
* Windows PowerShell
* ARP

---

## 12. Oluşturulan Veri

Bu çalışma sonucunda:

```text
arp_spoofing.pcapng
```

isimli ağ trafiği kaydı oluşturulmuştur.

PCAP dosyası saldırı sırasında üretilen ARP paketlerinin analiz edilmesi ve ileride CyberTrace veri setinin oluşturulması amacıyla kullanılacaktır.

---

## 13. Sonuç

Bu çalışmada ARP protokolünün normal çalışma şekli incelenmiş ve ardından izole bir laboratuvar ortamında ARP Spoofing / ARP Poisoning saldırısı gerçekleştirilmiştir.

Saldırı sırasında Kali Linux, `192.168.56.100` IP adresini kendi MAC adresiyle ilişkilendiren sahte ARP cevapları göndermiştir.

Wireshark üzerinde:

```text
192.168.56.100 is at 08:00:27:7d:bb:61
```

paketi gözlemlenmiş ve Windows'un ARP tablosunda `.100` adresinin Kali'nin MAC adresiyle eşleştiği görülmüştür.

Bu çalışma sonucunda ARP Spoofing saldırısının hem paket seviyesinde hem de hedef cihazın ARP tablosundaki değişiklik üzerinden tespit edilebileceği görülmüştür.

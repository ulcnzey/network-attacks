# ICMP Flood

## 1. Saldırının Amacı

ICMP Flood, hedef sisteme yüksek miktarda ICMP Echo Request paketi gönderilerek ağ ve hedef sistem üzerindeki kaynakların yoğun şekilde kullanılmasına neden olmayı amaçlayan bir DoS (Denial of Service) saldırısıdır.

ICMP protokolü normalde ağ üzerindeki cihazların erişilebilirliğini kontrol etmek için kullanılır. Örneğin `ping` komutu ICMP Echo Request gönderir ve hedef sistem ICMP Echo Reply ile cevap verir.

Normal kullanımda ICMP trafiği düşük miktardadır. ICMP Flood saldırısında ise kısa süre içerisinde çok sayıda Echo Request gönderilir.

Bu çalışmanın amacı, normal ICMP trafiği ile ICMP Flood trafiği arasındaki farkı Wireshark üzerinden gözlemlemek ve CyberTrace projesinde kullanılabilecek ağ trafiği özelliklerini belirlemektir.

---

## 2. ICMP Nedir?

ICMP (Internet Control Message Protocol), IP ağlarında hata bildirimleri ve ağ durumunun kontrol edilmesi amacıyla kullanılan bir protokoldür.

`ping` işlemi ICMP Echo Request ve Echo Reply mesajlarını kullanır.

Normal iletişim:

```text
Windows                         Kali
192.168.56.1                    192.168.56.101
     |                                |
     | ---- Echo Request ----------> |
     |                                |
     | <---- Echo Reply ------------ |
     |                                |
```

ICMP Echo Request:

```text
Type: 8
Code: 0
```

ICMP Echo Reply:

```text
Type: 0
Code: 0
```

---

## 3. ICMP Flood Nedir?

ICMP Flood saldırısında hedef sisteme çok sayıda ICMP Echo Request gönderilir.

Normal ping trafiğinde paket sayısı düşük ve paketler daha seyrek gelirken, ICMP Flood sırasında çok sayıda Echo Request kısa bir zaman aralığında hedefe ulaşır.

Basit şekilde:

```text
Normal ICMP:

Request
   ↓
Reply

Request
   ↓
Reply
```

ICMP Flood:

```text
Request Request Request Request Request
Request Request Request Request Request
Request Request Request Request Request
                    ↓
                   Kali
```

Bu nedenle ICMP Flood'un tespitinde yalnızca ICMP kullanılması değil, ICMP trafiğinin yoğunluğu ve paketlerin geliş sıklığı da önemlidir.

---

## 4. Laboratuvar Ortamı

Çalışma izole VirtualBox Host-Only Network ortamında gerçekleştirilmiştir.

| Sistem     | IP Adresi      | Rol          |
| ---------- | -------------- | ------------ |
| Windows    | 192.168.56.1   | Test kaynağı |
| Kali Linux | 192.168.56.101 | Hedef        |

Ağ yapısı:

```text
Windows
192.168.56.1
      |
      | Host-Only Network
      |
      v
Kali Linux
192.168.56.101
```

Çalışma yalnızca kontrol edilen laboratuvar ortamında gerçekleştirilmiştir.

---

## 5. Normal ICMP Trafiğinin Gözlemlenmesi

Saldırı gerçekleştirilmeden önce normal ICMP trafiği incelenmiştir.

Windows üzerinden Kali sistemine ping gönderilmiştir:

```powershell
ping 192.168.56.101
```

Wireshark üzerinde kullanılan filtre:

```text
icmp
```

Normal iletişim sırasında:

```text
192.168.56.1 → 192.168.56.101
ICMP Echo Request
```

ve:

```text
192.168.56.101 → 192.168.56.1
ICMP Echo Reply
```

paketleri gözlemlenmiştir.

Bu normal trafik daha sonra ICMP Flood trafiğiyle karşılaştırılmıştır.

### Ekran Görüntüsü

<img width="1363" height="334" alt="image" src="https://github.com/user-attachments/assets/220e196f-10a3-4019-8211-df08190e55b0" />


---

## 6. ICMP Flood Testi

Kontrollü laboratuvar ortamında Windows sisteminden Kali sistemine yüksek sayıda ICMP Echo Request gönderilmiştir.

Kullanılan test:

```powershell
1..500 | ForEach-Object { Test-Connection -ComputerName 192.168.56.101 -Count 1 -Quiet }
```

Bu test ile Kali sistemine çok sayıda ICMP Echo Request gönderilmiştir.

<img width="816" height="441" alt="image" src="https://github.com/user-attachments/assets/181cf7ef-9956-4575-a5e5-b1d6b8236479" />


Wireshark üzerinde trafik:

```text
icmp
```

filtresi kullanılarak izlenmiştir.

Daha sonra yalnızca ICMP Echo Request paketlerini incelemek için:

```text
icmp.type == 8
```

filtresi kullanılmıştır.

---

## 7. Wireshark Analizi

ICMP Flood testi sonrasında Wireshark üzerinde yüzlerce ICMP Echo Request paketi gözlemlenmiştir.

`icmp.type == 8` filtresi kullanıldığında capture içerisinde:

```text
504 Echo Request
```

paketi görüntülenmiştir.

Burada önemli olan nokta, bu değerin capture içerisindeki filtrelenmiş Echo Request paketlerini ifade etmesidir. Capture'ın saldırı öncesindeki normal ICMP paketlerini de içermesi nedeniyle bu sayı doğrudan yalnızca saldırı sırasında gönderilen paket sayısı olarak değerlendirilmemelidir.

### Ekran Görüntüsü

<img width="1361" height="625" alt="image" src="https://github.com/user-attachments/assets/b1008f8b-ce8f-4dee-8826-a8fa16db7a31" />


---

## 8. ICMP Paket Detayı

Örnek olarak incelenen bir ICMP Echo Reply paketinde aşağıdaki bilgiler görülmüştür:

```text
Internet Protocol Version 4
    Src: 192.168.56.101
    Dst: 192.168.56.1

Internet Control Message Protocol
    Type: 0 (Echo (ping) reply)
    Code: 0
    Sequence Number: 389
    Request frame: 684
    Response time: 0.026 ms
```

Bu paket Kali sisteminden Windows sistemine gönderilen bir ICMP Echo Reply paketidir.

`Request frame: 684` bilgisi, bu cevabın Wireshark'ta 684 numaralı frame içerisindeki Echo Request'e karşılık geldiğini göstermektedir.

`Response time: 0.026 ms` değeri ise istek ile cevap arasındaki ölçülen yanıt süresini göstermektedir.

### Ekran Görüntüsü

<img width="576" height="490" alt="image" src="https://github.com/user-attachments/assets/2b66124c-3d50-4895-b7c0-9ded5fbff98a" />


---

## 9. Normal ICMP ile ICMP Flood Karşılaştırması

| Özellik             | Normal Ping    | ICMP Flood                  |
| ------------------- | -------------- | --------------------------- |
| ICMP kullanımı      | Evet           | Evet                        |
| Echo Request        | Düşük sayıda   | Çok yüksek sayıda           |
| Paket geliş sıklığı | Daha düşük     | Çok yüksek                  |
| Kaynak IP           | 192.168.56.1   | 192.168.56.1                |
| Hedef IP            | 192.168.56.101 | 192.168.56.101              |
| Davranış            | Normal         | Şüpheli / saldırı davranışı |

Burada önemli olan yalnızca ICMP paketinin bulunması değildir.

Örneğin tek bir:

```text
ICMP Echo Request
```

paketi saldırı olarak değerlendirilemez.

Bunun yerine paket sayısı, zaman aralığı ve kaynak-hedef ilişkisi birlikte değerlendirilmelidir.

---

## 10. CyberTrace İçin Kullanılabilecek Özellikler

ICMP Flood tespitinde kullanılabilecek bazı ağ trafiği özellikleri:

```text
source_ip
destination_ip
icmp_request_count
icmp_reply_count
icmp_request_rate
icmp_reply_ratio
duration
```

Örneğin:

```text
source_ip = 192.168.56.1
destination_ip = 192.168.56.101
icmp_request_count = yüksek
icmp_request_rate = yüksek
```

şeklindeki bir davranış şüpheli olarak değerlendirilebilir.

Özellikle `icmp_request_rate` özelliği, belirli bir zaman aralığında kaç ICMP Echo Request gönderildiğini ifade eder.

Bu özellik ileride makine öğrenmesi modelinde kullanılabilecek bir özellik olabilir.

---

## 11. Tespit İçin Davranışsal Göstergeler

ICMP Flood tespitinde değerlendirilebilecek göstergeler:

* Kısa süre içerisinde çok sayıda ICMP Echo Request
* Aynı kaynak IP'den sürekli ICMP trafiği
* Aynı hedef IP'ye yoğun ICMP trafiği
* ICMP Request oranında ani artış
* Normal trafik seviyesinden belirgin şekilde yüksek ICMP yoğunluğu
* Request/Reply trafiğinde olağan dışı artış

Tek başına kaynak IP adresi saldırı göstergesi olarak değerlendirilmemelidir.

Davranışın zaman içerisindeki değişimi daha anlamlıdır.

---

## 12. CyberTrace Açısından Önemi

CyberTrace projesinin amacı ağ trafiğini analiz ederek şüpheli davranışları tespit etmektir.

ICMP Flood, bu amaç için önemli bir örnektir çünkü saldırının temel göstergesi belirli bir paket türünün olağan dışı yoğunluğudur.

CyberTrace aşağıdaki süreci uygulayabilir:

```text
PCAP
 ↓
Paketleri oku
 ↓
ICMP paketlerini belirle
 ↓
Echo Request paketlerini say
 ↓
Zaman aralığını hesapla
 ↓
ICMP Request Rate hesapla
 ↓
Normal trafik ile karşılaştır
 ↓
Risk değerlendirmesi
 ↓
ICMP Flood Alert
```

Bu yaklaşım ileride makine öğrenmesi tabanlı saldırı tespit sisteminde de kullanılabilir.

---

## 13. PCAP Verisi

Bu çalışma sonucunda ICMP Flood trafiğini içeren bir PCAP/PCAPNG dosyası oluşturulmuştur.

Dosya:

```text
icmp_flood.pcapng
```

PCAP dosyası daha sonra CyberTrace veri setinin oluşturulması ve saldırı trafiğinin analiz edilmesi amacıyla kullanılabilir.

---

## 14. Kullanılan Araçlar

* Kali Linux
* Windows
* VirtualBox
* Wireshark
* PowerShell
* ICMP / Ping

---

## 15. Sonuç

Bu çalışmada ICMP Flood saldırısının temel çalışma mantığı incelenmiş ve izole bir laboratuvar ortamında kontrollü olarak uygulanmıştır.

Öncelikle normal ICMP ping trafiği gözlemlenmiş, ardından yüksek sayıda ICMP Echo Request gönderilerek trafik yoğunluğu artırılmıştır.

Wireshark üzerinde `icmp.type == 8` filtresi kullanılarak yüzlerce Echo Request paketi gözlemlenmiştir.

Çalışma sonucunda ICMP Flood tespitinde yalnızca ICMP paketlerinin varlığının yeterli olmadığı, paket sayısı, paket geliş sıklığı, kaynak ve hedef bilgileri gibi davranışsal özelliklerin birlikte değerlendirilmesi gerektiği görülmüştür.

Bu çalışma sonucunda elde edilen PCAP verisi, CyberTrace projesinde ICMP Flood saldırılarının tespiti ve makine öğrenmesi veri setinin oluşturulması için kullanılacaktır.

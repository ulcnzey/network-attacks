# SSH Brute Force Saldırısı

## 1. Genel Bakış

SSH Brute Force, bir SSH servisine art arda giriş yapmayı deneyerek kullanıcı hesabının parolasını tahmin etmeye yönelik bir saldırı türüdür.

Bu çalışmada, kontrollü ve izole bir VirtualBox laboratuvar ortamında SSH Brute Force saldırı trafiği oluşturulmuş ve oluşan ağ trafiği Wireshark kullanılarak yakalanmıştır.

Elde edilen PCAP dosyası, CyberTrace Network Intrusion Detection System (NIDS) projesinde saldırı verisi olarak kullanılmak üzere hazırlanmıştır.

> **Not:** Bu çalışma yalnızca tarafıma ait ve izole edilmiş laboratuvar ortamında gerçekleştirilmiştir.

---

## 2. Saldırının Çalışma Mantığı

SSH (Secure Shell), Linux sistemlerine uzaktan güvenli şekilde bağlanmak için kullanılan bir protokoldür.

SSH Brute Force saldırısında saldırgan, SSH servisine çok sayıda bağlantı kurarak farklı parolalarla tekrar tekrar kimlik doğrulama denemesi gerçekleştirir.

Bu çalışmadaki temel iletişim yapısı:

```text
Windows Host
192.168.56.1
      |
      | TCP / 22
      v
Kali Linux
192.168.56.101
      |
      v
SSH Servisi
```

Laboratuvar çalışmasında kullanılan parola bilerek yanlış seçilmiş ve amaç sisteme giriş yapmak yerine **tekrarlanan başarısız SSH kimlik doğrulama trafiği oluşturmak** olmuştur.

---

## 3. Laboratuvar Ortamı

| Bileşen                         | Yapılandırma      |
| ------------------------------- | ----------------- |
| Saldırıyı gerçekleştiren sistem | Windows Host      |
| Kaynak IP                       | 192.168.56.1      |
| Hedef sistem                    | Kali Linux        |
| Hedef IP                        | 192.168.56.101    |
| Protokol                        | SSH               |
| Taşıma protokolü                | TCP               |
| Hedef port                      | 22                |
| Wireshark arayüzü               | eth1              |
| Sanallaştırma                   | VirtualBox        |
| Ağ türü                         | Host-Only Network |

Çalışma, yalnızca laboratuvar makinelerinin birbirleriyle iletişim kurduğu VirtualBox Host-Only ağı üzerinde gerçekleştirilmiştir.

---

## 4. SSH Servisinin Hazırlanması

Öncelikle Kali Linux üzerindeki SSH servisinin çalışıp çalışmadığı kontrol edilmiştir:

```bash
sudo systemctl status ssh
```

SSH servisinin 22 numaralı portu dinlediği aşağıdaki komut ile kontrol edilmiştir:

```bash
ss -lntp | grep :22
```

SSH servisi:

```text
TCP/22
```

üzerinden bağlantıları kabul etmektedir.

---

## 5. Laboratuvar Kullanıcısının Oluşturulması

Deney için ayrı bir kullanıcı hesabı oluşturulmuştur:

```bash
sudo useradd -m sshlab
```

Kullanıcıya parola atanmıştır:

```bash
sudo passwd sshlab
```

`sshlab` hesabı yalnızca bu laboratuvar çalışmasında kullanılmıştır.

---

## 6. Normal SSH Bağlantısının Test Edilmesi

Saldırı trafiği oluşturulmadan önce SSH servisinin düzgün çalıştığını doğrulamak amacıyla Windows sisteminden normal bir SSH bağlantısı kurulmuştur.

Windows PowerShell üzerinde:

```powershell
ssh sshlab@192.168.56.101
```

komutu kullanılmıştır.

İlk bağlantıda SSH sunucusunun anahtarının doğrulanması istenmiştir.

Bağlantının güvenilir olduğu doğrulandıktan sonra:

```text
yes
```

seçilmiştir.

Daha sonra `sshlab` kullanıcısının parolası girilerek Kali Linux sistemine başarıyla bağlanılmıştır.

Bağlantı testi tamamlandıktan sonra:

```bash
exit
```

komutu ile SSH oturumundan çıkılmıştır.

Bu adım, saldırı trafiği oluşturulmadan önce SSH servisinin ve kullanıcı hesabının düzgün çalıştığını doğrulamak açısından önemlidir.

---

## 7. Wireshark ile Trafik Yakalama

Kali Linux üzerinde Wireshark açılmış ve trafik yakalama arayüzü olarak:

```text
eth1
```

seçilmiştir.

Yakalama sırasında herhangi bir Capture Filter kullanılmamıştır.

Bunun amacı, oluşan trafiğin tamamını yakalayarak daha sonra farklı Wireshark filtreleriyle incelemektir.

SSH trafiğini görüntülemek için:

```text
tcp.port == 22
```

filtresi kullanılabilir.

SSH protokolünü doğrudan filtrelemek için:

```text
ssh
```

filtresi kullanılabilir.

Sadece Windows ile Kali arasındaki SSH trafiğini görmek için:

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && tcp.port == 22
```

filtresi kullanılabilir.

---

## 8. Kontrollü SSH Brute Force Saldırısı

Saldırı trafiğini oluşturmak amacıyla Windows sisteminden sınırlı sayıda başarısız SSH giriş denemesi gerçekleştirilmiştir.

Deney **20 başarısız giriş denemesi** ile sınırlandırılmıştır.

Kullanılan PowerShell komutu:

```powershell
1..20 | ForEach-Object {
    Write-Host "SSH Attempt $_"
    "WrongPassword123!" | ssh -o StrictHostKeyChecking=no -o PreferredAuthentications=password -o PubkeyAuthentication=no sshlab@192.168.56.101
}
```

Burada kullanılan:

```text
WrongPassword123!
```

parolası bilerek yanlış seçilmiştir.

Dolayısıyla bu çalışmanın amacı sisteme erişim sağlamak değil, **başarısız SSH kimlik doğrulama denemelerinden oluşan saldırı trafiğini üretmektir.**

---

## 9. Saldırı Trafiğinin Oluşturulması

Her denemede Windows sistemi ile Kali Linux üzerindeki SSH servisi arasında TCP bağlantısı oluşturulmuştur.

İletişim:

```text
Kaynak:
192.168.56.1

        ↓

Hedef:
192.168.56.101:22
```

şeklindedir.

Genel bağlantı yapısı:

```text
192.168.56.1
     |
     | TCP bağlantısı
     v
192.168.56.101:22
     |
     | SSH bağlantısı
     v
Kimlik doğrulama denemesi
     |
     | Başarısız
     v
Bağlantı sonlandırılır
```

Bu işlem 20 kez tekrarlandığında aynı kaynak IP adresinden SSH servisine yönelik tekrarlanan bağlantılar oluşmuştur.

---

## 10. Wireshark Trafik Analizi

Oluşturulan SSH trafiğini incelemek için aşağıdaki filtre kullanılmıştır:

```text
tcp.port == 22
```

Ayrıca:

```text
ssh
```

filtresi ile SSH trafiği incelenmiştir.

Yalnızca saldırgan ve hedef arasındaki trafiği görmek için:

```text
ip.src == 192.168.56.1 && ip.dst == 192.168.56.101 && tcp.port == 22
```

filtresi kullanılmıştır.

Yakalanan trafik içerisinde tekrar eden TCP bağlantıları gözlemlenmiştir.

Genel olarak bağlantı yapısı:

```text
TCP SYN
    ↓
TCP SYN/ACK
    ↓
TCP ACK
    ↓
SSH iletişimi
    ↓
Bağlantının sonlandırılması
```

şeklindedir.

Bu bağlantı modeli, kontrollü saldırı sırasında birçok kez tekrarlanmıştır.

---

## 11. Önemli Gözlem: SSH Trafiğinin Şifreli Olması

SSH güvenli ve şifreli bir protokol olduğu için Wireshark üzerinde kullanıcı parolası doğrudan okunamaz.

Örneğin deney sırasında kullanılan:

```text
WrongPassword123!
```

parolası paketlerin içerisinde açık metin olarak görüntülenmez.

Bu nedenle NIDS açısından yalnızca paket içeriğine bakmak yeterli değildir.

Bunun yerine aşağıdaki ağ davranışları incelenebilir:

* Kaynak IP adresi
* Hedef IP adresi
* Hedef port
* SSH bağlantı sayısı
* Bağlantı sıklığı
* Bağlantılar arasındaki zaman aralığı
* TCP bağlantı kurulma ve sonlandırılma davranışı
* Aynı kaynaktan gelen tekrarlanan SSH bağlantıları

Bu özellikler SSH Brute Force saldırısının tespit edilmesinde kullanılabilir.

---

## 12. SSH Brute Force Tespit Göstergeleri

Bir NIDS içerisinde SSH Brute Force tespiti için aşağıdaki davranışlar değerlendirilebilir:

```text
Aynı kaynak IP
       ↓
TCP/22 bağlantısı
       ↓
Tekrar
       ↓
Tekrar
       ↓
Tekrar
       ↓
Kısa zaman aralığında yüksek bağlantı sayısı
       ↓
Şüpheli SSH davranışı
       ↓
SSH Brute Force Alarmı
```

Örneğin basitleştirilmiş bir tespit mantığı:

```text
EĞER

hedef_port = 22

VE

aynı kaynak IP'den kısa süre içerisinde
çok sayıda SSH bağlantısı geliyorsa

O ZAMAN

SSH Brute Force şüphesi oluştur.
```

Tek bir başarısız SSH girişimi doğrudan saldırı anlamına gelmeyebilir.

Ancak aynı kaynaktan kısa zaman içerisinde çok sayıda başarısız giriş davranışının gözlemlenmesi saldırı şüphesini önemli ölçüde artırır.

---

## 13. CyberTrace İçin Kullanılabilecek Özellikler

Bu PCAP dosyası CyberTrace NIDS içerisinde saldırı verisi olarak kullanılabilir.

Önerilen saldırı etiketi:

```text
SSH_Brute_Force
```

Kullanılabilecek temel özellikler:

| Özellik          | Açıklama                                   |
| ---------------- | ------------------------------------------ |
| Source IP        | SSH bağlantısını başlatan kaynak IP        |
| Destination IP   | Hedef SSH sunucusunun IP adresi            |
| Destination Port | SSH portu, 22                              |
| Protocol         | TCP                                        |
| Connection Count | Toplam bağlantı sayısı                     |
| Connection Rate  | Belirli zaman aralığındaki bağlantı sayısı |
| Time Interval    | Bağlantılar arasındaki süre                |
| Flow Duration    | Bağlantı süresi                            |
| TCP Flags        | SYN, ACK, FIN vb. TCP bayrakları           |

Bu özellikler ilerleyen aşamada kural tabanlı veya makine öğrenmesi tabanlı tespit mekanizmalarında kullanılabilir.

---

## 14. Normal SSH ile SSH Brute Force Karşılaştırması

| Özellik            | Normal SSH            | SSH Brute Force        |
| ------------------ | --------------------- | ---------------------- |
| Bağlantı sayısı    | Düşük                 | Yüksek                 |
| Kimlik doğrulama   | Genellikle tek deneme | Çok sayıda deneme      |
| Kaynak IP          | Genellikle sabit      | Genellikle aynı kaynak |
| Hedef port         | 22                    | 22                     |
| Bağlantı sıklığı   | Düşük                 | Yüksek                 |
| Başarısız girişler | Az veya hiç           | Çok sayıda             |
| NIDS riski         | Düşük                 | Yüksek                 |

Buradaki temel fark, **tekrarlanan kimlik doğrulama ve bağlantı davranışıdır.**

---

## 15. PCAP Dosyası

Yakalanan trafik aşağıdaki isimle kaydedilmiştir:

```text
ssh_brute_force.pcapng
```

GitHub içerisinde planlanan konum:

```text
network-attacks/
└── ssh-brute-force/
    ├── ssh-brute-force.md
    ├── ssh_brute_force.pcapng
    └── screenshots/
```

PCAP dosyasının istatistiklerini kontrol etmek için:

```bash
capinfos ~/Desktop/pcapfiles/ssh_brute_force.pcapng
```

komutu kullanılabilir.

### PCAP Bilgileri

| Bilgi            | Değer                            |
| ---------------- | -------------------------------- |
| Dosya adı        | `ssh_brute_force.pcapng`         |
| Protokol         | SSH                              |
| Taşıma protokolü | TCP                              |
| Kaynak IP        | 192.168.56.1                     |
| Hedef IP         | 192.168.56.101                   |
| Hedef port       | 22                               |
| Saldırı denemesi | 20                               |
| Paket sayısı     | `capinfos` çıktısından eklenecek |
| Yakalama süresi  | `capinfos` çıktısından eklenecek |
| Dosya boyutu     | `capinfos` çıktısından eklenecek |

> Paket sayısı, süre ve dosya boyutu gibi değerler PCAP dosyasından alınmalı ve tahmin edilmemelidir.

---

## 16. Ekran Görüntüleri

### SSH Servis Durumu

```markdown
![SSH servis durumu](screenshots/ssh-server-status.png)
```

### Normal SSH Bağlantısı

```markdown
![Normal SSH bağlantısı](screenshots/normal-ssh-connection.png)
```

### SSH Brute Force Denemeleri

```markdown
![SSH Brute Force denemeleri](screenshots/ssh-brute-force-script.png)
```

### Wireshark SSH Trafiği

```markdown
![Wireshark SSH trafiği](screenshots/ssh-traffic.png)
```

### Tekrarlanan SSH Bağlantıları

```markdown
![Tekrarlanan SSH bağlantıları](screenshots/repeated-ssh-connections.png)
```

### PCAP İstatistikleri

```markdown
![PCAP istatistikleri](screenshots/pcap-statistics.png)
```

---

## 17. Saldırının Güvenlik Açısından Etkisi

Başarılı bir SSH Brute Force saldırısı sonucunda saldırgan bir kullanıcı hesabının parolasını ele geçirirse sisteme yetkisiz erişim sağlayabilir.

Ele geçirilen hesabın yetkilerine bağlı olarak saldırgan:

* Hassas dosyalara erişebilir.
* Komut çalıştırabilir.
* Sistem yapılandırmalarını değiştirebilir.
* Zararlı yazılım çalıştırabilir.
* Sistemde kalıcılık sağlamaya çalışabilir.
* Ağ içerisindeki diğer sistemlere yönelmeye çalışabilir.
* Ele geçirilen sistemi başka saldırılar için kullanabilir.

Bu nedenle SSH servislerine yönelik tekrarlanan başarısız giriş denemelerinin izlenmesi önemlidir.

---

## 18. Korunma Yöntemleri

### Güçlü Parola Kullanımı

Uzun, karmaşık ve tahmin edilmesi zor parolalar kullanılmalıdır.

### SSH Anahtarı Kullanımı

Parola tabanlı kimlik doğrulama yerine SSH public key authentication tercih edilebilir.

### Gereksiz SSH Servislerinin Kapatılması

SSH servisine ihtiyaç duyulmayan sistemlerde servis devre dışı bırakılabilir.

### Rate Limiting

Kısa süre içerisinde çok sayıda giriş denemesi yapan kaynaklara bağlantı sınırlaması uygulanabilir.

### Fail2Ban

Fail2Ban gibi araçlarla çok sayıda başarısız kimlik doğrulama gerçekleştiren IP adresleri otomatik olarak engellenebilir.

### NIDS ile İzleme

NIDS sistemleri, SSH bağlantılarındaki olağandışı artışları ve tekrarlanan bağlantı davranışlarını tespit edebilir.

---

## 19. CyberTrace Veri Seti Etiketi

Bu çalışmada oluşturulan trafik için önerilen veri seti etiketi:

```text
SSH_Brute_Force
```

Bu etiket daha sonra diğer saldırı PCAP'ları ile birleştirilerek CyberTrace veri setinin oluşturulmasında kullanılabilir.

---

## 20. Sonuç

Bu çalışmada, izole bir VirtualBox laboratuvar ortamında Kali Linux üzerindeki SSH servisine yönelik kontrollü bir SSH Brute Force saldırı simülasyonu gerçekleştirilmiştir.

İlk olarak SSH servisi kontrol edilmiş ve deney için ayrı bir kullanıcı hesabı oluşturulmuştur. Saldırı trafiği oluşturulmadan önce Windows sisteminden normal bir SSH bağlantısı gerçekleştirilerek servis doğrulanmıştır.

Daha sonra Windows sistemi üzerinden 20 adet kontrollü ve başarısız SSH kimlik doğrulama denemesi gerçekleştirilmiştir. Bu sırada oluşan ağ trafiği Wireshark kullanılarak yakalanmış ve `ssh_brute_force.pcapng` adıyla kaydedilmiştir.

SSH trafiği şifreli olduğundan parola bilgileri Wireshark üzerinde açık şekilde görüntülenememektedir. Buna rağmen kaynak IP, hedef IP, hedef port, bağlantı sayısı, bağlantı sıklığı ve zaman aralıkları gibi ağ özellikleri incelenerek Brute Force davranışı tespit edilebilir.

Elde edilen PCAP dosyası, CyberTrace projesinde `SSH_Brute_Force` etiketiyle kullanılabilecek saldırı verilerinden biri olarak değerlendirilmiştir.

---

## 21. GitHub Klasör Yapısı

```text
network-attacks/
└── ssh-brute-force/
    ├── ssh-brute-force.md
    ├── ssh_brute_force.pcapng
    └── screenshots/
        ├── ssh-server-status.png
        ├── normal-ssh-connection.png
        ├── ssh-brute-force-script.png
        ├── ssh-traffic.png
        ├── repeated-ssh-connections.png
        └── pcap-statistics.png
```

---

## 22. Sonraki Aşama

Saldırı çalışmalarındaki bir sonraki konu:

```text
FTP Brute Force
```

İzlenecek genel çalışma yöntemi:

```text
Laboratuvar Ortamının Hazırlanması
            ↓
Normal Trafiğin Oluşturulması
            ↓
Kontrollü Saldırı
            ↓
Wireshark ile Trafik Yakalama
            ↓
PCAP Kontrolü
            ↓
Saldırı Trafiğinin Analizi
            ↓
GitHub Dokümantasyonu
            ↓
CyberTrace Veri Setine Eklenmesi
```

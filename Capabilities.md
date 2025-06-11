### **1\. Teori: Linux Yetenekleri (Capabilities) Nedir ve Neden Önemlidir?**

Geleneksel UNIX izin modelinde, iki seviye vardır: **ayrıcalıksız kullanıcı** (regular user) ve **tam yetkili kullanıcı** (root). Bir işlemin ağ soketi açmak gibi ayrıcalıklı bir işlem yapması gerektiğinde, ona SUID biti ile tam `root` yetkisi vermek gerekirdi. Bu, "bir kapıyı açmak için binanın anahtarını vermek" gibidir; aşırı riskli ve gereksizdir.

**Capabilities** ise bu sorunu çözer. `root` kullanıcısının sahip olduğu tüm güçleri, daha küçük ve yönetilebilir "yeteneklere" böler.

- `CAP_NET_BIND_SERVICE`: 1024'ün altındaki portlara bağlanma yeteneği.
- `CAP_SETUID`: İşlemin kullanıcı kimliğini (UID) değiştirme yeteneği.
- `CAP_DAC_READ_SEARCH`: Dosya okuma ve dizin arama izinlerini baypas etme yeteneği.

Bir sistem yöneticisi, bir programa tam `root` yetkisi vermek yerine, sadece işini yapması için gereken spesifik yeteneği (`capability`) atayabilir. Bu, sistemin daha güvenli olmasını sağlar ama bizim gibi siber güvenlik uzmanları için de yeni bir saldırı yüzeyi oluşturur. 🎯

**Neden SUID Aramalarında Gözden Kaçar?** Senin de belirttiğin gibi en önemli nokta bu: `find / -perm -u=s` gibi komutlar sadece SUID bitine sahip dosyaları arar. Yetenekler (Capabilities) dosyanın izin bitlerinde değil, **genişletilmiş özniteliklerinde (extended attributes)** saklanır. Bu yüzden bu klasik komutlarla `vim` gibi bir dosyayı bulamazsın. Bu da onu daha gizli ve etkili bir vektör yapar.

* * *

### **2\. Pratik: Keşif ve Sömürü**

#### **Adım 1: Keşif (Enumeration)**

Sistemdeki tüm dosyaları tarayarak atanmış yetenekleri bulmamız gerekiyor. Bunun için `getcap` aracını kullanırız.

Bash

```
getcap -r / 2>/dev/null
```

Bu komutu biraz daha açalım:

- `getcap`: Yetenekleri okuyan ana komutumuz.
- `-r`: "Recursive" yani özyinelemeli. Belirtilen dizinden (`/`) başlayarak tüm alt dizinleri tarar.
- `/`: Kök dizin. Tüm sistemi taramak için buradan başlarız.
- `2>/dev/null`: Bu çok kritik bir kısım. `2>` standart hata (stderr) akışını yönlendirir. `/dev/null` ise "kara delik" olarak bilinen, üzerine yazılan her şeyi yok eden özel bir dosyadır. Ayrıcalıksız bir kullanıcı olarak tüm sistemi tararken "Permission denied" (İzin reddedildi) gibi binlerce hata mesajı alırız. Bu komut, bu hata mesajlarını ekranda göstermek yerine doğrudan çöpe atarak bize sadece temiz bir çıktı sunar.

**Örnek Çıktı:** Komutu çalıştırdığında şöyle bir şey görebilirsin:

```
/usr/bin/python3.9 = cap_setuid+ep
/usr/bin/vim.basic = cap_dac_read_search+ep
```

#### **Adım 2: Analiz ve Sömürü Planı**

Bu çıktıyı yorumlayalım:

- `python3.9` dosyası `cap_setuid` yeteneğine sahip. Bu, bu scriptin kendi kullanıcı kimliğini (UID) değiştirebileceği anlamına gelir. Hedefimiz `root` (UID=0) olmak olduğu için bu çok tehlikelidir! 💣
- `vim.basic` dosyası `cap_dac_read_search` yeteneğine sahip. Bu da onun sistemdeki tüm dosyaları okuyabileceği anlamına gelir. Örneğin `/etc/shadow` dosyasını okuyup parolaları kırmaya çalışabiliriz.

Şimdi GTFObins'e gitme zamanı. GTFObins, bu gibi durumlarda bir ikili dosyayı (binary) kötüye kullanarak nasıl yetki yükseltebileceğimizi gösteren komutları barındıran bir hazinedir.

1.  GTFObins sitesine git.
2.  Arama kutusuna `python` yaz.
3.  Fonksiyonlar listesinden "Capabilities"i bul.

Orada `cap_setuid` yeteneği için şu komutu göreceksin:

Python

```
./python3.9 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

- `-c '...'`: Python'a tek satırlık bir komut çalıştıracağını söyler.
- `import os`: İşletim sistemi fonksiyonlarını kullanmak için `os` modülünü içeri aktarır.
- `os.setuid(0)`: İşte sihirli kısım burası! `cap_setuid` yeteneğini kullanarak mevcut işlemin kullanıcı kimliğini `0` yani `root` olarak ayarlar.
- `os.system("/bin/bash")`: Yeni `root` kimliğiyle bir `bash` kabuğu başlatır.

* * *

### **3\. Pratik Senaryo: Adım Adım Yetki Yükseltme**

Hadi bunu baştan sona bir senaryo ile canlandıralım. Bir CTF makinesindesin ve `user` olarak kabuk aldın. Amacın `root` olmak.

1.  **Keşif:** İlk olarak SUID dosyalarını aradın ama işe yarar bir şey bulamadın. Sıradaki adayın Yetenekler (Capabilities).
    
    Bash
    
    ```
    user@ctf-box:~$ getcap -r / 2>/dev/null
    /usr/bin/tar = cap_dac_read_search+ep
    ```
    
    Bingo! `tar` komutunun tüm dosyaları okuma yeteneği var.
    
2.  **Planlama:** `tar` ile `/etc/shadow` gibi hassas dosyaları okuyabilirsin. `shadow` dosyasını okuyup kendi makinenize kopyaladıktan sonra John the Ripper veya Hashcat ile parolaları kırmayı deneyebilirsin.
    
3.  **GTFObins Kontrolü:** GTFObins'e gidip `tar`'ı arıyorsun ve "Capabilities" bölümünü kontrol ediyorsun. Orada `cap_dac_read_search` yeteneğiyle dosya okumak için bir komut buluyorsun:
    
    Bash
    
    ```
    ./tar -cf /dev/null /etc/shadow --checkpoint=1 --checkpoint-action=exec=/bin/cat /etc/shadow
    ```
    
    Bu komut, `tar`'ın checkpoint özelliğini kötüye kullanarak aslında bir dosyayı arşivlemek yerine başka bir komutu (`cat /etc/shadow`) çalıştırır. Ancak daha basit bir yol da var:
    
4.  **Uygulama:** `tar`'ın kendisi doğrudan dosya okuyabilir.
    
    Bash
    
    ```
    user@ctf-box:~$ /usr/bin/tar -cvf shadow.tar /etc/shadow
    /etc/shadow
    ```
    
    Bu komut, `/etc/shadow` dosyasını `shadow.tar` adlı bir arşive kaydeder. `cap_dac_read_search` yeteneği sayesinde okuma iznin olmasa bile bu işlem başarılı olur.
    
5.  **Sonuç Alma:** Şimdi `shadow.tar` arşivini açıp içindeki hash'leri alabilirsin.
    
    Bash
    
    ```
    user@ctf-box:~$ tar -xvf shadow.tar
    /etc/shadow
    user@ctf-box:~$ cat etc/shadow
    root:$6$.....:18628:0:99999:7:::
    daemon:*:18628:0:99999:7:::
    ...
    ```
    
    Artık `root` kullanıcısının parola hash'ine sahipsin!
    

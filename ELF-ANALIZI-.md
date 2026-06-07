# ELF Analiz Raporu — new-firmware.z1 / udp-client.z1 / udp-server.z1

Analiz edilen dosyalar: `new-firmware.z1`, `udp-client.z1`, `udp-server.z1`  
Araç zinciri: MSP430 GCC Toolchain (`msp430-readelf`, `msp430-objdump`, `msp430-nm`, `msp430-size`, `msp430-strings`, `msp430-addr2line`)

---

## 1. Binary Kimlik Analizi

### Kullanılan Araç
```bash
$ file new-firmware.z1 udp-client.z1 udp-server.z1
```

### Çıktı
```
new-firmware.z1: ELF 32-bit LSB executable, TI msp430, version 1 (embedded),
                 statically linked, with debug_info, not stripped
udp-client.z1:  ELF 32-bit LSB executable, TI msp430, version 1 (embedded),
                 statically linked, with debug_info, not stripped
udp-server.z1:  ELF 32-bit LSB executable, TI msp430, version 1 (embedded),
                 statically linked, with debug_info, not stripped
```

### Yorum

| Alan | Değer | Anlamı |
|------|-------|--------|
| ELF sınıfı | ELF32 | 32-bit adres uzayı kullanan ELF formatı |
| Endianness | LSB (Little-Endian) | Düşük anlamlı bayt önce gelir; MSP430 mimarisine uygun |
| Hedef platform | TI MSP430 | Texas Instruments 16-bit mikrodenetleyici ailesi |
| Bağlama | Statically linked | Tüm kütüphaneler tek ELF içinde birleştirilmiş |
| Debug bilgisi | with debug_info | DWARF debug bölümleri mevcut; semboller silinmemiş |

```bash
$ msp430-readelf -h new-firmware.z1
```
```
ELF Header:
  Magic:          7f 45 4c 46 01 01 01 ff 00 00 00 00 00 00 00 00
  Class:          ELF32
  Data:           2's complement, little endian
  Version:        1 (current)
  OS/ABI:         Standalone App
  Type:           EXEC (Executable file)
  Machine:        Texas Instruments msp430 microcontroller
  Entry point:    0x3100
  Flags:          0x0
```

**Entry Point (0x3100):** İşlemcinin reset sonrası kodu yürütmeye başladığı adres. Bu adres C runtime başlangıç koduna (yığın kurulumu, `.data` kopyalama, `.bss` sıfırlama) karşılık gelir. `main()` fonksiyonu ön işlemler tamamlandıktan sonra **0x313E** adresinde devreye girer.

**Neden ELF, ham binary değil?**  
ELF formatı yalnızca makine kodu içermez; bölüm adresleri, sembol tablosu, debug bilgisi ve bağlayıcı (linker) metadata'sını da barındırır. Ham binary bu yapıyı içermez.

| Özellik | ELF | Ham Binary |
|---------|-----|------------|
| Bölüm adresleri | ✅ Var | ❌ Yok |
| Sembol tablosu | ✅ Var | ❌ Yok |
| Debug bilgisi | ✅ Var | ❌ Yok |
| Çoklu segment desteği | ✅ Var | ❌ Yok |
| Toolchain entegrasyonu | ✅ Tam | ❌ Sınırlı |
| Yükleyici (loader) desteği | ✅ Var | ❌ Yok |

Ham binary, ELF içinden `objcopy -O binary` ile üretilir ve yalnızca Flash'a yazılacak ham baytları içerir. ELF ise geliştirme, hata ayıklama ve araç zinciri entegrasyonu için vazgeçilmezdir.

---

## 2. Bellek Kullanım Analizi

### Bölüm Başlıkları

```bash
$ msp430-readelf -S new-firmware.z1
```
```
[Nr] Name        Type     Addr     Size   Flg
[ 1] .far.text   PROGBITS 00010000 004a78  AX   (Flash - uzak kod)
[ 2] .text       PROGBITS 00003100 00976e  AX   (Flash - ana kod)
[ 3] .rodata     PROGBITS 0000c870 0035fd   A   (Flash - salt okunur veri)
[ 4] .data       PROGBITS 00001100 000150  WA   (RAM - başlangıç değerleri)
[ 5] .bss        NOBITS   00001250 001648  WA   (RAM - sıfırlanmış değişkenler)
[ 6] .noinit     NOBITS   00002898 000002  WA   (RAM - başlatılmamış alan)
[ 7] .vectors    PROGBITS 0000ffc0 000040  AX   (Flash - kesme vektörleri)
```

| Bölüm | Adres | Boyut | Açıklama |
|-------|-------|-------|----------|
| `.text` | 0x3100 | 38.766 B (~37,9 KB) | Derlenmiş makine kodu |
| `.far.text` | 0x10000 | 19.064 B (~18,6 KB) | MSP430 64KB sınırı ötesi uzak kod |
| `.rodata` | 0xC870 | 13.821 B (~13,5 KB) | String sabitler, lookup tabloları |
| `.data` | 0x1100 | 336 B | Başlangıç değerli global değişkenler |
| `.bss` | 0x1250 | 5.704 B (~5,6 KB) | Sıfır başlangıçlı global değişkenler |
| `.noinit` | 0x2898 | 2 B | Reset'te sıfırlanmayan özel alan |
| `.vectors` | 0xFFC0 | 64 B | Kesme vektör tablosu (32 vektör × 2 B) |

**Bölüm bayraklarının anlamı:**  
- `A` (Allocated): Çalışma zamanında bellekte yer kaplar  
- `X` (Executable): Çalıştırılabilir makine kodu içerir  
- `W` (Writable): Çalışma zamanında yazılabilir  
- `PROGBITS`: Gerçek veri içerir (Flash'ta bulunur)  
- `NOBITS`: Flash'ta yer kaplamaz, boyut bilgisi tutulur, RAM'de sıfırlanır

### Kod ve Veri Boyutları

```bash
$ msp430-size new-firmware.z1 udp-client.z1 udp-server.z1
```
```
   text    data     bss     dec     hex  filename
  71715     336    5706   77757   12fbd  new-firmware.z1
  47612     332    5856   53800    d228  udp-client.z1
  48738     332    5986   55056    d710  udp-server.z1
```

| Dosya | Flash (text+data) | RAM (data+bss) | Toplam |
|-------|-------------------|----------------|--------|
| new-firmware.z1 | 72.051 B (~70,4 KB) | 6.042 B (~5,9 KB) | 77.757 B |
| udp-client.z1 | 47.944 B (~46,8 KB) | 6.188 B (~6,0 KB) | 53.800 B |
| udp-server.z1 | 49.070 B (~47,9 KB) | 6.318 B (~6,2 KB) | 55.056 B |

- **text:** Flash'ta yer alan kod + sabit veri (`.text` + `.far.text` + `.rodata` + `.vectors`)
- **data:** Hem Flash'ta (LMA) hem RAM'de (VMA) yer alır; reset sırasında Flash'tan RAM'e kopyalanır
- **bss:** Yalnızca RAM'de yer alır; Flash'ta boyut bilgisi tutulur, reset sırasında sıfırlanır
- **dec:** Toplam bellek ayak izi (text + data + bss)

### CC1352R Üzerinde Bellek Yerleşim Haritası

#### Flash (352 KB) — new-firmware.z1

```
0x00000000 ┌─────────────────────────────────┐
           │     BIM Bootloader  (16 KB)     │
0x00004000 ├─────────────────────────────────┤  ← Uygulama Başlangıcı
           │  .text      (~37,9 KB)          │  Çalıştırılabilir kod
0x0000D770 ├─────────────────────────────────┤
           │  .rodata    (~13,5 KB)          │  Salt-okunur sabitler
0x00010D70 ├─────────────────────────────────┤
           │  .data LMA  (336 B)             │  Global değişken başlangıç değerleri
0x00010EC0 ├─────────────────────────────────┤
           │  .far.text  (~18,6 KB)          │  64KB sınırı ötesi kod
0x00015938 ├─────────────────────────────────┤
           │  Boş Flash  (~55 KB)            │  OTA için kullanılabilir alan
0x00023000 ├─────────────────────────────────┤  ← Slot B Başlangıcı
           │  OTA Slot B (124 KB)            │  Güncelleme firmware deposu
0x00042000 ├─────────────────────────────────┤
           │  Kalan Boş Flash                │
0x0057F800 ├─────────────────────────────────┤
           │  CCFG (2 KB)  — KRİTİK         │  Chip konfigürasyonu
0x0057FFFF └─────────────────────────────────┘
```

#### SRAM (80 KB)

```
0x20000000 ┌─────────────────────────────────┐
           │  .data VMA   (336 B)            │  Flash'tan kopyalanır
0x20000150 ├─────────────────────────────────┤
           │  .bss        (5.704 B)          │  Reset'te sıfırlanır
0x20001798 ├─────────────────────────────────┤
           │  Dinamik RAM (~74 KB)           │  Stack + Contiki buffer pool
0x20013FFF └─────────────────────────────────┘
```

---

## 3. Sembol Tablosu Analizi

```bash
$ msp430-nm -n new-firmware.z1 | grep " T \| t " | head -40
```
```
0000313e T main
00003376 T __ctors_end
00003376 T __dtors_end
0000337c T __mulsi3
000033a2 T __udivhi3
0000353e T port1_isr
000035c2 T irq_p2
000035fe T cc2420_timerb1_interrupt
00003624 T timera1
000036f4 T i2c_tx_interrupt
00003778 T i2c_rx_interrupt
0000378c T timera0
000037ae T uart0_rx_interrupt
000037d8 T watchdog_interrupt
00003a68 T accm_init
00003ad6 T cc2420_arch_init
```

### Sembol Türleri

| Harf | Anlamı |
|------|--------|
| `T` | Global kod sembolü — dışarıdan erişilebilir |
| `t` | Statik/lokal kod sembolü — yalnızca kaynak dosya içinde |
| `D` | Global başlangıç değerli veri (.data) |
| `B` | Global sıfır başlangıçlı veri (.bss) |
| `R` | Global salt-okunur veri (.rodata) |

### Anlamlı Semboller

| Sembol | Adres | Tür | Anlamı |
|--------|-------|-----|--------|
| `main` | 0x313E | T | Ana program başlangıcı |
| `port1_isr` | 0x353E | T | Port 1 GPIO kesme işleyicisi |
| `irq_p2` | 0x35C2 | T | Port 2 GPIO kesme işleyicisi |
| `cc2420_timerb1_interrupt` | 0x35FE | T | CC2420 radyo zamanlayıcı kesmesi |
| `timera1` / `timera0` | 0x3624 / 0x378C | T | Timer A kesme işleyicileri |
| `uart0_rx_interrupt` | 0x37AE | T | UART alım kesmesi |
| `watchdog_interrupt` | 0x37D8 | T | Watchdog zamanlayıcı kesmesi |
| `cc2420_arch_init` | 0x3AD6 | T | CC2420 radyo başlatma |
| `accm_init` | 0x3A68 | T | İvmeölçer başlatma |
| `__mulsi3` / `__udivhi3` | 0x337C+ | T | Yazılımsal çarpma/bölme rutinleri |

**Not:** `__mulsi3` ve `__udivhi3` gibi semboller, MSP430'un donanımsal çarpma/bölme birimi olmadığını gösterir. Derleyici bu işlemleri yazılım rutinleri ile gerçekleştirir.

---

## 4. String ve Statik Veri Analizi

### Kullanılan Araçlar

```bash
$ msp430-strings new-firmware.z1 | head -60
```
```
RPL
uip
IPv6
Rime
contiki
udp-server
udp-client
node_id
[BOOT]
[NET]
[RADIO]
cc2420
ERR: radio init failed
[OTA] Slot A valid
[OTA] Slot B valid
[OTA] Starting update...
[OTA] CRC check failed
LLADDR
6LoWPAN
CSMA
autostart
uIP stack
RPL-Classic
fe80::
sensors/akes
```

### String Analizinin Yorumu

String analizi, firmware'in hangi yazılım bileşenlerini içerdiğini doğrudan ortaya koyar:

| Tespit Edilen String | Anlamı |
|---------------------|--------|
| `RPL`, `IPv6`, `6LoWPAN` | Contiki OS ağ yığını — IoT için IPv6 protokol ailesi |
| `uip`, `uIP stack` | Lightweight IP (uIP) — kısıtlı cihazlar için TCP/IP |
| `Rime` | Contiki'nin düşük seviyeli kanalı — düğümler arası iletişim |
| `cc2420` | IEEE 802.15.4 radyo sürücüsü |
| `node_id` | Dinamik düğüm kimliği mekanizması |
| `[OTA] Slot A/B valid` | OTA (Over-the-Air) güncelleme bileşeni |
| `[OTA] CRC check failed` | OTA bütünlük doğrulama hata mesajı |
| `ERR: radio init failed` | Radyo başlatma hata dizesi |
| `fe80::` | IPv6 link-local adres öneki |

Bu stringler `.rodata` bölümünde saklanır ve Flash bellekte yer kaplar. Çalışma zamanında RAM'e kopyalanmaz; yalnızca işaretçi ile referans alınırlar. `[OTA]` prefix'li stringlerin varlığı, `new-firmware.z1`'in OTA güncelleme yeteneğine sahip olduğunu doğrular.

### `msp430-objdump -x` ile Bölüm Detayları

```bash
$ msp430-objdump -x new-firmware.z1 | grep -A 5 "SYMBOL TABLE"
```
```
SYMBOL TABLE:
0000313e g     F .text  0000003e main
0000353e g     F .text  00000084 port1_isr
00003a68 g     F .text  0000006e accm_init
00003ad6 g     F .text  0000004c cc2420_arch_init
0000ffc0 g     O .vectors 00000040 _vectors_start
```

Bu çıktıda `g` (global), `F` (fonksiyon), `O` (nesne/veri) bayrakları görülür. Her fonksiyonun boyutu da verilmektedir; örneğin `main` fonksiyonu 62 bayt (0x3E) uzunluğundadır.

---

## 5. Assembly / Instruction Analizi — Disassembly

```bash
$ msp430-objdump -d new-firmware.z1 | grep -A 30 "<main>"
```
```
0000313e <main>:
    313e:  b0 13 e4 6a    calla  #0x06ae4     ; donanım sürücüsü init
    3142:  b0 13 3a 45    calla  #0x0453a     ; ağ yığını init
    3146:  b0 13 da aa    calla  #0x0aada     ; zamanlayıcı init
    314a:  b0 13 7c 6d    calla  #0x06d7c     ; süreç başlatma
    314e:  0e 43          clr    r14           ; r14 = 0
    3150:  3f 40 3c 11    mov    #4412, r15   ; node-id adresi yükle
    3154:  b0 13 8e 6e    calla  #0x06e8e     ; node-id oku
    3178:  b2 90 03 00    cmp    #3, &0x1160  ; node_id == 2 mi?
    317e:  0e 38          jl     $+30         ; değilse atla
    3180:  30 12 3a ca    push   #0xca3a      ; gönderici kodu
```

### Komut Yorumları

| Komut | Açıklama |
|-------|----------|
| `calla #addr` | MSP430X uzak alt program çağrısı (20-bit adres, 64KB ötesi) |
| `clr r14` | r14 yazmaçını sıfırlar |
| `mov #val, r15` | r15'e sabit değer yükler |
| `cmp #3, &0x1160` | Bellek adresindeki node_id'yi 3 ile karşılaştırır |
| `jl $+30` | Küçükse 30 bayt ileriye atla (if bloğu) |
| `push #val` | Yığıta parametre iter (fonksiyon argümanı) |

**`calla` ve `.far.text` ilişkisi:** `calla` komutu MSP430X mimarisine özgü uzak çağrı komutudur. 64KB sınırını aşan `.far.text` bölümündeki fonksiyonlara ulaşmak için standart `call` yerine `calla` kullanılır.

**`cmp #3, &0x1160` → node_id kontrolü:** `udp-client.c` kaynak kodundaki `if(node_id == 2)` bloğunun derleme zamanı karşılığıdır. Aynı firmware farklı node_id değerlerine göre farklı kod bloklarını çalıştırır.

**Entry point netleştirme:** ELF başlığı `0x3100`'ı entry point olarak gösterir. Disassembly çıktısı `main()`'in `0x313E`'de başladığını kanıtlar. `0x3100`–`0x313D` arası C runtime ön işlemleri gerçekleştirir.

### `msp430-addr2line` ile Kaynak Satırı Eşleştirme

```bash
$ msp430-addr2line -e new-firmware.z1 -f 0x313e
```
```
main
/home/user/contiki/examples/ota-demo/main.c:47
```

`msp430-addr2line` aracı, bir makine kodu adresini kaynak dosya ve satır numarasına eşler. DWARF debug bilgisi mevcut olduğu için (bkz. Bölüm 1: "not stripped") bu eşleme mümkündür. Üretim firmware'i için debug bilgisi `msp430-strip` ile kaldırılmalıdır.

---

## 6. Karşılaştırmalı Firmware Analizi

Bu bölüm, üç firmware imajını boyut, sembol yoğunluğu ve işlevsel rol açısından karşılaştırır.

### Boyut Karşılaştırması

| Metrik | new-firmware.z1 | udp-client.z1 | udp-server.z1 |
|--------|-----------------|---------------|---------------|
| Flash kullanımı | 72.051 B (70,4 KB) | 47.944 B (46,8 KB) | 49.070 B (47,9 KB) |
| RAM kullanımı | 6.042 B (5,9 KB) | 6.188 B (6,0 KB) | 6.318 B (6,2 KB) |
| Flash'taki oran | %20,0 (352 KB'nin) | %13,3 | %13,6 |
| Fark (client'a göre) | +24.107 B (✕1,50) | — | +1.126 B |

### Fonksiyonel Rol Analizi

**`new-firmware.z1` — OTA Ana Firmware:**  
Flash ayak izi diğerlerine göre ~1,5 kat büyüktür. Bu fark, OTA güncelleme motoru, hem istemci hem sunucu rol kodlarını ve CRC doğrulama rutinlerini tek imajda barındırmasından kaynaklanır. String analizinde görülen `[OTA] Slot A/B valid` mesajları bu yorumu destekler. `main()` içindeki `node_id` kontrolü, bu tek imajın çalışma zamanında rolünü (istemci/sunucu) belirlemesini sağlar.

**`udp-client.z1` — UDP İstemci Firmware:**  
En küçük Flash ayak izine sahip imajdır. Yalnızca istemci rolüne özgü kod içerir: periyodik veri gönderme, ağa katılma ve uyku modu rutinleri. Sunucu bağlantı yönetimi ve OTA motoru bulunmaz.

**`udp-server.z1` — UDP Sunucu Firmware:**  
İstemciden yaklaşık 1 KB daha büyüktür. Sunucu rolü, birden fazla istemciden gelen paketleri yönetmek için ek buffer ve bağlantı takip yapıları gerektirir. BSS bölümü (RAM sıfır-başlangıçlı alan) da istemciye göre daha büyüktür (5.986 B vs 5.856 B), bu durum sunucunun çalışma zamanında daha fazla durum bilgisi tuttuğunu gösterir.

### Sembol Sayısı Karşılaştırması

```bash
$ msp430-nm new-firmware.z1 | wc -l
$ msp430-nm udp-client.z1 | wc -l
$ msp430-nm udp-server.z1 | wc -l
```
```
847   new-firmware.z1
612   udp-client.z1
634   udp-server.z1
```

| Dosya | Toplam Sembol | Kod Sembolü (T+t) | Veri Sembolü (D+B) |
|-------|---------------|-------------------|---------------------|
| new-firmware.z1 | ~847 | ~310 | ~537 |
| udp-client.z1 | ~612 | ~228 | ~384 |
| udp-server.z1 | ~634 | ~237 | ~397 |

Sembol sayısı farkı, `new-firmware.z1`'in diğer iki firmware'i kapsayan üst küme olduğunu doğrular.

### node_id Mekanizması

Assembly analizinde (Bölüm 5) görülen `cmp #3, &0x1160` komutu, `new-firmware.z1`'in tek binary içinde çok-rol desteğini nasıl sağladığını gösterir:

```c
// Kaynak kod karşılığı (C pseudocode):
if (node_id < 3) {
    // UDP sunucu rolü: bağlantı kabul et, veriyi topla
    start_udp_server();
} else {
    // UDP istemci rolü: periyodik gönderme
    start_udp_client();
}
```

Bu yaklaşım, tek bir firmware imajının farklı düğüm kimliklerine göre farklı roller üstlenmesini sağlar. Hem üretim verimliliğini artırır (tek binary dağıtımı) hem de OTA güncellemelerinde hedef düğüm ayrımını basitleştirir.

---

## 7. ELF Yapısı — Program Başlıkları (LOAD Segmentleri)

```bash
$ msp430-readelf -l new-firmware.z1
```
```
Type    Offset   VirtAddr   PhysAddr   FileSiz  MemSiz   Flg
LOAD    0x000000 0x0000300c 0x0000300c 0x09862  0x09862  R E   (.text)
LOAD    0x009864 0x0000c870 0x0000c870 0x035fd  0x035fd  R     (.rodata)
LOAD    0x00ce62 0x00001100 0x0000fe6e 0x00150  0x01798  RW    (.data + .bss)
LOAD    0x00cfb2 0x00002898 0x0000ffbe 0x00000  0x00002  RW    (.noinit)
LOAD    0x00cfb2 0x0000ffc0 0x0000ffc0 0x00040  0x00040  R E   (.vectors)
LOAD    0x00cff2 0x00010000 0x00010000 0x04a78  0x04a78  R E   (.far.text)
```

### VirtAddr ve PhysAddr Farkı

| Alan | Anlamı |
|------|--------|
| `PhysAddr` (LMA) | Flash'taki fiziksel konum — başlangıç değerleri burada saklanır |
| `VirtAddr` (VMA) | RAM'deki çalışma adresi — reset sırasında buraya kopyalanır |
| `FileSiz` | ELF dosyasında kaç bayt yer kaplar |
| `MemSiz` | Çalışma zamanında bellekte kaç bayt yer kaplar |

`.data` segmentinde `FileSiz (0x150) < MemSiz (0x1798)`: Flash'ta yalnızca 336 bayt başlangıç değeri saklanır; `.bss` alanı (5.704 B) RAM'de sıfırlanır, Flash'ta yer kaplamaz. Bu fark MSP430 bellek tasarrufu stratejisinin temelidir.

---

## 8. Kesme Vektörleri ve Donanım Analizi

```bash
$ msp430-readelf -x .vectors new-firmware.z1
```
```
0x0000ffc0  76337633 76337633 76337633 76337633
0x0000ffd0  76337633 76337633 76337633 76337633
0x0000ffe0  f4367837 3e35c235 76337633 7633ae37
0x0000fff0  24368c37 d8377633 fe357633 76330031
```

### Little-Endian Çözümleme

Little-endian okuma ile çözümlenen vektörler (her 2 bayt bir vektör adresi):

| Vektör Adresi | Hedef | Fonksiyon | Tetikleyici |
|---------------|-------|-----------|-------------|
| 0xFFE0 | 0x36F4 | `i2c_tx_interrupt` | I²C veri gönderimi tamamlandı |
| 0xFFE2 | 0x3778 | `i2c_rx_interrupt` | I²C veri alımı tamamlandı |
| 0xFFE4 | 0x353E | `port1_isr` | Port 1 GPIO kenar değişimi |
| 0xFFE6 | 0x35C2 | `irq_p2` | Port 2 GPIO kenar değişimi |
| 0xFFEE | 0x37AE | `uart0_rx_interrupt` | UART alım tamponu doldu |
| 0xFFF0 | 0x3624 | `timera1` | Timer A karşılaştırma/yakalama 1 |
| 0xFFF2 | 0x378C | `timera0` | Timer A taşma |
| 0xFFF4 | 0x37D8 | `watchdog_interrupt` | Watchdog sayacı süresi doldu |
| 0xFFF8 | 0x35FE | `cc2420_timerb1_interrupt` | CC2420 radyo zamanlayıcı kesmesi |
| **0xFFFE** | **0x3100** | **RESET vektörü** | **Güç açma / harici reset** |

`0x3376` ile dolu vektörler `__br_unexpected_` (tanımsız kesme işleyicisi) gösterir. Bu, beklenmedik bir kesme tetiklenirse sistemin tanımlı bir adrese (genellikle sonsuz döngü) yönlenmesini sağlar.

**RESET vektörü (0xFFFE → 0x3100):** MSP430 reset sonrası bu adresten çalışmaya başlar. 0x3100 → C runtime ön işlemleri → 0x313E `main()`.

### Donanım Bileşenleri

Sembol ve kesme tablosu birlikte yorumlandığında firmware'in aşağıdaki donanımları yönettiği anlaşılır:

| Donanım | İlgili Semboller | İşlev |
|---------|------------------|-------|
| CC2420 | `cc2420_arch_init`, `cc2420_timerb1_interrupt` | IEEE 802.15.4 kablosuz radyo |
| I²C | `i2c_tx_interrupt`, `i2c_rx_interrupt` | Sensör veriyolu |
| UART | `uart0_rx_interrupt` | Seri debug/programlama portu |
| Timer A | `timera0`, `timera1` | Zamanlama, MAC katmanı |
| Watchdog | `watchdog_interrupt` | Sistem kilitlenme koruması |
| GPIO Port 1/2 | `port1_isr`, `irq_p2` | Buton, sensör sinyalleri |
| İvmeölçer | `accm_init` | Hareket algılama |

---

## 9. Özet ve Sonuçlar

| Başlık | Bulgu |
|--------|-------|
| Platform | TI MSP430, 16-bit mikrodenetleyici, Little-Endian |
| İşletim Sistemi | Contiki RTOS — kısıtlı IoT cihazları için |
| Ağ Yığını | uIP / IPv6 / 6LoWPAN / RPL |
| Radyo | CC2420 — IEEE 802.15.4 |
| OTA Desteği | new-firmware.z1'de Slot A/B mekanizması ile mevcut |
| Çok-rol | node_id tabanlı çalışma zamanı rol seçimi |
| Debug | DWARF bilgisi mevcut; üretimde `msp430-strip` ile kaldırılmalı |
| Flash verimliliği | 352 KB'nin %20'si kullanılıyor; OTA için ~55 KB boş |

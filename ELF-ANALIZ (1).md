# ELF Analiz Raporu — new-firmware.z1 / udp-client.z1 / udp-server.z1

Analiz edilen dosyalar: `new-firmware.z1`, `udp-client.z1`, `udp-server.z1`
Araç zinciri: MSP430 GCC Toolchain (`msp430-readelf`, `msp430-objdump`, `msp430-nm`, `msp430-size`)

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
Entry point address: 0x3100
Machine:             Texas Instruments msp430 microcontroller
Type:                EXEC (Executable file)
OS/ABI:              Standalone App
```

**Entry Point (0x3100):** İşlemcinin reset sonrası kodu yürütmeye başladığı adres. Bu adres C runtime başlangıç koduna (yığın kurulumu, `.data` kopyalama, `.bss` sıfırlama) karşılık gelir. `main()` fonksiyonu ön işlemler tamamlandıktan sonra **0x313E** adresinde devreye girer.

**Neden ELF, ham binary değil?**
ELF formatı yalnızca makine kodu içermez; bölüm adresleri, sembol tablosu, debug bilgisi ve bağlayıcı (linker) metadata'sını da barındırır. Ham binary bu yapıyı içermez. Karşılaştırma:

| Özellik | ELF | Ham Binary |
|---------|-----|------------|
| Bölüm adresleri | ✅ Var | ❌ Yok |
| Sembol tablosu | ✅ Var | ❌ Yok |
| Debug bilgisi | ✅ Var | ❌ Yok |
| Çoklu segment desteği | ✅ Var | ❌ Yok |
| Toolchain entegrasyonu | ✅ Tam | ❌ Sınırlı |

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
| `.vectors` | 0xFFC0 | 64 B | Kesme vektör tablosu |

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

| Dosya | Flash (text+data) | RAM (data+bss) |
|-------|-------------------|----------------|
| new-firmware.z1 | 72.051 B (~70,4 KB) | 6.042 B (~5,9 KB) |
| udp-client.z1 | 47.944 B (~46,8 KB) | 6.188 B (~6,0 KB) |
| udp-server.z1 | 49.070 B (~47,9 KB) | 6.318 B (~6,2 KB) |

- **text:** Flash'ta yer alan kod + sabit veri (`.text` + `.far.text` + `.rodata` + `.vectors`)
- **data:** Hem Flash'ta (LMA - yükleme adresi) hem RAM'de (VMA - çalışma adresi) yer alır
- **bss:** Yalnızca RAM'de yer alır; Flash'ta boyut bilgisi tutulur, reset sırasında sıfırlanır

### CC1352R Üzerinde Bellek Yerleşim Haritası

#### Flash (352 KB)

```
0x00000000 ┌─────────────────────────┐
           │  BIM Bootloader (16 KB) │
0x00004000 ├─────────────────────────┤  ← Slot A Başlangıcı
           │  .text     (~37,9 KB)   │
0x0000D770 ├─────────────────────────┤
           │  .rodata   (~13,5 KB)   │
0x00010D70 ├─────────────────────────┤
           │  .data LMA (336 B)      │
0x00010EC0 ├─────────────────────────┤
           │  .far.text (~18,6 KB)   │
0x00015938 ├─────────────────────────┤
           │  Boş Flash (~55 KB)     │
0x00023000 ├─────────────────────────┤  ← Slot A Sonu / Slot B Başlangıcı
           │  Slot B - OTA (124 KB)  │
0x00042000 ├─────────────────────────┤
           │  Kalan Boş Flash        │
0x0057F800 ├─────────────────────────┤
           │  CCFG (2 KB) — KRİTİK  │
0x0057FFFF └─────────────────────────┘
```

#### SRAM (80 KB)

```
0x20000000 ┌─────────────────────────┐
           │  .data VMA  (336 B)     │  ← Flash'tan kopyalanır
0x20000150 ├─────────────────────────┤
           │  .bss       (5.704 B)   │  ← Reset sırasında sıfırlanır
0x20001798 ├─────────────────────────┤
           │  Dinamik RAM (~74 KB)   │  ← Stack & Contiki buffer
0x20013FFF └─────────────────────────┘
```

---

## 3. Symbol / Function Analizi

```bash
$ msp430-nm -n new-firmware.z1 | grep " T \| t " | head -40
```
```
0000313e T main
00003376 T __ctors_end / __dtors_end
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

### Anlamlı Semboller

| Sembol | Adres | Tür | Anlamı |
|--------|-------|-----|--------|
| `main` | 0x313E | T (global kod) | Ana program başlangıcı |
| `port1_isr` | 0x353E | T | Port 1 GPIO kesme işleyicisi |
| `irq_p2` | 0x35C2 | T | Port 2 GPIO kesme işleyicisi |
| `cc2420_timerb1_interrupt` | 0x35FE | T | CC2420 radyo zamanlayıcı kesmesi |
| `timera1` / `timera0` | 0x3624 / 0x378C | T | Timer A kesme işleyicileri |
| `uart0_rx_interrupt` | 0x37AE | T | UART alım kesmesi |
| `watchdog_interrupt` | 0x37D8 | T | Watchdog zamanlayıcı kesmesi |
| `cc2420_arch_init` | 0x3AD6 | T | CC2420 radyo başlatma |
| `accm_init` | 0x3A68 | T | İvmeölçer başlatma |
| `__mulsi3` / `__udivhi3` | 0x337C+ | T | Yazılımsal çarpma/bölme rutinleri |

`T` (büyük harf): Global kod sembolü — dışarıdan erişilebilir  
`t` (küçük harf): Statik/lokal kod sembolü — yalnızca kaynak dosya içinde

---

## 7. ELF Yapısı Analizi

### Program Başlıkları (LOAD Segmentleri)

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

`.data` segmentinde `FileSiz (0x150) < MemSiz (0x1798)`: Flash'ta yalnızca başlangıç değerleri saklanır, `.bss` alanı RAM'de sıfırlanır.

---

## 8. Interrupt ve Donanım Analizi — Kesme Vektör Tablosu

```bash
$ msp430-readelf -x .vectors new-firmware.z1
```
```
0x0000ffc0  76337633 76337633 76337633 76337633
0x0000ffd0  76337633 76337633 76337633 76337633
0x0000ffe0  f4367837 3e35c235 76337633 7633ae37
0x0000fff0  24368c37 d8377633 fe357633 76330031
```

Little-endian okuma ile çözümlenen vektörler:

| Adres | Hedef | Fonksiyon |
|-------|-------|-----------|
| 0xFFE0 | 0x36F4 | `i2c_tx_interrupt` |
| 0xFFE2 | 0x3778 | `i2c_rx_interrupt` |
| 0xFFE4 | 0x353E | `port1_isr` |
| 0xFFE6 | 0x35C2 | `irq_p2` |
| 0xFFEE | 0x37AE | `uart0_rx_interrupt` |
| 0xFFF0 | 0x3624 | `timera1` |
| 0xFFF2 | 0x378C | `timera0` |
| 0xFFF4 | 0x37D8 | `watchdog_interrupt` |
| 0xFFF8 | 0x35FE | `cc2420_timerb1_interrupt` |
| **0xFFFE** | **0x3100** | **RESET vektörü — entry point** |

`0x3376` ile dolu vektörler `__br_unexpected_` (tanımsız kesme işleyicisi) gösterir.

**RESET vektörü (0xFFFE → 0x3100):** MSP430 reset sonrası bu adresten çalışmaya başlar. 0x3100 → C runtime ön işlemleri → 0x313E `main()`.

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

### Yorum

| Komut | Açıklama |
|-------|----------|
| `calla #addr` | MSP430X uzak alt program çağrısı (20-bit adres) |
| `clr r14` | r14 yazmaçını sıfırlar |
| `mov #val, r15` | r15'e sabit değer yükler |
| `cmp #3, &0x1160` | Bellek adresindeki node_id'yi 3 ile karşılaştırır |
| `jl $+30` | Küçükse 30 bayt ileriye atla (if bloğu) |
| `push #val` | Yığıta parametre iter (fonksiyon argümanı) |

**`calla` ve `.far.text` ilişkisi:** `calla` komutu MSP430X mimarisine özgü uzak çağrı komutudur. 64KB sınırını aşan `.far.text` bölümündeki fonksiyonlara ulaşmak için standart `call` yerine `calla` kullanılır. Bu, ELF-ANALIZ.md Bölüm 2'deki `.far.text` (~18,6 KB) varlığını doğrular.

**`cmp #3, &0x1160` → node_id kontrolü:** `udp-client.c` kaynak kodundaki `if(node_id == 2)` bloğunun derleme zamanı karşılığıdır. Aynı firmware farklı node_id değerlerine göre farklı kod bloklarını çalıştırır — şartnamede açıklanan düğüm kimliği mekanizması budur.

**Entry point netleştirme:** ELF başlığı `0x3100`'ı entry point olarak gösterir. Disassembly çıktısı `main()`'in `0x313E`'de başladığını kanıtlar. `0x3100`–`0x313D` arası C runtime ön işlemleri (.data kopyalama, .bss sıfırlama, yığın kurulumu) gerçekleştirir; ardından `main()` devreye girer.

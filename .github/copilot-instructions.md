# Mountain Mining Protocol (MMP) — Proje Bağlamı

> Bu dosya GitHub Copilot'a (veya repoyu devralan başka bir yapay zekaya) projeyi tanıtmak
> için yazıldı. `.github/copilot-instructions.md` olarak repoya koyabilirsin; Copilot bu tür
> dosyaları otomatik okuyup bağlam olarak kullanır.
>
> **Önemli not:** Bu belge, projenin bende (bu sohbette) bilinen en güncel tasarım kararlarını
> özetliyor. Projede ayrı bir oturumda (muhtemelen Claude Code ile gerçek toolchain kullanılarak)
> ek revizyonlar yapıldığı görünüyor — on-chain class-capacity sayaçları, STOP komutunun
> otomatik ödül basımı yapması, gerçek Metaplex koleksiyon doğrulaması ve gerçek üretilmiş bir
> program keypair'i eklenmesi gibi. Bu belge o revizyonların **açıklamasını** içeriyor ama o
> oturumdaki gerçek kod dosyalarına erişimim yok — yani burada anlattığım mimari ile repodaki
> güncel dosyalar arasında küçük farklar olabilir. Copilot'a vermeden önce repodaki gerçek
> `programs/mountain_mining/src/` dosyalarıyla karşılaştırman iyi olur.

## 1. Proje Nedir

Mountain Mining Protocol, **Solana/Anchor** üzerinde çalışan, NFT tabanlı pasif bir madencilik
(mining) protokolü. 100.000 adet "Mining Pass" NFT'si, sabit 1.000.000.000 (1 milyar) arzlı bir
**MMP** token'ını 20 yıllık bir süreye yayarak madenciliğe açıyor. Proje başlangıçta Base
(Solidity/EVM) üzerinde tasarlanmıştı, sonra **Solana'ya pivot edildi** — mevcut ve güncel hedef
zincir Solana.

NFT bir finansal getiri vaadi değil; sadece madencilik sistemine erişim hakkı.

## 2. Teknoloji Yığını

- **On-chain program:** Rust + Anchor framework (Solana)
- **Token standardı:** Klasik SPL Token (Token-2022 değil — gereksiz extension riski almamak
  için bilinçli tercih)
- **NFT metadata:** Metaplex Token Metadata — koleksiyon doğrulaması **gerçek, bağımlılıksız
  bir implementasyonla** yapılıyor (`metaplex.rs`); `mpl-token-metadata` crate'i solana-program
  versiyon çakışması riski nedeniyle bilinçli olarak kullanılmadı, doğrulama mantığı elle
  (dependency-free) yazıldı.
- **Frontend:** Next.js + TypeScript, `@solana/wallet-adapter-*` ile cüzdan bağlantısı (Phantom
  minimum test edilen cüzdan)
- **Test:** Anchor/Mocha entegrasyon testleri + saf Rust `cargo test` ile reward-math testleri

## 3. Token Ekonomisi (MMP)

| Parametre | Değer |
|---|---|
| Toplam MMP arzı | 1.000.000.000 MMP (sabit tavan) |
| Mint/burn/pause/admin backdoor | Yok — tamamen trust-minimized |
| Mining süresi (protokol geneli hedef) | 20 yıl = 630.720.000 saniye |
| Mining Pass NFT toplam adedi | 100.000 |
| Sınıf sayısı | 7 (Stone, Obsidian, Iron, Steel, Titanium, Diamond, Mithril) |
| Toplam ağırlıklı güç (TOTAL_POWER) | 486.000 |

### Sınıf tablosu

| Sınıf | Adet | Çarpan (multiplier) | Adet × Çarpan |
|---|---|---|---|
| Stone | 40.000 | 1× | 40.000 |
| Obsidian | 25.000 | 2× | 50.000 |
| Iron | 15.000 | 4× | 60.000 |
| Steel | 10.000 | 8× | 80.000 |
| Titanium | 6.000 | 16× | 96.000 |
| Diamond | 3.000 | 32× | 96.000 |
| Mithril | 1.000 | 64× | 64.000 |
| **Toplam** | **100.000** | — | **486.000** |

Bu tablo on-chain'de değiştirilemez bir sabit (`CLASS_TABLE`) olarak tutuluyor ve derleme
zamanında (`const _: () = { ... }` ile) toplamların doğruluğu otomatik doğrulanıyor.

### NFT dağıtımı (3 faz)

- 10.000 ücretsiz airdrop
- 10.000 erken erişim (early access)
- 80.000 halka açık satış (~$2 hedef fiyat, SOL cinsinden; USD değeri on-chain hard-code
  edilmiyor)

Sınıf ataması **alıcı tarafından seçilemiyor** — protokol tarafından (verifiable randomness ile)
atanıyor.

## 4. Ödül Formülü (ÖNEMLİ — düzeltilmiş versiyon)

**İlk taslakta** 20 yıl "her NFT için sabit bir süre sonu" gibi ele alınmıştı. Bu **düzeltildi**:

> 20 yıl, TÜM 486.000 birimlik gücün kesintisiz madencilik yaptığı **teorik maksimum
> kullanım senaryosunda** Mountain'ın toplam ömrü için bir **hedef/asgari** süre — belirli bir
> NFT için zorunlu bir bitiş tarihi veya kişisel ömür sınırı DEĞİL. Madencilik oturumları
> (mining sessions) süresiz şekilde durdurulup yeniden başlatılabilir; kümülatif muhasebe
> (cumulative accounting) ile takip edilir.

Formül (integer/fixed-point, floating point yok):

```
reward = floor(
    elapsed_seconds * class_multiplier * TOTAL_SUPPLY
    / (MINING_PERIOD_SECONDS * TOTAL_POWER)
)
```

- `elapsed_seconds`: son claim'den (veya mining başlangıcından) bu yana geçen süre
- Tüm ara çarpımlar taşma riskine karşı `u128` ile hesaplanıyor (Rust `u64` girişler
  genişletiliyor)

### Global 1B tavan uygulaması — DÜZELTİLMİŞ yaklaşım

İlk taslakta tavan aşılırsa işlem **revert** ediliyordu. **Güncel/düzeltilmiş tasarım:**

> Global 1 milyar MMP tavanı, claim anında `min(calculated_reward, remaining_supply)` ile
> uygulanıyor — yani hesaplanan ödül kalan arzdan fazlaysa işlem başarısız olmuyor, sadece
> kalan arz kadar mint ediliyor. Bu, protokolün son aşamasında kullanıcıların "tavana çok
> yakınız, claim işlemim revert olabilir" belirsizliğiyle karşılaşmamasını sağlıyor.

## 5. Madencilik Yaşam Döngüsü (lifecycle)

```
NFT SAHİPLİĞİ → MINE (NFT program custody'sine kilitlenir, mining_state PDA açılır/yenilenir)
    → PASİF MADENCİLİK (blockchain'de sürekli işlem gerekmiyor, zaman damgasından hesaplanıyor)
    → CLAIM (tekrarlanabilir — istenildiği kadar claim edilebilir)
    → STOP/RELEASE (NFT sahibine geri döner)
```

### STOP komutu — GÜNCEL revizyon

İlk taslakta claim ve release ayrı komutlardı; release'den önce bekleyen ödülün claim
edilmiş olması gerekiyordu. **Güncel revizyonda STOP komutu artık otomatik "settle"
yapıyor**: NFT'yi kilitten çıkarmadan önce, o ana kadar biriken ödülü otomatik olarak mint
edip kullanıcıya gönderiyor — yani kullanıcının ayrıca claim çağırmasına gerek kalmadan tek
işlemde hem ödül alınıyor hem NFT serbest bırakılıyor.

### Kilitleme (lock) mekanizması

NFT, madencilik sırasında **programın kontrolündeki bir escrow/custody hesabına** taşınıyor
(PDA tarafından yönetilen bir token hesabı). Bu sayede:

- Sahibi imzalamadan normal transfer/satış imkansız (frontend'de buton devre dışı bırakmak
  değil, gerçek on-chain kilitleme)
- Çift madencilik (double mining), çift claim, başkasının ödülünü claim etme gibi saldırılar
  hesap kısıtlamalarıyla (constraint) engelleniyor

## 6. Güvenlik Mimarisi

- **Hiçbir admin/owner anahtarı** MMP mint edemiyor, kullanıcı bakiyesini değiştiremiyor,
  NFT çalamıyor veya claim'i override edemiyor.
- MMP mint authority'si **sadece bir program PDA'sı** — bu PDA'yı imzalayan tek kod yolu
  `claim`/`stop` instruction'ları, ve her ikisi de mint'ten önce tavan kontrolü yapıyor.
- **On-chain class-capacity enforcement** (güncel revizyon): `ProtocolConfig` hesabında her
  sınıf için canlı bir sayaç tutuluyor, mint sırasında sadece kalan kapasitesi olan sınıflar
  arasından seçim yapılıyor (ilk taslaktaki "orijinal oran üzerinden ağırlıklı çekiliş, kalan
  kapasiteyi takip etmeyen" zayıf yaklaşım düzeltildi).
- Upgrade authority / immutability kararı deployment aşamasında açıkça belgeleniyor (varsayılan
  olarak sessizce upgradeable bırakılmıyor).
- Program ID artık **sandbox'ta gerçekten üretilmiş bir ed25519 keypair** ile `declare_id!` ve
  `Anchor.toml` içine yazılı — ama bu **gerçek deploy öncesi yeniden üretilmesi gereken**,
  sandbox kökenli bir anahtar olarak işaretli (placeholder değil, gerçek keypair, ama üretim
  ortamı için güvenilir sayılmamalı).

## 7. Program Hesapları (PDA'lar) — genel şema

- `Mountain` / `ProtocolConfig` (singleton): global durum — toplam mint edilen MMP, mint
  fazları sayaçları, sınıf başına kalan kapasite sayaçları, mint authority bump'ları
- `MiningState` (her NFT için bir tane, `[mining_state_seed, pass_mint]`): sahip, sınıf ID,
  mining durumu, mining_start, last_claim_ts, lifetime_claimed
- Mint authority PDA: sadece imzalama yetkisi, veri tutmuyor
- Pass custody PDA: madencilik sırasında NFT'yi tutan escrow hesabının otoritesi

## 8. Yapılmaması Gerekenler (spesifikasyonun kesin kısıtları)

Staking yok, governance yok, DAO yok, referral ödülü yok, ikinci bir token yok, keyfi vergi/gizli
ücret yok, gizli admin yetkisi yok, upgrade backdoor yok. **Mainnet'e deploy YOK** — proje
bilinçli olarak devnet-ready seviyesinde tutuluyor; mainnet kararı ayrı, açık bir adım.

## 9. Bilinen Sınırlamalar / Dürüstçe Belirtilmiş Eksikler

- Sınıf atamasındaki rastgelelik kaynağı (`SlotHashes` tabanlı) production-grade bir VRF değil;
  mainnet öncesi Switchboard/ORAO gibi bir oracle VRF ile değiştirilmesi gerekiyor.
- Frontend'in mine/claim/stop butonları, gerçek bir `anchor build` sonucu üretilen IDL
  dosyasına (`target/idl/mountain_mining.json`) bağlı — bu dosya olmadan uçlar "wired değil"
  olarak işaretli, sahte/başarılı görünen bir UI ile gizlenmiyor.
- FAQ, Statistics, Project Info, How-It-Works sayfaları içerik kararları senden beklediği için
  iskelet halde.

## 10. Copilot'tan Beklenen Yaklaşım

- Ekonomik sabitleri (arz tavanı, sınıf tablosu, 20 yıllık süre, faz adetleri) **asla sessizce
  değiştirme** — belirsizlik varsa, ekonomik değişmezleri koruyan yorumu seç.
- Ödül matematiğinde floating point kullanma; `u128` ara hesaplama + `checked_*` operasyonları
  kullan.
- Yeni bir admin yetkisi, upgrade yolu veya gizli mint fonksiyonu ekleme.
- Var olan güvenlik kısıtlamalarını (constraint'leri) gevşetme; her yeni instruction için aynı
  düzeyde sahiplik/durum/tavan kontrolü ekle.

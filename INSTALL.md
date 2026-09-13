# INSTALL.md — Kaynaktan Derleme ve Kurulu Anytype'ı Güncelleme

Bu belge, **anytype-heart** (backend) + **anytype-ts** (arayüz) kaynaklarından derleyip
kurulu `/Applications/Anytype.app` içine güncelleme yapmanın tam akışını anlatır.

## Mimari

```
anytype-heart (Go)                     anytype-ts (TypeScript/Electron)
cmd/grpcserver ──derle──► dist/server ──► ../anytype-ts/dist/anytypeHelper
                                        ../anytype-ts/dist/lib/ (protolar)
                anytype-ts/dist/ (UI çıktısı)
                        │
                        ▼
         scripts/deploy.go  ──repack──►  /Applications/Anytype.app/Contents/Resources/
                                           ├── app.asar                (UI)
                                           └── app.asar.unpacked/dist/anytypeHelper (backend)
```

## Gereksinimler (macOS)

| Araç | Sürüm | Kullanım |
|---|---|---|
| Go | ≥ 1.26.5 (`go.mod`) | backend derlemesi (CGO: tantivy) |
| Xcode Command Line Tools | — | C derleyicisi (cc, clang) |
| bun | ≥ 1.4 | UI bağımlılıkları ve derleme |
| node + npm | — | `protos-js` eklentileri, `npx asar` |

Kontrol: `go version && bun --version && command -v npx`

## İlk Kurulum

```bash
# 1. Repoları klonlayın (yan yana durmalılar; anytype-heart Makefile'ı
#    varsayılan olarak ../anytype-ts hedefine kurar — CLIENT_DESKTOP_PATH)
git clone https://github.com/0xf61/anytype-heart.git   # kişisel fork (origin)
cd anytype-heart
git checkout GO-3192-l3-ttl                            # PR #2357 ile eşleşen branch (aşağıya bakın)
cd ..
git clone https://github.com/alicangnll/anytype-ts.git # kişisel fork (origin) = PR #2357

# 2. Kurulu Anytype uygulaması /Applications/Anytype.app konumunda olmalı.
#    (deploy.go --app-path ile başka konum belirtilebilir)
```

> **Not:** İlk backend derlemesinde `make` otomatik olarak `deps/libs/` altına
> tantivy ön-derleme kütüphanelerini indirir (`anyproto/tantivy-go` release'leri,
> `go.mod`'daki sürümle eşleşir) ve `govvv` / `goderive` / protoc eklentilerini
> `deps/` içine derler. İnternet bağlantısı gerekir.

## Backend Branch'i: `GO-3192-l3-ttl`

UI tarafındaki **PR #2357** (`alicangnll/anytype-ts` → `anyproto/anytype-ts`, `develop`)
ile backend tarafındaki **`GO-3192-l3-ttl`** branch'i (`0xf61/anytype-heart`) **eşleşen
bir settir** — ikisi birlikte kurulmalıdır. Branch, `develop`'ın üzerine 3 commit ekler:

| Commit | Getirdiği |
|---|---|
| `c4540187d` | VPN/overlay ağlar için opt-in statik peer'ler (`localdiscovery`) |
| `4c1adc0dc` | Statik peer kodunun refactor'u (düşük karmaşıklık) |
| `05c3b6b5a` | Yerel peer zaman aşımı ve ban TTL'nin `config.json`'dan ayarlanması |

Bu branch'in amacı **dosya/fotoğraf yüklememe sorununu** hedefler: L3 dosya katmanında
(`rpcstore`) yerel peer'lardan dosya çekme zaman aşımı (`LocalPeerTimeoutMs`, varsayılan
1000 ms) ve erişilemeyen peer'ın ne kadar banlanacağı (`LocalPeerBanTtlSec`, varsayılan
300 sn) yapılandırılabilir hale gelir. Bu ayarlar ve statik peer yönetimi, PR #2357'nin
Ayarlar → Network ekranından da yönetilebilir.

```bash
# Branch'e geçme (klon sonrası)
git fetch origin
git checkout GO-3192-l3-ttl            # yerel takip dalı otomatik oluşur

# Branch'i güncelleme
git pull origin GO-3192-l3-ttl

# develop'a geri dönme (branch'siz, resmi hale yakın kurulum için)
git checkout develop
```

> **Uyarı:** UI PR #2357 ile backend `develop`'ı karıştırmayın; statik peer ve
> TTL ayarları UI'da görünür ama backend'de karşılığı yoksa etkisiz kalır.

## Derleme + Kurulum Akışı (önerilen sıra)

```bash
# 1) UI'ı derle (anytype-ts)
cd ~/Documents/GitHub/anytype-ts
make build                                # bağımlılık + derleme → dist/

# 2) Backend'i derle ve anytype-ts'ye kur (anytype-heart)
cd ~/Documents/GitHub/anytype-heart
make install-dev-js                       # dist/server → ../anytype-ts/dist/anytypeHelper
                                           # + protolar → ../anytype-ts/dist/lib

# 3) Kurulu uygulamaya dağıt (anytype-ts)
cd ~/Documents/GitHub/anytype-ts
make copy                                 # = go run scripts/deploy.go --no-build
```

Kısa yol: anytype-ts içinde `make install` = adım 1 + adım 3 birlikte
(backend değiştiyse adım 2'yi yine ayrıca çalıştırın).

**Sıra neden önemli:** UI derlemesi `dist/` içindeki diğer dosyaları silmez,
ancak backend kurulumu (`install-dev-js`) `dist/anytypeHelper` ve `dist/lib`
yazar. En garantisi: önce UI, sonra backend, en son dağıt.

## Geliştirme Döngüsü

**Sadece arayüz değişti (TS):**
```bash
cd ../anytype-ts && make install          # derle + dağıt
```

**Sadece backend değişti (Go):**
```bash
cd ../anytype-heart && make install-dev-js
cd ../anytype-ts && make copy             # yeniden dağıt (helper'ı da taşır)
```

**Protolar değişti:** `make install-dev-js` protoları da yenileyip kopyalar;
UI tarafında `dist/lib` güncellenir.

**Sunucuyu elle çalıştırmak (istemci olmadan):**
```bash
cd anytype-heart && make run-server       # dist/server'ı başlatır
```

## Doğrulama

Dağıtımdan sonra kurulumun gerçekten gerçekleştiğini kontrol edin:

```bash
# 1) Yeni UI kodu asar içinde mi? (koddaki yeni bir string'i arayın;
#    örn. medya düzeltmesindeki localStorage anahtarı)
grep -ac "anytype_max_image_retries" /Applications/Anytype.app/Contents/Resources/app.asar
# > 0 dönmeli

# 2) Backend binary'si taze mi? (boyut, anytype-heart/dist/server ile aynı olmalı)
ls -la /Applications/Anytype.app/Contents/Resources/app.asar.unpacked/dist/anytypeHelper
ls -la ~/Documents/GitHub/anytype-heart/dist/server

# 3) Binary GO-3192-l3-ttl'den mi derlendi? (branch'e özgü sembol arayın)
strings -a /Applications/Anytype.app/Contents/Resources/app.asar.unpacked/dist/anytypeHelper \
  | grep -m1 "LocalPeerBanTtl"
# > "config.(*Config).LocalPeerBanTtl" satırı dönmeli

# Not: "build on ..." commit damgası govvv ile basılır; taze klonlarda
# ldflags boş kalabildiğinden en güvenilir yöntem sembol kontrolüdür.
```

Son adım zorunlu: **Anytype'ı tamamen kapatıp yeniden açın** (değişiklikler
ancak yeniden başlatmada yüklenir).

## Sorun Giderme

- **`dist/anytypeHelper` güncellenmiyor** — eski sürümlerdeki `deploy.go`,
  `asar pack --unpack` çıktısını `/tmp`'de bırakıp `app.asar.unpacked`'ı
  kopyalamıyordu. Betik düzeltildi; eski bir kopya kullanıyorsanız unpacked
  dizinini elle güncelleyin:
  ```bash
  cp ~/Documents/GitHub/anytype-ts/dist/anytypeHelper \
     /Applications/Anytype.app/Contents/Resources/app.asar.unpacked/dist/anytypeHelper
  ```
- **Geri alma** — `deploy.go` her dağıtımdan önce `app.asar.bak` yedeği alır:
  ```bash
  cp /Applications/Anytype.app/Contents/Resources/app.asar.bak \
     /Applications/Anytype.app/Contents/Resources/app.asar
  ```
- **`/Applications` yazma izni** — dağıtım komutunun izinle reddedilmesi
  durumunda yukarıdaki `cp` adımlarını elle çalıştırabilirsiniz.
- **Tantivy sürüm uyuşmazlığı** — `make`, `go.mod`'daki tantivy-go sürümünü
  `deps/libs/.verified` ile karşılaştırır; farklıysa otomatik indirir.
  Hash hatası alırsanız: `make download-tantivy-all-force`
- **Protoc eklentileri kurulamıyor** — `makefiles/setup.mk` içindeki
  `setup-protoc-js` hedefi `npm -D install` çalıştırır; node/npm kurulu olmalı.
- **Uygulama değişiklikleri görmüyor** — Anytype'ı yeniden başlattınız mı?
  (menüden Çık; arka planda kalırsa `killall Anytype`)

## Repoları Güncelleme

```bash
cd ~/Documents/GitHub/anytype-heart
git checkout GO-3192-l3-ttl                       # eşleşen backend branch'i
git fetch origin && git merge origin/GO-3192-l3-ttl
# develop tabanını güncellemek için:
#   git fetch upstream && git merge upstream/develop  (PR #2357 rebase'i gerektirebilir)
git fetch upstream                                # resmi anyproto (isteğe bağlı)

cd ~/Documents/GitHub/anytype-ts
git fetch origin && git merge origin/develop      # PR #2357 kaynağı
git fetch upstream
```

Her iki repo da derleme + dağıtım akışını yeniden çalıştırmayı gerektirir.

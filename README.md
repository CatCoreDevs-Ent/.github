# CatCoreDevs-Ent/.github

Org'un ortak GitHub ayarları. `profile/README.md` org sayfasında görünür.

## Ortak imaj hattı: `.github/workflows/build-image.yml`

Reusable workflow (`workflow_call`). Uygulama repoları commit SHA'sıyla sabitleyerek çağırır; hazır şablon
`workflow-templates/app-ci.yml` (Actions → New workflow → "CatCore uygulama CI'ı").

```
gitleaks → test (runner komutu ya da Dockerfile'ın test aşaması) → build (uzak BuildKit)
  → registry'ye ETİKETSİZ push (digest) → Trivy → cosign imza + CycloneDX SBOM (attest) → etiket sha-<12>
```

- Sadece varsayılan dala push'ta ve elle çalıştırmada yayımlar; PR'da ve diğer dallarda test + build.
- Etiket en son konur: taranmamış, imzasız ya da düzeltmesi olan kritik açığı bulunan digest etiket almaz ve
  dağıtıma gidemez (dağıtım yalnızca `sha-*` etiketlerini izler). İstisna: repo kökünde gerekçeli `.trivyignore`.
- İmza anahtarlı (cosign), şeffaflık kaydına yazılmaz; imza ve SBOM OCI 1.1 referrer'ı olarak imajın yanında durur.
  Doğrulama: `cosign verify --key cosign.pub --insecure-ignore-tlog <imaj>` (açık anahtar platform reposunda).
- Yalnızca org'un özel repolarında çalışır (self-hosted runner `catcore-arc`); herkese açık repodan çağrılırsa iş
  reddedilir. Kimlikler runner'ın ortamından gelir, çağıranın secret vermesi gerekmez.
- Araçlar (cosign, Trivy, crane, gitleaks) sabit sürüm ve sha256 ile indirilir; action'lar commit SHA'sıyla sabit.

| Girdi | Varsayılan | |
|---|---|---|
| `image` | (zorunlu) | `<uygulama>/<bileşen>` |
| `context`, `dockerfile`, `target` | `.`, `<context>/Dockerfile`, son aşama | |
| `platform` | `linux/amd64` | tek platform |
| `build-args` | | satır başına `KEY=VALUE` (sır değil) |
| `test-command` / `test-target` | | runner'da komut / BuildKit'te Dockerfile aşaması |
| `trivy-severity` | `CRITICAL` | ya da `HIGH,CRITICAL` |
| `push` | `true` | `false`: hiç yayımlama |
| `timeout-minutes` | `60` | |
| `private-repos` | | git bağımlılığı olarak okunan özel repolar (virgül ya da satırla, sahipsiz: `ccp-sdk`); BuildKit'te `id=github_token` ve `id=git_config` (v1.4.0) |
| `private-go-repos` | | `private-repos`'un eski adı (v1.3.0, Go modülleri); ikisi birleşir |
| `created-from-commit` | `false` | `true`: imajın `created` zamanı commit zamanı (`SOURCE_DATE_EPOCH`, v1.4.0); bedeli aşağıda |
| `npm-github-packages` | `false` | `true`: GitHub Packages (npm) için işin `GITHUB_TOKEN`'ı; BuildKit'te `id=npm_token` |

Çıktılar: `image`, `digest`, `tag`. Reusable workflow `secrets:` almaz; çağıranın secret vermesi gerekmez.

### Özel bağımlılıklar (git bağımlılıkları, Go modülü, GitHub Packages)

Girdi verilmezse hiçbir şey değişmez. Verilirse:

- **Git bağımlılıkları (`private-repos`; eski adı `private-go-repos`, ikisi birleşir):** `catcore-ci-deps` GitHub
  App'inin kimliği (App ID + özel anahtar) runner pod'unda (`CI_DEPS_APP_DIR`, infra: OpenTofu `ci_deps_github_app` →
  `arc-runners/github-app-ci-deps`), zot-ci ve cosign gibi. Adım içinde App anahtarıyla JWT (9 dk) imzalanır, org
  kurulumundan **yalnızca istenen repolara `contents: read`** kurulum token'ı alınır (ömrü en çok 1 saat) ve build
  biter bitmez iptal edilir. App yalnızca seçili repolara kurulu (`catcore-sdk`, `ccp-kit`, `ccp-sdk`), tek izni
  Contents: Read-only. Kurulumda olmayan bir repo istenirse adım GitHub'ın mesajıyla (HTTP 422) düşer.
- **`git_config` sırrı (v1.4.0):** token **içermeyen** hazır git ayarı. `GIT_CONFIG_GLOBAL=/run/secrets/git_config`
  ile pnpm, cargo ve go aynı satırla çalışır:
  - kimlik yardımcısı yalnızca `https://github.com/CatCoreDevs-Ent/…` yollarında ve yalnızca `get`'te yanıt verir;
    token'ı o anda `/run/secrets/github_token`'dan okur (başka yere bağladıysan `GITHUB_TOKEN_FILE=<yol>`). Token
    URL'ye, komut argümanına, `.git/config`'e ve git'in hata iletilerine girmez (BuildKit'in build geçmişi RUN
    log'larını maskesiz saklar);
  - `ssh://git@github.com/CatCoreDevs-Ent/…`, `git+ssh://…` ve `git@github.com:CatCoreDevs-Ent/…` adresleri
    https'e çevrilir (pnpm kilidindeki, Cargo'daki ssh adresleri de çalışır);
  - imajda git ≥ 2.26 gerekir (Debian bookworm 2.39 denendi).
- **npm (`npm-github-packages`):** işin `GITHUB_TOKEN`'ı. Bu workflow workflow düzeyinde `permissions` tanımlamaz: token
  çağıran işin izinleriyle gelir. İzni çağıran verir (`packages: read`); burada istenseydi izni vermeyen her çağıran
  başlamadan düşerdi. Paketin çağıran repoya Read erişimi olmalı (paket → Package settings → Manage Actions access).
- Token'lar `--secret id=…,src=…` ile yalnızca BuildKit'in `RUN --mount=type=secret` adımlarına gider: build-arg'a,
  log'a (maskeli, hiç basılmaz), cache export'a ve imaj katmanına girmez. Runner'da `0600` dosyada durur, `test-command`
  adımından sonra yazılır (o adım görmez), build biter bitmez silinir. `test-target` aşaması (BuildKit) da alır.
- Eksik kimlik açık hata verir: `private-repos` verildi ama runner'da App kimliği yoksa iş ilk adımda düşer; App
  repoya kurulu değilse token adımı GitHub'ın mesajıyla düşer.
- Token'ı katmana yazmak Dockerfile'ın elindedir: secret'ı yalnızca `RUN --mount=type=secret` ile kullan, dosyaya
  (`.npmrc`, `.netrc`, `.gitconfig`) ya da `ENV`'e yazma. RUN root değilse mount'a `uid=<n>` ver (varsayılan 0400
  root). İndirmeyi Dockerfile'ın başında yap: token en çok 1 saat geçerli.
- **Kabul edilen risk:** `catcore-arc`'ta koşan her org reposu (yalnızca özel repolar) bu App token'ını üretebilir;
  yetkisi `catcore-sdk`, `ccp-kit` ve `ccp-sdk`'yı okumak. Zot'a push ve cosign imzası gibi ortak CI kimliğiyle aynı
  sınıf. BuildKit'in önbellek bağları (`--mount=type=cache,id=…`) repolar arasında paylaşılır: aynı id'yi bağlayan
  başka bir Dockerfile indirilen kaynağı görebilir (token oraya girmez).

Çağıran (iş düzeyinde izinler; `secrets` yok):

```yaml
jobs:
  image:
    permissions:
      contents: read
      packages: read            # yalnızca npm-github-packages için
    uses: CatCoreDevs-Ent/.github/.github/workflows/build-image.yml@<commit-sha> # v1.4.0
    with:
      image: urun/api
      private-repos: catcore-sdk      # ya da ccp-sdk, ccp-kit (virgülle)
      npm-github-packages: true
```

Git bağımlılığı (pnpm, `git_config` ile; `required=true`: secret gelmezse build `secret … not found` ile düşer):

```dockerfile
# Derleme aşamasında git gerekir (apt-get install git); runtime aşamasına geçmez.
RUN --mount=type=cache,id=<uygulama>-pnpm,target=/pnpm-store,sharing=locked \
    --mount=type=secret,id=github_token,required=true \
    --mount=type=secret,id=git_config,required=true \
    GIT_CONFIG_GLOBAL=/run/secrets/git_config GIT_TERMINAL_PROMPT=0 \
    pnpm install --frozen-lockfile --store-dir /pnpm-store
```

`package.json`: `"@ccp/sdk": "git+ssh://git@github.com/CatCoreDevs-Ent/ccp-sdk.git#v0.N.0"` (alt dizin:
`#v0.N.0&path:/<dizin>`). `pnpm-lock.yaml`'da çözüm `type: git` olmalı (`resolution: {commit: …, repo: …, type:
git}`); `tarball: https://codeload.github.com/…` olursa build kırılır, codeload'a kimlik gitmez.

Cargo (`.cargo/config.toml` `[net] git-fetch-with-cli = true` ya da ortamda `CARGO_NET_GIT_FETCH_WITH_CLI=true`):

```dockerfile
RUN --mount=type=cache,id=<uygulama>-cargo-registry,target=/usr/local/cargo/registry,sharing=locked \
    --mount=type=cache,id=<uygulama>-cargo-git,target=/usr/local/cargo/git,sharing=locked \
    --mount=type=secret,id=github_token,required=true \
    --mount=type=secret,id=git_config,required=true \
    GIT_CONFIG_GLOBAL=/run/secrets/git_config GIT_TERMINAL_PROMPT=0 CARGO_NET_GIT_FETCH_WITH_CLI=true \
    cargo fetch --locked
```

Go modülü `git_config` ile (`GIT_CONFIG_GLOBAL=/run/secrets/git_config GOPRIVATE=github.com/CatCoreDevs-Ent/* go mod
download`) ya da aşağıdaki v1.3.0 örneğiyle çalışır; ikisi de desteklenir.

v1.3.0 örnekleri (`# syntax=docker/dockerfile:1`; `env=` için Dockerfile 1.10+). Token yalnızca o `RUN`'ın ortamında,
hiçbir dosyaya yazılmaz:

```dockerfile
# Go: git, CatCoreDevs-Ent URL'lerini token'lı adrese çevirir (yalnızca bu süreçte; Go'nun vcs önbelleğine girmez).
RUN --mount=type=secret,id=github_token,env=GITHUB_TOKEN,required=true \
    GIT_CONFIG_COUNT=1 \
    GIT_CONFIG_KEY_0="url.https://x-access-token:${GITHUB_TOKEN}@github.com/CatCoreDevs-Ent/.insteadOf" \
    GIT_CONFIG_VALUE_0="https://github.com/CatCoreDevs-Ent/" \
    GOPRIVATE=github.com/CatCoreDevs-Ent/* \
    go mod download

# npm (pnpm): geçici .npmrc tmpfs'te ve token'ı değil ${NPM_TOKEN} referansını taşır.
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN,required=true \
    --mount=type=tmpfs,target=/run/npmrc \
    printf '%s\n' '@catcoredevs-ent:registry=https://npm.pkg.github.com' \
                  '//npm.pkg.github.com/:_authToken=${NPM_TOKEN}' >/run/npmrc/.npmrc && \
    NPM_CONFIG_USERCONFIG=/run/npmrc/.npmrc pnpm install --frozen-lockfile
```

Repodaki `.npmrc` zaten `//npm.pkg.github.com/:_authToken=${NPM_TOKEN}` biçimindeyse tmpfs'e gerek yok:
`RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN,required=true pnpm install --frozen-lockfile`.
`required=true`: secret gelmezse build `secret … not found` ile düşer (boş token'la sessizce devam etmez).

### İmajın yaratılış zamanı (`created-from-commit`)

`true` verilirse checkout'tan sonra `SOURCE_DATE_EPOCH = git log -1 --format=%ct` ayarlanır; BuildKit imaj
config'inin ve geçmişinin `created` alanını commit zamanına eşitler (dosya zamanları değişmez). Kargo'nun NewestBuild
sırası böylece build sırasını değil commit sırasını izler.

- **Bedel:** BuildKit epoch'u `WORKDIR`'ın oluşturma işlemine yazar: ilk `WORKDIR`'dan sonraki katmanlar her
  commit'te önbelleksiz koşar. Bağımlılık indirmesi önbellek bağıyla (`--mount=type=cache`) hafifler.
- Committer zamanı eski bir commit (eski dalın fast-forward'ı) ya da geçişteki ilk build, önceki imajın build
  zamanlı `created`'ından eski görünebilir; o zaman Kargo yeni Freight çıkarmaz.
- `ARG SOURCE_DATE_EPOCH` diyen Dockerfile değeri RUN ortamında da görür; demeyen görmez.

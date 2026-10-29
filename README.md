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

Çıktılar: `image`, `digest`, `tag`.

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

## Score render hattı: `.github/workflows/score-render.yml`

Reusable workflow. Uygulamanın Score dosyalarını (`score.yaml`, `<bileşen>.score.yaml`) platformun Score paketiyle
(`cr.catcore.dev/score/k8s:<sürüm>`: infra'daki `kubernetes/score/k8s`, provisioner'lar + patch şablonu + `render.py`)
staging ve production için Kubernetes manifestlerine çevirir. Şablondaki `render` işi `needs: image` ile çağırır.

```
Score paketi (crane export) → imaj digest'leri (cr.catcore.dev/<app>/<bileşen>:sha-<12>) → render.py (staging, production)
  → cr.catcore.dev/<app>/rendered'a ETİKETSİZ push (digest) → cosign imza → etiket sha-<12>
```

- Paket tek katmanlı (FROM scratch): `staging/manifests.yaml`, `production/manifests.yaml`. Kargo `sha-*` etiketini
  görür ve katmanı deploy reposunun `rendered/<app>/<ortam>`'ına yazar.
- `metadata.name`: `<app>-<bileşen>`; `image: .` olan iş yükünün imajı aynı commit'in `sha-<12>` etiketinin digest'i.
  Etiket yoksa (imaj işi bitmemiş ya da düşmüş) render yayımlanmaz.
- Ortam override'ları (`<dosya>.staging.yaml`, `<dosya>.production.yaml`) `render.py` tarafından kendiliğinden uygulanır;
  `score-files`'a yazılmaz.
- Sadece varsayılan dala push'ta ve elle çalıştırmada yayımlar; PR'da ve diğer dallarda örnek digest'lerle render
  edilir (Score dosyaları ve provisioner'lar yine denetlenir, özet iş sayfasında). Aynı commit'in etiketi varsa atlanır.
- Araçlar (crane, cosign, score-k8s, PyYAML) sabit sürüm ve sha256 ile; PyYAML'ın saf Python modülü kaynak paketten
  açılır (runner imajında pip yok).

| Girdi | Varsayılan | |
|---|---|---|
| `app` | (zorunlu) | namespace `<app>-<ortam>` |
| `score-files` | `score.yaml` | satır başına bir yol (repo köküne göre) |
| `package` | `cr.catcore.dev/score/k8s:0.1.0` | Score paketinin sürümü ya da digest'i |
| `push` | `true` | `false`: hiç yayımlama |
| `timeout-minutes` | `20` | |

Çıktılar: `image` (`cr.catcore.dev/<app>/rendered`), `digest`, `tag` (yayımlanmadıysa boş).

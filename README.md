# EAS Local Runner — GitHub Actions şablonu

Expo / React Native uygulamalarını **kendi Mac'inizde** (self-hosted runner) `eas build --local`
ile derleyen, çıktıyı artifact ve tag'de **GitHub Release** olarak bırakan yeniden kullanılabilir
iş akışı. GitLab'daki
[eas-local-runner-template](https://gitlab.com/emrahyurttutan/eas-local-runner-template)'in
GitHub karşılığıdır; mağaza aşamaları (metadata, submit, sürüm notu) bu sürümde yoktur.

## Temel özellikler

- **Tek dosya, `uses:` ile çağrılır.** Job'lar, sıra ve artifact'ler şablonda; tüketici depo
  15 satır yazar.
- **Kendi makinenizde derleme.** EAS sunucusuyla tek temas: proje kimliği ve (EAS'ta tutuluyorsa)
  anahtar/profil indirme. Dakika ücreti yok; Gradle ve CocoaPods önbelleği diskte kalır.
- **Hızlı düşme.** Uzun kurulum başlamadan `app-dir`, `eas.json`, profil ve platform adları
  denetlenir.
- **Paket yöneticisi saptama.** bun / pnpm / yarn / npm lock dosyasından; monorepo'da kurulum
  repo kökünde yapılır.
- **Keychain ve JVM daemon temizliği.** GitLab şablonundaki iki tuzak burada da ele alınır:
  yarım kalan build'lerin login keychain'de bıraktığı sertifikalar (xcodebuild exit 65) ve
  2 saat ayakta kalıp RAM tutan Gradle/Kotlin daemon'ları.
- **Tag = sürüm.** `v1.2.0` push edilince çıktı `<repo>-<platform>-v1.2.0.<apk|aab|ipa>` adıyla
  tag'in Release'ine eklenir. Elle tetiklemede yalnız artifact üretilir.

## Kullanım

Tüketici depoya `.github/workflows/mobile-release.yml` (tam örnek:
[examples/mobile-release.yml](examples/mobile-release.yml)):

```yaml
name: Mobil sürüm

on:
  push:
    tags: ['v*']
  workflow_dispatch:
    inputs:
      platforms:
        description: 'android, ios ya da android,ios'
        type: string
        default: android

concurrency:
  group: mobile-release
  cancel-in-progress: false

jobs:
  mobile:
    uses: emrahyurttutan/eas-local-runner-template/.github/workflows/eas-build.yml@main
    permissions:
      contents: write # Release oluşturmak için
    with:
      app-dir: apps/mobile # uygulama repo kökündeyse kaldırın
      platforms: ${{ inputs.platforms || 'android' }}
      profile: production
    secrets:
      EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
      RUNNER_KC_PASSWORD: ${{ secrets.RUNNER_KC_PASSWORD }}
```

Sırlar tüketici deponun **Settings → Secrets and variables → Actions** bölümünde tanımlanır ve
`secrets:` bloğunda açıkça eşlenir; şablon hiçbir sırrı adıyla varsaymaz.

## Pipeline akışı

```text
plan      app-dir, eas.json, profil, platform adları              (~10 sn)
build     platform başına: kurulum → eas whoami → eas build --local → artifact
release   yalnız ref bir tag ise: artifact'ler → GitHub Release
```

- `build` job'ları platform başına ayrıdır ve biri düşerse diğeri sürer (`fail-fast: false`).
  Release yalnız hepsi başarılıysa oluşur.
- Tüketici iş akışındaki `concurrency` grubu, aynı deponun iki tag'inin iç içe girmesini önler.
  Depolar ve runner'lar arası sıralamayı **makine kilidi** sağlar: aynı Mac'te birden çok
  runner (başka depolar, GitLab şablonu) varsa her `build` job'ı önce `/tmp/eas-local-build.lock`
  kilidini alır ve diğer derleme bitene kadar bekler. Bekleme `timeout-minutes`'a sayılır.
  GitHub kuralı: bir grupta en fazla **bir çalışan ve bir bekleyen** iş olur; üçüncü tetikleme
  bekleyeni iptal eder. İptal edilen tag için iş akışı Actions sekmesinden yeniden çalıştırılır.

## Girdiler

| Girdi | Varsayılan | Anlamı |
|---|---|---|
| `platforms` | `android` | Virgülle ayrılmış: `android`, `ios`, `android,ios`. |
| `profile` | `production` | `eas.json` build profili. |
| `app-dir` | `.` | Monorepo'da uygulama kökü (`apps/mobile`). Sondaki `/` düşürülür. |
| `runs-on` | `["self-hosted","macOS"]` | JSON dizi. GitHub-hosted için `["macos-15"]`; yalnız Android ise `["ubuntu-latest"]` de olur. |
| `eas-cli-version` | `latest` | Sabitlemek için `22.5.0` gibi. |
| `node-version` | `22` | `actions/setup-node`. |
| `java-version` | `17` | macOS'ta `/usr/libexec/java_home -v` ile seçilir; yoksa job düşer. Linux'ta runner imajının JDK'sı kullanılır. |
| `package-manager` | boş | Saptamayı ezer: `bun`, `pnpm`, `yarn`, `npm`. |
| `install-command` | boş | Kurulum komutunun tamamını ezer; doluysa saptama yapılmaz. |
| `pnpm-version` | boş | `package.json` içinde `packageManager` alanı yoksa gerekir. |
| `clean-prebuild` | `false` | Derlemeden önce `expo prebuild --platform X --clean --no-install`. Yalnız `android/`, `ios/` klasörleri depoda commit'li projelerde gerekir (aşağıya bakın). |
| `release` | `auto` | `auto`: ref bir tag ise GitHub Release'e ekle. `never`: yalnız artifact. |
| `artifact-retention-days` | `7` | Artifact saklama süresi. |
| `timeout-minutes` | `90` | Build job'ı. |
| `unlock-keychain` | `true` | Self-hosted Mac'te login keychain kilidini açar (iOS). GitHub-hosted'da `false` verin. |
| `refresh-adhoc-profile` | `false` | iOS'ta `--refresh-ad-hoc-provisioning-profile`: son derlemeden sonra eklenen cihazları profile alır. ASC sırlarını ister. |
| `apple-team-type` | `INDIVIDUAL` | `EXPO_APPLE_TEAM_TYPE`; `COMPANY_OR_ORGANIZATION`, `IN_HOUSE`. Yalnız profil tazelemede. |

Çıktı: `release-url` (release job'ı çalıştıysa).

## Sırlar

| Sır | Zorunlu | Kullanım |
|---|---|---|
| `EXPO_TOKEN` | Evet | EAS kişisel erişim belirteci ([expo.dev → Access tokens](https://expo.dev/settings/access-tokens)). Anahtar ve profil bununla EAS'tan iner. |
| `RUNNER_KC_PASSWORD` | iOS + self-hosted | Runner Mac'inin kullanıcı parolası; login keychain'i açar. Boşsa ve `unlock-keychain` açıksa iOS job'ı derlemeye girmeden açık mesajla düşer. |
| `ASC_KEY_BASE64` | Hayır | App Store Connect API `.p8` anahtarı, base64. Yalnız `refresh-adhoc-profile` için. |
| `ASC_KEY_ID`, `ASC_ISSUER_ID` | Hayır | Aynı anahtarın kimlikleri. |
| `APPLE_TEAM_ID` | Hayır | Apple Team ID (10 karakter). |

`.p8` dosyası `RUNNER_TEMP` altına yazılır ve job sonunda `always()` ile silinir.

## Self-hosted Mac runner kurulumu

1. Tüketici depoda **Settings → Actions → Runners → New self-hosted runner** (macOS, ARM64).
   Kişisel hesaplarda runner depo düzeyindedir; birden fazla depo için her birine ayrı
   kaydedilir (aynı makine olabilir).
2. `config.sh` varsayılan etiketlerle (`self-hosted`, `macOS`, `ARM64`); `runs-on`
   varsayılanı bunlara uyar.
3. Servis: `./svc.sh install && ./svc.sh start`. launchd altında, **oturum açıkken** çalışır;
   iOS imzası için oturum şarttır. Durum: `./svc.sh status`.
4. Makinede hazır olması gerekenler: Xcode ve komut satırı araçları, CocoaPods, fastlane,
   JDK (`brew install openjdk@17`), Android SDK + NDK (`~/Library/Android/sdk`), git. Node,
   pnpm/bun ve eas-cli iş akışı tarafından kurulur (`actions/setup-node` araç önbelleği).
5. Runner servisinin login kabuğu yoktur; Homebrew, bun ve JDK yolları iş akışında eklenir.
   `~/.bash_profile` düzenlemek gerekmez.

> **Güvenlik:** Self-hosted runner'ı **yalnız private depolara** bağlayın. Public depoda fork
> PR'ları runner'da kod çalıştırabilir. Bu şablon deposunun kendi lint'i `ubuntu-latest`'te
> koşar; buraya runner bağlanmaz.

## Ön koşullar (bir kez)

- **Anahtarlar EAS'ta olmalı.** `--non-interactive` mod var olmayan Android keystore'unu ya da
  iOS sertifika/profilini üretmez. Her platform için bir kez yerelde interaktif çalıştırın:
  `eas build --local -p android --profile production` (iOS için `-p ios`). Paket adı veya bundle
  id değişince bu adım tekrarlanır.
- **`eas.json`'da profil.** Android çıktısının uzantısı `build.<profile>.android.buildType`
  alanından okunur (`apk` → `.apk`, aksi `.aab`; `extends` zinciri izlenir).
- **Sürüm sayacı.** `cli.appVersionSource: remote` + `autoIncrement: true` ile `versionCode` ve
  `buildNumber` EAS'ta artar; yerel derleme de bu sayacı kullanır. Tag ile `version` uyuşmazsa
  iş akışı uyarı basar, derleme sürer.

## Davranış notları

- **`clean-prebuild` ne zaman?** eas-cli çalışma ağacını `.gitignore`'a uyarak kopyalar ve
  native klasör yoksa prebuild'i kendisi yapar. `android/`, `ios/` **depoda commit'liyse**
  `app.json`'daki güncel sürüm numaralarının yansıması için `true` verin. Klasörler
  gitignore'daysa etkisizdir.
- **GitHub-hosted'a geçiş.** `runs-on: '["macos-15"]'` ve `unlock-keychain: false`. Private
  depoda macOS dakikası 10 kat sayılır; Expo bağımlılıkları her seferinde sıfırdan kurulur.
- **Keychain.** iOS job'ı başlamadan login keychain açılır ve `Distribution` adlı artık
  sertifikalar silinir; `eas build` kendi sertifikasını yeniden içe aktarır. Arama listesi sıfırlanmaz:
  login başa alınır, dosyası duran diğer kayıtlar (başka bir derlemenin geçici keychain'i) korunur;
  yarım kalmış EAS derlemelerinin `eas-build-<uuid>.keychain` kayıtları, başka bir EAS derlemesi
  koşmuyorsa çıkarılır.
- **Makine kilidi.** `/tmp/eas-local-build.lock`, GitLab şablonuyla ortak. Sahibi job'ın
  `Runner.Worker` sürecidir; job biterse ya da öldürülürse kilit sahipsiz kalır ve sonraki job kaldırır; `Temizlik`
  adımı da bırakır. Keychain ve daemon adımları yalnız kilit alındıktan sonra çalışır.
- **JVM daemon'ları.** Job öncesi ve sonrası (kilit bu job'dayken) `GradleDaemon` /
  `KotlinCompileDaemon` süreçleri kapatılır. Runner aynı zamanda geliştirme makinesiyse bu, bir sonraki build'in
  `OutOfMemoryError: Metaspace` ile düşmesini önler. Kilidi paylaşmayan bir derleme (Zeyrek
  mobile-runner, elle `eas build --local`) sürüyorsa — koşan bir `gradlew`, `xcodebuild` ya da EAS
  yerel derlemesi varsa — temizlik atlanır.
- **Artifact ve Release.** Artifact adı `build-<platform>`; Release yalnız `release: auto` ve
  ref `refs/tags/*` iken oluşur, `generate_release_notes` açıktır.

## Kapsam dışı

Mağaza metadata/submit/sürüm notu (GitLab şablonundaki `metadata_*`, `submit_*`,
`release_notes_*` job'ları), APK/IPA'nın bir sunucudan sunulması, iOS OTA kurulum manifesti.
Bunlar gerektiğinde `release` job'ından sonra ayrı job olarak eklenir.

# CLAUDE.md

## Bu proje nedir?

GitHub Actions **yeniden kullanılabilir iş akışı** (`on: workflow_call`). Expo/React Native
uygulamalarını kendi Mac'imizde (self-hosted runner) `eas build --local` ile derler, çıktıyı
artifact ve tag'de GitHub Release olarak bırakır. Mağaza aşamaları (metadata, submit) yoktur;
sonradan job olarak eklenebilir.

## Repo yapısı

| Dosya | Rol |
|---|---|
| `.github/workflows/eas-build.yml` | Tek gerçek kaynak: girdiler, sırlar, üç job, gömülü bash/node betikleri. |
| `.github/workflows/lint.yml` | actionlint; şablonun kendi denetimi. |
| `examples/mobile-release.yml` | Tüketici depoya kopyalanacak örnek. |
| `README.md` | Kullanım: girdiler, sırlar, runner kurulumu, ön koşullar. |

## Pipeline mimarisi

```text
plan     app-dir/eas.json ve profil var mı, platform adları geçerli mi → matrix JSON
build    (platform başına, fail-fast yok)
         ├─ makine kilidi /tmp/eas-local-build.lock (GitLab şablonuyla ortak)
         ├─ macOS hazırlığı: PATH, JAVA_HOME, ANDROID_HOME, artık JVM daemon temizliği
         ├─ paket yöneticisi saptama (lock dosyası) → araç kurulumu → kök dizinde install
         ├─ expo-github-action (eas-cli + EXPO_TOKEN) → eas whoami
         ├─ iOS + self-hosted: login keychain kilidi, artık Distribution sertifikaları
         ├─ (isteğe bağlı) expo prebuild --clean
         ├─ çıktı adı: <repo>-<platform>-<ref>.<apk|aab|ipa>
         ├─ tag ise: app version ↔ tag uyarısı
         ├─ eas build --local --non-interactive --output dist-ci/<ad>
         └─ upload-artifact, always(): temizlik + kilidi bırak (kilit bu job'daysa)
release  yalnız release=auto ve ref tag ise: artifact'leri indir → GitHub Release
```

## Değiştirirken dikkat edilecek kritik noktalar

- **Tüketici şablon deposunu checkout etmez.** Betikler `run:` içinde gömülü kalmak zorunda;
  ayrı `scripts/` dosyası çalışmaz.
- **macOS `/bin/bash` 3.2.** Boş dizi + `set -u` patlar; düz metin değişken kullan.
- **Eşzamanlılık: makine kilidi.** Runner Mac'te birden çok runner (bu şablonu kullanan başka
  depolar, GitLab'daki eşi) aynı anda derleme yapabilir; JVM daemon ve keychain adımları
  diğerinin canlı derlemesini bozar. Bu yüzden `build` job'ının ilk adımı
  `/tmp/eas-local-build.lock` dizinini alır (`pid`, `start` = `LC_ALL=C TZ=UTC ps -o lstart`, `who`; `pid` en son yazılır, sahipsiz kilit
  `<kilit>.reap` altında yeniden doğrulanıp kaldırılır). Kilit GitLab
  şablonuyla ORTAKTIR; adı ya da biçimi yalnız ikisinde birlikte değişir. Sahibi
  `Runner.Worker` sürecidir; job ölünce bekleyen kilidi kaldırır. Tüketici ayrıca iş akışı
  düzeyinde `concurrency` koyar. GitHub kuralı: grupta bir çalışan + bir bekleyen; üçüncü
  tetikleme bekleyeni iptal eder.
- **`$TMPDIR/eas-build-local-nodejs` silinmez.** eas-cli kendi `<uuid>` klasörünü başarıda,
  hatada ve SIGTERM/SIGINT'te siler; SIGKILL artıklarını runner Mac'in cron temizliği toplar.
- **`--non-interactive` anahtar üretmez.** İlk Android keystore'u ve iOS profili bir kez
  yerelde interaktif `eas build --local` ile oluşturulur.
- **eas-cli çalışma ağacını `.gitignore`'a uyarak kopyalar.** Yok sayılan `android/`, `ios/`
  derlemeye girmez; `clean-prebuild` yalnız native klasörleri commit'li projelerde anlamlı.
- **JVM daemon temizliği job öncesi ve sonrası.** Gradle/Kotlin daemon'ları 2 saat ayakta kalıp
  GB'larca RAM tutar; runner Mac aynı zamanda geliştirme makinesi. Makinede kilidi paylaşmayan bir
  derleme (`GradleWrapperMain`, `GradleMain`, `xcodebuild`, `eas-cli-local-build-plugin` süreci;
  ör. Zeyrek mobile-runner) sürüyorsa temizlik atlanır — onun daemon'ı canlıdır.
- **Keychain.** launchd altındaki runner'da login keychain kilitlidir; `RUNNER_KC_PASSWORD` ile
  açılır. Yarım kalan build'lerin bıraktığı "Distribution" sertifikaları xcodebuild exit 65'e
  yol açar; her iOS job'ı önce onları siler. Arama listesi sıfırlanmaz: login başa alınır, dosyası
  duran diğer kayıtlar korunur — başka bir derlemenin geçici keychain'i imzalama sürerken düşmesin.
  Yarım kalmış EAS derlemelerinin `eas-build-<uuid>.keychain` kayıtları (eas-cli temizlemez) başka bir
  EAS derlemesi koşmuyorsa atılır.

## Doğrulama (test yok, lint var)

```bash
actionlint -color .github/workflows/*.yml examples/*.yml
```

## Sürümleme ve commit

`main` doğrudan kullanılır; tüketiciler `@main` ile bağlıdır. Kırıcı değişiklikte tag
(`v2`) atılıp README'de yeni `uses:` satırı gösterilir. Commit mesajları Türkçe.

## İletişim dili

Türkçe.

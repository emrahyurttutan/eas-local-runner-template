# AGENTS.md

GitHub Actions için EAS yerel derleme şablonu. GitLab'daki
[eas-local-runner-template](https://gitlab.com/emrahyurttutan/eas-local-runner-template)'in
karşılığı: tüketici depo `uses:` ile çağırır, self-hosted Mac'te `eas build --local` çalışır.

## 30 saniyelik özet

- Tek gerçek kaynak `.github/workflows/eas-build.yml`. Job'lar: `plan` → `build` (platform
  başına) → `release` (yalnız tag'de).
- Tüketici örneği `examples/mobile-release.yml`; kullanım kılavuzu `README.md`.
- Sır yok; her şey tüketici deponun GitHub Secrets'ından `secrets:` ile eşlenir.

## Değişiklik yapmadan önce bilinmesi gerekenler

- **`main`'e commit = anında production.** Tüketiciler `@main` ile bağlı.
- **Sır ve proje adı bu depoya girmez.** Varsayılanlar genel değerlerdir.
- **Test yok, lint var:** `actionlint` (`.github/workflows/lint.yml`, ubuntu-latest). Mac runner bu
  depoya bağlanmaz: public depoya bağlı self-hosted runner fork PR'larından kod çalıştırır.
- **`eas-build.yml` ↔ `README.md` eş güncellenir.** Girdi ekleyince README tablosu da değişir.
- **Belge, yorum ve job adları Türkçedir.**

Ayrıntılı mimari ve tuzaklar için → [CLAUDE.md](CLAUDE.md)

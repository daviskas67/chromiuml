# chromiuml

Инженерная база для аудита старого Chromium APK и проверки возможности его безопасной перепаковки.

> **Текущий статус:** каркас проекта и CI. Исходный APK пока не получен из среды разработки: `curl` не смог разрешить `orchis.w0.am`. Поэтому фактические package/version/SDK/ABI/сертификат намеренно не указаны. Workflow скачивает оригинал на GitHub-hosted runner и формирует отчёт с измеренными значениями.

## Что проект делает сейчас

- скачивает ровно указанный исходный APK (URL можно переопределить через `APK_URL`);
- проверяет ZIP/APK, размер и SHA-256;
- извлекает package name, version code/name, min/target SDK, ABI и список `.so`;
- проверяет подпись через `apksigner` и сохраняет отпечаток сертификата;
- декодирует APK через Apktool;
- по явному запросу выполняет round-trip repack и подписывает результат отдельным debug-ключом;
- запускает статические smoke-проверки и сохраняет отчёты в GitHub Actions artifacts.

Проект **не** обновляет Chromium engine, не обещает поддержку современного веба и не содержит пока патчей Blink/V8/Net. Перепаковка APK не равна пересборке Chromium.

## Быстрый старт

Требуются Bash, Python 3.10+, `curl`, `unzip`, Android SDK Build Tools (`aapt`, `apksigner`, `zipalign`), `apkanalyzer` и Apktool.

```bash
scripts/download_apk.sh
scripts/analyze_apk.py --apk artifacts/upload.apk --outdir artifacts/report
scripts/decode_apk.sh artifacts/upload.apk artifacts/decoded
scripts/smoke_test.sh artifacts/upload.apk
```

Перепаковка выполняется только явно. Она меняет подпись и может изменить поведение/установимость APK:

```bash
scripts/repack_apk.sh artifacts/upload.apk artifacts/chromiuml-debug.apk
```

Результат подписывается debug-ключом, который генерируется локально и не коммитится. Он **не является обновлением**, подписанным оригинальным издателем. Android не позволит установить такой APK поверх оригинальной установки с другой подписью.

## GitHub Actions

Открой **Actions → APK audit and optional repack → Run workflow**. По умолчанию выполняется аудит. Для round-trip сборки включи `repack_apk`. Артефакт `chromiuml-audit` содержит исходный APK, JSON/Markdown-отчёт и декодированный манифест; при включённом repack — debug APK.

При обычном push/PR выполняются проверки скриптов и аудит исходного APK. APK не хранится в git.

## Стратегия

На первом этапе применяется **D: статический аудит + CI**, с ограниченной и опциональной **A: перепаковкой** только после фактического анализа. Для реальной поддержки ES2022+, новых CSS/API, WebGPU, актуальных TLS и исправлений безопасности нужен современный Chromium source build (стратегия B). Это отдельная большая работа: depot_tools, зафиксированный commit Chromium, GN/Ninja и совместимые Android toolchain/NDK.

## Важные ограничения

1. Без приватного исходного signing key сохранить подпись исходного APK невозможно. Package name можно оставить, подпись — нет.
2. APK мог быть собран из конкретного Chromium commit и toolchain. Их нельзя надёжно восстановить только по package/version строкам.
3. Apktool round-trip не доказывает корректность native libraries, sandbox, JNI, WebView/Content Shell integration или runtime.
4. `network_security_config.xml` в `configs/` — пример политики, не внедрённый в APK. Он не добавляет TLS 1.3 и не заменяет Chromium Net/BoringSSL.
5. В репозиторий не коммить оригинальный APK, keystore, токены или профили браузера.

## Документы

- [Фактический baseline APK](docs/APK_BASELINE.md)
- [Решение по архитектуре](docs/ARCHITECTURE_DECISION.md)
- [Матрица совместимости](docs/COMPATIBILITY.md)
- [Тест-план](docs/TEST_PLAN.md)
- [Сборка, подпись и откат](docs/RELEASE_AND_ROLLBACK.md)
- [Патчи](patches/README.md)

## Безопасность

Старый браузер следует считать неподдерживаемым и потенциально уязвимым до установления точной версии и прохождения тестов. Не используйте его для авторизации, платежей, конфиденциальных данных или повседневного просмотра. Проверяйте APK на тестовом устройстве/эмуляторе без личного аккаунта.

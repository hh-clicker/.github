# Выпуск HH Clicker

Release workflow находится в этом репозитории. Push тега `vX.Y.Z` запускает сборки Windows → Linux → macOS Intel → macOS Apple Silicon
и создаёт draft-релиз здесь. Обычный push ветки сборку релиза не запускает.
Каталог версий и pinned SHA остаётся в [launcher/releases](https://github.com/hh-clicker/launcher/tree/master/releases).

## Источники и совместимость

Workflow получает `hh-clicker/launcher` по тегу с тем же именем и проверяет совпадение
`hhClickerVersion`, версии release.json и тега. Затем получает backend, browser-agent,
frontend, generation-agent и infra по полным SHA из манифеста. Stable staging требует
чистые checkout и не подменяет их последним состоянием ветки.

Манифест должен правильно описывать minimumUpgradeVersion, configVersion, postgresMajor
и databaseMigration. При наличии миграций проверьте совместимость на отдельной копии БД.
Rollback установщика восстанавливает application, но не PostgreSQL.

## Подготовка

1. Завершите изменения и необходимые проверки компонентов.
2. Соберите и проверьте nightly. Зафиксируйте результаты проверки вне исходников проекта.
3. Создайте подписанные коммиты компонентов. Запишите окончательные SHA пяти сервисов
   в `launcher/releases/<version>/release.json`, обновите notes.ru.md и hhClickerVersion.
4. Создайте подписанный коммит launcher. Подпишите тег `v<version>` на согласованных
   коммитах шести компонентов. Проверьте подписи и соответствие pinned SHA.
5. Отправьте нужные ветки и конкретные теги компонентов. Не отправляйте все локальные теги.
6. Подготовьте пользовательское руководство и создайте подписанный тег той же версии
   в `.github`. Отправляйте его последним: это запускает release workflow.

Не переписывайте опубликованные теги и выпущенные каталоги. Исправление опубликованного
состава требует новой версии.

## Сборка и draft

Windows runner устанавливает Java 21 для launcher, Node.js 24 и Inno Setup. После создания draft
Linux runner собирает stable AppImage, затем отдельные macOS jobs собирают stable DMG для Intel и Apple Silicon.
Оба macOS jobs вызывают общий `macos-release.yml`: Intel использует `macos-15-intel`, ARM — `macos-15`.
Gradle собирает
артефакты из согласованного исходного дерева. Для публичных компонентов достаточно
стандартного github.token. При чтении приватных компонентов нужен отдельный secret
RELEASE_REPOSITORIES_TOKEN с правом чтения; не записывайте его в файлы проекта.

Workflow загружает EXE, AppImage и два DMG в draft-релиз; рядом с каждым файлом находится отдельный SHA-256.
macOS-файлы называются `HH-Clicker-<version>-macos-x64.dmg` и `HH-Clicker-<version>-macos-arm64.dmg`.
Архивировать DMG для скачивания не требуется. Workflow применяет пользовательские
release notes. Перед публикацией проверьте состав, checksum, версию и работу installer.
Публикация draft выполняется явно владельцем релиза. Ссылку скачивания и статус
руководства обновляйте после появления опубликованного релиза.

Каждая платформа собирается отдельным job. Если упала последняя сборка, используйте **Re-run failed jobs**:
завершённые Windows, Linux и Intel jobs пересобирать не нужно. Повторная загрузка файлов разрешена только в draft;
после публикации необходимо выпускать новую версию. Статус сборок владелец релиза проверяет в Actions.

## Локальные команды

Из корня launcher после подготовки окружения:

```powershell
.\gradlew.bat :shared:jvmTest --console=plain
.\gradlew.bat --% :desktopApp:packageWindowsInstaller -Pnightly -PnightlyVersion=2.0.0 --console=plain
```

Укажите числовую версию, совместимую с установленной. Nightly использует production-данные;
для разрушительных проверок нужна отдельная тестовая установка.


## Ручной запуск для существующего тега

Workflow можно запустить из Actions с параметром tag или через CLI:

```powershell
gh workflow run release.yml --repo hh-clicker/.github --ref master -f tag=v2.0.0
```

Используется текущий workflow выбранной ветки и исходники компонентов указанного тега.
Тег должен существовать в launcher и соответствовать его версии и манифесту.
Опубликованный релиз не изменяется; повторная загрузка допустима только для draft.

## Пробная Linux-сборка

Отдельный workflow `linux-appimage.yml` вручную собирает nightly AppImage из отправленных в GitHub
веток, тегов или SHA. Он сохраняет AppImage и SHA-256 в артефактах Actions на 7 дней,
не создаёт релиз и не загружает файлы в draft. Локальные незакоммиченные изменения в CI не попадают.
Для первого прогона, когда согласованные изменения отправлены в `master` шести репозиториев:

```powershell
gh workflow run linux-appimage.yml --repo hh-clicker/.github --ref master -f version=2.0.1 -f launcher_ref=master -f service_ref=master
```

Значение `version` должно быть не ниже базовой версии в `launcher/gradle.properties`.
Перед добавлением Linux-файла к draft-релизу проверьте его запуск на Debian 13 и Arch.

## Пробные macOS-сборки

Два независимых workflow запускаются вручную и собирают неподписанные nightly DMG:

- `macos-dmg-intel.yml` — Intel x64, runner `macos-15-intel`.
- `macos-dmg-arm64.yml` — Apple Silicon arm64, runner `macos-15`.

Параметры `version`, `launcher_ref` и `service_ref` совпадают с пробной Linux-сборкой.
Launcher и пять сервисов берутся из GitHub; локальные изменения сначала нужно отправить
в выбранные ветки. Для launcher требуется версия кода с macOS-упаковкой.

```powershell
gh workflow run macos-dmg-intel.yml --repo hh-clicker/.github --ref master -f version=2.1.1 -f launcher_ref=master -f service_ref=master
gh workflow run macos-dmg-arm64.yml --repo hh-clicker/.github --ref master -f version=2.1.1 -f launcher_ref=master -f service_ref=master
```

Каждый workflow проверяет архитектуру приложения и встроенной JVM, наличие Browser Agent,
минимальную macOS 14 и целостность DMG. Файл `HH-Clicker-<version>-nightly-macos-<arch>.dmg`
и SHA-256 хранятся в artifacts 7 дней. Теги и релизы эти workflow не создают.
Пакеты не подписаны и не нотарифицированы. Успешная сборка не заменяет ручную проверку
установки и работы приложения на соответствующей архитектуре.

## Критические обновления

Лаунчер сверяет последнюю опубликованную стабильную версию с `update-policy.json` в этом репозитории.
Если старая версия больше не работает (например, после несовместимого изменения HH), поднимите
`minimumSupportedVersion` до первого релиза с исправлением и отправьте изменение в `master`.
Порог можно подготовить вместе с релизом: лаунчер применяет его только если версия уже доступна
среди опубликованных стабильных релизов. Пока 3.0.0 находится в draft, порог 3.0.0 не делает обновление
критическим. Обычное обновление не требует изменения файла. Понижать порог после следующего релиза
не нужно: он относится ко всем старым версиям.

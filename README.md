# iOS on Redmi 4X (santoni) — test checkpoint

[Русский](#русский) / [English](#english)

## Русский

**Чекпоинт 22.11.2026.** Проверено на Redmi 4X **santoni, 2/16 ГБ**, разблокированный загрузчик. iOS 16.5 / XNU 8796.121.2, Mesa 25.0.7, Adreno 505. Экспериментальный порт, не повседневная прошивка.

Работают SpringBoard, сенсор, Калькулятор и кнопки питания/громкости; Home проверялся в прогонах. Аппаратный FD505 прошёл тесты GLES2-шейдеров, пикселей треугольника, очереди GPU и IOKit trap. Эта сборка использует прямое рисование в IOSurface, номинальную настройку GPU 100 МГц и автоматический выбор GMEM/sysmem.

Плавность нестабильна. Максимальное сохранённое окно вывода последнего прогона — **10,79 FPS**. Прежние отдельные сценарии достигали 20–25 FPS, но устойчивые 20 FPS всего интерфейса пока не подтверждены. «Настройки» ранее падали. Камера, звук, сеть, связь, сон и прочие функции не рабочие. Часть WidgetKit-расширений временно скрывается read-only bind-mount. GPU-ускорение не означает полной поддержки Metal.

### Скачать

[Текущая тестовая сборка](ios-santoni-test-build.zip), около 11 МБ. SHA-256 архива:

```text
45f845cc0a122c63c78546c1720e464d3f841807916fe8c845ab6fc569c84422
```

Архив содержит только актуальные компоненты:

| Файл | Назначение |
|---|---|
| `boot.img` | XNU, загрузчик, небольшие загрузочные APFS-окружения; **не полная система iOS** |
| `lk2nd.img` | Временный загрузчик с инициализацией microSD и питанием сенсора |
| `mesa-runtime.apfs` | Mesa и проверочная программа для подготовленной карты |
| `SHA256SUMS` | Контрольные суммы распакованных файлов |

Старых диагностических образов, дампов и журналов в архиве нет.

### Образ microSD для записи

[Скачать SD-образ и контрольные суммы из Releases](https://github.com/egortestf/ios-santoni/releases/tag/checkpoint-2026-11-22).

Файл `ios-santoni-checkpoint-sd.img.gz` содержит **весь образ носителя**: MBR, подготовленную систему iOS и Mesa на нужных секторах. Он собран с нуля из системных файлов, не является дампом личной карты. Для GitHub файл разделён на `.001` и `.002`, суммарно около 2,66 ГБ. Распакованный `.img` занимает **13 778 288 640 байт**; нужна microSD от **16 ГБ** (64 ГБ также подходит). Это та же тестовая система с теми же ограничениями и таймером.

**Запись полностью уничтожит содержимое выбранной карты.** Сохраните нужные файлы и тщательно выберите устройство.

1. Скачайте **обе части** `.img.gz.001` и `.img.gz.002`, `SHA256SUMS` и свежий `ios-santoni-test-build.zip`. Старый lk2nd из первой версии архива принимал только ёмкость карты стенда; используйте обновлённый архив.
2. Проверьте SHA-256 частей по `SHA256SUMS`, затем объедините их в папке загрузки. В PowerShell:

```powershell
cmd /c copy /b ios-santoni-checkpoint-sd.img.gz.001+ios-santoni-checkpoint-sd.img.gz.002 ios-santoni-checkpoint-sd.img.gz
Get-FileHash .\ios-santoni-checkpoint-sd.img.gz -Algorithm SHA256
```

В Linux/macOS: `cat ios-santoni-checkpoint-sd.img.gz.001 ios-santoni-checkpoint-sd.img.gz.002 > ios-santoni-checkpoint-sd.img.gz`. SHA-256 объединённого gzip:

```text
c8524bbc414e16f4bac384c4a4b853bab443271b19cf62a2efaab273d85541ee
```

Освободите место для частей, объединённого gzip и распакованного образа (около 20 ГБ). Не запускайте объединение повторно поверх нужного файла без проверки.
3. Распакуйте gzip в `.img`. В программе записи сырых образов, например [balenaEtcher](https://etcher.balena.io/) ([документация](https://etcher-docs.balena.io/)), выберите **файл `.img`**, нужную microSD и запишите образ с последующей проверкой. Это запись на **всё устройство**, не копирование файла на раздел. Предварительное форматирование не требуется.
4. После записи Windows/Android могут предложить форматировать неизвестные APFS-разделы — **откажитесь**. Остаток ёмкости больших карт пока не используется; разделы не перемещайте.
5. Безопасно извлеките карту, вставьте в Redmi и выполните раздел «Запуск» ниже. Инициализация через `fastboot oem santoni-sd-init` по-прежнему нужна.

Содержимое и контрольные суммы разделов проверены в файле-образе. Запись этого файла на отдельную новую физическую карту и её загрузка пока не проверялись; исходные системные байты совпадают с рабочей картой побайтно. Новый образ **не заменяет Android на внутренней памяти** и сам по себе не запускается без RAM boot-образа.

### Размещение данных (для ручной подготовки)

Готовый SD-образ из Releases уже содержит нужное размещение. Сведения ниже нужны при ручной подготовке: boot-образ читает фиксированные сектора, поэтому простое копирование файлов на произвольную карту не обеспечивает запуск.

Системный APFS около 2,5 ГБ включён **в SD-образ из Releases**, но не в маленький ZIP с `boot.img`. При ручной подготовке нужен совместимый APFS iOS 16.5 с изменениями порта: произвольный IPSW/стоковый rootfs не является готовой заменой.

Карта стенда: 64 ГБ, 122138624 сектора по 512 байт. Адреса, встроенные в `boot.img`:

| Данные | Имя на карте стенда | Байт | Начальный LBA | Секторов |
|---|---|---:|---:|---:|
| Подготовленная iOS | `santoni-ios-system.apfs` | 2497955840 | 22031296 | 4878820 |
| Mesa из архива | `santoni-mesa-fastsubmit-72584b80.apfs` | 5320704 | 22008976 | 10392 |

Нужно проверять **фактические непрерывные экстенты**, а не только имена: XNU читает секторы напрямую и не ищет NTFS-пути. Не записывайте файлы через `dd` по этим LBA в занятую файловую систему — это может повредить её. Для другой карты требуется проверить размещение или пересобрать образ с её адресами.

SHA-256 требуемого Mesa-файла:

```text
72584b801177796315ac120dbaa6eb4c92e7b7701d8fdfc05bb199b5436720dd
```

### Запуск на подготовленном стенде

Нужны разблокированный santoni, рабочий Android для возврата, USB-кабель, [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools), подготовленная microSD. Другие модели и конфигурации памяти не проверены.

Распакуйте архив. Команды выполняйте из его папки. Это **загрузка в RAM**: `flash`, `erase` и блокировка загрузчика не нужны. Не прошивайте lk2nd в разделы телефона.

1. Вставьте карту. Войдите в штатный fastboot: питание + громкость вниз.
2. Проверьте телефон:

```powershell
fastboot devices
fastboot getvar product
```

Ожидается `product: santoni`. Если устройств несколько, добавляйте `-s <серийный_номер>` к каждой команде.

3. Загрузите временный lk2nd:

```powershell
fastboot boot lk2nd.img
```

Подождите несколько секунд, затем:

```powershell
fastboot getvar product
```

Ожидается `product: lk2nd-msm8952`.

4. Инициализируйте карту:

```powershell
fastboot oem santoni-sd-init
```

Продолжайте только после `OKAY`, `MBR=55aa`, `CTRL=00000d12`, `CLK=000f0207`. Команда ограничена Santoni и проверенной геометрией карты; она не форматирует карту и не записывает пользовательские секторы. При ошибке проверьте карту и повторите через Android/fastboot.

5. Включите питание сенсора:

```powershell
fastboot oem santoni-touch-power
```

Ожидается `RPM acknowledged LDOA10 enabled at 2800mV` и `OKAY`.

6. Запустите образ:

```powershell
fastboot boot boot.img
```

ADB/fastboot обычно исчезает на время XNU. Загрузка с карты может занять несколько минут. После появления интерфейса проверьте свайпы, Калькулятор и Home.

### Завершение

- Удерживайте **обе кнопки громкости 3–4 секунды**, пока не появится текст загрузчика.
- После фиксации результата удерживайте **питание + громкость вниз** до обычного fastboot.
- Выполните `fastboot reboot` и отпустите кнопки для возврата в Android. Автоматический возврат в fastboot пока ненадёжен.
- Сборка ограничена по времени: GPU-сервис работает около 900 секунд после инициализации, возврат ядра запланирован примерно через 960 секунд. После отключения GPU интерфейс может перестать работать. Завершайте проверку раньше лимита. Это не постоянная установка iOS.

Эти команды не заменяют Android на внутренней памяти. XNU открывает системный и Mesa-файлы карты read-only; это не защищает от ошибок ручной переразметки.

Основа: [HakiOS](https://github.com/oosexgang/hakios-xnu-kernel), [Apple XNU](https://github.com/apple-oss-distributions/xnu/tree/xnu-8796.121.2), [Mesa](https://www.mesa3d.org/), [lk2nd](https://github.com/msm8916-mainline/lk2nd). Применяются лицензии соответствующих компонентов. README не перелицензирует их и не делает проприетарную iOS открытым ПО. Это бинарный чекпоинт, не полный комплект исходников и средств воспроизводимой сборки.

Следующая публикация — только после следующего согласованного чекпоинта.

---

## English

**Checkpoint: 22 November 2026.** Tested on an unlocked **Redmi 4X santoni, 2/16 GB**. iOS 16.5 / XNU 8796.121.2, Mesa 25.0.7, Adreno 505. Experimental port, not daily-driver firmware.

SpringBoard, touch, Calculator and power/volume buttons work in test runs; Home has also been exercised. FD505 hardware rendering passed GLES2 shader, triangle-pixel, GPU queue and IOKit trap tests. This build uses direct IOSurface rendering, a nominal 100 MHz GPU setting and automatic GMEM/sysmem selection.

Smoothness remains inconsistent. The user noted a small improvement; the latest run's highest saved panel-output window was **10.79 FPS**. Earlier isolated scenarios reached 20–37 FPS, but stable 20 FPS throughout the UI has not been established. Settings has previously crashed. Camera, audio, networking, cellular, sleep and other features are not claimed to work. Some WidgetKit extensions are temporarily hidden using read-only bind mounts. GPU acceleration does not imply full Metal support.

### Download

[Current test build](ios-santoni-test-build.zip), about 11 MB. Archive SHA-256:

```text
45f845cc0a122c63c78546c1720e464d3f841807916fe8c845ab6fc569c84422
```

Only current components are included:

| File | Purpose |
|---|---|
| `boot.img` | XNU, loader and small APFS boot environments; **not the complete iOS system** |
| `lk2nd.img` | Temporary loader providing microSD initialization and touch power |
| `mesa-runtime.apfs` | Mesa and a probe needed on the prepared card |
| `SHA256SUMS` | Extracted-file checksums |

Old diagnostic images, dumps and logs are excluded.

### Whole-card microSD image

[Download the SD image and checksums from Releases](https://github.com/egortestf/ios-santoni/releases/tag/checkpoint-2026-11-22).

`ios-santoni-checkpoint-sd.img.gz` contains the **whole device image**: MBR, the prepared iOS system and Mesa at the required sectors. It is assembled from system files, not a dump of a personal card. GitHub downloads are split into `.001` and `.002`, about 2.66 GB combined. The extracted `.img` is **13,778,288,640 bytes**; use a microSD of **16 GB or larger** (64 GB also works). The same test limitations and timer still apply.

**Writing this image destroys the selected card's contents.** Back up files and carefully identify the target device.

1. Download **both parts**, `.img.gz.001` and `.img.gz.002`, `SHA256SUMS` and the updated `ios-santoni-test-build.zip`. The initial ZIP's lk2nd accepted only the original card capacity; use the updated archive.
2. Verify the part hashes using `SHA256SUMS`, then join them in the download directory. PowerShell:

```powershell
cmd /c copy /b ios-santoni-checkpoint-sd.img.gz.001+ios-santoni-checkpoint-sd.img.gz.002 ios-santoni-checkpoint-sd.img.gz
Get-FileHash .\ios-santoni-checkpoint-sd.img.gz -Algorithm SHA256
```

Linux/macOS: `cat ios-santoni-checkpoint-sd.img.gz.001 ios-santoni-checkpoint-sd.img.gz.002 > ios-santoni-checkpoint-sd.img.gz`. Combined gzip SHA-256:

```text
c8524bbc414e16f4bac384c4a4b853bab443271b19cf62a2efaab273d85541ee
```

Allow about 20 GB free space for parts, combined gzip and extracted image. Check existing output before repeating the join command.
3. Extract gzip to `.img`. Use a raw-image writer such as [balenaEtcher](https://etcher.balena.io/) ([documentation](https://etcher-docs.balena.io/)): select the **`.img`**, select the microSD, write and validate. Target the **whole device**, not a partition; do not merely copy the image file onto the card. No prior formatting is needed.
4. Decline any Windows/Android prompt to format the unknown APFS partitions. Extra capacity on larger cards remains unused; do not move the partitions.
5. Safely eject, insert into Redmi and follow the boot instructions below. `fastboot oem santoni-sd-init` is still required.

The image's partition contents and hashes have been verified. Writing it to a separate new physical card and booting that card have not yet been tested; the system bytes match the working card exactly. The SD image **does not replace internal Android** and still requires the RAM boot image.

### Data layout (manual provisioning)

The release SD image already supplies the required layout. For manual provisioning, the following fixed sector mappings matter; copying files to an arbitrary card does not provide them.

The approximately 2.5 GB prepared APFS system is included **in the release SD image**, not in the small boot ZIP. Manual provisioning requires the compatible modified iOS 16.5 system; a stock IPSW/rootfs is not a drop-in replacement.

Project card: 64 GB, 122138624 sectors of 512 bytes. Hardcoded mappings:

| Data | Project-card filename | Bytes | Starting LBA | Sectors |
|---|---|---:|---:|---:|
| Prepared iOS system | `santoni-ios-system.apfs` | 2497955840 | 22031296 | 4878820 |
| Included Mesa | `santoni-mesa-fastsubmit-72584b80.apfs` | 5320704 | 22008976 | 10392 |

Verify **actual contiguous extents**, not just filenames. XNU reads raw sectors, not NTFS paths. Do not blindly `dd` files onto these LBAs in an existing filesystem: this can overwrite metadata and user data. Another card requires verified placement or a rebuilt boot image using its mappings.

Required Mesa file SHA-256:

```text
72584b801177796315ac120dbaa6eb4c92e7b7701d8fdfc05bb199b5436720dd
```

### Booting the prepared setup

Requirements: unlocked santoni, working Android to return to, USB cable, [Android Platform Tools](https://developer.android.com/tools/releases/platform-tools) and the prepared microSD. Other models/memory configurations are untested.

Extract the archive and run commands from that directory. This is **RAM boot**: no `flash`, `erase` or bootloader relocking is required. Do not flash lk2nd to phone partitions.

1. Insert the card; enter stock fastboot with power + volume down.
2. Check the phone:

```powershell
fastboot devices
fastboot getvar product
```

Expect `product: santoni`. With multiple devices, add `-s <serial>` to every command.

3. Start the temporary loader:

```powershell
fastboot boot lk2nd.img
```

Wait a few seconds, then:

```powershell
fastboot getvar product
```

Expect `product: lk2nd-msm8952`.

4. Initialize the card:

```powershell
fastboot oem santoni-sd-init
```

Continue only after `OKAY`, `MBR=55aa`, `CTRL=00000d12`, `CLK=000f0207`. The command is restricted to Santoni and the verified card geometry. It does not format the card or write user sectors. On failure, check the card and retry through Android/fastboot.

5. Enable touch power:

```powershell
fastboot oem santoni-touch-power
```

Expect `RPM acknowledged LDOA10 enabled at 2800mV` and `OKAY`.

6. Boot:

```powershell
fastboot boot boot.img
```

ADB/fastboot normally disappears while XNU runs. Allow several minutes for loading from the card, then test swipes, Calculator and Home.

### Stopping

- Hold **both volume buttons for 3–4 seconds** until loader text appears.
- After recording results, hold **power + volume down** to enter stock fastboot.
- Run `fastboot reboot` and release the buttons to return to Android. Automatic fastboot return is unreliable.
- This is a timed build: the GPU service runs about 900 seconds after initialization; kernel return is scheduled around 960 seconds. The UI may stop when the GPU service ends. Finish testing before the limit. This is not a permanent iOS installation.

These commands do not replace Android on internal storage. XNU opens the system/Mesa card views read-only; that does not protect against manual partitioning mistakes.

Based on [HakiOS](https://github.com/oosexgang/hakios-xnu-kernel), [Apple XNU](https://github.com/apple-oss-distributions/xnu/tree/xnu-8796.121.2), [Mesa](https://www.mesa3d.org/) and [lk2nd](https://github.com/msm8916-mainline/lk2nd). Their respective licenses apply. This README does not relicense them or make proprietary iOS open source. This is a binary checkpoint, not the complete source/build reproduction package.

Further publication requires the next agreed checkpoint.

# Чистка Android TV по ADB

> [!NOTE]
> Документ написан ИИ-агентом для ИИ-агентов. Человеку достаточно включить отладку на телевизоре и дать ссылку на этот файл своему агенту.

Проверено на ТВ с 1–2 ГБ RAM: **Haier/CANDY MT5867** (913 МБ RAM, Android 14) и двух **TCL** (`BeyondTV2` Android 9 и `G08 Smart TV Pro` Realtek RTD2880 Android 11/12).

### Почему Android TV тормозит
На ТВ с 1–1.5 ГБ физической памяти стоковый комбайн (Google TV Home + рекомендации + Google Assistant + Chromecast + `DroidGuard` + 5–10 предустановленных кинотеатров и OEM-сервисов) съедает всю свободную RAM ещё до открытия любого приложения.
Ядро вытесняет 400–600 МБ в **zram** (сжатый swap **в той же оперативке**, тратящий CPU на сжатие/распаковку). В итоге:
1. Любое нажатие кнопки на пульте вызывает лавину page-fault'ов и распаковку страниц из zram.
2. Демон `lmkd` (Low Memory Killer) непрерывно убивает фоновые сервисы, а `ActivityManager` тут же поднимает их заново (`Scheduling restart of crashed service`) — процессор крутит бесконечный цикл смертей и перезапусков.

### `disable-user` vs `uninstall --user 0`
Из системного раздела (`/system`, `/product`, `/vendor`) без root ничего физически не удаляется — обе команды обратимы, но работают по-разному:
* **`pm disable-user --user 0 <pkg>`** — замораживает пакет. Его `Activity`, `Service`, `ContentProvider` и `BOOT_COMPLETED` больше не запускаются, **данные и настройки сохраняются**. Возвращается через `pm enable <pkg>`. **Для чего:** стоковый лаунчер, `setupwraith` и системные компоненты.
* **`pm uninstall --user 0 <pkg>`** — удаляет пакет для пользователя `0` **вместе с данными (`data/user/0/<pkg>`), кэшем и задачами в `JobScheduler` / `AlarmManager`**. Базовый APK остаётся в read-only разделе и возвращается чистым через `pm install-existing --user 0 <pkg>`. **Для чего:** кинотеатры, рекламные службы и блоатварь, распухшие в `/data`.

### Протокол для ИИ-агента
1. **Снять исходное состояние (до первого изменения):** сохранить вывод `pm list packages -f`, `pm list packages -d`, `getprop`, `settings list global/secure/system` и текущий `HOME` (`cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME`).
2. **Замер «ДО» и опрос владельца одним сообщением:** снять память (`/proc/[0-9]*/status`) и CPU за 60 с (`/proc/[0-9]*/stat`), показать топ пожирателей и **одним списком спросить**:
   * Какими кинотеатрами пользуетесь (YouTube/SmartTube, Кинопоиск, Okko, Wink, KION, Иви, Zona, Stremio, Lampa)?
   * Нужен ли голосовой поиск с микрофона на пульте (`katniss` + `tv.remote.service`)?
   * Нужна ли трансляция с телефона кнопкой Cast (`mediashell`)?
   * Используются ли антенна (Live TV) и порты HDMI (приставка, PS5, саундбар)?
   * Какой лаунчер ставим (п. 4) и ставим ли SmartTube?
3. **Проверки безопасности:** прогнать кандидатов через проверки п. 2.2 (`dumpsys wifi`, `TV_INPUT_HARDWARE`, `PERSISTENT`, порядок `setupwraith` → стоковый лаунчер).
4. **Применение и проверка после ребута:** выполнить `adb shell sync`, подождать 15 секунд, перезагрузить ТВ, проверить что `HOME` открывает новый лаунчер, снять замер «ПОСЛЕ» и **вернуть настройки сна (п. 0)**.

## 0. Поиск ТВ в сети, доступ и защита от засыпания

Если человек не знает IP телевизора — не спрашивать его сразу, а за 2 секунды просканировать локальную подсеть `/24` на открытый порт `5555` и mDNS:

```sh
# 1. Проверить mDNS (Android 11+ Wireless Debugging / Chromecast) и ARP-таблицу:
adb mdns services 2>/dev/null; ip neigh

# 2. Найти порт 5555 в локальной подсети (убедившись, что трафик в 192.168.x.x не перехватывает VPN на компьютере):
SUB=$(ip -4 route show default | awk '{print $3}' | cut -d. -f1-3)
for i in $(seq 1 254); do (nc -z -w1 $SUB.$i 5555 2>/dev/null && echo "ADB open: $SUB.$i") & done; wait

adb connect <TV_IP>:5555
adb devices
```
*(Если порт 5555 нигде не открыт — подсказать человеку включить отладку по шагам из п. 2.1 или посмотреть IP на ТВ в `Настройки → Сеть и интернет → <имя Wi-Fi сети>`).*

Если телик уходит в стендбай — Wi-Fi/eth0 гаснет, ADB отваливается, разбудить по сети нельзя.
Поэтому **первым действием** агент должен запретить сон на время работы, а **последним** — вернуть обратно:

```sh
# В начале работы:
adb shell 'svc power stayon true; settings put global stay_on_while_plugged_in 3; settings put secure sleep_timeout -1; settings put system screen_off_timeout 2147483647'

# В конце работы (обязательно вернуть!):
adb shell 'svc power stayon false; settings put global stay_on_while_plugged_in 0; settings delete secure sleep_timeout; settings delete system screen_off_timeout'
```

И сразу отключить проверку Play Protect для ADB-установок (иначе `adb install` повиснет в ожидании нажатия кнопки «Разрешить» с пульта на экране ТВ):

```sh
adb shell settings put global verifier_verify_adb_installs 0
```

## 1. Сначала измерить, потом лечить

`load average` на таких ТВ **врёт**: 50-60 дают десятки потоков драйверов MTK в состоянии D
(`DMX_Filt_Task`, `VDEC_*`, `VGA Mon`, `VeOutTtxTask`). Это не загрузка.

Реальные показатели:

```sh
# 1. Память и swap (главный индикатор: если SwapFree мало, а MemAvailable < 200 МБ — ТВ в thrashing'е)
adb shell 'grep -E "MemTotal|MemAvailable|SwapTotal|SwapFree" /proc/meminfo'

# 2. Топ процессов по реальной памяти (VmRSS + VmSwap в МБ) напрямую из /proc за 0.3 секунды:
adb shell 'for p in /proc/[0-9]*; do
  awk "/^Name:/{n=\$2} /^VmRSS:/{r=\$2} /^VmSwap:/{s=\$2} END{if(r+s>10240) printf \"%6.1f MB (RSS %5.1f + Swap %5.1f)  PID %-6s %s\n\", (r+s)/1024, r/1024, s/1024, substr(FILENAME,7,index(FILENAME,\"/status\")-7), n}" "$p/status" 2>/dev/null
done | sort -rn'

# 3. Честная загрузка CPU: дельта по /proc/*/stat за 60 секунд
# (одиночный top врёт или ловит пик от самого запуска sh; а разность utime+stime за 60 с даёт точный средний расход)
adb shell 'for p in /proc/[0-9]*; do n=$(sed -n "s/^Name:[[:space:]]*//p" $p/status 2>/dev/null); s=$(awk "{print \$14+\$15}" $p/stat 2>/dev/null); [ -n "$n" ] && [ -n "$s" ] && echo "$n $s"; done'
# два снимка с интервалом 60 с, разницу тиков (USER_HZ=100) разделить на 60 → проценты одного ядра

# 4. Если жрёт конкретный процесс (например, системный провайдер или кинотеатр) — найти виновный поток внутри него:
adb shell 'for t in /proc/<PID>/task/*; do echo "$(cut -d" " -f14,15,3 $t/stat) $(basename $t) $(cat $t/comm)"; done'
```

**Почему НЕЛЬЗЯ вызывать `dumpsys meminfo` и `dumpsys cpuinfo` на лагающем ТВ:**
`dumpsys meminfo` без аргументов обходит все процессы через Binder и заставляет ядро считать PSS каждого маппинга в `/proc/<pid>/smaps`. На ТВ с забитым swap это вгоняет `system_server` в дисковый/zram шторм и вешает ADB на 30–90 секунд. Чтение `/proc/[0-9]*/status` из шелла обходит `system_server` и работает мгновенно.

Полезно также: `dumpsys usagestats` (чем реально пользуются), `dumpsys netstats detail`
(трафик по uid), `logcat -d -b crash` (кто падает по `OutOfMemoryError`).

## 2. Деблоат (обратимый): почему базы UAD недостаточно

База [Universal Android Debloater (`uad_lists.json`)](https://github.com/0x192/universal-android-debloater) (и её живой форк `UAD-ng`) заточена под **смартфоны**, а не под Android TV.
На ТВ Haier/CANDY из **155 пакетов** в UAD как `Recommended` нашлось всего **4 пакета** (`play.games`, `videos`, `feedback`, `partnersetup`), которые почти не влияли на память.
А **20 из 24 реально удалённых пакетов** (включая всех главных пожирателей RAM — `katniss`, `mediashell`, `tvrecommendations`, `backdrop`, `netflix.ninja`, ACR-телеметрию `tv.anoki.acr` и весь софт Haier/РФ-кинотеатры) **в базе UAD отсутствуют вообще**.
Та же картина на **TCL BeyondTV2**: из **66 вычищенных пакетов** (26 пакетов `com.tcl.*`, 14 `uk.co.freeview.*`, Google TV-пакеты) в базе UAD было всего **9** базовых телефонных пакетов Google/AOSP и **ноль** пакетов TCL или Android TV.

Поэтому рабочий подход для агента:
1. Использовать `uad_lists.json` главным образом как **стоп-лист** (`Unsafe`/`Expert` — то, что трогать нельзя) + забрать базовый телефонный хлам Google.
2. Главных пожирателей искать **по живой памяти** (`/proc/[0-9]*/status` → `VmRSS + VmSwap`), а не по статическим спискам из интернета.

```sh
adb shell pm uninstall --user 0 <package>        # удалить для юзера 0 (данные стираются, автостарт гаснет)
adb shell pm install-existing --user 0 <package> # вернуть удалённый пакет обратно
adb shell pm disable-user --user 0 <package>     # заморозить без удаления данных (вернуть: pm enable)
```

Как посмотреть, что на ТВ уже было удалено/отключено до тебя:

```sh
# Отключённые через disable-user:
adb shell pm list packages -d

# Удалённые через uninstall --user 0 (есть в -u, но нет в активных):
comm -13 <(adb shell pm list packages | sed 's/package://' | sort) \
         <(adb shell pm list packages -u | sed 's/package://' | sort)
```

Что на Haier/CANDY дало главный эффект (swap 532 → 213 МБ) и **почему**:

| пакет | что это и почему жрёт / что отвалится |
|---|---|
| `com.google.android.katniss` | **Google Ассистент (100–140 МБ в фоне всегда).** ⚠️ **Спросить владельца перед сносом:** если снести — перестанет работать кнопка микрофона (голосовой ввод) на пульте. Если голосом не пользуются — главный кандидат на снос. |
| `com.google.android.apps.mediashell` | **Google Cast / встроенный Chromecast (60–95 МБ в фоне всегда).** ⚠️ **Спросить владельца:** если с телефона видео/музыку кнопкой Cast на ТВ не кидают — сносить; если кидают — оставить. |
| `com.google.android.tvrecommendations`, `com.google.android.backdrop` | Полки «Рекомендаций» Android TV и фоновое слайд-шоу заставки Daydream. Нужны только стоковому лаунчеру; с FLauncher бесполезно держат ~50–80 МБ. |
| `com.netflix.ninja` | **Netflix — двойной выигрыш (~150+ МБ):** сам висит в фоне ради быстрого старта с кнопки пульта, а главное — именно он регулярно будит `com.google.android.gms.unstable` (**`DroidGuard`**, ещё **106 МБ RAM**) для проверки аттестации/DRM. После сноса Netflix процесс `DroidGuard` исчез совсем. |
| `ru.more.play`, `ru.mts.mtstv`, `mobi.zona.lite`, `ru.ivi.client` | Предустановленные кинотеатры (Okko, KION, Zona, IVI). Даже если их ни разу не открывали, они стартуют по `BOOT_COMPLETED` и `ChannelsWorker`, чтобы пушить рекламные баннеры на главный экран, и остаются в RAM навсегда. |
| `com.google.android.videos`, `com.google.android.play.games` | Мёртвый Google Play Фильмы и Play Игры (фоновые синхронизации). |
| `ru.haier.evo`, `com.haier.apps.evoSmart`, `com.haierubic.usmartdevice` | Вендорская рекламная оболочка Haier EVO и демон умного дома Haier (постоянно висят в памяти и шлют телеметрию). |
| `tv.anoki.acr.*` | **ACR (Automatic Content Recognition)** — шпионский модуль распознавания того, что сейчас показывают на экране (снимает отпечатки кадров для рекламного таргетинга). Сносить безусловно. |
| `com.avo.vpn`, `fusion.android.tv.demo`*, `de.benzac.msx`, `com.cltv.fast` | Вшитый VPN, магазинное демо (*для `fusion.android.tv.demo` сначала проверить п. 2.2!), Media Station X, FAST-каналы. |

**Не трогать ни в коем случае:**
* `com.google.android.gms`, `com.google.android.gsf`, `com.android.vending` — Google Play Services и Play Store (без них падают YouTube/Кинопоиск и авторизация).
* `com.android.systemui`, `com.android.tv.settings`, `com.mediatek.wwtv.tvcenter` (эфирное ТВ MediaTek).
* **Экранную клавиатуру** (`com.google.android.inputmethod.latin` / Gboard, `com.tcl.inputmethod.international`) — если снести единственную IME-клавиатуру, не получится ввести ни пароль от Wi-Fi, ни запрос в поиске.
* **Аудио-сервисы и TV-middleware:** `com.dolby.android.audio.service` (на части ТВ пропадает звук), **`com.cltv.mal`** на Haier/MediaTek (выглядит как стороннее приложение, но это системный TV-middleware — его отключение на Haier H50H5F вызывает бесконечный бутлуп), **`mitv.service`** на Xiaomi (держит `TvService` — без него настройки Звука и Яркости виснут на 30–40 с и падают по `NumberFormatException: "NULL"`), **`com.heytap.tv.support`** на OnePlus/OPPO (выглядит как «техподдержка», но запрашивает `TV_INPUT_HARDWARE` и биндит HDMI-сессию — без него все HDMI пишут «No Signal»).
* **Связка микрофона на пульте (`katniss` + `tv.remote.service`):** если владелец пользуется голосовым поиском, нужны **оба** пакета — `com.google.android.katniss` (сам ассистент) и `com.google.android.tv.remote.service` (мост, пересылающий аудиопоток с микрофона Bluetooth-пульта). Если отключить `tv.remote.service`, ассистент будет открываться по кнопке, но записывать абсолютную тишину.
* **Связка `mediashell` и Stremio:** `com.google.android.apps.mediashell` (Google Cast) нужен не только для трансляции с телефона — через него же воспроизводит поток Android TV-версия **Stremio** (без `mediashell` каталог Stremio открывается, но видео бесконечно крутит спиннер).
* Любые пакеты ядра Android (`permissioncontroller`, `packageinstaller`, `ext.services`, `ext.shared`, `modulemetadata`, `com.google.android.overlay.modules.*`, `com.android.providers.*` — без них ТВ не загрузится после ребута!) и заводские сервисные меню (`*factory*`, `*facmenu*`, `*testpq*`).

Что вычищается на **TCL** (проверено на двух моделях TCL — Android 9 и Android 11/12) и **в чём там нюансы**:
- **Что на TCL трогать НЕЛЬЗЯ:**
  - `com.tcl.tv`, `com.tcl.tvinput`, **`com.tcl.channelplus`** — стек эфирного ТВ и HDMI-входов (если отключить `channelplus`, то Live TV открывается с чёрным экраном).
  - `com.tcl.system.server`, `com.tcl.systemui.plugin`, `com.tcl.settings`, `com.tcl.common.shortcutmenu` — системные настройки изображения/звука и быстрое меню.
  - `com.tcl.tcl_bt_rcu_service`, `com.tcl.autopair`, `com.tcl.keycustomfunctionservice`, `com.tcl.micmanager` — Bluetooth-пульт, его автоспаривание и обработка горячих клавиш.
- **Что на TCL безопасно вычищается:**
  - `com.tcl.waterfall.overseas` (рекламный «водопад» TCL Channel), `com.tcl.bootadservice` (буквально сервис показа рекламы при включении ТВ!), `com.tcl.bi` (сбор телеметрии), `com.tcl.esticker` / `com.tcl.exhibit` / `com.tcl.hotelmenu` (магазинные бейджи, витринный и отельный режимы).
  - `com.tcl.tvweishi` («Safety Guard / антивирус-оптимизатор» — сам жрёт RAM и CPU фоновым сканированием).
  - `com.tcl.MultiScreenInteraction_TV` / `com.tcl.magiconnectfree` (приёмник фирменного T-Cast — постоянно держит сетевой демон и шлёт броадкасты; если T-Cast с телефона не нужен — под снос).
  - `com.tcl.appmarket2`, `com.tcl.assistant`, `com.tcl.smartalexa`, `com.tcl.messagebox`, `com.tcl.notereminder`, `com.tcl.tv.tclhome_master`, `com.tcl.tv.tclhome_passive`, `com.tcl.usercenter`.
  - ⚠️ Нюанс с `partnercustomizer` (`com.tcl.partnercustomizer` / `com.xiaomi.android.tvsetup.partnercustomizer`): на старых TCL (Android 9) он только засевал рекламные каналы в стоковый лаунчер и спокойно удаляется, но **на Xiaomi и некоторых свежих ТВ** внутри `partnercustomizer` живёт `GlobalKeyReceiver`, который обрабатывает цветные и брендовые кнопки пульта (включая кнопку `Mi` вместо `Home`). Поэтому на Xiaomi его не трогать!
- Региональный мёртвый груз в прошивках TCL: `uk.co.freeview.*` (14 пакетов британского гида телепередач, которые крутятся даже в РФ), `au.com.stan.and`, `com.eros.now`, `com.linecorp.linetv`, `tv.wuaki.apptv`, `com.graymatrix.did`.
- Официальный YouTube (`com.google.android.youtube.tv`, `youtube.tvmusic`) по желанию заменяется на **SmartTube** (`org.smarttube.stable` — легче по памяти, без рекламы и со встроенным SponsorBlock).
  - ⚠️ **Важно про SmartTube:** в конце 2025 года после компрометации сборочной машины автор SmartTube сменил ключ подписи и имя пакета со старого `com.teamsmart.videomanager.tv` на **`org.smarttube.stable`**. Если на ТВ стоит старый `com.teamsmart.videomanager.tv` — снести его (`pm uninstall com.teamsmart.videomanager.tv`) и поставить свежий `org.smarttube.stable` (`armeabi-v7a`) с официального GitHub.

### 2.1. Российская специфика: Яндекс ТВ (YaOS), Салют ТВ (Sber), Xiaomi РСТ и локальный софт

В России огромная доля бюджетных телевизоров (DEXP, Hartens, Tuvio, Hyundai, Leff, Novex, Starwind, Sber, BBK, Hi) работает не на чистом Google TV, а на **Яндекс ТВ (YaOS)** или **Салют ТВ (StarOS)** на китайских системных платах **CVTE (Cultraview)**. У них свои правила и свои смертельные ловушки:

#### А. Яндекс ТВ / YaOS (Hartens, Tuvio, DEXP, Hyundai, Leff, Novex, Starwind)
1. **Как вообще включить ADB (на прошивках 2025–2026 Яндекс вырезал «Отладку по USB» из меню разработчика):**
   Включить порт `5555` можно через **инженерное меню платы CVTE**:
   * С комплектного ИК-пульта (нужна физическая кнопка `Menu` с тремя полосками; с Bluetooth-пульта без неё не выйдет): зажать `Menu` на ~1 с, отпустить и без пауз нажать **`← ↑ ← ↑`**, затем **`Назад`** (другие коды шасси CVTE: **`Menu → 1 1 4 7`** или **`Input/Source → 2 5 8 0`**). В открывшемся сине-сером меню: `Other options / Common options / DEBUG` → **`ADB Mode` (`ADB ENABLE`) → `On`**.
   * Если комбинация с пульта не срабатывает — поставить с флешки утилиту `FactoryMenu.apk` с 4PDA (для Android 11 нужна версия `1.1`/`1.2` под пакет `com.cvte.fac.menu`) → `DEBUG → ADB ENABLE → On`.
   * После включения ADB обязательно **выдернуть ТВ из розетки на 10 секунд** — на многих платах CVTE окно «Разрешить отладку?» начинает всплывать только после холодного ребута.
   * ⚠️ В инженерном меню CVTE **ни в коем случае не трогать** `Panel Type`, `LVDS`, `Shipping Init`, `Board Init`, `Aging mode` — любое нажатие там даёт чёрный экран или перевёрнутую/битую матрицу.
2. **Что на Яндекс ТВ отключать НЕЛЬЗЯ (ведёт в чёрный экран / Recovery):**
   * `com.yandex.tv.services.platform` и `com.yandex.tv.services` — сразу после отключения ТВ падает в `Entering recovery system` и чёрный экран.
   * `com.yandex.tv.live` и `com.yandex.tv.input.efir` — в `com.yandex.tv.live` зашито переключение видеовходов: если его отключить, **перестают переключаться HDMI-порты**.
   * `ru.yandex.androidkeyboard.tv` — единственная клавиатура в системе.
   * На «Яндекс ТВ по подписке» (купленных в рассрочку Плюса) установка сторонних APK заблокирована на уровне прошивки до полного погашения.
3. **Что на Яндекс ТВ безопасно отключать:**
   * Главный генератор баннеров и видеорекламы: **`com.yandex.android.advid`** + телеметрия `com.yandex.tv.bugreportsender`.
   * Пакет `com.yandex.io.sdk` тоже тянет рекламу, но на нём живёт голосовой ассистент **Алиса** (`com.yandex.tv.alice`) — отключать только если Алиса не нужна.
   * Обновления (`com.yandex.launcher.updaterapp`, `com.yandex.tv.appudater`, `com.cultraview.osupdate`): при прилёте апдейта прошивка включает рекламные пакеты обратно, поэтому апдейтер отключают по согласованию с владельцем.
   * Штатный лаунчер (`com.yandex.tv.home`) отключать **только после** установки нового лаунчера и проверки, что кнопка `Home` открывает новый.

#### Б. Салют ТВ / StarOS (телевизоры Sber, SberBox)
1. **Как включается ADB:** на StarOS 1.71+ штатный путь — переписать `ID устройства` из `Настройки → Об устройстве`, зайти на `developers.sber.ru/studio` под тем же Сбер ID (`Настройки сервисов → SmartApp → Мои устройства`), добавить свой ID, включить тумблер ADB и перезагрузить ТВ. (На свежих Sber SBX с Android 13 / StarOS 1.103 снова работает `Номер сборки ×7 → Отладка по Wi-Fi`, но порт 5555 закрывается после каждого ребута).
2. **Главный капкан Салют ТВ — НЕ ТРОГАТЬ `ru.sberdevices.starlauncher`:**
   Даже если вы поставили сторонний лаунчер, **отключать `ru.sberdevices.starlauncher` через `pm disable-user` или `uninstall` НЕЛЬЗЯ** — проверка целостности StarOS уводит телевизор в бесконечную `Recovery`-петлю! На Салют ТВ сторонний лаунчер ставят только поверх штатного (через перехват кнопки Home в Projectivy / tvQuickActions), оставляя `starlauncher` включённым.
3. **Не трогать:** `ru.sberdevices.SberDevicesServiceSystem`, `ru.sberdevices.settings`, `ru.sberdevices.starlivetv` (эфир и HDMI).
4. **Безопасно отключаются:** `ru.sberdevices.crashreporter`, `ru.sberdevices.smarthome.promo`, `ru.sberdevices.trafficwidget`, `ru.sberdevices.newswidget`, а также неиспользуемые `sberplay`, `music`, `smotreshka` (`ru.sberdevices.TV`, `tv.lfstrm.smotreshka_stb`).

#### В. Xiaomi РСТ (двойной лаунчер `PatchWall` + `Android TV Home`) и ноунейм-платы CVTE
* **Синдром двух лаунчеров на Xiaomi:** на российских Xiaomi в оперативке параллельно живут сразу два тяжеловеса — фирменный `com.mitv.tvhome.atv` (PatchWall, ~140 МБ) и гугловский `com.google.android.tvlauncher` (~95 МБ). Плюс фоновая автоустановка партнёрских приложений (`android.autoinstalls.config.xiaomi.*`), всплывающие промо-окна (`com.xiaomi.floatingframe`), ACR-шпион `tv.alphonso.alphonso_eula`, аналитика (`com.miui.tv.analytics`, `com.xiaomi.statistic`) и псевдо-очиститель `com.xiaomi.mitv.tvmanager`.
  * Если заморозить `com.mitv.tvhome.atv`, `com.mitv.tvhome.mitvplus` и `com.mitv.tvhome.michannel`, освобождается ~140 МБ RAM.
  * Если после отключения `tvrecommendations` и `videos` на стандартном лаунчере остались пустые серые полосы-призраки («Рекомендуемое»), они убираются сбросом данных лаунчера: `adb shell pm clear com.google.android.tvlauncher`.
* **Ловушка ноунейм-телевизоров на голом AOSP (шасси CVTE `com.cvte.tv.launcher.*`):**
  Дешёвые российские/белорусские бренды на платах CVTE с оболочкой `Wisdom Share` используют лаунчеры `com.cvte.tv.launcher.highend` / `saturn` / `space` / `mars` / `mosaix`. На многих таких платах отключение стокового лаунчера вешает ТВ на бут-анимации — поэтому сначала переключать `cmd package set-home-activity`, перезагружать и проверять, и без крайней нужды стоковый `com.cvte.tv.launcher.*` не отключать.

#### Г. Фоновый жор в `mobi.zona`, локальный ByeDPI/VPN и сеть
* **`mobi.zona` (полная версия Zona):** в отличие от лёгкой веб-обёртки `mobi.zona.lite`, полная `mobi.zona` прописывается в автозапуск и висит в фоне сутками (реальный замер на TCL G08: 5.5 суток в фоне, два процесса `mobi.zona` + `mobi.zona:Metrica` на **173 МБ RSS**, падения по `OutOfMemoryError: pthread_create failed` и фоновая раздача торрентов).
  * **Без спроса не удалять** (если через неё смотрят кино). Если владелец ей не пользуется — сносить (`pm uninstall --user 0 mobi.zona`); если пользуется, но ТВ лагает в других приложениях — принудительно останавливать после просмотра (`am force-stop mobi.zona`) или перейти на связку `Lampa + TorrServer`.
* **Ловушка с локальным ByeDPI / VPN на самом ТВ после холодной перезагрузки:**
  Если для обхода замедления YouTube на сам телевизор ставят локальный `ByeDPI` (`io.github.romanvht.byedpi`) или VPN-клиент (Happ / v2rayNG / NekoBox), который поднимает локальный `tun0`, то при холодном старте на ТВ с 1–1.5 ГБ RAM `ActivityManager` часто убивает его `BootReceiver` через 10–12 секунд из-за жёсткого лимита кэшированных процессов (`CUR_MAX_EMPTY_PROCESSES=4`). В результате `tun0` не поднялся, а правила маршрутизации (`ip rule` для UID YouTube в таблицу `tun0`) остались — и YouTube пишет «Нет подключения к сети» при работающем интернете.
  * Чтобы Android сам гарантированно поднимал локальный VPN/ByeDPI при загрузке (но не блокировал сеть при его падении), включать его через системный `always_on_vpn_app` **строго с `lockdown=0`**:
    ```sh
    adb shell "settings put secure always_on_vpn_app <pkg>; settings put secure always_on_vpn_lockdown 0"
    ```
* **Почему компьютер не видит ТВ по ADB (`connection timed out`), хотя они в одном Wi-Fi:**
  В РФ почти у всех на ноутбуке крутится VPN-клиент (AmneziaVPN, Outline, Happ, Hiddify, WireGuard), который перехватывает маршруты в локальную подсеть `192.168.x.x` и уводит пакеты `adb connect` в туннель. Перед подключением к ТВ проверить `ip route get <TV_IP>` (на Linux/macOS) или `Find-NetRoute -RemoteIPAddress <TV_IP>` (на Windows).

### 2.2. Шесть обязательных проверок перед отключением пакетов

1. **Проверка владельца Wi-Fi сети (`cname=` в `dumpsys wifi`):**
   Если при первой настройке ТВ Wi-Fi подключали внутри стартового визарда (`com.google.android.tungsten.setupwraith`) или стокового лаунчера, Android записывает этот пакет владельцем сети. Как только сделать ему `pm disable-user` или `pm uninstall` — **Android молча удалит сохранённую Wi-Fi сеть**, и ТВ мгновенно и навсегда отвалится от Wi-Fi посреди ADB-сессии:
   ```sh
   adb shell "dumpsys wifi | grep -oE 'cname=[^ ]*'"
   ```
   Если там светится `cname=com.google.android.tungsten.setupwraith` (или лаунчер) — **не трогать его**, пока пользователь на ТВ вручную не нажмёт «Забыть сеть» и не переподключит Wi-Fi через обычные `Settings → Network & Internet`.

2. **Строгий порядок отключения `setupwraith` и стокового лаунчера:**
   Внутри `com.google.android.tungsten.setupwraith` на Google TV сидит `RecoveryActivity` с `CATEGORY_HOME`.
   Если отключить стоковый лаунчер (`launcherx` / `tvlauncher`), а `setupwraith` оставить включённым — `RecoveryActivity` перехватывает кнопку Home. Перезагрузка на этом экране на Xiaomi (`MiTV-MOSR4`) вызывала **полный Factory Reset**, а на Kalley 43G200 (`R3_GTV`) — вечный бутлуп.
   * **Правило:** `setupwraith` всегда отключается **ДО** стокового лаунчера, а при возврате в сток включается **ПОСЛЕ** него.
   * **Исключения, где менять стоковый лаунчер нельзя вообще:** телевизоры **Huawei (HarmonyOS)** и **Kalley 43G200 (`R3_GTV`)** — после отключения стокового лаунчера не загружаются дальше логотипа (лечится только полным сбросом/USB-прошивкой); на **Amazon Fire OS** кнопку Home удерживает лончер Amazon с `priority=950`.

3. **Автопроверка пакета на связь с HDMI-входами, пультом и флагом `PERSISTENT` (`TV_INPUT_HARDWARE` / `TV_INPUT_BUTTON` / `PERSISTENT`):**
   Даже пакет с названием `fusion.android.tv.demo` на части плат MediaTek (Multilaser/CVTE, проекторы Xiaomi) держит кнопку `Input` пульта и Bluetooth-пейринг, отключение пакета с `TV_INPUT_HARDWARE` приводит к вечному «Нет сигнала» на HDMI, а системные пакеты с флагом `PERSISTENT` (`pkgFlags=[ ... PERSISTENT ... ]`) ядро всё равно принудительно перезапускает при старте. Перед сносом любого сомнительного вендорского пакета проверить его через `dumpsys package`:
   ```sh
   adb shell 'for p in <pkg1> <pkg2>; do d="$(dumpsys package $p)"; if printf "%s\n" "$d" | grep -m1 pkgFlags | grep -q PERSISTENT; then echo "KEEP (PERSISTENT): $p"; elif printf "%s\n" "$d" | sed -n -r "/requested permissions:/,/(install|runtime) permissions:|User [0-9]+:/p" | grep -q TV_INPUT_HARDWARE || printf "%s\n" "$d" | grep -qE "TV_INPUT_BUTTON|CONNECT_INPUT|AddAccessory"; then echo "KEEP (inputs/remote): $p"; else echo "OK to disable: $p"; fi; done'
   ```

4. **Сохранять список отключённого прямо в `settings global` самого телевизора:**
   Чтобы через год (или с другого компьютера без локальных логов) за секунду узнать, что именно отключал скрипт/агент, записывать список прямо на ТВ:
   ```sh
   adb shell settings put global debloat_disabled "pkg1,pkg2,pkg3"
   adb shell settings get global debloat_disabled
   ```

5. **Фирменные ловушки TCL (`TclAppBoot`, `TGuard`, апдейтер и баг громкости на Android 11):**
   * На TCL фоновый автозапуск сторонних приложений блокируется по умолчанию (`TclAppBoot`). Для нужного приложения (например, лаунчера или сервиса) давать разрешение вручную (переустановка APK его сбрасывает):
     ```sh
     adb shell "appops set <pkg> AUTO_START allow || appops set <pkg> APP_AUTO_START allow"
     ```
   * На TCL с **Android 11 на Realtek RTD2880** (`Smart TV Pro` G08, билд `RP1A200622.001`) живёт второй агрессивный киллер `TGuard` (`com.tcl.guard`), убивающий фоновые процессы через ~1 с после ухода с экрана, и есть **баг прошивки с `AccessibilityService`**: если включить *любой* сервис перехвата кнопок (`flagRequestFilterKeyEvents`, вроде Button Mapper), громкость с пульта начинает прыгать **по 2 деления за одно нажатие** и залипать при удержании (а на TCL G09 Google TV 12 наоборот — зажатая кнопка громкости перестаёт повторяться).
   * Если на TCL отключить `com.tcl.xian.StartandroidService`, проверка обновлений прошивки (`Settings → About → System Update`) будет падать с ошибкой «Local network or server abnormal» (`dnum is null!` в `logcat`), потому что апдейтер читает из этого сервиса серийный ID устройства.

6. **Проверка вторых профилей (`pm list users`):**
   `pm disable-user --user 0` отключает пакеты только для основного пользователя `0`. Если на Google TV когда-то заводили второй профиль (например, `UserInfo{10:...}`), весь мусор для него остаётся включённым. Хуже того, на платах Skyworth/Coocaa при наличии второго пользователя (`UserManager.getUsers().size() > 1`) и отключённом `launcherx` нажатие кнопки `Input` или `Settings` на пульте роняет системный сервис `com.smartdevice.global.service` по `NullPointerException` (он лезет проверять детский режим в `launcherx.LiveTvContentProvider`), после чего Android запускает `RescueParty` вплоть до сброса настроек!
   * Перед заменой лаунчера проверить `adb shell pm list users`: если там висит ненужный второй профиль — удалить его (`adb shell pm remove-user <id>`).

## 3. Технические грабли для агента

1. **Почти все ТВ с 1–2 ГБ RAM работают в 32-битном режиме (`armeabi-v7a`)**, даже если процессор 64-битный (Cortex-A53/A55) и стоит Android 12 или 14. Перед скачиванием любого APK (SmartTube, FLauncher и т.д.) проверять `adb shell getprop ro.product.cpu.abilist` — `arm64-v8a` не встанет (`INSTALL_FAILED_NO_MATCHING_ABIS`).
2. **`adb exec-out screencap -p > file.png` на MediaTek бьёт PNG-файл**, а на HDMI/LiveTV/DRM возвращает **0 байт или старый закэшированный кадр**:
   * На MediaTek вендорские библиотеки пишут в stdout `Init wrapper sys mutex successful. Pid:XXXXX` прямо перед заголовком PNG — снимать скриншоты только через файл (`adb shell screencap -p /sdcard/s.png && adb pull /sdcard/s.png`).
   * На некоторых ТВ (Fire OS, часть MediaTek) после переключения экрана `screencap` может возвращать **байт-в-байт предыдущий кадр** (с тем же MD5). Поэтому факт того, какое окно реально на экране, всегда перепроверять через `adb shell "dumpsys activity activities | grep -iE 'ResumedActivity|mFocusedApp'"`.
3. **Инъекция нажатий (`input tap x y`) работает в настройках и боковых панелях**, но в главной сетке Android TV только переводит фокус, а не открывает карточку — для запуска использовать `input keyevent` (`19/20/21/22` DPAD, `23` CENTER, `3` HOME, `4` BACK) или `am start -n <pkg>/<activity>`.
4. **НИКОГДА не запускать заводские пакеты (`*factory*`, `com.cvte.tv.hkcfac`, `com.at.factory.*`) «просто посмотреть, что это за приложение»:**
   На платах Haier/CVTE один только запуск `com.cvte.tv.hkcfac/.MainActivity` выставляет флаг `customer_factory_mode=true` в `/vendor/tvcertificate/fdata.ini` и `vendor.mtk.factory.disable.input=1`. В ту же секунду умирают кнопка `Input` и HDMI-CEC, а при следующей загрузке заводской менеджер сам отключает Google Play Services, Netflix, `katniss` и `mediashell`, причём `pm enable` из шелла на них перестаёт работать! Изучать незнакомый пакет можно только пассивно через `dumpsys package <pkg>`, не запуская его. (Если уже запустили `hkcfac`: вывести его на экран, послать код выхода из factory-режима `adb shell "input keyevent 218; input keyevent 218"`, перезагрузить и вернуть `settings put global hdmi_control_enabled 1`).
5. **Не перезагружать ТВ сразу после `pm disable-user` (`WRITE_SETTINGS_DELAY`):**
   `PackageManagerService` в Android сбрасывает обновлённый `package-restrictions.xml` на флеш с задержкой **~10 секунд**. Если сразу после пачки `pm disable-user` послать `adb reboot`, изменения пропадут и после ребута пакеты снова окажутся включёнными. Правило: выполнять `pm` по одному вызову (цепочка `pm disable a; pm disable b` на некоторых ТВ молча обрывается после первого пакета), а в конце выполнить `adb shell sync` и подождать 15 секунд перед ребутом.
6. **Осторожно с USB-Ethernet адаптерами при включении «Отладки по USB»:**
   На ряде приставок и ТВ включение тумблера `USB debugging` переводит USB-контроллер из режима host в device и обесточивает USB-порт. Если интернет в ТВ заведён через **USB-Ethernet адаптер**, в момент включения отладки проводная сеть отвалится (а при выключении отладки в конце — наоборот).
7. **Ошибка `INSTALL_FAILED_INSUFFICIENT_STORAGE` при установке APK (раздел `/data` всего 4–8 ГБ):**
   На многих ТВ раздел `/data` забит под 98% кэшами и обновлениями предустановленных кинотеатров, из-за чего не ставится SmartTube (~80 МБ) или лаунчер. Сначала проверить `adb shell df -h /data`, снести ненужные кинотеатры через `pm uninstall --user 0` (это удалит и их распухшие обновления из `/data/app/`), и одной командой сбросить все системные кэши:
   ```sh
   adb shell pm trim-caches 999G
   ```
8. **Лайфхак: как перенести приложение (например, Кинопоиск) с одного своего ТВ на другой без входа в Google Play:**
   Современные приложения из Google Play ставятся как Split APK (4–6 файлов в одной папке). Чтобы вытащить приложение с первого ТВ и поставить на второй:
   ```sh
   # На ТВ-доноре узнать пути ко всем split-файлам пакета и скачать их:
   adb shell pm path ru.kinopoisk.tv | sed 's/^package://' | xargs -n1 adb pull
   # На целевом ТВ установить все скачанные .apk одной транзакцией:
   adb install-multiple -r base.apk split_config.*.apk
   ```

## 4. Выбор и установка лаунчера

Стоковый Google TV Home (`launcherx`) даже в режиме «Apps only» перерисовывает экран 60 раз в секунду и съедает **~180–250 МБ RAM + 43% ядра CPU + 32% `SurfaceFlinger`**.

Четыре варианта на замену:

| Лаунчер | APK / RAM (PSS) | CPU в простое | Кому подходит | Плюсы и подводные камни |
|---|---|---|---|---|
| **1. `FLauncher`** (оригинал, `me.efesser.flauncher`) | ~8 МБ / ~85 МБ (Flutter) | **0%** *(после отключения анимации рамки!)* | Базовый выбор для ТВ, где смотрят только приложения (или входы HDMI переключаются кнопкой `Input` на пульте). | Никаких разрешений на сеть, умеет категории, обои и запуск не-TV приложений. **Минусы:** не обновлялся с 2022 г. (v0.18.0), **нет встроенного меню переключения HDMI**, из коробки включена прожорливая анимация рамки (53% ядра!), баг при удалении категорий/смене сортировки. |
| **2. `LTvLauncher`** (живой форк FLauncher 2026, `com.leanbitlab.ltvL`) | ~12 МБ / ~95 МБ (Flutter) | **~0%** (без погоды и анимаций) | Если нравится простота FLauncher, но **нужно переключение HDMI 1/2/AV с экрана** или перехват кнопки Home без отключения стокового лаунчера. | Open-source (GPLv3), починены баги старого FLauncher, есть **меню HDMI-входов**, строка `Continue Watching`, бэкап настроек и встроенный `Home Button Fix` (для Салют ТВ / Fire TV). На слабом ТВ не включать виджет погоды. |
| **3. `just-a-launcher`** (`tw.hungmi.justalauncher`) | **35 КБ (!)** / **39 МБ** (Native Kotlin) | **0% CPU, 0% SurfaceFlinger** | Экстремально слабые ТВ/проекторы (1 ГБ RAM), где дорог каждый мегабайт. | 200 строк чистого Kotlin (`GridView`) без шрифтов и текстур (экономит ~45 МБ RAM относительно Flutter). Долгое нажатие `OK` меняет порядок иконок, последняя плитка открывает системный выбор HDMI-входов. **Минус:** нет категорий и не показывает телефонные APK без TV-баннера. |
| **4. `Projectivy Launcher`** (`com.spocky.projengmenu`) | ~10 МБ / **78 МБ** | **~6% ядра** + 1.7% SF | Если ТВ используется как монитор для приставки/PS5 (нужен **автозапуск конкретного HDMI при включении**) или это **Салют ТВ**, где нельзя гасить стоковый лаунчер. | Ставится прямо из Google Play, умеет HDMI-входы с CEC-именами, родительский PIN и перехват Home через Accessibility. **Минусы:** закрытый код, часть фич платная, даже с выключенными анимациями будит UI-поток ~80 раз/с (~6% ядра). |

Установка (на примере FLauncher или LTvLauncher):

```sh
adb install -r flauncher-armeabi-v7a.apk   # или LTvLauncher-armeabi-v7a-release.apk / just-a-launcher.apk
adb shell cmd package set-home-activity me.efesser.flauncher/.MainActivity
adb shell cmd role add-role-holder android.app.role.HOME me.efesser.flauncher
# Сначала проверить п. 2.2 (что setupwraith не владеет Wi-Fi!), затем отключать setupwraith ДО стокового лаунчера:
adb shell pm disable-user --user 0 com.google.android.tungsten.setupwraith
adb shell pm disable-user --user 0 com.google.android.tvlauncher   # или com.google.android.apps.tv.launcherx на Google TV
```

**Критично для FLauncher:** в настройках (шестерёнка справа сверху) сразу выключить
**«App card highlight animation»** и Crash/Analytics Reporting.
* **Почему это критично:** пульсирующая рамка вокруг выбранной иконки во Flutter-движке FLauncher крутит бесконечный `AnimationController.repeat()` на 60 FPS даже когда пульт лежит на столе. На телевизионном Mali GPU эта одна рамка непрерывно жгла **31% ядра в `me.efesser.flauncher` + 22% в `SurfaceFlinger`** (пол-ядра впустую!). После снятия галочки расход на статичном экране падает ровно в **0%**.
* **Критично для Projectivy:** в настройках (`Projectivy Settings → Appearance`) отключить динамические обои и анимацию фокуса, иначе он тоже крутит 60 FPS в простое.

**Проверка HDMI-входов после смены лаунчера:**
Поскольку в классическом FLauncher нет плиток HDMI-портов, сразу после отключения стокового лаунчера нажмите физическую кнопку `Input`/`Source` на пульте. Если на вашей прошивке меню входов открывалось через отключённый лаунчер и кнопка перестала работать — не прячьте в FLauncher системную иконку `TV` (`tvcenter` / `Live TV`), либо поставьте `LTvLauncher` / `just-a-launcher` / `Projectivy`, у которых переключение HDMI встроено.

**Побочные эффекты отключения стокового лаунчера (`launcherx`) на Google TV:**
В Google TV (Android 11–14) Google зашил внутрь APK стокового лаунчера (`com.google.android.apps.tv.launcherx`) несколько системных провайдеров, поэтому при его отключении:
1. **Кнопка «Шестерёнка» (`Settings`) на пульте** на некоторых ТВ (TCL, Hisense, Sony) шлёт `KEYCODE_NOTIFICATION`, который обрабатывал сам `launcherx`. Когда он отключён, шестерёнка на пульте может перестать открывать шторку — тогда в настройки заходят через иконку `Settings` в FLauncher (поэтому не прячьте её совсем, не проверив кнопку на пульте).
2. **Пункт «Аккаунты и профили» (`Settings → Accounts & Profiles`) становится пустым** (и нельзя выбрать альбомы Google Фото для заставки): страница управления аккаунтами и `AccountModeContentProvider` живут внутри `launcherx`. Сами аккаунты на месте (`dumpsys account`); если понадобится добавить/сменить аккаунт — временно включить `pm enable com.google.android.apps.tv.launcherx`, переключить аккаунт и снова отключить.
3. **Глобальный голосовой поиск фильмов** может отвечать «Sorry, not available in this profile yet» или «having trouble» (так как результаты поиска отрисовываются через `launcherx/.gateway.LauncherDeepLink`, а проверка детского режима идёт через `launcherx`). При этом голосовой поиск **внутри самих приложений** (иконка микрофона в YouTube / SmartTube / Кинопоиске) продолжает работать нормально.
4. **На Chromecast 4K/HD и Google TV Streamer** после перезагрузки с отключённым `setupwraith` телефон при попытке Cast может написать «Untrusted device»: приёмник `mediashell` при старте читает ключ аттестации из `CastInfoProvider` внутри `setupwraith`. Лечится включением `setupwraith` на 30 секунд после ребута (пока ключ не закэшируется в памяти `mediashell`) или оставлением `setupwraith` + `launcherx` включёнными на этих донглах.

Мусорные системные ярлыки в FLauncher / LTvLauncher убирать **скрытием в лаунчере** (долгое нажатие `OK` на иконке → `Hide`), а не удалением самих системных пакетов: `Videos`, `Factory Menu`, `HKCFactoryAT`, сам `FLauncher`, `HaiCast`, `TV Assistant`.
Пустую категорию «Non-TV Applications» можно удалить в `Settings → Categories`.

### 4.1. Известные баги лаунчеров

* **`FLauncher` (оригинал 0.18.0, GitLab `#109–#126`):**
  1. **Кнопка `OK` на некоторых пультах не открывает приложения (`#113`, `#118`):** старый движок Flutter в FLauncher 0.18.0 ждёт `KEYCODE_DPAD_CENTER` (`23`), а часть китайских пультов и аэромышей шлёт `KEYCODE_ENTER` (`66`). В итоге при проверке агентом через `adb shell input keyevent 23` всё открывается, а когда владелец жмёт реальный пульт — иконка только дёргается! **Правило:** после установки FLauncher всегда просить владельца нажать физическую кнопку `OK` на пульте. Если не открывает — ставить `Projectivy` или `just-a-launcher`.
  2. **Аэромыши (`air-mouse`) и обычные мыши не кликают (`#119`, `#124`):** левый клик только переводит фокус, но не запускает приложение.
  3. **Баг при удалении категорий / смене сортировки (`#109`, `#110`):** внутренний список карточек бьётся — плитки рисуются пустыми или перестают появляться новые приложения из Play Store. Лечится `adb shell pm clear me.efesser.flauncher` (после чего сортировку Алфавит ↔ Ручная лучше не дёргать).
  4. **Скрытые иконки вылезают обратно после автообновления приложения (`LTvLauncher #158`, общий код `AppsService`):** когда скрытое приложение обновляется в фоне через Google Play, обработчик `PACKAGE_CHANGED` не находит его в списке видимых (`indexOf == -1`) и добавляет в конец экрана до следующего рестарта лаунчера.
  5. **Встроенные обои Unsplash («Random») не работают (`#108`, `#126`):** API давно изменился, кнопка падает с ошибкой.

* **`LTvLauncher` (форк FLauncher, GitHub `#134–#158`):**
  1. **Полка `Continue Watching` не работает без ручного `pm grant` (`#133`, `#134`):** на многих ТВ (Sony, Fire OS) системный диалог выдачи разрешения `READ_TV_LISTINGS` вырезан, поэтому включать его нужно по ADB:
     ```sh
     adb shell pm grant com.leanbitlab.ltvL android.permission.READ_TV_LISTINGS
     ```
  2. **Зависание на `Loading...` при обновлении поверх старой версии (`#143`, `#145`):** лечится `adb shell pm clear com.leanbitlab.ltvL`.

* **`Projectivy Launcher` (GitHub `spocky/miproja1` `#646–#669`):**
  1. **НЕ включать Accessibility-сервис Projectivy, если стоковый лаунчер уже отключён по ADB (`#669`):** перехватчик специальных возможностей в Projectivy нужен *только* тогда, когда стоковый лаунчер нельзя отключить (например, на Салют ТВ). Если вы уже переключили `HOME` через `cmd package set-home-activity` и погасили стоковый лаунчер, Accessibility-сервис Projectivy включать не надо — он перехватывает все нажатия пульта, ломает ползунки яркости в Google Home и вызывает двойной шаг громкости на TCL Android 11.
  2. **Если Accessibility всё же нужен (Салют ТВ), а APK поставлен в обход Play Store (`#655`):** на Android 13/14 система блокирует включение Accessibility для sideload-приложений («Restricted setting»). Разблокируется через ADB:
     ```sh
     adb shell appops set com.spocky.projengmenu ACCESS_RESTRICTED_SETTINGS allow
     adb shell settings put secure enabled_accessibility_services com.spocky.projengmenu/com.spocky.projengmenu.services.ProjectivyAccessibilityService
     adb shell settings put secure accessibility_enabled 1
     ```
  3. **Выбор HDMI-входа открывает файловый менеджер (`#646`):** если на ТВ установлен сторонний проводник (ES File Explorer / CX Проводник), он перехватывает `ACTION_VIEW` для `vnd.android.cursor.item/channel` — и нажатие на `HDMI 1` в Projectivy открывает проводник вместо входа.

* **`just-a-launcher` (GitHub `#1`):**
  * **Нельзя скрыть отдельные иконки (`#1`):** он выводит на экран абсолютно все установленные приложения с `LEANBACK_LAUNCHER`. Если на ТВ есть неотключаемая системная иконка (например, сервисное приложение), спрятать её с экрана, как в FLauncher через `Hide`, не получится.

## 5. Чего не делать

- Никаких «чистильщиков кэша» и «ускорителей» из Play Store — они и есть источник лагов.
- Не гоняться за `load average` — см. п. 1.
- **Не включать бездумно `Private DNS` (`dns.adguard-dns.com`) на самом ТВ**, как советуют многие гайды:
  1. **Ломает роутерный обход блокировок:** DNS-over-TLS с телевизора идёт напрямую на внешний резолвер в обход `dnsmasq` домашнего роутера. Если на роутере крутится точечная маршрутизация по доменам (OpenWrt `dnsr`/`podkop`/`mihomo`/Pi-hole), роутер перестаёт видеть DNS-запросы телевизора и перестаёт заворачивать трафик в туннель.
  2. **Вешает резолвинг при включённом VPN-клиенте на ТВ** (Tailscale, AdGuard VPN и т.п.): Android пытается валидировать DoT внутри VPN-туннеля, получает ошибку и блокирует вообще все DNS-запросы (`unknown host`).
  3. **Кладёт интернет после выключения ТВ из розетки:** на приставках без батарейки часов после подачи питания время сбрасывается на дату сборки прошивки → сертификат DoT-сервера отвергается как «ещё не наступивший» → DNS не работает → NTP не может отрезолвить сервер времени, чтобы подвести часы (взаимная блокировка). Лечится возвратом `adb shell settings put global private_dns_mode opportunistic`.
- **В конце работы закрыть за собой сетевой ADB (если он больше не нужен постоянно):** порт `5555` работает без шифрования, а на некоторых боксах и проекторах свойство `ro.adb.secure=0` даёт любому устройству в локальной сети (или гостевом Wi-Fi) root/shell без подтверждения на экране.
- Kernel-потоки (`VeOutTtxTask` и пр.) и системный ТВ-стек (`com.android.providers.tv` ↔ `com.tcl.tvinput`) из userspace не убить — только ребут:
  - На Haier/CANDY `VeOutTtxTask` (телетекст эфирного ТВ) на плохом сигнале уходит в busy-loop до 98% одного ядра.
  - На TCL G08 (`Smart TV Pro`) связка `com.tcl.tvinput` → `com.android.providers.tv` (`Binder:1660_A`) точно так же залипает в 100% ядра (намотала 49 часов CPU за 8 суток аптайма).
- **Не оставлять «про запас» онлайн-кинотеатры, которыми не пользуются** (IVI, Zona, Wink и т.д.). На Android TV «не открывал приложение» ≠ «оно не работает»: они сами поднимаются при загрузке через `BOOT_COMPLETED` / `FirebaseInitProvider` / `ChannelsWorker` (чтобы пушить свои полки рекомендаций в домашний экран) и остаются в RAM навсегда. Реальный замер на TCL G08: ни разу не запускавшийся `ru.ivi.client` висел 4.3 суток в фоне (105 МБ RSS, 93 потока) и жёг **85% ядра** потоком `prefetcher`, долбясь в мёртвый `localhost:443`. Не смотришь — `pm disable-user --user 0` или `pm uninstall --user 0`.

## 6. Живые базы и ссылки

При работе с незнакомой моделью ТВ чужие списки из этих баз обязательно пропускать через проверки п. 2.2:

1. **Каталог 430+ протестированных моделей ТВ (`CleanYourTV`):**
   * Поиск по бренду/модели: **https://cleanyour.tv/tvs**
   * Папки с `debloat.txt` и заметками по конкретным шасси/прошивкам: **https://github.com/Gathin23/cleanyourtv/tree/main/examples**
   * База реальных поломок и пост-мортемов со стектрейсами (обновляется ежедневно): **https://github.com/Gathin23/cleanyourtv/blob/main/docs/TROUBLESHOOTING.md**
2. **Актуальный стоп-лист `PROTECTED` пакетов и особенностей пультов/тюнеров (`Free My TV`):**
   * Код проверок и список неприкасаемых пакетов: **https://github.com/simbasjungles/free-my-tv/blob/main/app/src/io/freemytv/HomeMode.java**
   * Инженерные заметки по железу разных брендов: **https://github.com/simbasjungles/free-my-tv/blob/main/CLAUDE.md**
3. **Российские платформы (Яндекс ТВ / YaOS, Салют ТВ / StarOS, коды сервисных меню плат CVTE):**
   * Живой JSON со списками пакетов для Яндекс ТВ, Салют ТВ и РФ-кинотеатров: **https://raw.githubusercontent.com/by-sonic/chistytv/main/data/packages.json** *(⚠️ осторожно: `com.tcl.channelplus` там ошибочно помечен как `disable` — на TCL его не трогать!)*
   * Сводка способов включения скрытого ADB на Яндекс ТВ и Салют ТВ: **https://github.com/by-sonic/chistytv/blob/main/docs/ADB.md**
   * Профильные ветки 4PDA (если нужна прошивка или код входа в сервисное меню конкретной платы):
     * Яндекс ТВ (YaOS): **https://4pda.to/forum/index.php?showtopic=1000256**
     * Телевизоры на шасси CVTE (`CV9632xx` и др.): **https://4pda.to/forum/index.php?showtopic=1058309**
     * Салют ТВ / SberBox / Sber TV: **https://4pda.to/forum/index.php?showtopic=1005242** и **https://4pda.to/forum/index.php?showtopic=1044662**
4. **База AOSP / Google пакетов (`UAD-ng`):**
   * Живой форк Universal Android Debloater (использовать для справки по базовым пакетам Android и стоп-листа `Unsafe`): **https://raw.githubusercontent.com/Universal-Debloater-Alliance/universal-android-debloater-next-generation/main/resources/assets/uad_lists.json**
5. **Официальные релизы базовых APK (качать сборку под `armeabi-v7a`):**
   * **FLauncher** (классический минималистичный лаунчер): **https://gitlab.com/flauncher/flauncher/-/releases**
   * **LTvLauncher** (свежий форк FLauncher с переключением HDMI и перехватом Home): **https://github.com/leanbitlab-org/LtvLauncher/releases/latest**
   * **just-a-launcher** (ультралёгкий нативный лаунчер 35 КБ / 39 МБ RAM): **https://github.com/hungmi/just-a-launcher/releases/latest**
   * **Projectivy Launcher** (функциональный лаунчер с автозапуском HDMI): **https://github.com/spocky/miproja1/releases**
   * **SmartTube** (`org.smarttube.stable` — YouTube без рекламы и со SponsorBlock): **https://github.com/yuliskov/SmartTube/releases/latest**

## 7. Аварийное спасение

Поскольку `pm disable-user --user 0` и `pm uninstall --user 0` не трогают системный раздел `/system`, любой сбой обратим:
1. **Чёрный экран после смены лаунчера, но ТВ пингуется по Wi-Fi:**
   Подключиться по ADB и включить обратно стоковый лаунчер (сначала лаунчер, потом `setupwraith`):
   ```sh
   adb connect <TV_IP>:5555
   adb shell pm enable com.google.android.apps.tv.launcherx
   adb shell pm enable com.google.android.tvlauncher
   adb shell cmd package set-home-activity com.google.android.apps.tv.launcherx/.home.HomeActivity
   adb shell pm enable com.google.android.tungsten.setupwraith
   ```
2. **Отвалился Wi-Fi (забыли проверить `cname=` перед отключением `setupwraith`) и на экране ничего нельзя нажать:**
   Протянуть обычный **патч-корд Ethernet** от роутера к телевизору. Проводной интерфейс `eth0` поднимается по DHCP автоматически без всяких паролей и сохранённых сетей. Посмотреть новый IP телевизора в админке роутера, подключиться по `adb connect <NEW_IP>:5555` и выполнить `pm enable` из п. 1.
3. **Зависло на бут-анимации и ADB не отвечает:**
   * Попробовать **Безопасный режим (Safe Mode)** Android: во время анимации загрузки зажать и держать кнопку `Power` или `Громкость −` (`Vol-`) на пульте — Android загрузится с отключёнными сторонними приложениями, после чего в `Настройки → Приложения` можно включить штатный лаунчер.
   * На телевизорах **Яндекс ТВ (YaOS)** аппаратный сброс без пульта: выдернуть вилку из розетки, зажать физическую кнопку `Power` (или `CH+`) **на корпусе ТВ**, не отпуская воткнуть вилку обратно и держать ~20 секунд до старта сброса.


### Процесс установки через archinstall

1. Модифицировать скрипт `archinstall/lib/installer.py` в системной папке `site-packages`
    - Чтобы EFI-загрузчик установился в `/boot`, а не в папку/раздел EFI, в функции `_add_grub_bootloader()` переменная `boot_dir` должна остаться "`/boot`": выключить ветку

        ```
        if boot_partition.mountpoint and boot_partition.mountpoint != boot_dir:
            boot_dir_arg.append(f'--boot-directory={boot_partition.mountpoint}')
            boot_dir = boot_partition.mountpoint
        ```

    - Чтобы EFI-загрузчик установился не в папку/EFI-запись BOOT, а в папку с осмысленным именем, в этой же функции убрать из списка `add_options` элемент `--removable` и установить желаемое имя в `--bootloader-id=<имя>`. Запись BOOT может понадобиться только на некоторых системах, которые без неё не читают остальные записи; но тогда можно вручную скопировать папку EFI-загрузчика любой ОС на место BOOT.
1. Разметить диск (мой вариант, не как в Arch Wiki)
    - Корневой раздел
        + тип `Linux x86-64 root`
        + форматировать командой `mkfs.ext4 /dev/root_partition`
        + примонтировать к `/mnt/archinstall`
    - EFI-раздел, на нём будет или уже есть папка EFI
        + тип `EFI system partition`
        + форматировать (только если надо) командой `mkfs.fat -F 32 /dev/efi_system_partition`
        + монтировать к `/mnt/archinstall/boot/efi` (получится конфигурация `/mnt/archinstall/boot/efi/EFI/`<папки EFI-записей разных ОС>)
2. Настроить зеркала для своей страны
3. Disk configuration
    - Pre-mounted - `/mnt/archinstall`
4. Swap
    - Удалить swap on zram
5. Bootloader
    - Grub
6. Hostname
7. Root password
8. User account
    - Добавить своего пользователя в администраторы
9. Profile
    - Desktop - KDE Plasma
10. Audio
    - Pipewire
11. Kernel
    - `linux` и `linux-lts`
12. Network
    - Network manager
13. Additional packages
    - Подготовить список через пробел и добавить потом в сохранённый конфиг
14. Optional repositories
    - multilib
15. Timezone
16. Save configuration, чтобы подредактировать список пакетов

### Настройка свежей системы

- Раскомментировать "Color" и "VerbosePkgLists" в `/etc/pacman.conf`
- Обновить список зеркал: `sudo reflector --country Russia --sort rate --save /etc/pacman.d/mirrorlist`
- Выключить swap и zram ("`Just more ram + earlyoom ᕕ( ᐛ )ᕗ`")
    + `sudo systemctl enable --now earlyoom`
    + Выключить zram (см. "`zram-generator`", "`systemd.zram=0`" и [ссылку](https://askubuntu.com/a/1419240))
    + Если надо, настроить swap-раздел/файл нужного размера для гибернации
- Включить AUR
- Настроить локали, чтобы был нормальный порядок символов при сортировке, метрическая система мер, размеры бумаги и формат записи времени:
    + ~~Расскомментировать "`ru_RU.UTF-8`" в файле `/etc/locale.gen` и выполнить `sudo locale-gen`~~ (уже не надо: решил использовать `C.UTF-8`, т.к. она не переводит названия месяцев на русский)
    + Для всей системы - изменить файл `/etc/locale.conf` (или подать нужные значения в команду `sudo localectl set-locale ...=...`), а для одного пользователя - создать файл `$HOME/.config/locale.conf`

        ```
        LANG=en_US.UTF-8  # Распространяется на всё, что не задано
        LANGUAGE=  # Список fallback-языков через двоеточие (см. ниже)
        LC_ADDRESS=
        LC_COLLATE=C.UTF-8  # Для сортировки точек в ls
        LC_CTYPE=
        LC_IDENTIFICATION=
        LC_MEASUREMENT=C.UTF-8
        LC_MESSAGES=
        LC_MONETARY=
        LC_NAME=
        LC_NUMERIC=
        LC_PAPER=C.UTF-8
        LC_TELEPHONE=
        LC_TIME=C.UTF-8
        LC_ALL=  # Перезаписывает всё остальное
        ```
        
        Если в системе используется не английский язык (например, `LANG=ru_RU.UTF-8`), то можно заставить программы, которые ориентируются на локализацию через `gettext`, сначала искать его, а потом определённые другие языки, задав

        ```
        LANGUAGE=ru_RU:en:C:zh_CN
        ```

        Здесь сначала будет искаться "русский язык для России", потом "любой английский", потом "стандартный Си" (некоторые программы используют его указания сохранения своих английских переводов вместо "`en...`"), а потом упрощённый китайский. И уже если ничего не нашлось, то обращаются к дефолтному языку - `C` (*Наверное, иногда дефолтный задан в системе иначе? Не важно.*). Здесь `C` помещается вперёд, т.к. иногда английский может лежать в нём, а не в "`en...`". \
        Для английского это не актуально, т.к. если не задано, то fallback-языком становится "`C`".
        + После всего этого записать эти же переменные в `$HOME/.config/plasma-localerc` (простого удаления недостаточно)
        + В LibreOffice надо менять локаль в его собственных настройках из-за, возможно, того, что у него нет локали `C.UTF-8` и он откатывается на значение `LC_LANG`.
- Обновить базу данных документации: `tldr --update`
- Если будет мало места, то можно рискнуть и перенести часть системных папок на другой раздел/диск: [раз](https://superuser.com/a/529441), [два](https://askubuntu.com/a/668/969154), [три](https://serverfault.com/a/737336), [четыре](https://serverfault.com/a/597780); для безопасности стоит изучить, [как делать это с командой `init 1`](https://unix.stackexchange.com/a/131318/408809); [нельзя переносить `/usr`](https://askubuntu.com/a/1394599/969154)
- Пройтись по списку каждого из программ и пакетов и настроить её по документации
- Настроить ОС и программы под себя

#### Настройка KDE Plasma 6

- Ressources monitor (2 "S"; [GitHub](https://github.com/orblazer/plasma-applet-resources-monitor), [KDE Store on OpenDesktop.org](https://store.kde.org/p/2143899))
- Панель задач - "Icons-and-Text task manager"
    + Закрытие программ на панели задач средней кнопкой мыши
    + Выключить "Keep launchers separate", чтобы окна открывались на месте своих ярлыков
- Модификатор для движения окон - Meta вместо Alt (для возможности выделения ссылок через Alt)
- Настроить поведение средней кнопки мыши ~~(с ней творится что-то непонятное: вставка из буфера обмена, установка курсора, автоматическая прокрутка нигде не работает)~~
    + Выключить вставку из буфера обмена средней кнопкой мыши (опция появляется в Wayland-сеансе, но работает и для X11)
    + Включить "Press middle button and move mouse to scroll" ~~(непонятно, где это работает)~~
- Настроить тачпад для ноутбука (касание 2 и 3 пальцами, блокировка перетаскивания для автоматической прокрутки)
    + Если не работает средняя кнопка мыши по нажатию тремя пальцами на тачпаде, то попробовать переключить эту настройку и вернуть назад
- В настройках подсвечивать изменённые пункты
- Выключить KWallet
- Выключить индексацию Baloo и удалить кэш в настройках поиска
- Выключить, если надо, блокировку экрана после бездействия и пробуждения; выключение экране, сон и гибернацию при бездействии; включить автоматический вход в систему
- Добавить Yakuake в автозапуск
- Поменять тему экрана загрузки (SDDM): "Colors & Themes" - "Login Screen"
- Настроить приложения по умолчанию
- [Обои Linux Mint](http://packages.linuxmint.com/pool/main/m/); сакура в Sonya
- Если нет опций регулировки режима производительности, то см. [ссылку](https://wiki.archlinux.org/title/KDE#No_power_profile_options)
- Свои обои можно хранить в `$HOME/.local/share/wallpapers`, т.к. они не копируются туда при установке автоматически
- Настроить "Screen Edges"
    + Включить "Peek at Desktop" в правом нижнем углу и удалить соответствующую кнопку с панели задач
    + Включить "Overview" в левом верхнем углу
- Настроить системный трей
    + Уменьшить расстояние между значками
    + Спрятать значки Bluetooth, буфера обмена, яркости
- Включить проверку орфографии и, возможно, выключить автоматическое определение языка
- Включить эффект "Magic Lamp"

#### Настройка Dolphin

- При старте открывать домашнюю папку
- Добавить "/" в левую панель
- Показывать путь к папке сверху целиком (Show full path inside location bar)
- Показывать панель фильтрации
- Показывать скрытые файлы и папки
    + Сортировка - "Hidden Files last"
- Показать панель информации
    + Выключить "Show item on hover"
- В настройках Confirmation поменять "When opening an executable file"
- Включить панель меню (Show menu bar with all actions)
- Добавить кнопку "Up" и "Refresh" (скрыть её подпись)
- Установить для корзины вид с группировкой по исходному пути
    + "Show Additional Information" - Path и "Deletion Time"
    + View - "Show in Groups" и "Adjust View Display Style"
    + Убедиться, что это применилось только к корзине

#### Настройка Gwenview

- Колесико мыши для пролистывания картинок
- Сохранять зум и позицию между картинками
- Патч на интерполяцию "nearest neighbors" при зуме не от x4, а от x2
    + [Изначальный коммит](https://invent.kde.org/graphics/gwenview/-/commit/f88db58a516bcd76e8ea14d28881e5f2ea1d9d61)
    + Более новые версии кода: [22.12](https://invent.kde.org/graphics/gwenview/-/blob/release/22.12/lib/documentview/rasterimageitem.cpp#L122) и [24.12](https://invent.kde.org/graphics/gwenview/-/blob/release/24.12/lib/documentview/rasterimageitem.cpp#L144)
    + Надо искать в `libgwenviewlib.so.4.97.0` (или другой версии)

        ```
        movsd xmm0, qword...
        comisd xmm0, ...
        ```

        между вызовами `imp.QImage::format()` и `imp.QImage::size()` или константу 4.0 (`0x00 00 00 00 00 00 10 40h`) и менять адрес на любую константу 2.0 (`0x00 00 00 00 00 00 00 40h`) из read-only сегмента памяти.
        Пример для версии 24.12. Заменить `04fe0a00` на `14fe0a00`, чтобы перенести переменную на 16 байт дальше и попасть на 2.0 (конкретные адреса могут менять в зависимости от версии или даже дистрибутива Linux, так что надо искать через Cutter, Ghidra или другой дизассемблер/декомпилятор).

        ```
        0x000c2e19      ff1511410f00           call    qword [QImage::size() const] ; 0x1b6f30
        0x000c2e1f      f20f101589fb0a00       movsd   xmm2, qword [data.001729b0] ; 0x1729b0
        0x000c2e27      4531c0                 xor     r8d, r8d
        0x000c2e2a      4c89f6                 mov     rsi, r14
        0x000c2e2d      66480f6ee8             movq    xmm5, rax
        0x000c2e32      f20f100566fb0a00       movsd   xmm0, qword [data.001729a0] ; 0x1729a0
        0x000c2e3a      488b95c8feffff         mov     rdx, qword [rbp - data.00000138]
        0x000c2e41      f30fe6cd               cvtdq2pd xmm1, xmm5
        0x000c2e45      660f598dd0feffff       mulpd   xmm1, xmmword [rbp - data.00000130]
        0x000c2e4d      660f14d2               unpcklpd xmm2, xmm2
        0x000c2e51      488bbd00ffffff         mov     rdi, qword [rbp - data.00000100]
        0x000c2e58      660f14c0               unpcklpd xmm0, xmm0
        0x000c2e5c      660f54d1               andpd   xmm2, xmm1
        0x000c2e60      660f56c2               orpd    xmm0, xmm2
        0x000c2e64      660f58c1               addpd   xmm0, xmm1
        0x000c2e68      660fe6c0               cvttpd2dq xmm0, xmm0
        0x000c2e6c      660fd68550ffffff       movq    qword [rbp - 0xb0], xmm0
        
        0x000c2e74      f20f100504fe0a00       movsd   xmm0, qword [data.00172c80] ; 0x172c80
        
        0x000c2e7c      660f2f85f0feffff       comisd  xmm0, xmmword [rbp - 0x110]
        0x000c2e84      410f97c0               seta    r8b
        0x000c2e88      31c9                   xor     ecx, ecx
        0x000c2e8a      ff15482c0f00           call    qword [QImage::scaled(QSize const&, Qt::AspectRatioMode, Qt::TransformationMode) const] ; 0x1b5ad8
        0x000c2e90      488bbd08ffffff         mov     rdi, qword [rbp - 0xf8]
        0x000c2e97      ff15cb480f00           call    qword [QPaintDevice::QPaintDevice()] ; 0x1b7768
        0x000c2e9d      498d4510               lea     rax, [r13 + 0x10]
        0x000c2ea1      488b5590               mov     rdx, qword [rbp - 0x70]
        0x000c2ea5      488bbd08ffffff         mov     rdi, qword [rbp - 0xf8]
        0x000c2eac      488945a0               mov     qword [rbp - 0x60], rax
        0x000c2eb0      488b8570ffffff         mov     rax, qword [rbp - 0x90]
        0x000c2eb7      48c7459000000000       mov     qword [rbp - 0x70], 0
        0x000c2ebf      48899570ffffff         mov     qword [rbp - 0x90], rdx
        0x000c2ec6      488945b0               mov     qword [rbp - 0x50], rax
        0x000c2eca      ff15804c0f00           call    qword [QImage::~QImage()] ; 0x1b7b50
        0x000c2ed0      488bbd00ffffff         mov     rdi, qword [rbp - data.00000100]
        0x000c2ed7      ff15734c0f00           call    qword [QImage::~QImage()] ; 0x1b7b50
        0x000c2edd      4c89f7                 mov     rdi, r14
        0x000c2ee0      ff159a220f00           call    qword [QImage::format() const] ; 0x1b5180
        ```

#### Настройка fcitx

- В X11 и Wayland инициализация разная.
    + В X11 надо добавить в файл `~/.xprofile`:

        ```
        export GTK_IM_MODULE=fcitx
        export QT_IM_MODULE=fcitx
        export XMODIFIERS=@im=fcitx
        ```

    + В Wayland, вроде, достаточно выбрать Fcitx в "Virtual Keyboard" в системных настройках.
- В X11 на русской раскладке, возможно, не работают сочетания клавиш из-за перевода нажатий в русские буквы.
    + Возможно, поможет "Disable overriding XKB settings"
- Выключить "Global options" - Behavior - "Show preedit in application"
- Включить "Global options" - Behavior - "Share input state"
- Сделать так, чтобы русский и английский не переключались через Shift, а азиатские, возможно, переключались (один из двух вариантов)
    + Разнести русский и английский по разным группам
    + Выключить Global - Hotkey - "Temporarily switch between..."
- Можно поменять комбинацию клавиш включения метода ввода на две вместе: Alt+Shift и Shift+Alt, как это работает в Windows
- Включить "Configure addons" - "Classic User Interface" - "Prefer text icon"

#### Настройка шрифтов и font fallback

Установить шрифты из списка приложений.

Создать файл "/etc/fonts/local.conf" (глобально) или "$USER/.config/fontconfig/fonts.conf" или , где для каждого типа шрифтов указать по порядку желаемые шрифты:

```
<?xml version='1.0'?>
<!DOCTYPE fontconfig SYSTEM 'urn:fontconfig:fonts.dtd'>
<fontconfig>
 <!-- settings go here -->

 <alias>
  <family>serif</family>
  <prefer>
   <family>Noto Serif</family>
   <family>Noto Serif CJK SC</family>

   <family>Jigmo</family>
   <family>Jigmo2</family>
   <family>Jigmo3</family>

   <family>Noto Color Emoji</family>
   <family>Noto Sans Symbols</family>
   <family>Noto Sans Symbols 2</family>
  </prefer>
 </alias>

 <alias>
  <family>sans-serif</family>
  <prefer>
   <family>Noto Sans</family>
   <family>Noto Sans CJK SC</family>

   <family>Jigmo</family>
   <family>Jigmo2</family>
   <family>Jigmo3</family>

   <family>Noto Color Emoji</family>
   <family>Noto Sans Symbols</family>
   <family>Noto Sans Symbols 2</family>
  </prefer>
 </alias>

 <alias>
  <family>monospace</family>
  <prefer>
   <family>Noto Sans Mono</family>
   <family>Noto Sans Mono CJK SC</family>

   <family>Jigmo</family>
   <family>Jigmo2</family>
   <family>Jigmo3</family>

   <family>Noto Color Emoji</family>
   <family>Noto Sans Symbols</family>
   <family>Noto Sans Symbols 2</family>
  </prefer>
 </alias>

 <dir>~/.local/share/fonts</dir>
</fontconfig>
```

Потом выполнить:
```
fc-cache -fv
```

*Непонятно, надо ли в самом начале указывать дефолтный шрифт или этот файл задаёт только fallback-шрифты.*

- Чтобы проверить, какие шрифты содержат символ (например, 0x1F60A), выполнить

    ```
    fc-list :charset=1f60a
    ```

- Поиск шрифтов: `fc-match -s monospace`

#### Настройка Firefox

- Общие настройки
    + При запуске открывать последние закрытые вкладки
    + Предпочитаемый язык для сайтов
    + Всегда спрашивать, куда сохранять файл
    + Всегда спрашивать, сохранять ли или открывать файл
    + Проигрывать DRM-контент
    + Прокрутка средней кнопкой мыши
    + Всегда показывать полосу прокрутки
    + Выключить Picture-in-Picture
- Настройки домашней страницы
    + Выключить рекламу
- Настройки поиска
    + Показывать предложения и в приватном режиме
	+ Не показывать популярные результаты
	+ Выключить ненужные поисковики
- Настройки приватности
    + Блокировка автозапуска аудио и видео
- Дополнения
    + uBlock Origin
    + Return YouTube dislikes
    + To Google Translate

#### Настройка LibreOffice

- Везде
    + Тема иконок Colibre
    + В том же меню настроить поведение средней кнопки мыши
    + Настроить единицы измерения, формат даты, размеры бумаги, разделитель дробной части, сортировку списков (дни недели с понедельника, а не воскресенья): можно установить локаль на англоязычную с метрической системой и неделей с понедельника или же делать вручную (опасно из-за неучтённых параметров внутри остающейся американской локали)
    + Включить азиатские языки
    + Выключить автоматические заглавные буквы в начале предложения, автоматические маркированные и нумерованные списки, автоматический верхний регистр в порядковых числительных вида "1st" (выключить в Writer, т.к. там 2 по варианта этих опций)
    + Базовые шрифты для западных языков и CJK
- В Writer
    + Добавить "Entire page", "Page Width", "Character", "Paragraph", "Phonetic Guide"
    + Добавить проверку грамматики
- В Calc добавить на полосу:
    + Добавить строку, удалить столбец, удалить строку, удалить столбец
    + Оптимальная ширина столбца, оптимальная высота строки
    + Заполнить вверх/вниз/влево/вправо

#### Настройка Docker

- Добавить пользователя в группу для запуска Docker без `sudo`:

    ```
    sudo usermod -aG docker $USER
    ```

- Включить сервис, но не обычный:

    ```
    sudo systemctl enable docker.socket
    ```

    > Note that `docker.service` starts the service on boot, whereas `docker.socket` starts docker on first usage which can decrease boot times.
- `docker info`

#### Настройка VirtualBox Guest Additions

- Выполнить `sudo usermod -aG vboxsf $USER`
- Выполнить

    ```
    sudo systemctl enable vboxservice.service
    sudo modprobe -a vboxguest vboxsf vboxvideo
    ```

#### Настройка Ghostwriter

- Выключить: View - "Dark mode"
- Включить табуляцию через пробелы
- Добавить в файл `$HOME/.config/kde.org/ghostwriter.conf`

    ```
    [Preview]
    lastUsedExporter=Pandoc
    ```
    
  Когда этот параметр не задан (часто слетает сам после захода в настройки Preview), то при открытии настроек Preview программа будет вылетать.

### Приложения для установки

Искать пакеты можно [здесь](https://archlinux.org/packages/).

Звёздочка\* значит, что пакет из AUR.

#### Утилиты
- less
- fuse (для запуска AppImage; fuse2, не fuse3; запрашивать как просто "fuse")
- flatpak flatpak-kcm
- kwalletmanager
- git base-devel (для установки пакетов из AUR)
- mc
- fcitx5-im fcitx5-chinese-addons fcitx5-anthy
- usbutils (для usbpci)
- earlyoom
- man-db man-pages-ru tealdeer (tldr) (`man --locale=ru <query>`)
- sysstat (ради iostat)
- lm_sensors
- unrar zip unzip 7zip
- yt-dlp
- virtualbox-guest-utils (если на виртуальной машине VirtualBox)
- reflector (управляет списком серверов для загрузки пакетов) rsync
- intel-ucode или amd-ucode
- powerdevil power-profiles-daemon

#### Офис
- libreoffice-fresh hunspell hunspell-ru hunspell-en_us hunspell-en_gb hyphen hyphen-en hyphen-ru\*

#### Браузеры
- firefox
- chromium

#### Мессенджер
- telegram-desktop

#### Шрифты
- noto-fonts noto-fonts-cjk noto-fonts-emoji noto-fonts-extra ttf-jigmo

#### Инструменты
- yakuake
- qbittorrent
- gimp
- filelight
- kcharselect
- kcolorchooser
- kcalc
- krfb krdc (для подключения и "раздачи" удалённого рабочего стола)
- kdeconnect
- partitionmanager ntfs-3g
- okteta
- kfind
- docker docker-compose lazydocker\*
- kclock
- bpytop
- tor torbrowser-launcher(надо ли?)
- nekoray\* v2rayn\* hiddify\*
- openvpn networkmanager-openvpn (не уверен, что KDE сам не справится)
- rz-cutter(разобраться с декомпилятором: в версии из репозитория его нету) ghidra(опционально)

#### Markdown (*выбрать один*)
- ~~Kate (markdownpart marksman)~~
    + не отображает формулы
    + не поддерживает grid tables
    + нет синхронного скролинга
    + надо настроить font fallback в ОС
    + чуть-чуть кривая автоматическая мультиязычная проверка правописания: активен только 1 язык, и автоматическая детекция работает построчно
- **ghostwriter (pandoc-cli mathjax)**
    + нет синхронного скролинга
    + надо настроить font fallback в ОС
    + чуть-чуть кривая автоматическая мультиязычная проверка правописания: активен только 1 язык, и автоматическая детекция работает построчно
    + падает при открытии настроек превью
- ~~zettr~~
    + не поддерживает grid tables
    + подглючивают таблицы (например, строка | a | | | | | b | c |)
- apostrophe
    + нет мультиязычной проверки правописания
    + GTK

| Программа | Формулы | grid tables | Синхронный скролинг | Подхватывает все шрифты | Проверка правописания | Таблицы работают корректно |
|---|---|---|---|---|---|---|
| Kate |  |  |  | надо настроить <br/> font fallback | одновременно 1 язык | v |
| ghostwriter | v | v |  | надо настроить <br/> font fallback | одновременно 1 язык | v |
| zettr | v |  | v | v | v |  |
| apostrophe | v | v | v | v |  | v |

#### Проигрыватели
- vlc
- okular
- gwenview

#### TeX (1.1 или 2.8 Гб)
- texstudio texlive texlive-langcyrillic texlive-mathscience biber kbibtex
- Для `texlive` можно сделать `--ignore texlive-fontsextra`, т.к. он очень большой (1.7 Гб).

#### Wine (2 Гб)
- wine wine-gecko wine-mono winetricks

#### Редактор звука и диктофон (*выбрать один*)
- ~~audacity~~
- **tenacity**

#### Разработка
- python-pip

#### Сравнение файлов и папок (*выбрать один*)
- **meld**
- ~~kompare~~
- *kdiff3* (?)

### TODO

- Драйвера Bluetooth (
    + Работа Wi-Fi и Bluetooth одновременно (*bt_coex_active*) или Bluetooth вообще
    + Что-то из этого может помочь:
        * Ссылки: [установка драйверов](https://unix.stackexchange.com/questions/700030/no-default-controller-available-bluetooth-arch-linux), [Wi-Fi&Bluetooth coexistence](https://wiki.archlinux.org/title/Network_configuration/Wireless#Bluetooth_coexistence), [hardware](https://linux-hardware.org/?id=usb:8087-07da).
        * Просмотр информации:

            ```
            sudo lsusb -v | grep -i blue
            lsmod | grep btusb
            lsmod | grep -i blue
            modinfo btusb | less
            sudo dmesg | grep iwlwifi | less
            sudo dmesg | grep -i blue | less
            journalctl -b | grep -iE 'error|firmware' | less
            rfkill list
            bluetoothctl  # show
            ```

        * Установка драйверов:

            ```
            sudo pacman --needed -Syu bluez bluez-utils
            sudo systemctl enable --now bluetooth.service
            sudo systemctl restart --now bluetooth.service
            sudo systemctl stop bluetooth.service
            sudo /usr/lib/bluetooth/bluetoothd -n -d
            sudo nano /etc/modprobe.d/iwlwifi.conf
            sudo systemctl restart --now bluetooth.service
            sudo /etc/init.d/bluetooth restart
            sudo systemctl restart bluetooth
            sudo rmmod btusb && sudo modprobe btusb
            bluetoothctl: power on; agent on; scan on; pair MAC_address
            ```

- Firewall (ufw, firewalld, ...)
- Микрофон

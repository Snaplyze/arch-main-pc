# Безопасная настройка Arch Linux с GNOME 48 (март 2025)

Инструкция по настройке системы с Intel Core i7 13700k, RTX 4090, 32 ГБ ОЗУ, 4 NVME Gen4 и 2 HDD на Arch Linux с GNOME 48 в окружении Wayland с использованием systemd-boot и ядра linux-zen.

## ⚠️ Важные предупреждения перед началом

1. **Создайте резервную копию критических файлов**:
   ```bash
   sudo cp /etc/fstab /etc/fstab.bak
   sudo cp -r /boot/loader /boot/loader.bak
   ```

2. **Этапы** — выполняйте инструкцию последовательно, проверяя результат каждой команды.

3. **Осторожно с системными файлами** — при сомнениях создавайте резервные копии.

## 1. Обновление системы и базовая настройка

```bash
# Обновление системы
sudo pacman -Syu

# Установка intel-ucode (микрокод процессора Intel)
sudo pacman -S intel-ucode

# Установка базовых утилит
sudo pacman -S base-devel git curl wget bash-completion
```

## 2. Установка драйверов NVIDIA и настройка для Wayland

```bash
# Установка драйверов NVIDIA и необходимых пакетов (проверьте наличие пакетов)
sudo pacman -S nvidia-dkms nvidia-utils nvidia-settings libva-nvidia-driver

# Создаём конфигурационный файл для NVIDIA
sudo mkdir -p /etc/modprobe.d/
cat << EOF | sudo tee /etc/modprobe.d/nvidia.conf
options nvidia-drm modeset=1
options nvidia NVreg_PreserveVideoMemoryAllocations=1
EOF

# Добавляем модули в initramfs для ядра linux-zen
# Сначала сделаем резервную копию
sudo cp /etc/mkinitcpio.conf /etc/mkinitcpio.conf.bak

# Проверяем текущие модули
current_modules=$(grep "^MODULES=" /etc/mkinitcpio.conf)
echo "Текущие модули: $current_modules"

# Добавляем модули NVIDIA, сохраняя существующие
if ! grep -q "nvidia" /etc/mkinitcpio.conf; then
    sudo sed -i 's/^MODULES=(/MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm /' /etc/mkinitcpio.conf
    echo "Модули NVIDIA добавлены в mkinitcpio.conf"
else
    echo "Модули NVIDIA уже присутствуют в конфигурации"
fi

# Пересоздаем образы initramfs
sudo mkinitcpio -P linux-zen
```

## 3. Оптимизация NVMe и дисков

```bash
# Установка необходимых утилит
sudo pacman -S nvme-cli hdparm smartmontools

# Проверка и включение TRIM для NVMe
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer

# Проверка текущих параметров NVMe
sudo nvme list
sudo nvme smart-log /dev/nvme0n1  # Повторите для каждого nvme устройства

# Проверим текущие настройки BTRFS (только вывод информации, без изменений!)
sudo btrfs filesystem df /
sudo btrfs filesystem usage /

# Оптимизация кэша для улучшения производительности без изменения монтирования
cat << EOF | sudo tee /etc/sysctl.d/60-btrfs-performance.conf
# Увеличение лимита кэша метаданных для BTRFS
vm.dirty_bytes = 4294967296
vm.dirty_background_bytes = 1073741824
EOF

sudo sysctl --system
```

## 4. Форматирование и монтирование дополнительных дисков (NVMe и HDD)

> ⚠️ **ВНИМАНИЕ**: Следующие команды полностью удалят данные на указанных дисках.  
> Убедитесь, что работаете с правильными дисками! Проверьте их с помощью `lsblk`.

```bash
# Перед форматированием проверим список всех дисков в системе
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS,LABEL

# ВАЖНО: Выполняйте следующие команды ТОЛЬКО если уверены, что диски пусты/не нужны!
```

### 4.1 Форматирование свободных NVMe дисков в ext4

```bash
# Создаем подкаталог для хранения скриптов
mkdir -p ~/setup_scripts

# Создаем скрипт для форматирования NVMe
cat << 'EOF' > ~/setup_scripts/format_nvme.sh
#!/bin/bash
# Этот скрипт форматирует NVMe диски в ext4

# Список NVMe дисков для форматирования (ИЗМЕНИТЬ ПРИ НЕОБХОДИМОСТИ)
NVME_DISKS=("/dev/nvme1n1" "/dev/nvme2n1" "/dev/nvme3n1")

# Функция для подтверждения действия
confirm() {
    local prompt="$1 (y/N): "
    local response
    read -p "$prompt" response
    case "$response" in
        [yY][eE][sS]|[yY]) return 0 ;;
        *) return 1 ;;
    esac
}

# Покажем текущее состояние дисков
echo "Текущее состояние дисков:"
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS,LABEL

# Запрашиваем подтверждение
if ! confirm "Внимание! Все данные на указанных NVMe дисках будут уничтожены. Продолжить?"; then
    echo "Операция отменена."
    exit 0
fi

# Форматирование каждого диска
for disk in "${NVME_DISKS[@]}"; do
    if [ ! -b "$disk" ]; then
        echo "ОШИБКА: Диск $disk не существует!"
        continue
    fi
    
    label=$(basename "$disk" | tr -d '0123456789/')
    mount_point="/mnt/$label"
    
    echo "Форматирование $disk (метка: $label, точка монтирования: $mount_point)"
    
    # Создание таблицы разделов GPT
    sudo parted "$disk" mklabel gpt
    
    # Создание одного большого раздела
    sudo parted -a optimal "$disk" mkpart primary ext4 0% 100%
    
    # Форматирование в ext4
    sudo mkfs.ext4 -L "$label" "$disk"
    
    # Создание точки монтирования
    sudo mkdir -p "$mount_point"
    
    # Добавление записи в fstab, если её ещё нет
    if ! grep -q "LABEL=$label" /etc/fstab; then
        echo "LABEL=$label  $mount_point  ext4  defaults,noatime,x-gvfs-show  0 2" | sudo tee -a /etc/fstab
    fi
done

# Монтирование всех дисков
sudo mount -a

echo "Форматирование NVMe завершено. Новое состояние дисков:"
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS,LABEL
EOF

# Делаем скрипт исполняемым
chmod +x ~/setup_scripts/format_nvme.sh

# ВНИМАНИЕ: Запустите скрипт вручную когда будете готовы!
echo "Скрипт для форматирования NVMe создан в ~/setup_scripts/format_nvme.sh"
echo "Запустите его вручную, когда будете готовы форматировать диски."
```

### 4.2 Форматирование HDD дисков

```bash
# Создаем скрипт для форматирования HDD
cat << 'EOF' > ~/setup_scripts/format_hdd.sh
#!/bin/bash
# Этот скрипт форматирует HDD диски в ext4

# Список HDD дисков для форматирования (ИЗМЕНИТЬ ПРИ НЕОБХОДИМОСТИ)
HDD_DISKS=("/dev/sda" "/dev/sdb")

# Функция для подтверждения действия
confirm() {
    local prompt="$1 (y/N): "
    local response
    read -p "$prompt" response
    case "$response" in
        [yY][eE][sS]|[yY]) return 0 ;;
        *) return 1 ;;
    esac
}

# Покажем текущее состояние дисков
echo "Текущее состояние дисков:"
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS,LABEL

# Запрашиваем подтверждение
if ! confirm "Внимание! Все данные на указанных HDD дисках будут уничтожены. Продолжить?"; then
    echo "Операция отменена."
    exit 0
fi

# Оптимизация HDD дисков перед форматированием
for disk in "${HDD_DISKS[@]}"; do
    if [ ! -b "$disk" ]; then
        echo "ОШИБКА: Диск $disk не существует!"
        continue
    fi
    
    echo "Применение оптимизаций для $disk"
    sudo hdparm -W 1 "$disk"  # Включение кэша записи
    sudo hdparm -B 127 -S 120 "$disk"  # Настройка энергосбережения
done

# Форматирование каждого диска
for disk in "${HDD_DISKS[@]}"; do
    if [ ! -b "$disk" ]; then
        echo "ОШИБКА: Диск $disk не существует!"
        continue
    fi
    
    label="hdd$(echo "$disk" | grep -o '[a-z]$')"
    mount_point="/mnt/$label"
    
    echo "Форматирование $disk (метка: $label, точка монтирования: $mount_point)"
    
    # Создание таблицы разделов GPT
    sudo parted "$disk" mklabel gpt
    
    # Создание одного большого раздела
    sudo parted -a optimal "$disk" mkpart primary ext4 0% 100%
    
    # Форматирование в ext4
    partition="${disk}1"  # sda → sda1
    sudo mkfs.ext4 -L "$label" "$partition"
    
    # Создание точки монтирования
    sudo mkdir -p "$mount_point"
    
    # Добавление записи в fstab, если её ещё нет
    if ! grep -q "LABEL=$label" /etc/fstab; then
        echo "LABEL=$label  $mount_point  ext4  defaults,noatime,x-gvfs-show  0 2" | sudo tee -a /etc/fstab
    fi
done

# Монтирование всех дисков
sudo mount -a

echo "Форматирование HDD завершено. Новое состояние дисков:"
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS,LABEL
EOF

# Делаем скрипт исполняемым
chmod +x ~/setup_scripts/format_hdd.sh

# ВНИМАНИЕ: Запустите скрипт вручную когда будете готовы!
echo "Скрипт для форматирования HDD создан в ~/setup_scripts/format_hdd.sh"
echo "Запустите его вручную, когда будете готовы форматировать диски."
```

## 5. Настройка чистого экрана при загрузке и выключении

```bash
# Создадим резервную копию загрузчика
sudo cp -r /boot/loader /boot/loader.bak

# Создаем или редактируем файл параметров ядра (сначала проверим)
sudo mkdir -p /etc/kernel/cmdline.d/

# Проверка существующих параметров
if [ -f /etc/kernel/cmdline.d/quiet.conf ]; then
    echo "Файл /etc/kernel/cmdline.d/quiet.conf уже существует:"
    cat /etc/kernel/cmdline.d/quiet.conf
    echo -e "\nВы хотите заменить его? (y/N): "
    read replace_file
    if [[ ! "$replace_file" =~ ^[Yy]$ ]]; then
        echo "Файл не изменен."
    else
        cat << EOF | sudo tee /etc/kernel/cmdline.d/quiet.conf
quiet loglevel=3 rd.systemd.show_status=false rd.udev.log_level=3 vt.global_cursor_default=0 splash plymouth.enable=1
EOF
        echo "Файл параметров обновлен."
    fi
else
    cat << EOF | sudo tee /etc/kernel/cmdline.d/quiet.conf
quiet loglevel=3 rd.systemd.show_status=false rd.udev.log_level=3 vt.global_cursor_default=0 splash plymouth.enable=1
EOF
    echo "Файл параметров создан."
fi

# Отключение журналирования на tty (с проверкой)
sudo mkdir -p /etc/systemd/journald.conf.d/
if [ -f /etc/systemd/journald.conf.d/quiet.conf ]; then
    echo "Файл /etc/systemd/journald.conf.d/quiet.conf уже существует:"
    cat /etc/systemd/journald.conf.d/quiet.conf
    echo -e "\nПропускаем изменение файла журналирования."
else
    cat << EOF | sudo tee /etc/systemd/journald.conf.d/quiet.conf
[Journal]
TTYPath=/dev/null
EOF
    echo "Отключение журналирования на tty настроено."
fi

# Проверка наличия plymouth
if pacman -Qs plymouth > /dev/null; then
    echo "Plymouth уже установлен."
else
    # Установка Plymouth для красивой загрузки
    sudo pacman -S plymouth
fi

# Проверка hooks в mkinitcpio.conf
if ! grep -q "plymouth" /etc/mkinitcpio.conf; then
    # Резервная копия перед изменением
    sudo cp /etc/mkinitcpio.conf /etc/mkinitcpio.conf.plymouth.bak
    
    # Добавляем plymouth в HOOKS
    sudo sed -i 's/^HOOKS=.*/HOOKS=(base udev plymouth autodetect modconf kms keyboard keymap consolefont block filesystems fsck)/' /etc/mkinitcpio.conf
    
    # Пересоздаем образы
    sudo mkinitcpio -P linux-zen
    echo "Plymouth добавлен в initramfs."
else
    echo "Plymouth уже присутствует в HOOKS mkinitcpio.conf."
fi

# Проверка конфигурации systemd-boot
echo "Проверка loader.conf..."
if [ -f /boot/loader/loader.conf ]; then
    echo "Текущее содержимое /boot/loader/loader.conf:"
    cat /boot/loader/loader.conf
    
    echo -e "\nВы хотите обновить файл конфигурации загрузчика? (y/N): "
    read update_loader
    if [[ "$update_loader" =~ ^[Yy]$ ]]; then
        sudo cp /boot/loader/loader.conf /boot/loader/loader.conf.bak
        cat << EOF | sudo tee /boot/loader/loader.conf
default arch-zen.conf
timeout 0
console-mode max
editor no
EOF
        echo "Файл loader.conf обновлен."
    else
        echo "Файл loader.conf оставлен без изменений."
    fi
else
    sudo mkdir -p /boot/loader/
    cat << EOF | sudo tee /boot/loader/loader.conf
default arch-zen.conf
timeout 0
console-mode max
editor no
EOF
    echo "Файл loader.conf создан."
fi

# Создание загрузочной записи только если она не существует
if [ ! -f /boot/loader/entries/arch-zen.conf ]; then
    sudo mkdir -p /boot/loader/entries/
    cat << EOF | sudo tee /boot/loader/entries/arch-zen.conf
title Arch Linux Zen
linux /vmlinuz-linux-zen
initrd /intel-ucode.img
initrd /initramfs-linux-zen.img
options $(cat /etc/kernel/cmdline.d/quiet.conf)
EOF
    echo "Создана загрузочная запись arch-zen.conf."
else
    echo "Загрузочная запись arch-zen.conf уже существует. Проверьте её содержимое:"
    cat /boot/loader/entries/arch-zen.conf
    
    echo -e "\nХотите обновить запись? (y/N): "
    read update_entry
    if [[ "$update_entry" =~ ^[Yy]$ ]]; then
        sudo cp /boot/loader/entries/arch-zen.conf /boot/loader/entries/arch-zen.conf.bak
        cat << EOF | sudo tee /boot/loader/entries/arch-zen.conf
title Arch Linux Zen
linux /vmlinuz-linux-zen
initrd /intel-ucode.img
initrd /initramfs-linux-zen.img
options $(cat /etc/kernel/cmdline.d/quiet.conf)
EOF
        echo "Загрузочная запись обновлена."
    else
        echo "Загрузочная запись оставлена без изменений."
    fi
fi

# Обновление загрузчика
sudo bootctl update

echo "Настройка чистого экрана загрузки завершена."
```

## 6. Установка Paru в скрытую папку и настройка

```bash
# Проверка, установлен ли paru
if command -v paru &> /dev/null; then
    echo "Paru уже установлен. Пропускаем установку."
else
    echo "Установка Paru..."
    # Создание скрытой папки для Paru
    mkdir -p ~/.local/paru
    
    # Клонирование репозитория Paru
    git clone https://aur.archlinux.org/paru.git ~/.local/paru/build
    cd ~/.local/paru/build
    
    # Сборка и установка Paru
    makepkg -si
fi

# Настройка Paru
mkdir -p ~/.config/paru
if [ -f ~/.config/paru/paru.conf ]; then
    echo "Файл конфигурации paru уже существует."
    echo "Проверьте его содержимое: ~/.config/paru/paru.conf"
else
    cat << EOF > ~/.config/paru/paru.conf
[options]
BottomUp
SudoLoop
Devel
CleanAfter
BatchInstall
NewVersion
UpgradeMenu
CombinedUpgrade
RemoveMake
KeepRepoCache
Redownload 
NewsOnUpgrade

# Папка для скачивания
CloneDir = ~/.local/paru/packages
EOF
    echo "Файл конфигурации paru создан."
fi
```

## 7. Включение Flathub и установка GNOME Software

```bash
# Проверка, установлен ли Flatpak
if command -v flatpak &> /dev/null; then
    echo "Flatpak уже установлен."
else
    echo "Установка Flatpak..."
    sudo pacman -S flatpak
fi

# Добавление репозитория Flathub (если его нет)
if flatpak remotes | grep -q flathub; then
    echo "Репозиторий Flathub уже добавлен."
else
    echo "Добавление репозитория Flathub..."
    flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
fi

# Установка GNOME Software
if pacman -Qs gnome-software > /dev/null; then
    echo "GNOME Software уже установлен."
else
    echo "Установка GNOME Software..."
    sudo pacman -S gnome-software
fi

# Установка платформы GNOME (проверяем версию)
echo "Установка GNOME Platform через Flatpak..."
flatpak install -y flathub org.gnome.Platform//45
```

## 8. Установка Steam и необходимых библиотек

```bash
# Проверка наличия multilib репозитория
if grep -q "^\[multilib\]" /etc/pacman.conf; then
    echo "Репозиторий multilib уже включен."
else
    echo "Включение multilib репозитория..."
    sudo sed -i "/\[multilib\]/,/Include/"'s/^#//' /etc/pacman.conf
    sudo pacman -Syu
fi

# Проверка наличия Steam
if pacman -Qs steam > /dev/null; then
    echo "Steam уже установлен."
else
    echo "Установка Steam и необходимых библиотек..."
    # Установка по порядку, с проверкой ошибок
    sudo pacman -S --needed steam lib32-nvidia-utils \
      lib32-vulkan-icd-loader vulkan-icd-loader \
      lib32-vulkan-intel vulkan-intel \
      lib32-mesa vulkan-tools \
      lib32-libva-mesa-driver lib32-mesa-vdpau \
      libva-mesa-driver mesa-vdpau \
      lib32-openal lib32-alsa-plugins
fi
```

## 9. Установка Proton GE

```bash
# Создание директории для Proton GE, если её нет
mkdir -p ~/.steam/root/compatibilitytools.d/

# Проверка, установлен ли proton-ge-custom
if paru -Qs proton-ge-custom-bin > /dev/null; then
    echo "Proton GE уже установлен через paru."
else
    echo "Установка Proton GE..."
    
    # Спрашиваем, какой метод установки предпочтителен
    echo "Установить Proton GE через:"
    echo "1) paru (рекомендуется, автоматическое обновление)"
    echo "2) ручная загрузка (последняя версия)"
    read -p "Выберите метод (1/2): " ge_method
    
    if [ "$ge_method" = "1" ] || [ -z "$ge_method" ]; then
        paru -S proton-ge-custom-bin
    else
        # Ручная установка последней версии
        echo "Скачивание последней версии Proton GE..."
        PROTON_VERSION=$(curl -s https://api.github.com/repos/GloriousEggroll/proton-ge-custom/releases/latest | grep "tag_name" | cut -d\" -f4)
        wget -O /tmp/proton-ge.tar.gz https://github.com/GloriousEggroll/proton-ge-custom/releases/download/${PROTON_VERSION}/GE-Proton${PROTON_VERSION:1}.tar.gz
        
        echo "Распаковка Proton GE..."
        tar -xzf /tmp/proton-ge.tar.gz -C ~/.steam/root/compatibilitytools.d/
        rm /tmp/proton-ge.tar.gz
        
        echo "Proton GE успешно установлен вручную."
    fi
fi
```

## 10. Оптимизация для Wayland

```bash
# Проверка существующего файла environment
if [ -f /etc/environment ]; then
    echo "Текущее содержимое /etc/environment:"
    cat /etc/environment
    echo ""
fi

# Проверяем, нужно ли добавить переменные окружения
echo "Добавить переменные окружения для Wayland и NVIDIA? (y/N): "
read add_env
if [[ "$add_env" =~ ^[Yy]$ ]]; then
    # Создаем резервную копию
    sudo cp /etc/environment /etc/environment.bak
    
    # Добавляем переменные, которых еще нет
    echo "# Настройки Wayland и NVIDIA" | sudo tee -a /etc/environment
    
    grep -q "LIBVA_DRIVER_NAME" /etc/environment || echo "LIBVA_DRIVER_NAME=nvidia" | sudo tee -a /etc/environment
    grep -q "XDG_SESSION_TYPE" /etc/environment || echo "XDG_SESSION_TYPE=wayland" | sudo tee -a /etc/environment
    grep -q "GBM_BACKEND" /etc/environment || echo "GBM_BACKEND=nvidia-drm" | sudo tee -a /etc/environment
    grep -q "__GLX_VENDOR_LIBRARY_NAME" /etc/environment || echo "__GLX_VENDOR_LIBRARY_NAME=nvidia" | sudo tee -a /etc/environment
    grep -q "WLR_NO_HARDWARE_CURSORS" /etc/environment || echo "WLR_NO_HARDWARE_CURSORS=1" | sudo tee -a /etc/environment
    grep -q "MOZ_ENABLE_WAYLAND" /etc/environment || echo "MOZ_ENABLE_WAYLAND=1" | sudo tee -a /etc/environment
    grep -q "MOZ_WEBRENDER" /etc/environment || echo "MOZ_WEBRENDER=1" | sudo tee -a /etc/environment
    grep -q "QT_QPA_PLATFORM" /etc/environment || echo "QT_QPA_PLATFORM=wayland" | sudo tee -a /etc/environment
    grep -q "QT_WAYLAND_DISABLE_WINDOWDECORATION" /etc/environment || echo "QT_WAYLAND_DISABLE_WINDOWDECORATION=1" | sudo tee -a /etc/environment
fi

# Установка дополнительных полезных пакетов для Wayland
echo "Установка библиотек для Wayland..."
packages_to_install=()

# Проверяем наличие каждого пакета
for pkg in qt6-wayland qt5-wayland xorg-xwayland; do
    if ! pacman -Qi $pkg &> /dev/null; then
        packages_to_install+=($pkg)
    fi
done

# Устанавливаем пакеты, если есть отсутствующие
if [ ${#packages_to_install[@]} -gt 0 ]; then
    echo "Устанавливаем: ${packages_to_install[*]}"
    sudo pacman -S --needed ${packages_to_install[*]}
else
    echo "Все необходимые библиотеки Wayland уже установлены."
fi

# Предлагаем установку xwaylandvideobridge из AUR
echo -e "\nХотите установить xwaylandvideobridge для захвата экрана XWayland-приложений? (y/N): "
read install_xwayland
if [[ "$install_xwayland" =~ ^[Yy]$ ]]; then
    paru -S xwaylandvideobridge
else
    echo "Пропускаем установку xwaylandvideobridge."
fi
```

## 11. Управление питанием и оптимизации

```bash
# Установка power-profiles-daemon
if systemctl status power-profiles-daemon.service &> /dev/null; then
    echo "power-profiles-daemon уже работает."
else
    echo "Установка и настройка power-profiles-daemon..."
    sudo pacman -S --needed power-profiles-daemon
    sudo systemctl enable power-profiles-daemon.service
    sudo systemctl start power-profiles-daemon.service
fi

# Настройка автоматического перехода HDD в спящий режим
# Проверяем, установлен ли hdparm
if ! command -v hdparm &> /dev/null; then
    echo "Установка hdparm..."
    sudo pacman -S hdparm
fi

# Создание правил для перевода HDD в спящий режим
echo "Настройка правил для HDD..."
cat << EOF | sudo tee /etc/udev/rules.d/69-hdparm.rules
# Правила для перевода HDD в спящий режим при простое
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", RUN+="/usr/bin/hdparm -B 127 -S 120 /dev/%k"
EOF

# Настройка systemd timer для регулярного применения настроек
echo "Настройка systemd timer для HDD..."
cat << EOF | sudo tee /etc/systemd/system/hdparm-idle.service
[Unit]
Description=Set hard disk spindown timeout

[Service]
Type=oneshot
ExecStart=/usr/bin/hdparm -B 127 -S 120 /dev/sda
ExecStart=/usr/bin/hdparm -B 127 -S 120 /dev/sdb
EOF

cat << EOF | sudo tee /etc/systemd/system/hdparm-idle.timer
[Unit]
Description=Run hdparm spindown setting regularly

[Timer]
OnBootSec=1min
OnUnitActiveSec=60min

[Install]
WantedBy=timers.target
EOF

sudo systemctl enable hdparm-idle.timer
sudo systemctl start hdparm-idle.timer

# Настройка планировщика для NVMe и HDD
echo "Настройка планировщиков ввода-вывода..."
cat << EOF | sudo tee /etc/udev/rules.d/60-ioschedulers.rules
# Планировщик для NVMe
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"

# Планировщик для HDD
ACTION=="add|change", KERNEL=="sd[a-z]|hd[a-z]", ATTR{queue/scheduler}="bfq"
EOF

# Настройка swappiness для оптимизации использования ОЗУ
echo "Настройка swappiness..."
cat << EOF | sudo tee /etc/sysctl.d/99-swappiness.conf
vm.swappiness=10
EOF

# Применяем настройки сразу
sudo sysctl vm.swappiness=10

# Включение автоматической очистки кэша
echo "Настройка очистки кэша..."
cat << EOF | sudo tee /etc/systemd/system/clear-cache.service
[Unit]
Description=Clear Memory Cache

[Service]
Type=oneshot
ExecStart=/usr/bin/sh -c "sync; echo 3 > /proc/sys/vm/drop_caches"
EOF

cat << EOF | sudo tee /etc/systemd/system/clear-cache.timer
[Unit]
Description=Clear Memory Cache Timer

[Timer]
OnBootSec=15min
OnUnitActiveSec=1h

[Install]
WantedBy=timers.target
EOF

sudo systemctl enable clear-cache.timer
sudo systemctl start clear-cache.timer
```

## 12. Настройка локализации и безопасности

```bash
# Проверка локализации
if grep -q "ru_RU.UTF-8 UTF-8" /etc/locale.gen; then
    echo "Локаль ru_RU.UTF-8 уже добавлена в locale.gen."
else
    echo "Добавление русской локали..."
    echo "ru_RU.UTF-8 UTF-8" | sudo tee -a /etc/locale.gen
    sudo locale-gen
fi

# Проверка файла locale.conf
if [ -f /etc/locale.conf ]; then
    echo "Текущее содержимое /etc/locale.conf:"
    cat /etc/locale.conf
    
    echo -e "\nХотите установить русскую локаль? (y/N): "
    read set_locale
    if [[ "$set_locale" =~ ^[Yy]$ ]]; then
        echo "LANG=ru_RU.UTF-8" | sudo tee /etc/locale.conf
        echo "Установлена русская локаль."
    fi
else
    echo "LANG=ru_RU.UTF-8" | sudo tee /etc/locale.conf
    echo "Установлена русская локаль."
fi

# Настройка часового пояса (с подтверждением)
echo -e "\nУстановить часовой пояс для Москвы? (y/N): "
read set_timezone
if [[ "$set_timezone" =~ ^[Yy]$ ]]; then
    sudo ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
    sudo hwclock --systohc
    echo "Установлен часовой пояс для Москвы."
fi

# Настройка базового файрвола
if systemctl status ufw &> /dev/null; then
    echo "UFW уже запущен."
else
    echo "Настройка UFW..."
    sudo pacman -S --needed ufw
    sudo systemctl enable ufw
    sudo systemctl start ufw
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow ssh
    sudo ufw enable
fi

# Улучшение безопасности системы
echo "Настройка дополнительных параметров безопасности..."

# Отключение core dumps
if grep -q "hard core 0" /etc/security/limits.conf; then
    echo "Ограничения core dumps уже настроены."
else
    echo "* hard core 0" | sudo tee -a /etc/security/limits.conf
    echo "* soft core 0" | sudo tee -a /etc/security/limits.conf
fi

if [ -f /etc/sysctl.d/51-coredump.conf ]; then
    echo "Файл конфигурации core dumps уже существует."
else
    echo "kernel.core_pattern=/dev/null" | sudo tee -a /etc/sysctl.d/51-coredump.conf
    sudo sysctl -p /etc/sysctl.d/51-coredump.conf
fi
```

## 13. Установка дополнительных программ и утилит

```bash
# Установка популярных утилит
echo "Установка дополнительных утилит..."
packages_to_install=()

# Проверяем наличие каждого пакета
for pkg in htop neofetch bat exa ripgrep fd; do
    if ! pacman -Qi $pkg &> /dev/null; then
        packages_to_install+=($pkg)
    fi
done

# Устанавливаем пакеты, если есть отсутствующие
if [ ${#packages_to_install[@]} -gt 0 ]; then
    echo "Устанавливаем: ${packages_to_install[*]}"
    sudo pacman -S --needed ${packages_to_install[*]}
else
    echo "Все базовые утилиты уже установлены."
fi

# Настройка Gnome-keyring для хранения паролей
if ! pacman -Qi gnome-keyring &> /dev/null; then
    echo "Установка gnome-keyring и seahorse..."
    sudo pacman -S --needed gnome-keyring seahorse
fi

# Проверка наличия настроек keyring в bash_profile
if grep -q "gnome-keyring-daemon" ~/.bash_profile; then
    echo "Настройки gnome-keyring уже добавлены в bash_profile."
else
    echo "Добавление настроек gnome-keyring в bash_profile..."
    echo "eval \$(gnome-keyring-daemon --start)" >> ~/.bash_profile
    echo "export SSH_AUTH_SOCK" >> ~/.bash_profile
fi

# Настройка автоматических снапшотов BTRFS
if command -v snapper &> /dev/null; then
    echo "Snapper уже установлен."
else
    echo "Установка Snapper для автоматических снапшотов BTRFS..."
    sudo pacman -S --needed snapper
    
    # Проверка, существует ли конфигурация для root
    if [ -d /etc/snapper/configs/root ]; then
        echo "Конфигурация Snapper для root уже существует."
    else
        sudo snapper -c root create-config /
        sudo systemctl enable snapper-timeline.timer
        sudo systemctl start snapper-timeline.timer
    fi
fi

# Создание сервиса для снапшотов перед обновлениями
mkdir -p ~/.config/systemd/user/
if [ -f ~/.config/systemd/user/pacman-snapshot.service ]; then
    echo "Сервис pacman-snapshot уже существует."
else
    cat << EOF > ~/.config/systemd/user/pacman-snapshot.service
[Unit]
Description=Create BTRFS snapshot before system upgrade
Before=pacman.service

[Service]
Type=oneshot
ExecStart=/usr/bin/snapper -c root create -d "Before system upgrade"

[Install]
WantedBy=pacman.service
EOF

    systemctl --user enable pacman-snapshot.service
    echo "Сервис pacman-snapshot создан и включен."
fi
```

## 14. Файловый менеджер и интеграция GVFS

```bash
# Установка gvfs для отображения дисков в файловом менеджере
if pacman -Qs gvfs > /dev/null; then
    echo "GVFS уже установлен."
else
    echo "Установка GVFS для интеграции с файловым менеджером..."
    sudo pacman -S gvfs
fi

# Дополнительные компоненты GVFS для работы с внешними устройствами
echo "Установка дополнительных компонентов GVFS..."
sudo pacman -S --needed gvfs-mtp gvfs-smb gvfs-nfs

echo "Интеграция GVFS завершена. Диски должны теперь отображаться в файловом менеджере."
```

## 15. Финальная проверка и перезагрузка

```bash
# Проверка критических компонентов перед перезагрузкой
echo "Проверка системных компонентов..."

# Проверка инициализации
if [ ! -f /boot/initramfs-linux-zen.img ]; then
    echo "ОШИБКА: Отсутствует образ initramfs. Выполните: sudo mkinitcpio -P linux-zen"
    exit 1
fi

# Проверка загрузчика
if [ ! -f /boot/loader/entries/arch-zen.conf ]; then
    echo "ОШИБКА: Отсутствует конфигурация загрузчика. Выполните настройку systemd-boot."
    exit 1
fi

# Проверка fstab
if ! sudo findmnt -n -o SOURCE / &> /dev/null; then
    echo "ОШИБКА: Проблема с fstab. Проверьте монтирование корневого раздела."
    exit 1
fi

echo -e "\nВсе проверки пройдены успешно!\n"

echo "Настройка системы завершена. Рекомендуется перезагрузить систему."
echo "Выполните 'sudo reboot' для применения всех изменений."
```

После перезагрузки у вас будет полностью настроенная система Arch Linux с GNOME 48 на ядре linux-zen, работающая в окружении Wayland, с драйверами NVIDIA-DKMS для RTX 4090, оптимизированной системой и дополнительными дисками на ext4, чистым экраном загрузки/выключения, управлением питанием для автоматического перехода HDD в спящий режим, а также всеми необходимыми приложениями и утилитами.
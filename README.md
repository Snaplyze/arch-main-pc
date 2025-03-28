# Настройка Arch Linux с GNOME 48 после чистой установки

Подробная инструкция по настройке системы с Intel Core i7 13700k, RTX 4090, 32 ГБ ОЗУ, 4 NVME Gen4 и 2 HDD на Arch Linux с GNOME 48 в окружении Wayland с использованием systemd-boot и ядра linux-zen.

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
# Установка драйверов NVIDIA и необходимых пакетов
sudo pacman -S nvidia-dkms nvidia-utils nvidia-settings libva-nvidia-driver

# Создаём конфигурационный файл для NVIDIA
sudo mkdir -p /etc/modprobe.d/
cat << EOF | sudo tee /etc/modprobe.d/nvidia.conf
options nvidia-drm modeset=1
options nvidia NVreg_PreserveVideoMemoryAllocations=1
EOF

# Добавляем модули в initramfs для ядра linux-zen
sudo sed -i 's/^MODULES=.*/MODULES=(btrfs nvidia nvidia_modeset nvidia_uvm nvidia_drm)/' /etc/mkinitcpio.conf
sudo mkinitcpio -P linux-zen
```

## 3. Оптимизация хранилища (NVMe и HDD)

### Оптимизация NVMe

```bash
# Установка необходимых утилит
sudo pacman -S nvme-cli hdparm

# Проверка и включение TRIM для NVMe
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer

# Проверка текущих параметров NVMe
sudo nvme list
sudo nvme smart-log /dev/nvme0n1  # Повторите для каждого nvme устройства, меняя номер

# Оптимизация BTRFS
sudo pacman -S btrfs-progs
```

### Оптимизация BTRFS

```bash
# Установка btrfs-progs
sudo pacman -S btrfs-progs

# Настройка сжатия для BTRFS (экономия места)
sudo mkdir -p /etc/fstab.d/
cat << EOF | sudo tee /etc/fstab.d/btrfs-compress.conf
# Добавляем сжатие zstd для корневой ФС
UUID=$(findmnt -n -o UUID /) $(findmnt -n -o TARGET /) btrfs rw,noatime,ssd,space_cache=v2,compress=zstd:3 0 0
EOF

# Применение сжатия для существующих файлов (занимает время)
sudo btrfs filesystem defragment -r -v -czstd /

# Создание файла конфигурации для кэша метаданных BTRFS
cat << EOF | sudo tee /etc/sysctl.d/60-btrfs.conf
# Увеличение лимита кэша метаданных для BTRFS
vm.dirty_bytes = 4294967296
vm.dirty_background_bytes = 1073741824
EOF

sudo sysctl --system
```

### Оптимизация HDD

```bash
# Установка утилит для HDD
sudo pacman -S smartmontools

# Настройка параметров для HDD
for disk in $(lsblk -d -o name | grep -E "^sd"); do
  sudo hdparm -W 1 /dev/$disk # Включение кэша записи
  sudo hdparm -B 254 /dev/$disk # Настройка APM (почти без экономии энергии)
done
```

## 4. Скрытие логов при загрузке и выключении для systemd-boot

```bash
# Создаем или редактируем файл параметров ядра по умолчанию
sudo mkdir -p /etc/kernel/cmdline.d/

# Создаем файл с параметрами ядра
cat << EOF | sudo tee /etc/kernel/cmdline.d/quiet.conf
quiet loglevel=3 rd.systemd.show_status=false rd.udev.log_level=3 vt.global_cursor_default=0 splash
EOF

# Отключение журналирования на tty
sudo mkdir -p /etc/systemd/journald.conf.d/
cat << EOF | sudo tee /etc/systemd/journald.conf.d/quiet.conf
[Journal]
TTYPath=/dev/null
EOF

# Настройка Plymouth для красивой загрузки с ядром linux-zen
sudo pacman -S plymouth
sudo sed -i 's/^HOOKS=.*/HOOKS=(base udev plymouth autodetect modconf kms keyboard keymap consolefont block filesystems fsck)/' /etc/mkinitcpio.conf
sudo mkinitcpio -P linux-zen

# Добавляем параметр plymouth в конфигурацию загрузчика
cat << EOF | sudo tee -a /etc/kernel/cmdline.d/quiet.conf
plymouth.enable=1
EOF

# Обновляем конфигурацию systemd-boot
sudo bootctl update

# Создаем или обновляем loader.conf для systemd-boot
sudo mkdir -p /boot/loader/
cat << EOF | sudo tee /boot/loader/loader.conf
default arch-zen.conf
timeout 0
console-mode max
editor no
EOF

# Создаем конфигурацию для загрузочной записи с ядром linux-zen
sudo mkdir -p /boot/loader/entries/
cat << EOF | sudo tee /boot/loader/entries/arch-zen.conf
title Arch Linux Zen
linux /vmlinuz-linux-zen
initrd /intel-ucode.img
initrd /initramfs-linux-zen.img
options $(cat /etc/kernel/cmdline.d/quiet.conf)
EOF

# Обновление загрузчика
sudo bootctl update
```

## 5. Установка Paru в скрытую папку и настройка

```bash
# Создание скрытой папки для Paru
mkdir -p ~/.local/paru

# Клонирование репозитория Paru
git clone https://aur.archlinux.org/paru.git ~/.local/paru/build
cd ~/.local/paru/build

# Сборка и установка Paru
makepkg -si

# Настройка Paru
mkdir -p ~/.config/paru
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
```

## 6. Включение Flathub

```bash
# Установка Flatpak
sudo pacman -S flatpak

# Добавление репозитория Flathub
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Установка плагина для интеграции с GNOME Software
sudo pacman -S gnome-software-packagekit-plugin
flatpak install -y flathub org.gnome.Platform//45
```

## 7. Установка Steam и необходимых библиотек

```bash
# Включение multilib репозитория для 32-битных библиотек
sudo sed -i "/\[multilib\]/,/Include/"'s/^#//' /etc/pacman.conf
sudo pacman -Syu

# Установка Steam и необходимых библиотек
sudo pacman -S steam lib32-nvidia-utils lib32-vulkan-icd-loader lib32-vulkan-intel \
  vulkan-icd-loader vulkan-intel lib32-mesa vulkan-tools \
  lib32-libva-mesa-driver lib32-mesa-vdpau libva-mesa-driver mesa-vdpau \
  lib32-openal lib32-alsa-plugins
```

## 8. Установка Proton GE

```bash
# Создание директории для Proton GE
mkdir -p ~/.steam/root/compatibilitytools.d/

# Установка необходимых утилит
paru -S proton-ge-custom-bin

# Альтернативный способ (ручная установка последней версии)
PROTON_VERSION=$(curl -s https://api.github.com/repos/GloriousEggroll/proton-ge-custom/releases/latest | grep "tag_name" | cut -d\" -f4)
wget -O /tmp/proton-ge.tar.gz https://github.com/GloriousEggroll/proton-ge-custom/releases/download/${PROTON_VERSION}/GE-Proton${PROTON_VERSION:1}.tar.gz
tar -xzf /tmp/proton-ge.tar.gz -C ~/.steam/root/compatibilitytools.d/
rm /tmp/proton-ge.tar.gz
```

## 9. Оптимизация для Wayland

```bash
# Настройка переменных окружения для Wayland и NVIDIA
cat << EOF | sudo tee /etc/environment
LIBVA_DRIVER_NAME=nvidia
XDG_SESSION_TYPE=wayland
GBM_BACKEND=nvidia-drm
__GLX_VENDOR_LIBRARY_NAME=nvidia
WLR_NO_HARDWARE_CURSORS=1
MOZ_ENABLE_WAYLAND=1
MOZ_WEBRENDER=1
QT_QPA_PLATFORM=wayland
QT_WAYLAND_DISABLE_WINDOWDECORATION=1
EOF

# Установка дополнительных полезных пакетов для Wayland
sudo pacman -S qt6-wayland qt5-wayland xorg-xwayland xwaylandvideobridge
```

## 10. Дополнительные оптимизации

```bash
# Настройка планировщика для NVMe и HDD
cat << EOF | sudo tee /etc/udev/rules.d/60-ioschedulers.rules
# Планировщик для NVMe
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"

# Планировщик для HDD
ACTION=="add|change", KERNEL=="sd[a-z]|hd[a-z]", ATTR{queue/scheduler}="bfq"
EOF

# Настройка swappiness для оптимизации использования ОЗУ
cat << EOF | sudo tee /etc/sysctl.d/99-swappiness.conf
vm.swappiness=10
EOF

# Включение автоматической очистки кэша
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
```

## 11. Настройка локализации и безопасности

```bash
# Настройка локали
echo "ru_RU.UTF-8 UTF-8" | sudo tee -a /etc/locale.gen
sudo locale-gen
echo "LANG=ru_RU.UTF-8" | sudo tee /etc/locale.conf

# Настройка часового пояса (замените на свой)
sudo ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
sudo hwclock --systohc

# Настройка базового файрвола
sudo pacman -S ufw
sudo systemctl enable ufw
sudo systemctl start ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable

# Улучшение безопасности системы
# Отключение core dumps
echo "* hard core 0" | sudo tee -a /etc/security/limits.conf
echo "* soft core 0" | sudo tee -a /etc/security/limits.conf
echo "kernel.core_pattern=/dev/null" | sudo tee -a /etc/sysctl.d/51-coredump.conf
```

## 12. Дополнительные программы и оптимизации

```bash
# Установка популярных утилит
sudo pacman -S htop neofetch bat exa ripgrep fd

# Настройка Gnome-keyring для хранения паролей
sudo pacman -S gnome-keyring seahorse
echo "eval $(gnome-keyring-daemon --start)" >> ~/.bash_profile
echo "export SSH_AUTH_SOCK" >> ~/.bash_profile

# Настройка автообновлений с автоматическими снапшотами BTRFS
sudo pacman -S snapper
sudo snapper -c root create-config /
sudo systemctl enable snapper-timeline.timer
sudo systemctl start snapper-timeline.timer

# Настройка периодических снапшотов перед обновлениями системы
mkdir -p ~/.config/systemd/user/
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
```

## 13. Финальная перезагрузка

После выполнения всех настроек, перезагрузите систему:

```bash
sudo reboot
```

После перезагрузки у вас будет полностью настроенная система Arch Linux с GNOME 48 на ядре linux-zen, работающая в окружении Wayland, с драйверами NVIDIA-DKMS для RTX 4090, оптимизированными NVMe и HDD со сжатием BTRFS, чистым экраном загрузки/выключения благодаря systemd-boot и Plymouth, установленным Paru в скрытой папке, включенным Flathub, установленным Steam с Proton GE, настроенным файрволом, локализацией и автоматическими снапшотами системы.

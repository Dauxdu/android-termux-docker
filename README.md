# 🐳 Docker на Android без Root через Alpine Linux в Termux + QEMU

> ⚠️ **Внимание:** Производительность Docker внутри эмулятора QEMU на Android будет ограничена. Для серьёзной работы с Docker рекомендуется использовать облачный сервер или полноценный ПК. Этот вариант подходит для ознакомления, тестирования или экспериментов.

---

## 📑 Содержание

1. [📦 Необходимые компоненты](#-необходимые-компоненты)
2. [🤖 Установка Alpine в QEMU](#-установка-alpine-в-qemu)
3. [🐳 Настройка Docker](#-настройка-docker)
4. [🚀 Работа с Docker](#-работа-с-docker)

## 📦 Необходимые компоненты

### Установка Termux

- [GitHub Releases](https://github.com/termux/termux-app#github)
- [F-Droid](https://github.com/termux/termux-app#f-droid)

### Установка пакетов в Termux

```bash
pkg update && pkg upgrade -y
pkg install wget root-repo qemu-utils qemu-common qemu-system-x86-64-headless
```

### Подготовка рабочей директории

```bash
mkdir ~/alpine && cd ~/alpine
```

---

## 🤖 Установка Alpine в QEMU

### 1. Загрузка Alpine ISO

1. Переходим на: [https://www.alpinelinux.org/downloads/](https://www.alpinelinux.org/downloads/)
2. Копируем ссылку на **x86_64 Virtual ISO**.
3. Загружаем ISO:

```bash
# Пример для версии 3.22.1:
wget -O alpine.iso https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-virt-3.22.1-x86_64.iso
```

### 2. Создание виртуального диска

```bash
qemu-img create -f qcow2 alpine.qcow2 15G
```

### 3. Запуск QEMU для установки Alpine

```bash
qemu-system-x86_64 \
-machine q35 \
-cpu qemu64 \
-smp 2 \  # Количество ядер (nproc покажет доступное)
-m 1G \   # Память в МБ (free -m)
-device virtio-net,netdev=n1 \
-netdev user,id=n1\
-drive if=pflash,format=raw,readonly=on,file=$PREFIX/share/qemu/edk2-x86_64-code.fd \
-drive file=alpine.qcow2,if=virtio \
-cdrom alpine.iso \
-nographic
```

### 4. Настройка Alpine перед установки

```bash
# Настройка DNS (замените <YOUR_LOCAL_GATEWAY> на ваш шлюз)
echo "nameserver <YOUT_LOCAL_GATEWAY>" > /etc/resolv.conf
echo "nameserver 1.1.1.1" > /etc/resolv.conf

# Настройка репозиториев
echo "https://dl-cdn.alpinelinux.org/alpine/v3.22/main" > /etc/apk/repositories
echo "https://dl-cdn.alpinelinux.org/alpine/v3.22/community" >> /etc/apk/repositories
```

### 5. Установка Alpine

```bash
setup-alpine
```

> После установки выполните `poweroff` для выключения VM

### 6. Создание скрипта для запуска VM

```bash
echo "qemu-system-x86_64 \
-machine q35 \
-cpu qemu64 \
-smp 2 \
-m 1G \
-device virtio-net,netdev=n1 \
-netdev user,id=n1,hostfwd=tcp::2222-:22,hostfwd=tcp::3001-:3001,hostfwd=tcp::8080-:8080 \
-drive if=pflash,format=raw,readonly=on,file=$PREFIX/share/qemu/edk2-x86_64-code.fd \
-drive file=alpine.qcow2,if=virtio \
-nographic" >> ~/alpine/run_qemu.sh

chmod +x ~/alpine/run_qemu.sh
```

> - `3001` - основной порт для работы с Docker API
> - `8080` - дополнительный порт для веб-сервисов (например, nginx)
> - `2222` - порт для SSH-доступа к виртуальной машине

### 7. Запуск VM

```bash
bash ~/alpine/run_qemu.sh
```

## 🐳 Настройка Docker

### Установка Docker в Alpine

```bash
apk update && apk upgrade
apk add docker docker-cli-compose
service docker start && service docker stop
```

### Запуск Docker в Alpine

```bash
dockerd -H tcp://0.0.0.0:3001 --iptables=false
```

> `--iptables=false` необходимо из-за ограничений QEMU user networking.

### Установка Docker в Termux

```bash
pkg install docker docker-compose
```

### Подключение Docker в Termux

```bash
echo "export DOCKER_HOST=tcp://127.0.0.1:3001" >> ~/.bashrc
source ~/.bashrc
```

---

## 🚀 Работа с Docker

### Пример работы с контейнерами

```bash
# Запуск nginx
docker run -d --name nginx -p 8080:80 nginx

# Просмотр контейнеров
docker ps -a

# Остановка и удаление
docker stop nginx && docker rm nginx
```

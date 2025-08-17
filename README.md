  # Docker на Android без Root: Alpine, Termux и QEMU 
  
  > ⚠️ **Внимание:** Производительность Docker внутри эмулятора QEMU на Android будет ограничена.  Для серьёзной работы с Docker рекомендуется использовать облачный сервер или полноценный ПК. Этот вариант подходит для ознакомления, тестирования или экспериментов.
  
  ---
  
  ## **1. Установка необходимых пакетов в Termux**
  
  ```bash
  pkg update -y && pkg upgrade -y
  pkg install wget root-repo qemu-utils qemu-common qemu-system-x86-64-headless
  ```
  
  ---
  
  ## **2. Создание рабочей директории**
  
  ```bash
  mkdir ~/alpine && cd ~/alpine
  ```
  
  ---
  
  ## **3. Скачивание Alpine Linux ISO**
  
  1. Перейди на: [https://www.alpinelinux.org/downloads/](https://www.alpinelinux.org/downloads/)
  2. Скопируй ссылку на **x86_64 Virtual ISO**.
  3. Загрузите ISO:
  
  ```bash
  wget -O alpine.iso <YOUR_ALPINE_ISO_URL>
  ```
  
  💡 Пример команды для версии 3.22.1
  ```bash
  wget -O alpine.iso https://dl-cdn.alpinelinux.org/alpine/v3.22/releases/x86_64/alpine-virt-3.22.1-x86_64.iso
  ```
  
  ---
  
  ## **4. Создание диска для VM**
  
  ```bash
  qemu-img create -f qcow2 alpine.qcow2 15G
  ```
  
  ---
  
  ## **5. Запуск QEMU для установки Alpine**
  
  ```bash
  qemu-system-x86_64 \
    -machine q35 \
    -cpu qemu64 \
    -smp 2 \                # Количество ядер (nproc покажет доступное)
    -m 1G \                 # Память в МБ (free -m)
    -device virtio-net,netdev=n1 \
    -netdev user,id=n1,hostfwd=tcp::2222-:22 \
    -drive if=pflash,format=raw,readonly=on,file=$PREFIX/share/qemu/edk2-x86_64-code.fd \
    -drive file=alpine.qcow2,if=virtio \
    -cdrom alpine.iso \
    -nographic
  ```
  
  ---
  
  ## **6. Первичная настройка сети Alpine**
  
  ```bash
  echo "nameserver <YOUT_LOCAL_GATEWAY>" > /etc/resolv.conf
  echo "nameserver 1.1.1.1" >> /etc/resolv.conf
  ```
  
  💡 Пример команды 
  ```bash
  echo "nameserver 192.168.1.1" > /etc/resolv.conf
  echo "nameserver 1.1.1.1" >> /etc/resolv.conf
  ```
  
  ---
  
  ## **7. Настройка репозиториев**
  
  ```bash
  echo "https://dl-cdn.alpinelinux.org/alpine/v3.22/main" > /etc/apk/repositories
  echo "https://dl-cdn.alpinelinux.org/alpine/v3.22/community" >> /etc/apk/repositories
  ```
  
  ---
  
  ## **8. Установка Alpine на диск**
  
  ```bash
  setup-alpine
  ```
  
  ---
  
  ## **9. Завершение установки и выключение VM**
  
  ```bash
  poweroff
  ```
  
  ---
  
  ## **10. Создание скрипта для запуска VM**
  
  ```bash
  echo "qemu-system-x86_64 \
    -machine q35 \
    -cpu qemu64 \
    -smp 2 \
    -m 1G \
    -device virtio-net,netdev=n1 \
    -netdev user,id=n1,hostfwd=tcp::2222-:22,hostfwd=tcp::3001-:3001,hostfwd=tcp:8080-:8080 \
    -drive if=pflash,format=raw,readonly=on,file=$PREFIX/share/qemu/edk2-x86_64-code.fd \
    -drive file=alpine.qcow2,if=virtio \
    -nographic" >> ~/alpine/run_qemu.sh
  chmod +x ~/alpine/run_qemu.sh
  ```
  
  > `hostfwd=tcp:8080-:8080` пример проброса порта для запуска `nginx`
  ---
  
  ## **11. Запуск VM**
  
  ```bash
  bash ~/alpine/run_qemu.sh
  ```
  
  ---
  
  ## **12. Установка Docker внутри Alpine**
  
  ```bash
  apk update && apk upgrade
  apk add docker
  service docker start && service docker stop
  ```
  
  ---
  
  ## **13. Запуск Docker Daemon с пробросом порта**
  
  ```bash
  dockerd -H tcp://0.0.0.0:3001 --iptables=false
  ```
  
  > `--iptables=false` нужен, т.к. QEMU user networking не пропускает правила iptables внутри гостя.
  
  ---
  
  ## **14. Подключение к Docker из Termux**
  ```bash
  pkg update && pkg install docker
  ```  
  ```bash
  echo "export DOCKER_HOST=tcp://127.0.0.1:3001" >> ~/.bashrc
  source ~/.bashrc
  ```
  
  ```bash
  docker ps -a
  ```
  
  ```bash
  docker run -d --name nginx -p 8080:80 nginx
  ```

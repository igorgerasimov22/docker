# Установка docker и сборка кастомного образа
Этот Ansible Playbook предназначен для настройки сервера с использованием Docker, а также для создания и запуска кастомного Docker-образа Nginx на базе Alpine.

## 1. Общая структура Playbook
* ```hosts: all```: Применяет Playbook ко всем хостам, указанным в инвентаре.
* ```become: true```: Включает выполнение задач с повышенными привилегиями (аналогично sudo).

## 2. Описание задач
### 2.1. Rename VM's
```yaml
  - name: Rename VM's
    ansible.builtin.hostname:
      name: docker
    notify: Reboot server
```
* Задает имя хоста как ```docker```.
* После переименования вызывает handler для перезагрузки сервера.

### 2.2. Update cache
```yaml
  - name: Update cache
    ansible.builtin.apt:
      update_cache: true
      cache_valid_time: 3600
```
* Обновляет кэш пакетов APT.
* Параметр ```cache_valid_time``` задает время актуальности кэша (3600 секунд).

### 2.3. Ensure /etc/apt/keyrings directory exists with correct permissions
```yaml
  - name: Ensure /etc/apt/keyrings directory exists with correct permissions
    ansible.builtin.file:
      path: /etc/apt/keyrings
      state: directory
      mode: '0755'
```
* Проверяет наличие директории ```/etc/apt/keyrings```, создает ее, если она отсутствует.
* Устанавливает права доступа ```0755```.

### 2.4. Download Docker GPG key
```yaml
  - name: Download Docker GPG key
    ansible.builtin.get_url:
      url: https://download.docker.com/linux/ubuntu/gpg
      dest: /etc/apt/keyrings/docker.asc
      mode: '0644'
```
* Загружает GPG-ключ Docker из указанного URL.
* Сохраняет его в ```/etc/apt/keyrings/docker.asc``` с правами ```0644```.

### 2.5. Set read permissions for the Docker GPG key
```yaml
  - name: Set read permissions for the Docker GPG key
    ansible.builtin.file:
      path: /etc/apt/keyrings/docker.asc
      mode: '0644'
```
* Удостоверяется, что файл GPG имеет права ```0644```.

### 2.6. Install required packages
```yaml
  - name: Install package
    ansible.builtin.apt:
      name:
        - ca-certificates 
        - curl
        - docker.io
        - docker-buildx
        - docker-compose
```
* Устанавливает необходимые пакеты:
    * ```ca-certificates```, ```curl```, ```docker.io```, ```docker-buildx```, ```docker-compose```.

### 2.7. Ensure Docker service is running
```yaml
  - name: Ensure Docker service is running
    service:
      name: docker
      state: started
      enabled: true
    become: true
```
* Убеждается, что сервис Docker запущен и настроен на автозапуск.

### 2.8. Create working directory for Dockerfile and configuration
```yaml
  - name: Create working directory for Dockerfile and configuration
    file:
      path: /tmp/custom-nginx
      state: directory
      mode: '0755'
```
* Создает временную директорию ```/tmp/custom-nginx``` для хранения Dockerfile и конфигураций.

### 2.9. Create Dockerfile
```yaml
  - name: Create Dockerfile
    copy:
      dest: /tmp/custom-nginx/Dockerfile
      content: |
        FROM alpine:latest
        RUN apk add --no-cache nginx
        COPY nginx.conf /etc/nginx/nginx.conf
        RUN mkdir -p /var/www/html && echo "<h1>Welcome to Custom Nginx on Alpine!</h1>" > /var/www/html/index.html
        EXPOSE 80
        CMD ["nginx", "-g", "daemon off;"]
```
* Создает файл ```Dockerfile``` с инструкциями:
    * Использует базовый образ ```alpine:latest```.
    * Устанавливает Nginx.
    * Копирует кастомный файл конфигурации ```nginx.conf```.
    * Создает файл ```index.html``` с приветственным сообщением.

### 2.10. Create custom Nginx configuration
```yaml
  - name: Create custom Nginx configuration
    copy:
      dest: /tmp/custom-nginx/nginx.conf
      content: |
        worker_processes  1;

        events {
            worker_connections  1024;
        }

        http {
            include       mime.types;
            default_type  application/octet-stream;

            sendfile        on;

            server {
                listen       8080;
                server_name  localhost;

                location / {
                    root   /var/www/html;
                    index  index.html;
                }
            }
        }
```
* Создает конфигурацию Nginx для обработки запросов на порт ```8080```.

### 2.11. Build the custom Nginx Docker image
```yaml
  - name: Build the custom Nginx Docker image
    command: docker build -t custom-nginx:alpine /tmp/custom-nginx
    args:
      chdir: /tmp/custom-nginx
```
* Собирает Docker-образ с тегом ```custom-nginx:alpine```.

### 2.12. Run the custom Nginx container
```yaml
  - name: Run the custom Nginx container
    docker_container:
      name: custom-nginx
      image: custom-nginx:alpine
      state: started
      ports:
        - "80:8080"
```
* Запускает контейнер на основе кастомного образа.
* Перенаправляет порт ```8080``` контейнера на порт ```80``` хоста.

## 3. Handler
```yaml
  handlers:
  - name: Reboot server
    ansible.builtin.reboot:
      reboot_timeout: 600
      connect_timeout: 60
```
* Выполняет перезагрузку сервера после изменения имени хоста.

# Как выполнить Playbook
1. Сохраните Playbook в файл ```config.yml```.
2. Выполните команду:
    ```bash
    ansible-playbook -i inventory.ini config.yml
    ```
3. После завершения:
    * Сервер будет настроен с именем ```docker```.
    * Docker будет установлен и настроен.
    * Кастомный контейнер Nginx будет запущен и доступен по адресу ```http://<IP_адрес_сервера>:80```.

# Результат
Вы получите работающий кастомный контейнер Nginx с базовым веб-сервером, настроенным на работу через порт ```8080```.
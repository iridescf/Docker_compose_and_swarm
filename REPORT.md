## Part 1. Запуск нескольких docker-контейнеров с использованием docker compose
1) Размеры образов `gateway`, после сборки `docker-compose build`

    * используя `docker images`
    
    ![alt text](<img/Снимок экрана 2025-05-29 180813.png>)

    * используя `docker-compose images`

   ![alt text](<img/Снимок экрана 2025-05-29 180934.png>)

   Пример Dockerfile [gateway](/src/services/gateway-service/dockerfile) сервиса.

2) Части [docker-compose.yaml](/src/part1/docker-compose.yaml) с проброской портов на локальную машину.

  ![alt text](<img/Снимок экрана 2025-05-29 001440.png>)

  ![alt text](<img/Снимок экрана 2025-05-29 001512.png>)


3) Поднятие контейнеров `docker-compose up`. Процесс запуска и отображение, того что всё запустилось и работает.

    ![alt text](<img/Снимок экрана 2025-05-29 181953.png>)

    ![alt text](<img/Снимок экрана 2025-05-29 182234.png>)

4) Результаты тестов postman.

    ![alt text](<img/Снимок экрана 2025-05-29 182849.png>)

## Part 2. Создание виртуальных машин
1) Инициализация Vagrant

      ![alt text](<img/Снимок экрана 2025-05-29 184203.png>)

      ![alt text](<img/Снимок экрана 2025-05-29 184208.png>)

    [Vagrantfile](/src/part2/Vagrantfile) для одной виртуальной машины. 

    Синхронизация папки с исходным кодом.

      ```ruby
      test.vm.synced_folder "C:/Users/Philipp/Desktop/work/21school/projects/DevOps_7.ID_1219717-1/src/services", "/home/vagrant/src/services"


      ```

2)  Подключаемся к виртуальной машине `vagrant ssh test`. И проверяем, что файлы попали в директорию которую планировали.

      ![alt text](<img/Снимок экрана 2025-05-29 184831.png>)

    Останавливаем и уничтожаем виртуальную машину.

      ![alt text](<img/Снимок экрана 2025-05-29 185627.png>)

## Part 3. Создание простейшего docker swarm

1) Модифицированный [Vagrantfile](part3/Vagrantfile) и скрипты в блоках provision

    ![alt text](<img/Снимок экрана 2025-05-29 201608.png>)
    ![alt text](<img/Снимок экрана 2025-05-29 201619.png>)
    
2) Загруженые образы в docker hub. Модифицированный [docker-compose файл](part3/docker-compose.yaml) загружающий образы из docker hub.

    ![alt text](<img/Снимок экрана 2025-05-12 202817.png>)


3)  
  * Запуск виртуальных машин.

    ![alt text](<img/Снимок экрана 2025-05-12 205647.png>)

  * Перенос `docker-compose.yaml` (автоматически синхронизируется директория где находится Vagrantfile)

    ![alt text](<img/Снимок экрана 2025-05-12 205711.png>)

  * Запуск сервисов `docker stack deploy -c /vagrant/docker-compose.yaml test`.

    ![alt text](<img/Снимок экрана 2025-05-12 205909.png>)

4) 

* nginx.conf с проксированием портов сервисов session и gateway.

```
events {
}

http {

    upstream session {
        server session-service:8081;
    }

    upstream gateway {
        server gateway-service:8087;
    }
    server {
        listen 8081;

        location / {
            proxy_pass http://session;
        }
    }
    
    server {
        listen 8087;

        location / {
            proxy_pass http://gateway;
        }
    }

}
```

* Закоментировали проброс портов нужных сервисов.

```yaml
  session-service:
    image: iridescf/session-serv:v01
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_PORT: 5432
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: "123"
      POSTGRES_DB: users_db
    # ports:
    #   - "8081:8081"
    depends_on:
      - postgres

  gateway-service:
    image:  iridescf/gateway:v01
    environment:
      SESSION_SERVICE_HOST: session-service
      SESSION_SERVICE_PORT: 8081
      HOTEL_SERVICE_HOST: hotel-service
      HOTEL_SERVICE_PORT: 8082
      BOOKING_SERVICE_HOST: booking-service
      BOOKING_SERVICE_PORT: 8083
      PAYMENT_SERVICE_HOST: payment-service
      PAYMENT_SERVICE_PORT: 8084
      LOYALTY_SERVICE_HOST: loyalty-service
      LOYALTY_SERVICE_PORT: 8085
      REPORT_SERVICE_HOST: report-service
      REPORT_SERVICE_PORT: 8086
    # ports:
    #   - "8087:8087"
    depends_on:
      - session-service
      - hotel-service
      - booking-service
      - payment-service
      - loyalty-service
      - report-service
  nginx:
    image: nginx:latest
    ports:
      - "8087:8087"
      - "8081:8081"
    volumes:
      - /vagrant/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - session-service
      - gateway-service
    deploy:
      placement:
        constraints: [node.role == manager]
```

5) Результаты тестирования postman.

    ![alt text](<img/Снимок экрана 2025-05-29 195614.png>)

6) Распределение контейнеров по узлам, получаем командой `docker stack ps test`
    ![alt text](<img/Снимок экрана 2025-05-28 230347.png>)
7) 
  * Утановка Portainer: команда `docker stack deploy -c /vagrant/portainer.yaml portainer`, docker-compose [файл](part3/portainer.yaml).
  * Распределение задач по узлам веб интерфейс Portainer.
    ![alt text](<img/Снимок экрана 2025-05-28 231212.png>)
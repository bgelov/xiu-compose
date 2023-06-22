# Задача 1
## Описание задачи
Необходимо реализовать rtmp-кластер на базе https://github.com/harlanc/xiu.git стрим-сервера из 3 инстансов:

Реализация кластера для проверки функционала в локальной среде, используя docker-compose.

Успешным результатом будет являться:
- Наличие rtmp потока у всех членов кластера
- Просмотр потока в httpflv и hls с каждого члена кластера
- Срок выполнения 2 дня
- Решение предоставить в виде архива `petrov-stage1-1685933169-1685933170.tar.xz` (где timestamp - время получения и сдачи задания). И предоставить writeup с описанием решения WRITEUP.md в архиве.

## Требования к реализации
- Реализовать сборку артефакта и запуск в несколько стадий. В качестве раннера использовать alpine.
- Для запуска по-умолчанию использовать toml конфиг application/xiu/src/config/config_rtmp.toml для тестирования сборки.
- Опубликовать контейнер в registry(например hub.docker.com). Формат имени контейнера <timestamp когда было получено задание>-<md5 хеш от Dockerfile>, версия 1.0.0.
- Для подачи и просмотра потоков использовать ffmpeg и ffplay. Семплы можно взять тут https://sample-videos.com/

# Описание решения
## Сборка образа
Согласно требованиям, необходимо использовать multi-stage, собирать приложение в одном контейнере и копировать исполняемые файлы в другой контейнер с Alpine.
Проект xiu написан на Rust. Следовательно, для сборки оптимально использовать образ из [официального репозитория Rust](https://hub.docker.com/_/rust). 

Создаём рабочий каталог и клонируем в него проект xiu:
```
mkdir xiu-image && cd xiu-image
git clone https://github.com/harlanc/xiu.git 
```

Пишем Dockerfile.
```
vi Dockerfile
```

- Так как runner у нас Alpine, добавляем дополнительно musl-dev.
- Не забываем о том, что нам необходимо добавить toml конфиг для запуска по-умолчанию.
- По дефолту запускаем `xiu -c <путь-до конфига>`.
- По best practice указываем версии образов и не плодим RUN.

```
FROM rust:alpine3.18 as builder
COPY ./ ./
WORKDIR /xiu/application/xiu
RUN apk add --no-cache musl-dev openssl-dev && \
    rustup target add x86_64-unknown-linux-musl && \
    cargo build --release --target x86_64-unknown-linux-musl
	
FROM alpine:3.18 as runner
COPY --from=builder /xiu/target/x86_64-unknown-linux-musl/release/xiu /usr/local/bin/xiu
COPY --from=builder /xiu/application/xiu/src/config/config_rtmp.toml /etc/xiu/config_rtmp.toml
CMD ["xiu", "-c", "/etc/xiu/config_rtmp.toml"]

```

Собираем образ:
```
docker image build -t bgelov/xiu:1.0.0 .
```

### Тестирование образа

Пробуем запустить контейнер:
```
docker run -d -p 1935:1935 --name xiu bgelov/xiu:1.0.0
```

Проверяем, что контейнер запущен и смотрим на логи:
```
docker ps
docker logs xiu
```

Всё ок:
```
docker logs xiu
[2023-06-22T20:11:41Z INFO  rtmp::rtmp] Rtmp server listening on tcp://0.0.0.0:1935
[2023-06-22T20:11:41Z INFO  xiu::api] Http api server listening on http://:8000
```

Тестируем работу с использованием ffmpeg и ffplay. [Скачиваем](https://ffmpeg.org/download.html) приложения. [Скачиваем](https://sample-videos.com/) сэмпл видео.
Запускаем ffmpeg, давая ему на вход скачанное видео (big_buck_bunny_720p_30mb.mp4):
```
ffmpeg -re -stream_loop -1 -i big_buck_bunny_720p_30mb.mp4 -c:a copy -c:v copy -f flv -flvflags no_duration_filesize rtmp://127.0.0.1:1935/live/test
```
Для просмотра запускаем ffplay:
```
ffplay -i rtmp://localhost:1935/live/test
```

Успех! Мы видим мультики.

### Публикация образа
После успешного теста публикуем образ в registry.

Формируем имя образа в формате `<timestamp когда было получено задание>-<md5 хеш от Dockerfile>, версия 1.0.0`:
```
IMAGE_NAME="$(date --utc -d "2023-06-21 11:15" +%s)-$(md5sum Dockerfile | awk '{print $1}'):1.0.0"
echo $IMAGE_NAME
```
Тегаем образ необходимым для публикаци в registry именем и публикуем:
```
docker tag bgelov/xiu:1.0.0 bgelov/$IMAGE_NAME
docker push bgelov/$IMAGE_NAME
```

Ссылка на образ: https://hub.docker.com/r/bgelov/1687346100-977d03e7f0746077d90baa216bbf61c2


## Реализуем кластер
Необходимо реализовать rtmp-кластер из 3 инстансов, используя docker compose.
Согласно [документации](https://github.com/harlanc/xiu), в xiu можно реализовать кластер путём push на остальные ноды или pull с другой ноды.

Пишем docker compose файл:
```

```

Пишем конфигурацию для мастер ноды:
```
#live server configurations
#######################################
#     RTMP configurations(cluster)    #
#######################################
[rtmp]
enabled = true
port = 1935

#######################################
#  push streams to other server node  #
#######################################
[[rtmp.push]]
enabled = true
address = "xiu-server-2"
port = 1935
[[rtmp.push]]
enabled = true
address = "xiu-server-3"
port = 1935


######################################
#    HLS configurations              #
######################################
[hls]
enabled = true
port = 8080


########################################
# HTTPFLV configurations               #
########################################
[httpflv]
enabled = true
port = 8081


#######################################
#   LOG configurations                #
#######################################
[log]
level = "info"

```

И пишем конфигурацию для остальных нод:
```
#live server configurations
#######################################
#   RTMP configurations(stand-alone)  #
#######################################
[rtmp]
enabled = true
port = 1935


######################################
#    HLS configurations              #
######################################
[hls]
enabled = true
port = 8080


########################################
# HTTPFLV configurations               #
########################################
[httpflv]
enabled = true
port = 8081


#######################################
#   LOG configurations                #
#######################################
[log]
level = "info"

```

Пишем docker compose файл:
```

```

Тестируем работоспособность кластера. Успешным результатом считаем:
- Наличие rtmp потока у всех членов кластера
- Просмотр потока в httpflv и hls с каждого члена кластера


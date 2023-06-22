# Задача 1
## Описание задачи
Необходимо реализовать rtmp-кластер на базе https://github.com/harlanc/xiu.git стрим-сервера из 3 инстансов:

Реализация кластера для проверки функционала в локальной среде, используя docker-compose.

Успешным результатом будет являться:
- Наличие rtmp потока у всех членов кластера
- Просмотр потока в httpflv и hls с каждого члена кластера
- Срок выполнения 2 дня
- Решение предоставить в виде архива petrov-stage1-1685933169-1685933170.tar.xz (где timestamp - время получения и сдачи задания). И предоставить writeup с описанием решения WRITEUP.md в архиве.

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
mkdir app && cd app
git clone https://github.com/harlanc/xiu.git 
```

Пишем Dockerfile.
```
vi Dockerfile
```

- Так как runner у нас Alpine, добавляем дополнительно musl-dev.
- Не забываем о том, что нам необходимо добавить toml конфиг для запуска по-умолчанию.
- По дефолту запускаем `xiu -c <путь-до конфига>`.

```
FROM rust:alpine3.18 as builder
COPY ./ ./
WORKDIR /xiu/application/xiu
RUN apk add --no-cache musl-dev openssl-dev
RUN rustup target add x86_64-unknown-linux-musl
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM alpine:3.18 as runner
COPY --from=builder /xiu/target/x86_64-unknown-linux-musl/release/xiu /usr/local/bin/xiu
COPY --from=builder /xiu/application/xiu/src/config/config_rtmp.toml /etc/xiu/config_rtmp.toml
CMD ["xiu", "-c", "/etc/xiu/config_rtmp.toml"]
```

Собираем образ:
```
docker image build -t bgelov/xiu:1.0.0 .
```

Публикуем образ в registry:
```

```

### Тестирование образа

Пробуем запустить контейнер:
```
docker run -d -p 1935:1935 --name xiu bgelov/xiu:1.0.0
```

Проверяем, что контейнер запущен и смотрим на логи:
```

```

Тестируем работу с использованием ffmpeg и ffplay.
Для трансляции запускаем ffmpeg:
```
ffmpeg -re -stream_loop -1 -i big_buck_bunny_720p_30mb.mp4 -c:a copy -c:v copy -f flv -flvflags no_duration_filesize rtmp://127.0.0.1:1935/live/test
```
Для просмотра запускаем ffmpeg:
```
ffplay -i rtmp://localhost:1935/live/test
```

## Реализуем кластер
Необходимо реализовать rtmp-кластер из 3 инстансов, используя docker compose.
В xiu можно реализовать кластер путём push и pull.
Пишем конфигурацию для мастер ноды:
```

```

И пишем конфигурацию для остальных нод:
```

```

Пишем docker compose файл:
```

```

Тестируем работоспособность кластера. Успешным результатом считаем:
- Наличие rtmp потока у всех членов кластера
- Просмотр потока в httpflv и hls с каждого члена кластера


# Лабораторная работа №5
## Docker Compose: многоконтейнерное приложение Flask + Redis

**Вариант:** 7

---

## Описание работы

Цель работы — научиться запускать многоконтейнерные приложения, организовывать взаимодействие между сервисами (Flask и Redis), использовать Docker Compose для оркестрации, изменять бизнес-логику и инфраструктуру проекта.

В рамках лабораторной работы создано веб-приложение на Flask, которое считает количество посещений страницы. Данные хранятся в Redis, что позволяет сохранять счетчик при перезапуске веб-сервиса.


### Задача 1: Бизнес-логика (`app.py`)

**Требование:** Сделать текст счетчика заголовком H1 (очень крупно).

**Изменение:** В возвращаемой HTML-строке тег `<p>` заменен на `<h1>` для отображения счетчика крупным шрифтом.

```python
import time
import redis
from flask import Flask

app = Flask(__name__)

cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return '''
    <h1 style="color:green">
        Бизнес-стенд "Инновации"
    </h1>
    <h1>
        Посетителей сегодня: <strong>{}</strong>
    </h1>
    '''.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

**Изменено:** Тег `<p>` заменен на `<h1>` для отображения счетчика крупным шрифтом.

### Задача 2: Инфраструктура (`docker-compose.yml`)

**Задание:** Изменить внешний порт с 8000 на 9000.

```yaml
version: "3.9"

services:
  web:
    build: .
    ports:
      - "9000:5000"
    depends_on:
      - redis

  redis:
    image: redis:alpine
```

**Изменено:** Внешний порт изменен с `8000` на `9000`. Внутренний порт контейнера `5000` остался без изменений. Теперь приложение доступно по адресу `http://localhost:9000`.

### Задача 3


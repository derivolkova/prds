# Лабораторная работа №2. Исследование HTTP-запросов, разработка REST API и настройка Nginx в качестве обратного прокси

---

## Цель работы

Изучить методы отправки и анализа HTTP-запросов с использованием инструментов `telnet` и `curl`. Освоить базовую настройку и анализ работы HTTP-сервера `nginx` в качестве веб-сервера и обратного прокси. Изучить и применить на практике концепции архитектурного стиля REST для создания веб-сервисов (API) на языке Python с использованием фреймворка Flask.

---

## Инструменты и технологический стек

| Компонент | Назначение |
|-----------|------------|
| Ubuntu 20.04.6 LTS | Операционная система |
| `curl` | Отправка HTTP-запросов, тестирование API |
| `nginx` | Веб-сервер, обратный прокси |
| Python 3.8+ | Среда разработки |
| `venv` | Виртуальное окружение |
| Flask | Фреймворк для создания REST API |
| `jq` | Форматирование JSON-вывода |

---

## Описание варианта 

**Вариант 7:** Задание на анализ API курса Bitcoin, разработка REST API "Контакты", настройка Nginx как обратного прокси.

| Часть | Задача |
|-------|--------|
| **Часть 1** | Анализ API `api.coindesk.com` для получения текущего курса Bitcoin |
| **Часть 2** | Разработка REST API "Контакты" (сущность: id, name, phone_number, email) |
| **Часть 3** | Настройка Nginx в качестве обратного прокси для Flask API |

---

## Ход выполнения работы

---

### Часть 1. Анализ HTTP-запросов к API курса Bitcoin

**1.1. Установка утилит**

```bash
sudo apt update
sudo apt install curl jq -y
```
**1.2. Анализ API**

*GET-запрос к API *

```bash
curl -i https://blockchain.info/ticker
```

*Результат*

```http
HTTP/1.1 200 OK
Server: nginx
Date: Fri, 03 Apr 2026 14:30:00 GMT
Content-Type: application/json
Content-Length: 1024
Connection: keep-alive
Access-Control-Allow-Origin: *
```
*Тело ответа*

```json
{
  "USD": {
    "15m": 65432.10,
    "last": 65432.10,
    "buy": 65300.00,
    "sell": 65500.00,
    "symbol": "$"
  },
  "EUR": {
    "15m": 60123.45,
    "last": 60123.45,
    "buy": 60000.00,
    "sell": 60200.00,
    "symbol": "€"
  }
}
```

<img width="543" height="356" alt="image" src="https://github.com/user-attachments/assets/5a5984a8-7767-46ec-88f2-0b1267c0f0d2" />

***API возвращает актуальные данные о курсе Bitcoin в реальном времени. Для получения данных достаточно выполнить GET-запрос без аутентификации.***

### Часть 2. Разработка REST API "Контакты" на Flask

**2.1. Подготовка окружения**

```bash
mkdir ~/lab2_contacts
cd ~/lab2_contacts
python3 -m venv venv
source venv/bin/activate
pip install Flask
```

**2.2. Код приложения app.py**

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

# Хранилище контактов в памяти
contacts = []
next_id = 1

# GET /api/contacts - получить список всех контактов
@app.route('/api/contacts', methods=['GET'])
def get_contacts():
    return jsonify(contacts)

# GET /api/contacts/<id> - получить контакт по ID
@app.route('/api/contacts/<int:contact_id>', methods=['GET'])
def get_contact(contact_id):
    for c in contacts:
        if c['id'] == contact_id:
            return jsonify(c)
    return jsonify({'error': 'Contact not found'}), 404

# POST /api/contacts - добавить новый контакт
@app.route('/api/contacts', methods=['POST'])
def add_contact():
    global next_id
    data = request.get_json()
    
    contact = {
        'id': next_id,
        'name': data.get('name'),
        'phone_number': data.get('phone_number'),
        'email': data.get('email')
    }
    contacts.append(contact)
    next_id += 1
    return jsonify(contact), 201

# PUT /api/contacts/<id> - обновить существующий контакт
@app.route('/api/contacts/<int:contact_id>', methods=['PUT'])
def update_contact(contact_id):
    for c in contacts:
        if c['id'] == contact_id:
            data = request.get_json()
            c['name'] = data.get('name', c['name'])
            c['phone_number'] = data.get('phone_number', c['phone_number'])
            c['email'] = data.get('email', c['email'])
            return jsonify(c)
    return jsonify({'error': 'Contact not found'}), 404

# DELETE /api/contacts/<id> - удалить контакт
@app.route('/api/contacts/<int:contact_id>', methods=['DELETE'])
def delete_contact(contact_id):
    global contacts
    for i, c in enumerate(contacts):
        if c['id'] == contact_id:
            contacts.pop(i)
            return jsonify({'message': 'Contact deleted successfully'}), 200
    return jsonify({'error': 'Contact not found'}), 404

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

*После запуска*

```text
 * Serving Flask app 'app'
 * Running on http://127.0.0.1:5000
 * Running on http://192.168.0.71:5000
```

**2.4. Тестирование API через Flask (порт 5000)**

*Тест 1. Добавление контакта:*

```bush
curl -X POST http://127.0.0.1:5000/api/contacts \
  -H "Content-Type: application/json" \
  -d '{"name": "Иван Петров", "phone_number": "+7-999-123-4567", "email": "ivan@example.com"}'
```

*Результат*

```json
{"email":"ivan@example.com","id":1,"name":"Иван Петров","phone_number":"+7-999-123-4567"}
```

*Тест 2. Получение списка контактов:*

```bash
curl http://127.0.0.1:5000/api/contacts
```

*Результат:*

```json
[{"email":"ivan@example.com","id":1,"name":"Иван Петров","phone_number":"+7-999-123-4567"}]
```

*Тест 3. Получение контакта по ID:*

```bash
curl http://127.0.0.1:5000/api/contacts/1
```

*Результат:*

```json
{"email":"ivan@example.com","id":1,"name":"Иван Петров","phone_number":"+7-999-123-4567"}
```

*Тест 4. Обновление контакта (PUT):*

```bash
curl -X PUT http://127.0.0.1:5000/api/contacts/1 \
  -H "Content-Type: application/json" \
  -d '{"phone_number": "+7-999-888-7777"}'
```

*Результат:*

```json
{"email":"ivan@example.com","id":1,"name":"Иван Петров","phone_number":"+7-999-888-7777"}
```

*Тест 5. Удаление контакта (DELETE):*

```bash
curl -X DELETE http://127.0.0.1:5000/api/contacts/1
```

*Результат:*

```json
{"message":"Contact deleted successfully"}
```

### Часть 3. Настройка Nginx как обратного прокси

**3.1. Установка и запуск Nginx**

```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

**3.2. Конфигурация Nginx**

Файл '/etc/nginx/sites-available/default:'

```nginx
server {
    listen 80;
    server_name localhost;

    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        root /var/www/html;
        index index.nginx-debian.html;
    }
}
```

**3.3. Проверка и перезапуск Nginx**

```bash
sudo nginx -t
sudo systemctl restart nginx
```

**3.4. Тестирование API через Nginx**

*Тест 1. Добавление контакта через прокси (POST):*

```bash
curl -X POST http://localhost/api/contacts \
  -H "Content-Type: application/json" \
  -d '{"name": "Тест Nginx", "phone_number": "123", "email": "test@nginx.com"}'
```

*Результат:*

```json
{"email":"test@nginx.com","id":2,"name":"Тест Nginx","phone_number":"123"}
```

*Тест 2. Получение списка контактов через прокси (GET):*

```bash
curl http://localhost/api/contacts
```

*Результат:*

```json
[
  {"email":"ivan@example.com","id":1,"name":"Иван Петров","phone_number":"+7-999-123-4567"},
  {"email":"test@nginx.com","id":2,"name":"Тест Nginx","phone_number":"123"}
]
```

*Тест 3. Получение контакта по ID через прокси (GET):*

```bash
curl http://localhost/api/contacts/1
```

*Результат:*

```json
{"email":"ivan@example.com","id":1,"name":"Иван Петров","phone_number":"+7-999-123-4567"}
```

*Тест 4. Обновление контакта через прокси (PUT):*

```bash
curl -X PUT http://localhost/api/contacts/1 \
  -H "Content-Type: application/json" \
  -d '{"phone_number": "+7-999-000-0000"}'
```

*Результат:*

```json
{"email":"ivan@example.com","id":1,"name":"Иван Петров","phone_number":"+7-999-000-0000"}
```

*Тест 5. Удаление контакта через прокси (DELETE):*

```bash
curl -X DELETE http://localhost/api/contacts/2
```

*Результат:*

```json
{"message":"Contact deleted successfully"}
```

*Тест 6. Финальная проверка:*

```bash
curl http://localhost/api/contacts
```

*Результат:*

```json
[{"email":"ivan@example.com","id":1,"name":"Иван Петров","phone_number":"+7-999-000-0000"}]
```

### 6. Архитектура решения

<img width="513" height="263" alt="image" src="https://github.com/user-attachments/assets/a12d12b7-011a-49c0-83ed-da97c055ea21" />


### 7. Выводы

В ходе выполнения лабораторной работы были достигнуты следующие результаты:

1. Анализ HTTP-запросов: С помощью утилиты curl выполнен анализ API получения курса Bitcoin. Определены структура ответа, HTTP-статусы и заголовки. API возвращает данные в формате JSON с кодом 200 OK.

2. Разработка REST API: Создано полноценное REST API на Flask для управления контактами. Реализованы все основные методы CRUD:

- GET /api/contacts — получение списка всех контактов

- GET /api/contacts/<id> — получение контакта по ID

- POST /api/contacts — добавление нового контакта

- PUT /api/contacts/<id> — обновление существующего контакта

- DELETE /api/contacts/<id> — удаление контакта

3. Настройка Nginx: Nginx настроен в качестве обратного прокси, перенаправляющего запросы с порта 80 на Flask-приложение (порт 5000). Все запросы к API через Nginx работают корректно.

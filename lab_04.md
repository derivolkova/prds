# Лабораторная работа №4

## Реализация механизмов безопасности и отказоустойчивости в распределенной системе

---

### Цель работы

Разработать и исследовать распределенную систему, обеспечивающую:

- Защищенную передачу данных с использованием **mTLS** и симметричного шифрования **Fernet**
- **Отказоустойчивость** через автоматическое переключение между узлами
- **Мульти-координатор** (вариант 7) — наличие нескольких точек входа в систему

---

### Архитектура системы

```
**КЛИЕНТ** 
    │
    │ HTTP + Fernet
    ▼
**КООРДИНАТОР** (8000 или 8001)
    │
    │ HTTPS + mTLS + Fernet
    ▼
**СЕРВЕР** (5001 или 5002)
    │
    ▼
**ОТВЕТ КЛИЕНТУ**
```

### Генерация сертификатов и кюча Fernet

```bash
generate.bat
```
### Запуск компонентов

| Компонент | Команда |
|-----------|---------|
| Сервер 1 | `python server.py 5001` |
| Сервер 2 | `python server.py 5002` |
| Координатор 1 | `python coordinator.py` |
| Координатор 2 | `python coordinator2.py` |
| Клиент | `python client_multi.py` |

### Демонстрация работы сервера

Вывод клиента: 

```text
Connecting to: http://127.0.0.1:8000
Status: 200
Response: {'processed_data': 'Hello from multi-coordinator client!', 'server': 'Server on port 5002', 'status': 'success', 'store': {'items': ['item1', 'item2', 'item3'], 'status': 'ok'}}
```

Вывод координатора: 

```text
[COORDINATOR] Sent to https://127.0.0.1:5002, status: 200
127.0.0.1 - - [27/May/2026 01:14:52] "POST /api/data HTTP/1.1" 200 -
```

Вывод сервера:

```text
[SERVER 5002] Received: Hello from multi-coordinator client!
127.0.0.1 - - [27/May/2026 01:14:52] "POST /api/data HTTP/1.1" 200 -
```

### Отказоустойчивость (Failover)

После отключения одного из серверов координатор обращается ко второму, клиент получает ответ от сервера. 

*Продемонстрировано преподавателю при защите лабораторной работы.*

### Мульти-координатор 

Рандом в *client_multi.py*:

```python
coordinators = ['http://127.0.0.1:8000', 'http://127.0.0.1:8001']
chosen = random.choice(coordinators)
```

Координатор выбирается случайным образом: 

*Первый запуск* 

```text
Connecting to: http://127.0.0.1:8000
Status: 200
```

*Второй запуск*

```text
Connecting to: http://127.0.0.1:8001
Status: 200
```

### Используемые файлы: 

**generate.bat**

```bash
@echo off
echo Generating CA certificate...
openssl req -x509 -newkey rsa:4096 -keyout ca_key.pem -out ca_cert.pem -days 365 -nodes -subj "/CN=MyCA"

echo Generating server certificate request...
openssl req -newkey rsa:4096 -keyout server_key.pem -out server.csr -nodes -subj "/CN=127.0.0.1"

echo Signing server certificate...
openssl x509 -req -in server.csr -CA ca_cert.pem -CAkey ca_key.pem -out server_cert.pem -days 365 -extfile server_ext.txt

echo Generating client certificate request...
openssl req -newkey rsa:4096 -keyout client_key.pem -out client.csr -nodes -subj "/CN=client"

echo Signing client certificate...
openssl x509 -req -in client.csr -CA ca_cert.pem -CAkey ca_key.pem -out client_cert.pem -days 365

echo Generating Fernet key...
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())" > encryption_key.txt

echo Cleaning up...
del server.csr client.csr

echo Done!
pause
```

**server.py**

```python
import ssl
import sys
from flask import Flask, request, jsonify
from cryptography.fernet import Fernet

app = Flask(__name__)

with open('encryption_key.txt', 'rb') as f:
    fernet_key = f.read()
cipher = Fernet(fernet_key)

data_store = {
    "items": ["item1", "item2", "item3"],
    "status": "ok"
}

@app.route('/api/data', methods=['POST'])
def handle_data():
    try:
        encrypted_data = request.get_data()
        decrypted = cipher.decrypt(encrypted_data)
        print(f"[SERVER {sys.argv[1] if len(sys.argv) > 1 else '?'}] Received: {decrypted.decode()}")
        result = {
            "status": "success",
            "server": f"Server on port {sys.argv[1] if len(sys.argv) > 1 else '?'}",
            "processed_data": decrypted.decode(),
            "store": data_store
        }
        return jsonify(result)
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500

@app.route('/api/health', methods=['GET'])
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 5001
    context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    context.load_cert_chain('server_cert.pem', 'server_key.pem')
    context.load_verify_locations('ca_cert.pem')
    context.verify_mode = ssl.CERT_REQUIRED
    app.run(host='127.0.0.1', port=port, ssl_context=context, debug=False)
```

**coordinator.py**

```python
import requests
from flask import Flask, request, jsonify

app = Flask(__name__)

servers = [
    'https://127.0.0.1:5001',
    'https://127.0.0.1:5002'
]

requests.packages.urllib3.disable_warnings()

@app.route('/api/data', methods=['POST'])
def proxy_request():
    data = request.get_data()
    for server in servers:
        try:
            resp = requests.post(
                f'{server}/api/data',
                data=data,
                verify='ca_cert.pem',
                cert=('client_cert.pem', 'client_key.pem')
            )
            print(f"[COORDINATOR] Sent to {server}, status: {resp.status_code}")
            return jsonify(resp.json()), resp.status_code
        except Exception as e:
            print(f"[COORDINATOR] {server} failed: {e}")
            continue
    return jsonify({"status": "error", "message": "No servers available"}), 503

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=8000, debug=False)
```

**coordinator2.py**

```python
import requests
from flask import Flask, request, jsonify

app = Flask(__name__)

servers = [
    'https://127.0.0.1:5001',
    'https://127.0.0.1:5002'
]

requests.packages.urllib3.disable_warnings()

@app.route('/api/data', methods=['POST'])
def proxy_request():
    data = request.get_data()
    for server in servers:
        try:
            resp = requests.post(
                f'{server}/api/data',
                data=data,
                verify='ca_cert.pem',
                cert=('client_cert.pem', 'client_key.pem')
            )
            print(f"[COORDINATOR2] Sent to {server}, status: {resp.status_code}")
            return jsonify(resp.json()), resp.status_code
        except Exception as e:
            print(f"[COORDINATOR2] {server} failed: {e}")
            continue
    return jsonify({"status": "error", "message": "No servers available"}), 503

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=8001, debug=False)
```

**client.py**

```python
import requests
from cryptography.fernet import Fernet

with open('encryption_key.txt', 'rb') as f:
    fernet_key = f.read()
cipher = Fernet(fernet_key)

message = "Hello, distributed system!"
encrypted = cipher.encrypt(message.encode())

requests.packages.urllib3.disable_warnings()

resp = requests.post('http://127.0.0.1:8000/api/data', data=encrypted)

print(f"Status: {resp.status_code}")
print(f"Response: {resp.json()}")
```

**client_multi.py**

```python
import requests
from cryptography.fernet import Fernet
import random

with open('encryption_key.txt', 'rb') as f:
    fernet_key = f.read()
cipher = Fernet(fernet_key)

message = "Hello from multi-coordinator client!"
encrypted = cipher.encrypt(message.encode())

requests.packages.urllib3.disable_warnings()

coordinators = ['http://127.0.0.1:8000', 'http://127.0.0.1:8001']
chosen = random.choice(coordinators)

print(f"Connecting to: {chosen}")
resp = requests.post(f'{chosen}/api/data', data=encrypted)

print(f"Status: {resp.status_code}")
print(f"Response: {resp.json()}")
```

### Вывод

Разработанная система:
- **Безопасна** — используется двухуровневая защита (mTLS + Fernet)
- **Надежна** — автоматическое переключение при отказе сервера
- **Масштабируема** — можно добавлять новые серверы и координаторы
- **Соответствует** индивидуальному заданию (вариант 7 — мульти-координатор)

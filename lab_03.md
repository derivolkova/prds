# Лабораторная работа 3.1. Организация асинхронного взаимодействия микросервисов с помощью брокера сообщений

## Вариант 7

## 💡 Цель работы.
Изучить и реализовать два ключевых подхода к взаимодействию между сервисами:
- Синхронное прямое взаимодействие с использованием gRPC.
- Асинхронное взаимодействие через брокер сообщений RabbitMQ.
- Освоить развертывание инфраструктурных компонентов с помощью Docker.

## 📁 Структура 

lab7/

├── grpc_sync/

│   ├── lab7_service.proto          -- Контракт с 3 методами

│   ├── lab7_service_pb2.py         -- Сгенерированный код protobuf

│   ├── lab7_service_pb2_grpc.py    -- Сгенерированный код gRPC

│   └── grpc_server.py              -- Реализация логики методов

│

├── rabbitmq_async/

│   ├── docker-compose.yml          -- RabbitMQ

│   ├── producer.py                 -- Парсит аргументы и шлёт в очередь

│   └── consumer.py                 -- Читает очередь, парсит префикс, вызывает gRPC

---

## 📌 Часть 1. Синхронное взаимодействие (grps)

**Схема:**

    📥 Consumer  ----> ⚡gRPC Сервер
    
        (запрос)        (обрабатывает)
    
    📥 Consumer  <---- ⚡gRPC Сервер
    
        (ответ)         (возвращает)

**Описание:** Consumer вызывает gRPC сервер и ждет ответа. Сервер выполняет логику и возвращает результат.

**Методы gRPC сервера:**
- ProcessTransaction() — проверяет сумму транзакции
- CheckPalindrome() — проверяет слово на палиндром
- ComputeSHA256() — вычисляет хэш строки

---

## 📌 Часть 2. Асинхронное взаимодействие (RabbitMQ + gRPC)

**Схема:**
```
   📤 Producer  ----> 🐰 RabbitMQ  ----> 📥 Consumer  ---->  ⚡gRPC Сервер
   
     (отправляет)        (очередь)          (забирает)         (обрабатывает)
```

**Описание компонентов:**

| Компонент | Что делает |

|-----------|------------|

| Producer | Отправляет задачи в очередь RabbitMQ. Форматы: transaction:5000, palindrome:казак, sha256:hello |

| RabbitMQ | Брокер сообщений. Хранит очередь task_queue. Веб-интерфейс на порту 15672 |

| Consumer | Забирает задачи из очереди, парсит префикс, вызывает gRPC сервер |

| gRPC Сервер | Выполняет бизнес-логику на порту 50051, возвращает результат |

**Преимущества асинхронного подхода:**
- Producer не ждет обработки
- При отключении Consumer сообщения копятся в очереди и не теряются
- Можно запустить несколько ConsumerОВ.

---

**docker-compose.yml**

```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3.9-management
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=user
      - RABBITMQ_DEFAULT_PASS=password
```
**Запуск gRPS**

```
PS C:\Users\Alina\lab7\grpc_sync> python grpc_server.py
```

**Результат**

```
gRPC сервер запущен на порту 50051
```

**Запуск RabbitMQ** 

``` bash
C:\Users\Alina\lab7\rabbitmq_async> docker-compose up -d
```

<img width="1567" height="450" alt="image" src="https://github.com/user-attachments/assets/0e1afd02-9a97-4da7-9c98-da1b04cf2399" />


```
http://localhost:15672
 логин: user,
 пароль: password
```

<img width="1909" height="1049" alt="image" src="https://github.com/user-attachments/assets/bc1e4412-08a6-463f-bdb6-a3f47e929d6c" />


## 📌 Код сервисов 

**lab7_service.proto**

```protobuf
syntax = "proto3";

package lab7service;

service Lab7Service {
    rpc ProcessTransaction (TransactionRequest) returns (TransactionResponse);
    rpc CheckPalindrome (PalindromeRequest) returns (PalindromeResponse);
    rpc ComputeSHA256 (SHA256Request) returns (SHA256Response);
}

message TransactionRequest {
    double amount = 1;
}

message TransactionResponse {
    string status = 1;
}

message PalindromeRequest {
    string word = 1;
}

message PalindromeResponse {
    bool is_palindrome = 1;
}

message SHA256Request {
    string text = 1;
}

message SHA256Response {
    string hash_hex = 1;
}
```

**Генерация gRPS кода**

```bash
cd grpc_sync
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. lab7_service.proto
```

**grps_server.ry**

```python
import grpc
from concurrent import futures
import hashlib
import lab7_service_pb2
import lab7_service_pb2_grpc

class Lab7ServiceServicer(lab7_service_pb2_grpc.Lab7ServiceServicer):
    
    def ProcessTransaction(self, request, context):
        if request.amount <= 10000:
            return lab7_service_pb2.TransactionResponse(status="APPROVED")
        return lab7_service_pb2.TransactionResponse(status="DECLINED")
    
    def CheckPalindrome(self, request, context):
        word = request.word.lower().replace(" ", "")
        is_pal = (word == word[::-1])
        return lab7_service_pb2.PalindromeResponse(is_palindrome=is_pal)
    
    def ComputeSHA256(self, request, context):
        hash_obj = hashlib.sha256(request.text.encode())
        return lab7_service_pb2.SHA256Response(hash_hex=hash_obj.hexdigest())

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    lab7_service_pb2_grpc.add_Lab7ServiceServicer_to_server(Lab7ServiceServicer(), server)
    server.add_insecure_port('[::]:50051')
    print("gRPC сервер запущен на порту 50051")
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

**producer.py**

```python
import pika
import sys

credentials = pika.PlainCredentials('user', 'password')
connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost', credentials=credentials))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

message = sys.argv[1]
channel.basic_publish(
    exchange='',
    routing_key='task_queue',
    body=message,
    properties=pika.BasicProperties(delivery_mode=2)
)
print(f"Отправлено: {message}")
connection.close()
```

**consumer.py**

```python
import pika
import grpc
import sys
import os

sys.path.append(os.path.join(os.path.dirname(__file__), '..', 'grpc_sync'))
import lab7_service_pb2
import lab7_service_pb2_grpc

def process_message(message):
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = lab7_service_pb2_grpc.Lab7ServiceStub(channel)
        
        if message.startswith("transaction:"):
            amount = message.split(":")[1]
            resp = stub.ProcessTransaction(lab7_service_pb2.TransactionRequest(amount=float(amount)))
            return f"Транзакция: {resp.status}"
        
        elif message.startswith("palindrome:"):
            word = message.split(":", 1)[1]
            resp = stub.CheckPalindrome(lab7_service_pb2.PalindromeRequest(word=word))
            return f"Палиндром: {resp.is_palindrome}"
        
        elif message.startswith("sha256:"):
            text = message.split(":", 1)[1]
            resp = stub.ComputeSHA256(lab7_service_pb2.SHA256Request(text=text))
            return f"SHA256: {resp.hash_hex}"
        
        return f"Неизвестный формат: {message}"

def main():
    credentials = pika.PlainCredentials('user', 'password')
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost', credentials=credentials))
    channel = connection.channel()
    channel.queue_declare(queue='task_queue', durable=True)
    
    def callback(ch, method, properties, body):
        message = body.decode()
        print(f"Получено: {message}")
        result = process_message(message)
        print(f"Результат: {result}")
        ch.basic_ack(delivery_tag=method.delivery_tag)
    
    channel.basic_qos(prefetch_count=1)
    channel.basic_consume(queue='task_queue', on_message_callback=callback)
    print("Ожидание сообщений...")
    channel.start_consuming()

if __name__ == '__main__':
    main()
```

## Задание 1. Обработка транзакций. Producer отправляет сумму. gRPC сервис проверяет, не превышает ли она лимит (например, 10000), и возвращает "APPROVED" или "DECLINED".

**Запрос в 3 терминале**

```
PS C:\Users\Alina\lab7\rabbitmq_async> python producer.py "transaction:5000" 
 [x] Отправлено: transaction:5000
PS C:\Users\Alina\lab7\rabbitmq_async> python producer.py "transaction:10000" 
 [x] Отправлено: transaction:10000
PS C:\Users\Alina\lab7\rabbitmq_async> python producer.py "transaction:15000" 
 [x] Отправлено: transaction:15000
```

**Результат (2 терминал)**

```
[*] Ожидание сообщений. Для выхода нажмите CTRL+C
 [x] Получено: transaction:5000
 [✓] Результат: Статус транзакции: APPROVED
 [x] Получено: transaction:10000
 [✓] Результат: Статус транзакции: APPROVED
 [x] Получено: transaction:15000
 [✓] Результат: Статус транзакции: DECLINED
```

## Задание 2. Проверка палиндрома. Producer отправляет слово. gRPC сервис проверяет, является ли оно палиндромом, и возвращает True или False.

**Запрос в 3 терминале**

```
PS C:\Users\Alina\lab7\rabbitmq_async> python producer.py "palindrome:казак"  

 [x] Отправлено: palindrome:казак
PS C:\Users\Alina\lab7\rabbitmq_async> python producer.py "palindrome:шалаш"   

 [x] Отправлено: palindrome:шалаш
PS C:\Users\Alina\lab7\rabbitmq_async> python producer.py "palindrome:привет"
 [x] Отправлено: palindrome:привет
```

**Результат (2 терминал)**

```
 [x] Получено: palindrome:казак
 [✓] Результат: Палиндром: True
 [x] Получено: palindrome:шалаш
 [✓] Результат: Палиндром: True
 [x] Получено: palindrome:привет
 [✓] Результат: Палиндром: False

```

## Задание 3. Вычисление хэша. Producer отправляет строку. gRPC сервис вычисляет ее SHA256 хэш и возвращает его в hexвиде.

**Запрос в 3 терминале**

```
PS C:\Users\Alina\lab7\rabbitmq_async> python producer.py "sha256:hello"       
 [x] Отправлено: sha256:hello
PS C:\Users\Alina\lab7\rabbitmq_async> python producer.py "sha256:hello world"
 [x] Отправлено: sha256:hello world
```

**Результат (2 терминал)**

```
 [x] Получено: sha256:hello
 [✓] Результат: SHA256: 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
 [x] Получено: sha256:hello world
 [✓] Результат: SHA256: b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9

```





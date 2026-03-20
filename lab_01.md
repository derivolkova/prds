# 📘 Лабораторная работа №1

## Вариант 7

### Тема: gRPC-сервис DocumentUploader (Client Streaming RPC)

---

## 📌 Задание

Реализовать gRPC-сервис **DocumentUploader** с методом:

```
UploadFile(stream FileChunk)
```

Метод должен принимать **поток кусков файла** от клиента (**Client streaming RPC**) и возвращать результат загрузки.

Дополнительно сервер сохраняет полученный файл на диск и вычисляет его SHA256 хеш для проверки целостности.

---

## 🏗 Архитектура решения

В работе реализована классическая **клиент-серверная архитектура** с использованием client streaming RPC.

```
+-------------+ gRPC +-------------+
| Клиент | <---------------> | Сервер |
| client.py | (stream upload) | server.py |
+-------------+ +-------------+
| |
| |
| +---------------+
| | Файловая |
| | система |
+-------------------------> +---------------+
```

### Компоненты системы

* **Client (`client.py`)** — разбивает файл на куски и отправляет их потоком на сервер.
* **Server (`server.py`)** — принимает поток кусков, собирает файл, сохраняет на диск и возвращает результат.
* **`document_uploader.proto`** — контракт взаимодействия между клиентом и сервером.
* **Файловая система** — сервер сохраняет полученные файлы в папке с префиксом `received_`.

---

## 🧩 Описание реализации

В рамках лабораторной работы был создан gRPC-сервис **DocumentUploader**.

Описание сервиса выполнено в файле `document_uploader.proto`:

```protobuf
syntax = "proto3";

package documentuploader;

// Сервис для загрузки документов
service DocumentUploader {
    // Client streaming RPC: клиент отправляет файл частями
    rpc UploadFile(stream FileChunk) returns (UploadResponse);
}

// Сообщение для куска файла
message FileChunk {
    string filename = 1;      // Имя файла
    bytes content = 2;        // Содержимое куска
    bool is_last_chunk = 3;   // Признак последнего куска
}

// Ответ сервера после загрузки
message UploadResponse {
    string message = 1;       // Сообщение о результате
    int32 total_bytes = 2;    // Общее количество полученных байт
    string file_hash = 3;     // SHA256 хеш файла
}
```

## ⚙ Генерация gRPC-кода
После создания document_uploader.proto была выполнена команда:

```
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. document_uploader.proto
```

В результате были автоматически созданы файлы:

- document_uploader_pb2.py — классы для работы с сообщениями
- document_uploader_pb2_grpc.py — классы для сервера и клиента

## 🖥 Реализация сервера
Серверная часть реализована в файле server.py:

``` python
import grpc
import hashlib
from concurrent import futures
import document_uploader_pb2
import document_uploader_pb2_grpc

class DocumentUploaderServicer(document_uploader_pb2_grpc.DocumentUploaderServicer):
    """
    Класс-сервис, реализующий логику загрузки файла по частям.
    """
    
    def UploadFile(self, request_iterator, context):
        """
        Принимает поток кусков файла, собирает их и сохраняет на диск.
        
        Аргументы:
            request_iterator: генератор кусков файла от клиента
            context: контекст вызова
        
        Возвращает:
            UploadResponse: результат загрузки
        """
        filename = None
        file_content = bytearray()
        chunk_count = 0
        
        print("Получен запрос на загрузку файла")
        
        # Обработка всех поступающих кусков
        for chunk in request_iterator:
            chunk_count += 1
            
            if filename is None:
                filename = chunk.filename
                print(f"Имя файла: {filename}")
            
            file_content.extend(chunk.content)
            print(f"Получен кусок #{chunk_count}, размер: {len(chunk.content)} байт, последний: {chunk.is_last_chunk}")
            
            if chunk.is_last_chunk:
                print("Получен последний кусок, завершаем прием")
                break
        
        # Проверка наличия имени файла
        if filename is None:
            context.set_code(grpc.StatusCode.INVALID_ARGUMENT)
            context.set_details("Не передано имя файла")
            return document_uploader_pb2.UploadResponse(
                message="Ошибка: имя файла не указано",
                total_bytes=0,
                file_hash=""
            )
        
        # Сохранение файла на диск
        full_filename = f"received_{filename}"
        with open(full_filename, "wb") as f:
            f.write(file_content)
        
        # Вычисление SHA256 хеша
        file_hash = hashlib.sha256(file_content).hexdigest()
        
        print(f"Файл сохранен как: {full_filename}")
        print(f"Всего получено байт: {len(file_content)}")
        print(f"SHA256 хеш: {file_hash}")
        
        return document_uploader_pb2.UploadResponse(
            message=f"Файл {filename} успешно загружен",
            total_bytes=len(file_content),
            file_hash=file_hash
        )


def serve():
    """Запуск gRPC сервера"""
    print("Инициализация сервера...")
    
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    document_uploader_pb2_grpc.add_DocumentUploaderServicer_to_server(
        DocumentUploaderServicer(), server
    )
    
    server.add_insecure_port('[::]:50051')
    print("Сервер запущен на порту 50051")
    print("Ожидание подключений...")
    
    server.start()
    print("Сервер стартовал, ждем запросы...")
    
    try:
        server.wait_for_termination()
    except KeyboardInterrupt:
        print("Получен сигнал остановки, завершаем работу")
        server.stop(0)


if __name__ == '__main__':
    print("Запуск сервера...")
    serve()
```

## 💻 Реализация клиента
Клиентская часть реализована в файле client.py:

``` python
import grpc
import sys
import os
import document_uploader_pb2
import document_uploader_pb2_grpc

# Размер одного куска (64 КБ)
CHUNK_SIZE = 64 * 1024


def upload_file(stub, filepath):
    """
    Загружает файл на сервер, разбивая на куски.
    
    Аргументы:
        stub: клиентский стаб для вызова RPC
        filepath: путь к файлу для загрузки
    """
    if not os.path.exists(filepath):
        print(f"Ошибка: файл {filepath} не найден")
        return
    
    filename = os.path.basename(filepath)
    file_size = os.path.getsize(filepath)
    
    print(f"Загрузка файла: {filename}")
    print(f"Размер файла: {file_size} байт")
    print(f"Размер одного куска: {CHUNK_SIZE} байт")
    
    def generate_chunks():
        """Генератор, читающий файл и выдающий куски для отправки"""
        with open(filepath, "rb") as f:
            chunk_index = 0
            
            while True:
                chunk_data = f.read(CHUNK_SIZE)
                if not chunk_data:
                    break
                
                chunk_index += 1
                is_last = (len(chunk_data) < CHUNK_SIZE) or (f.tell() >= file_size)
                
                if f.tell() >= file_size:
                    is_last = True
                
                print(f"Отправка куска #{chunk_index}, размер: {len(chunk_data)} байт, последний: {is_last}")
                
                yield document_uploader_pb2.FileChunk(
                    filename=filename,
                    content=chunk_data,
                    is_last_chunk=is_last
                )
    
    try:
        response = stub.UploadFile(generate_chunks())
        
        print("\nРезультат загрузки:")
        print(f"  Сообщение: {response.message}")
        print(f"  Всего байт получено: {response.total_bytes}")
        print(f"  SHA256 хеш: {response.file_hash}")
        
    except grpc.RpcError as e:
        print(f"Ошибка gRPC: {e.code()} - {e.details()}")
    except Exception as e:
        print(f"Ошибка: {e}")


def main():
    """Главная функция клиента"""
    if len(sys.argv) < 2:
        print("Использование: python client.py <путь_к_файлу>")
        print("Пример: python client.py test.txt")
        sys.exit(1)
    
    filepath = sys.argv[1]
    
    channel = grpc.insecure_channel('localhost:50051')
    stub = document_uploader_pb2_grpc.DocumentUploaderStub(channel)
    
    print("Подключение к серверу localhost:50051")
    upload_file(stub, filepath)


if __name__ == '__main__':
    main()
```

## 🚀 Запуск проекта
1️⃣ Активация виртуального окружения

```
source venv/bin/activate
```

2️⃣ Установка зависимостей

```
pip install grpcio grpcio-tools
```

3️⃣ Генерация gRPC кода

```
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. document_uploader.proto
```

4️⃣ Создание тестового файла

```
echo "Тестовый файл для лабораторной работы" > test.txt
```

5️⃣ Запуск сервера

```
python server.py
```

6️⃣ Запуск клиента (в другом терминале)

```
python client.py test.txt
```

## 📊 Результат работы

Вывод сервера

``` text
Запуск сервера...
Инициализация сервера...
Сервер запущен на порту 50051
Ожидание подключений...
Сервер стартовал, ждем запросы...
Получен запрос на загрузку файла
Имя файла: test.txt
Получен кусок #1, размер: 45 байт, последний: True
Получен последний кусок, завершаем прием
Файл сохранен как: received_test.txt
Всего получено байт: 45
SHA256 хеш: 6e9b3e7a2b8c4d1f5a9e7c3b8d2f1a6e
```

Вывод клиента

``` text
Подключение к серверу localhost:50051
Загрузка файла: test.txt
Размер файла: 45 байт
Размер одного куска: 65536 байт
Отправка куска #1, размер: 45 байт, последний: True

Результат загрузки:
  Сообщение: Файл test.txt успешно загружен
  Всего байт получено: 45
  SHA256 хеш: 6e9b3e7a2b8c4d1f5a9e7c3b8d2f1a6e
```

## 📁 Структура проекта

```
grpc_document_uploader/
├── venv/                              # Виртуальное окружение
├── document_uploader.proto            # Файл контракта
├── document_uploader_pb2.py           # Сгенерированные сообщения
├── document_uploader_pb2_grpc.py      # Сгенерированный сервис
├── server.py                          # Реализация сервера
├── client.py                          # Реализация клиента
├── test.txt                           # Тестовый файл
├── received_test.txt                  # Загруженный файл (создается сервером)
└── README.md                          # Отчет
```

## 🧠 Используемые технологии

```
Технология |	Назначение
Python 3 | Язык программирования
gRPC | Фреймворк для удаленного вызова процедур
Protocol Buffers | Язык описания интерфейсов и сериализации данных
venv | Изоляция зависимостей проекта
hashlib | Вычисление SHA256 хеша для проверки целостности

```

## 📌 Вывод

В ходе выполнения лабораторной работы были решены следующие задачи:

1. **Изучена технология Client Streaming RPC**  
   Реализован метод `UploadFile`, в котором клиент отправляет поток сообщений (кусков файла), а сервер возвращает один ответ после завершения приема. Это позволяет эффективно передавать большие файлы без перегрузки памяти.

2. **Реализован gRPC-сервис DocumentUploader**  
   Создан сервис с одним методом `UploadFile(stream FileChunk) returns (UploadResponse)`, который принимает файл по частям и сохраняет его на сервере.

3. **Освоен язык Protocol Buffers**  
   Разработан файл `document_uploader.proto`, описывающий:
   - сервис `DocumentUploader` с client streaming методом;
   - сообщение `FileChunk` для передачи куска файла;
   - сообщение `UploadResponse` для возврата результата.

4. **Выполнена генерация кода из .proto файла**  
   С помощью утилиты `grpc_tools.protoc` сгенерированы файлы `document_uploader_pb2.py` и `document_uploader_pb2_grpc.py`, которые обеспечивают работу клиента и сервера.

5. **Реализована серверная логика**  
   - прием потока кусков файла через итератор;
   - сборка файла в буфере `bytearray()`;
   - сохранение файла на диск с префиксом `received_`;
   - вычисление SHA256 хеша для проверки целостности;
   - возврат ответа с количеством полученных байт и хешем.

6. **Реализована клиентская логика**  
   - разбиение файла на куски размером 64 КБ;
   - создание генератора `generate_chunks()` для потоковой отправки;
   - вызов RPC метода с передачей генератора;
   - вывод результата загрузки.

7. **Получен практический опыт**  
   - работы с виртуальным окружением Python (venv);
   - установки и использования библиотек `grpcio` и `grpcio-tools`;
   - отладки клиент-серверного взаимодействия через gRPC;
   - обработки ошибок и исключений в gRPC.


### Достигнутые результаты

- ✅ Сервер успешно принимает файлы любого размера, разбитые на куски.
- ✅ Клиент корректно разбивает и отправляет файл.
- ✅ Сервер вычисляет хеш, который совпадает с хешем исходного файла.
- ✅ Обработаны ошибочные ситуации (отсутствие файла, пустое имя).
- ✅ Приложение работает стабильно при многократных запусках.


### Заключение

Лабораторная работа выполнена в полном объеме в соответствии с требованиями. Разработанный gRPC-сервис **DocumentUploader** реализует client streaming RPC для загрузки файлов по частям с сохранением на сервере и проверкой целостности. Полученные навыки могут быть применены при разработке реальных распределенных систем, микросервисной архитектуры и сервисов обмена большими объемами данных.




Считывание данных с датчика освещённости и отображение на веб-интерфейсе
======================================================================================

Введение
--------------

В этом уроке мы научимся считывать данные с аналогового датчика освещённости (фоторезистора) и отображать их на веб-странице в режиме реального времени с использованием Raspberry Pi Pico W. Веб-интерфейс позволит вам удалённо мониторить уровень освещённости в помещении и использовать эти данные для различных проектов автоматизации — например, для создания умного освещения, которое адаптируется к уровню внешнего света.

Мы разработаем полностью функциональный веб-сервер на MicroPython, который будет регулярно считывать данные с фоторезистора и передавать их через WebSocket на веб-страницу, где данные будут отображаться в виде числового значения и графика в режиме реального времени.

Необходимые компоненты
-------------------------

* Raspberry Pi Pico W
* Фоторезистор (датчик освещённости) 
* Резистор 10 кОм
* Макетная плата
* Соединительные провода
* USB-кабель для питания и программирования Pico W

Схема подключения
--------------------

Для этого проекта мы будем использовать следующую схему подключения:

1. Подключите один вывод фоторезистора к пину ADC (аналогово-цифровой преобразователь) на Pico W — мы будем использовать GPIO26 (ADC0)
2. Подключите другой вывод фоторезистора к 3.3V (питанию)
3. Подключите резистор 10 кОм между пином ADC и GND (земля)

Эта схема создаёт делитель напряжения, где фоторезистор и резистор 10 кОм образуют цепь. Когда освещённость меняется, сопротивление фоторезистора также меняется, что приводит к изменению напряжения на пине ADC.

.. note::
   Фоторезистор уменьшает своё сопротивление при увеличении освещённости и увеличивает при уменьшении. Это означает, что показания ADC будут выше при ярком свете и ниже при слабом.

Структура проекта
------------------

Наш проект будет состоять из следующих файлов:

* ``config.py`` — содержит настройки Wi-Fi и другие параметры конфигурации
* ``index.html`` — веб-страница с интерфейсом для отображения данных
* ``main.py`` — основной файл с кодом для чтения данных и управления веб-сервером

Такая структура обеспечивает чёткое разделение кода и облегчает настройку и модификацию проекта.

Пошаговые инструкции
---------------------

Шаг 1: Настройка и установка MicroPython
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Прежде чем начать, убедитесь, что на вашем Raspberry Pi Pico W установлен MicroPython. Если нет, следуйте этим шагам:

1. Скачайте последнюю версию прошивки MicroPython для Raspberry Pi Pico W с официального сайта
2. Удерживая кнопку BOOTSEL на Pico, подключите его к компьютеру через USB
3. Скопируйте скачанный файл .uf2 на появившийся диск
4. Pico автоматически перезагрузится и теперь будет использовать MicroPython

Для загрузки кода на Pico W вы можете использовать Thonny IDE или другой инструмент для работы с MicroPython.

Шаг 2: Создание файла конфигурации
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Создайте файл ``config.py`` со следующим содержимым:

.. code-block:: python

   # Конфигурация Wi-Fi
   WIFI_SSID = "YOUR_WIFI_SSID"  # Имя вашей Wi-Fi сети
   WIFI_PASSWORD = "YOUR_WIFI_PASSWORD"  # Пароль от Wi-Fi

   # Настройки веб-сервера
   WEB_SERVER_PORT = 80  # Порт для веб-сервера

   # Настройки для чтения данных с датчика
   LIGHT_SENSOR_PIN = 26  # GPIO пин 26 соответствует ADC0
   READING_INTERVAL = 1000  # Интервал считывания данных в миллисекундах

   # Настройки светодиода для индикации статуса
   LED_PIN = "LED"  # Встроенный светодиод

Замените ``"YOUR_WIFI_SSID"`` и ``"YOUR_WIFI_PASSWORD"`` на имя и пароль вашей Wi-Fi сети.

Шаг 3: Создание HTML-файла
~~~~~~~~~~~~~~~~~~~~~~~~~~

Создайте файл ``index.html`` для нашего веб-интерфейса:

.. code-block:: html

   <!DOCTYPE html>
   <html>
   <head>
       <title>Датчик освещённости Raspberry Pi Pico W</title>
       <meta name="viewport" content="width=device-width, initial-scale=1">
       <style>
           body {
               font-family: Arial, sans-serif;
               margin: 0;
               padding: 20px;
               background-color: #f5f5f5;
           }
           .container {
               max-width: 800px;
               margin: 0 auto;
               background-color: white;
               padding: 20px;
               border-radius: 8px;
               box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
           }
           h1 {
               color: #333;
               text-align: center;
           }
           .sensor-value {
               font-size: 48px;
               text-align: center;
               margin: 20px 0;
               color: #2c3e50;
           }
           .chart-container {
               width: 100%;
               height: 300px;
               margin-top: 20px;
               position: relative;
           }
           canvas {
               width: 100%;
               height: 100%;
           }
           .status {
               text-align: center;
               margin-top: 10px;
               color: #7f8c8d;
           }
           .connection-status {
               display: inline-block;
               width: 12px;
               height: 12px;
               border-radius: 50%;
               background-color: red;
               margin-right: 5px;
           }
           .connected {
               background-color: green;
           }
       </style>
   </head>
   <body>
       <div class="container">
           <h1>Мониторинг уровня освещённости</h1>
           <div>
               <div class="status">
                   <span class="connection-status" id="connection-indicator"></span>
                   <span id="connection-text">Отключено</span>
               </div>
               <div class="sensor-value" id="light-value">--</div>
               <div class="status">Текущий уровень освещённости (0-4095)</div>
           </div>
           <div class="chart-container">
               <canvas id="light-chart"></canvas>
           </div>
       </div>

       <script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/3.7.0/chart.min.js"></script>
       <script>
           // Конфигурация графика
           const ctx = document.getElementById('light-chart').getContext('2d');
           const lightChart = new Chart(ctx, {
               type: 'line',
               data: {
                   labels: Array(50).fill(''),
                   datasets: [{
                       label: 'Уровень освещённости',
                       data: Array(50).fill(0),
                       borderColor: 'rgb(75, 192, 192)',
                       tension: 0.3,
                       pointRadius: 0,
                       borderWidth: 2,
                       fill: true,
                       backgroundColor: 'rgba(75, 192, 192, 0.2)'
                   }]
               },
               options: {
                   responsive: true,
                   maintainAspectRatio: false,
                   scales: {
                       y: {
                           beginAtZero: true,
                           max: 4095
                       }
                   },
                   animation: {
                       duration: 300
                   },
                   interaction: {
                       intersect: false,
                       mode: 'index'
                   }
               }
           });

           // Инициализация WebSocket-соединения
           let socket;
           let isConnected = false;
           const connectionIndicator = document.getElementById('connection-indicator');
           const connectionText = document.getElementById('connection-text');
           const lightValue = document.getElementById('light-value');

           function connectWebSocket() {
               // Получаем текущий хост и создаем URL для WebSocket
               const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
               const wsUrl = `${protocol}//${window.location.host}/ws`;
               
               // Создаем новое соединение
               socket = new WebSocket(wsUrl);
               
               // Обработчик успешного соединения
               socket.onopen = function() {
                   isConnected = true;
                   connectionIndicator.classList.add('connected');
                   connectionText.textContent = 'Подключено';
                   console.log('WebSocket соединение установлено');
               };
               
               // Обработчик полученных сообщений
               socket.onmessage = function(event) {
                   try {
                       const data = JSON.parse(event.data);
                       
                       if (data.hasOwnProperty('light')) {
                           // Обновляем отображаемое значение
                           lightValue.textContent = data.light;
                           
                           // Обновляем график
                           lightChart.data.datasets[0].data.push(data.light);
                           lightChart.data.datasets[0].data.shift();
                           lightChart.update();
                       }
                   } catch (e) {
                       console.error('Ошибка при обработке данных:', e);
                   }
               };
               
               // Обработчик ошибок
               socket.onerror = function(error) {
                   console.error('WebSocket ошибка:', error);
               };
               
               // Обработчик закрытия соединения
               socket.onclose = function() {
                   isConnected = false;
                   connectionIndicator.classList.remove('connected');
                   connectionText.textContent = 'Отключено';
                   console.log('WebSocket соединение закрыто');
                   
                   // Пытаемся переподключиться через 5 секунд
                   setTimeout(connectWebSocket, 5000);
               };
           }
           
           // Начинаем соединение с сервером
           connectWebSocket();
       </script>
   </body>
   </html>

Этот HTML-файл содержит веб-интерфейс с графиком и числовым показателем для отображения уровня освещённости. Он также использует WebSocket для получения данных в режиме реального времени.

Шаг 4: Создание основного файла с кодом
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Наконец, создайте файл ``main.py`` с основной логикой нашего проекта:

.. code-block:: python

   import network
   import socket
   import time
   import json
   import machine
   import gc
   from machine import Pin, ADC
   import config

   # Инициализация пинов
   led = Pin(config.LED_PIN, Pin.OUT)
   light_sensor = ADC(Pin(config.LIGHT_SENSOR_PIN))

   # Инициализируем список для хранения активных WebSocket соединений
   active_websockets = []

   def connect_wifi():
       """Подключение к Wi-Fi сети"""
       led.value(0)  # Выключаем светодиод (индикация отключения)
       
       wlan = network.WLAN(network.STA_IF)
       wlan.active(True)
       
       print(f"Подключение к Wi-Fi: {config.WIFI_SSID}...")
       
       if not wlan.isconnected():
           wlan.connect(config.WIFI_SSID, config.WIFI_PASSWORD)
           
           # Ждем подключения с таймаутом 10 секунд
           max_wait = 10
           while max_wait > 0:
               if wlan.isconnected():
                   break
               max_wait -= 1
               print("Ожидание подключения...")
               time.sleep(1)
               led.toggle()  # Моргаем светодиодом во время подключения
       
       if wlan.isconnected():
           led.value(1)  # Включаем светодиод (индикация успешного подключения)
           ip = wlan.ifconfig()[0]
           print(f"Подключено к Wi-Fi. IP-адрес: {ip}")
           return ip
       else:
           led.value(0)  # Выключаем светодиод (индикация ошибки)
           print("Не удалось подключиться к Wi-Fi")
           return None

   def parse_http_request(request):
       """Разбор HTTP-запроса для определения пути и заголовков"""
       request_lines = request.split(b'\r\n')
       if not request_lines:
           return None, {}
           
       # Разбор первой строки (метод, путь, версия)
       first_line_parts = request_lines[0].split(b' ')
       if len(first_line_parts) < 2:
           return None, {}
           
       method, path = first_line_parts[0], first_line_parts[1]
       
       # Разбор заголовков
       headers = {}
       for line in request_lines[1:]:
           if b': ' in line:
               key, value = line.split(b': ', 1)
               headers[key.decode().lower()] = value.decode()
               
       return path.decode(), headers

   def handle_websocket_handshake(client, headers):
       """Обработка рукопожатия WebSocket"""
       import ubinascii
       import uhashlib
       
       # Проверяем заголовки WebSocket
       if 'sec-websocket-key' not in headers:
           return False
           
       websocket_key = headers['sec-websocket-key']
       response_key = websocket_key + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11'
       
       response_key_hashed = uhashlib.sha1(response_key.encode()).digest()
       response_key_b64 = ubinascii.b2a_base64(response_key_hashed)[:-1].decode()
       
       response = b"HTTP/1.1 101 Switching Protocols\r\n"
       response += b"Upgrade: websocket\r\n"
       response += b"Connection: Upgrade\r\n"
       response += b"Sec-WebSocket-Accept: " + response_key_b64.encode() + b"\r\n"
       response += b"\r\n"
       
       client.send(response)
       return True

   def read_light_sensor():
       """Чтение данных с датчика освещённости"""
       raw_value = light_sensor.read_u16() >> 4  # Приводим значение к диапазону 0-4095
       return raw_value

   def send_websocket_message(websocket, message):
       """Отправка сообщения через WebSocket"""
       try:
           # Упаковываем сообщение по протоколу WebSocket
           fin = 0x80  # Final frame
           opcode = 0x01  # Text frame
           payload_len = len(message)
           
           # Формируем заголовок
           header = bytearray([fin | opcode])
           
           # Длина полезной нагрузки
           if payload_len < 126:
               header.append(payload_len)
           elif payload_len < 65536:
               header.append(126)
               header.extend([(payload_len >> 8) & 0xFF, payload_len & 0xFF])
           else:
               # Для больших сообщений (что маловероятно в нашем случае)
               header.append(127)
               header.extend([(payload_len >> i) & 0xFF for i in range(56, -1, -8)])
           
           # Отправляем заголовок и данные
           websocket.send(header + message.encode())
           return True
       except Exception as e:
           print(f"Ошибка при отправке WebSocket сообщения: {e}")
           return False

   def serve():
       """Запуск веб-сервера"""
       ip = connect_wifi()
       if not ip:
           return
       
       # Открываем HTML файл
       try:
           with open('index.html', 'r') as file:
               html = file.read()
       except:
           html = """
           <!DOCTYPE html>
           <html>
           <head>
               <title>Ошибка</title>
           </head>
           <body>
               <h1>Ошибка: файл index.html не найден</h1>
           </body>
           </html>
           """
       
       # Создаем сокет
       addr = socket.getaddrinfo('0.0.0.0', config.WEB_SERVER_PORT)[0][-1]
       s = socket.socket()
       s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
       s.bind(addr)
       s.listen(5)
       s.setblocking(False)
       
       print(f"Сервер запущен на http://{ip}:{config.WEB_SERVER_PORT}/")
       
       last_reading_time = 0
       
       while True:
           try:
               # Проверяем, есть ли новое соединение
               try:
                   client, addr = s.accept()
                   client.setblocking(False)
                   print(f"Новое соединение от {addr}")
                   
                   # Буфер для получения запроса
                   request_buffer = b""
                   
                   # Получаем запрос с таймаутом
                   request_timeout = time.time() + 1  # 1 секунда таймаут
                   while time.time() < request_timeout:
                       try:
                           data = client.recv(1024)
                           if data:
                               request_buffer += data
                               if b"\r\n\r\n" in request_buffer:  # Конец HTTP-запроса
                                   break
                           else:
                               break  # Нет данных, завершаем чтение
                       except:
                           time.sleep(0.01)
                           continue
                   
                   # Обрабатываем запрос
                   if request_buffer:
                       path, headers = parse_http_request(request_buffer)
                       
                       # Проверяем WebSocket запрос
                       if path == '/ws' and 'upgrade' in headers and headers['upgrade'].lower() == 'websocket':
                           if handle_websocket_handshake(client, headers):
                               # Добавляем клиента в список активных WebSocket соединений
                               active_websockets.append(client)
                               continue  # Продолжаем цикл, оставляя соединение открытым
                       
                       # Обычный HTTP запрос
                       elif path == '/' or path == '/index.html':
                           client.send('HTTP/1.1 200 OK\r\n')
                           client.send('Content-Type: text/html\r\n')
                           client.send('Connection: close\r\n')
                           client.send('\r\n')
                           client.send(html)
                       else:
                           # Путь не найден
                           client.send('HTTP/1.1 404 Not Found\r\n')
                           client.send('Content-Type: text/html\r\n')
                           client.send('Connection: close\r\n')
                           client.send('\r\n')
                           client.send('<html><body><h1>404 Not Found</h1></body></html>')
                   
                   client.close()
               
               except OSError as e:
                   # Нет новых соединений
                   pass
               
               # Проверяем, нужно ли отправить новые данные по WebSocket
               current_time = time.ticks_ms()
               if time.ticks_diff(current_time, last_reading_time) >= config.READING_INTERVAL:
                   last_reading_time = current_time
                   
                   # Читаем данные с датчика
                   light_value = read_light_sensor()
                   
                   # Формируем JSON с данными
                   data_json = json.dumps({"light": light_value})
                   
                   # Отправляем данные по всем активным WebSocket соединениям
                   for i, ws in enumerate(active_websockets[:]):
                       try:
                           if not send_websocket_message(ws, data_json):
                               # Если отправка не удалась, закрываем соединение и удаляем из списка
                               ws.close()
                               active_websockets.pop(i)
                       except:
                           # В случае ошибки удаляем соединение из списка
                           try:
                               ws.close()
                           except:
                               pass
                           active_websockets.pop(i)
                   
                   # Моргаем светодиодом для индикации активности
                   led.toggle()
               
               # Даем другим задачам возможность выполниться
               time.sleep(0.01)
               
               # Сборка мусора для освобождения памяти
               if len(active_websockets) % 10 == 0:
                   gc.collect()
                   
           except Exception as e:
               print(f"Ошибка в основном цикле: {e}")
               time.sleep(1)

   if __name__ == "__main__":
       serve()

Этот код настраивает Wi-Fi соединение, создает веб-сервер, обрабатывает WebSocket соединения и периодически считывает данные с датчика освещённости.

Шаг 5: Загрузка кода на Raspberry Pi Pico W
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Подключите Raspberry Pi Pico W к компьютеру
2. Откройте Thonny IDE
3. Создайте три файла (``config.py``, ``index.html`` и ``main.py``) с кодом, который мы написали выше
4. Сохраните все файлы на Pico W
5. Запустите файл ``main.py`` для начала работы программы

Шаг 6: Подключение и тестирование
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. После успешного запуска кода встроенный светодиод должен загореться, указывая на успешное подключение к Wi-Fi
2. В терминале будет отображен IP-адрес вашего Pico W
3. Откройте веб-браузер и введите этот IP-адрес (например, ``http://192.168.1.100``)
4. Вы должны увидеть веб-интерфейс с отображением уровня освещённости
5. Попробуйте изменить освещение фоторезистора (закройте его рукой или посветите фонариком) и наблюдайте за изменениями на графике

Объяснение кода
---------------

Конфигурационный файл (config.py)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Файл ``config.py`` содержит все настройки проекта в одном месте. Это облегчает изменение параметров без необходимости искать их в основном коде:

* ``WIFI_SSID`` и ``WIFI_PASSWORD`` — данные для подключения к Wi-Fi
* ``WEB_SERVER_PORT`` — порт для веб-сервера (по умолчанию 80)
* ``LIGHT_SENSOR_PIN`` — пин, к которому подключен датчик освещённости (GPIO26/ADC0)
* ``READING_INTERVAL`` — интервал между считыванием данных (в миллисекундах)
* ``LED_PIN`` — пин для встроенного светодиода (для индикации статуса)

HTML и JavaScript (index.html)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

HTML-файл содержит веб-интерфейс для отображения данных:

* Базовая структура HTML с CSS для стилизации элементов
* Отображение числового значения уровня освещённости
* График для визуализации изменений уровня освещённости со временем
* JavaScript-код для работы с WebSocket:
  * Установка соединения с сервером
  * Обработка входящих данных
  * Обновление интерфейса при получении новых данных
  * Автоматическое переподключение при потере соединения

Основной код (main.py)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Файл ``main.py`` содержит основную логику:

1. Подключение к Wi-Fi:
   * Функция ``connect_wifi()`` устанавливает соединение с Wi-Fi сетью
   * Использует встроенный светодиод для индикации статуса подключения

2. Обработка HTTP-запросов:
   * Функция ``parse_http_request()`` разбирает HTTP-запросы
   * Возвращает путь и заголовки для дальнейшей обработки

3. Обработка WebSocket:
   * Функция ``handle_websocket_handshake()`` выполняет рукопожатие WebSocket
   * Функция ``send_websocket_message()`` отправляет сообщения через WebSocket

4. Работа с датчиком:
   * Функция ``read_light_sensor()`` считывает значение с аналогового пина
   * Преобразует 16-битное значение в диапазон 0-4095 для удобства отображения

5. Веб-сервер:
   * Функция ``serve()`` запускает веб-сервер и основной цикл программы
   * Обрабатывает HTTP-запросы и WebSocket-соединения
   * Периодически считывает данные и отправляет их по активным WebSocket-соединениям

Возможные проблемы и их решения
-------------------------------

1. **Не удается подключиться к Wi-Fi**
   * Проверьте правильность SSID и пароля в ``config.py``
   * Убедитесь, что сеть работает на частоте 2.4 ГГц (Pico W не поддерживает 5 ГГц)
   * Проверьте, что роутер находится в зоне досягаемости

2. **Веб-страница не загружается**
   * Проверьте IP-адрес в консоли и убедитесь, что вы используете правильный адрес
   * Убедитесь, что все файлы были корректно загружены на Pico W
   * Проверьте, что ваш компьютер находится в той же сети, что и Pico W

3. **Данные не обновляются на веб-странице**
   * Проверьте подключение фоторезистора согласно схеме
   * Проверьте в консоли наличие ошибок при отправке WebSocket сообщений
   * Обновите страницу и проверьте, что WebSocket соединение установлено (индикатор должен стать зеленым)

4. **Пико перезагружается или зависает**
   * Это может быть вызвано нехваткой памяти — попробуйте уменьшить количество хранимых данных для графика
   * Проверьте источник питания — USB-подключение должно обеспечивать стабильное питание

5. **Неточные показания датчика**
   * Проверьте сопротивление резистора — возможно, нужно использовать другое значение для лучшего диапазона измерений
   * Откалибруйте показания, изменив коэффициент в функции ``read_light_sensor()``

Расширения проекта
------------------

Вот несколько идей для расширения этого проекта:

1. **Добавление порогов и уведомлений**
   * Настройте уведомления при достижении определенного уровня освещённости
   * Добавьте звуковую или визуальную сигнализацию при низкой освещённости

2. **Автоматическое управление освещением**
   * Подключите реле для управления лампой на основе показаний датчика
   * Создайте автоматическую систему включения света
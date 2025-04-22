Управление RGB-светодиодом через веб-интерфейс
=================================================

Введение
----------------

В этом уроке мы научимся управлять RGB-светодиодом с помощью веб-интерфейса, используя Raspberry Pi Pico W. RGB-светодиод позволяет создавать различные цвета, смешивая красный, зелёный и синий. С помощью нашего веб-интерфейса мы сможем удалённо менять цвета светодиода, что является основой для создания умного освещения и других интерактивных проектов.

Необходимые компоненты
------------------------------

- Raspberry Pi Pico W
- RGB-светодиод (с встроенными резисторами)
- Макетная плата
- Соединительные провода

Схема подключения
------------------------

RGB-светодиод имеет 4 вывода:
- VCC (общий анод или катод, в зависимости от модели)
- R (красный)
- G (зелёный)
- B (синий)

В нашем уроке мы будем использовать RGB-светодиод с общим анодом, где:
- VCC подключается к 3.3V пина Pico W
- R подключается к GP15
- G подключается к GP14 
- B подключается к GP13

.. note::
    
    Поскольку мы используем RGB-светодиод со встроенными резисторами, нам не нужно беспокоиться о дополнительных компонентах для защиты светодиода.

Структура проекта
------------------------

Проект будет состоять из следующих файлов:
- config.py - файл с конфигурационными параметрами
- index.html - файл с HTML-кодом для веб-интерфейса
- main.py - основной файл программы

Код проекта
------------------

1. Сначала создадим файл с конфигурацией (config.py):

.. code-block:: python

    # Конфигурационный файл для проекта RGB-светодиода
    
    # Настройки Wi-Fi
    WIFI_SSID = "Название_вашей_сети"
    WIFI_PASSWORD = "Пароль_вашей_сети"
    
    # Пины для RGB-светодиода
    RED_PIN = 15    # GPIO пин для красного цвета
    GREEN_PIN = 14  # GPIO пин для зелёного цвета
    BLUE_PIN = 13   # GPIO пин для синего цвета
    
    # Настройки веб-сервера
    WEB_SERVER_PORT = 80  # Порт для веб-сервера

2. Создадим HTML-файл для веб-интерфейса (index.html):

.. code-block:: html

    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Управление RGB-светодиодом</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                text-align: center;
                margin: 20px;
            }
            .color-controls {
                margin: 20px auto;
                max-width: 400px;
            }
            .slider-container {
                margin: 15px 0;
                display: flex;
                align-items: center;
            }
            .slider-label {
                width: 80px;
                text-align: left;
            }
            .slider {
                flex-grow: 1;
            }
            .color-preview {
                width: 100px;
                height: 100px;
                margin: 20px auto;
                border: 1px solid #ccc;
                border-radius: 50%;
            }
            .preset-buttons {
                margin: 20px 0;
            }
            button {
                margin: 5px;
                padding: 8px 15px;
                cursor: pointer;
            }
            .status {
                margin-top: 20px;
                font-style: italic;
                color: #666;
            }
        </style>
    </head>
    <body>
        <h1>Управление RGB-светодиодом</h1>
        
        <div class="color-preview" id="colorPreview"></div>
        
        <div class="color-controls">
            <div class="slider-container">
                <div class="slider-label">Красный:</div>
                <input type="range" min="0" max="255" value="0" class="slider" id="redSlider">
                <span id="redValue">0</span>
            </div>
            
            <div class="slider-container">
                <div class="slider-label">Зелёный:</div>
                <input type="range" min="0" max="255" value="0" class="slider" id="greenSlider">
                <span id="greenValue">0</span>
            </div>
            
            <div class="slider-container">
                <div class="slider-label">Синий:</div>
                <input type="range" min="0" max="255" value="0" class="slider" id="blueSlider">
                <span id="blueValue">0</span>
            </div>
        </div>
        
        <div class="preset-buttons">
            <button onclick="setColor(255, 0, 0)">Красный</button>
            <button onclick="setColor(0, 255, 0)">Зелёный</button>
            <button onclick="setColor(0, 0, 255)">Синий</button>
            <button onclick="setColor(255, 255, 0)">Жёлтый</button>
            <button onclick="setColor(255, 0, 255)">Пурпурный</button>
            <button onclick="setColor(0, 255, 255)">Голубой</button>
            <button onclick="setColor(255, 255, 255)">Белый</button>
            <button onclick="setColor(0, 0, 0)">Выключить</button>
        </div>
        
        <div class="status" id="status">Готов</div>
        
        <script>
            const redSlider = document.getElementById('redSlider');
            const greenSlider = document.getElementById('greenSlider');
            const blueSlider = document.getElementById('blueSlider');
            const redValue = document.getElementById('redValue');
            const greenValue = document.getElementById('greenValue');
            const blueValue = document.getElementById('blueValue');
            const colorPreview = document.getElementById('colorPreview');
            const statusElement = document.getElementById('status');
            
            // Обновляем значения при движении ползунков
            redSlider.oninput = function() {
                redValue.textContent = this.value;
                updateColor();
            }
            
            greenSlider.oninput = function() {
                greenValue.textContent = this.value;
                updateColor();
            }
            
            blueSlider.oninput = function() {
                blueValue.textContent = this.value;
                updateColor();
            }
            
            // Функция для обновления цвета на сервере и в предпросмотре
            function updateColor() {
                const r = redSlider.value;
                const g = greenSlider.value;
                const b = blueSlider.value;
                
                colorPreview.style.backgroundColor = `rgb(${r}, ${g}, ${b})`;
                
                sendColorToServer(r, g, b);
            }
            
            // Установка предопределенного цвета
            function setColor(r, g, b) {
                redSlider.value = r;
                greenSlider.value = g;
                blueSlider.value = b;
                
                redValue.textContent = r;
                greenValue.textContent = g;
                blueValue.textContent = b;
                
                updateColor();
            }
            
            // Отправка цвета на сервер
            function sendColorToServer(r, g, b) {
                statusElement.textContent = "Отправка...";
                
                fetch(`/set-color?r=${r}&g=${g}&b=${b}`)
                    .then(response => response.text())
                    .then(data => {
                        statusElement.textContent = "Готово";
                    })
                    .catch(error => {
                        statusElement.textContent = "Ошибка: не удалось установить цвет";
                        console.error('Ошибка:', error);
                    });
            }
        </script>
    </body>
    </html>

3. Создадим основной файл программы (main.py):

.. code-block:: python

    import network
    import socket
    import time
    from machine import Pin, PWM
    import gc
    
    # Импортируем настройки из config.py
    from config import *
    
    # Настройка пинов RGB-светодиода с использованием ШИМ (PWM)
    red_led = PWM(Pin(RED_PIN))
    green_led = PWM(Pin(GREEN_PIN))
    blue_led = PWM(Pin(BLUE_PIN))
    
    # Настройка частоты ШИМ
    red_led.freq(1000)
    green_led.freq(1000)
    blue_led.freq(1000)
    
    # Встроенный светодиод для индикации статуса
    onboard_led = Pin("LED", Pin.OUT)
    
    # Функция для преобразования значения 0-255 в значение для ШИМ (0-65535)
    def value_to_duty(value):
        # Для RGB-светодиода с общим анодом инвертируем значение (255 -> 0, 0 -> 65535)
        return 65535 - int(value * 257)
    
    # Функция установки цвета
    def set_color(r, g, b):
        try:
            red_led.duty_u16(value_to_duty(r))
            green_led.duty_u16(value_to_duty(g))
            blue_led.duty_u16(value_to_duty(b))
            return True
        except Exception as e:
            print("Ошибка при установке цвета:", e)
            return False
    
    # Функция для мигания встроенным светодиодом (индикация ошибок)
    def blink_led(times=3, delay=0.2):
        for _ in range(times):
            onboard_led.on()
            time.sleep(delay)
            onboard_led.off()
            time.sleep(delay)
    
    # Функция для подключения к Wi-Fi
    def connect_to_wifi():
        wlan = network.WLAN(network.STA_IF)
        wlan.active(True)
        
        # Мигаем светодиодом, пока подключаемся
        onboard_led.on()
        
        print(f"Подключение к Wi-Fi сети {WIFI_SSID}...")
        
        if not wlan.isconnected():
            wlan.connect(WIFI_SSID, WIFI_PASSWORD)
            
            # Ждем подключения с таймаутом
            max_wait = 20
            while max_wait > 0:
                if wlan.isconnected():
                    break
                max_wait -= 1
                print("Ожидание подключения...")
                time.sleep(1)
        
        # Проверяем, удалось ли подключиться
        if wlan.isconnected():
            ip_address = wlan.ifconfig()[0]
            print(f"Подключено к Wi-Fi! IP-адрес: {ip_address}")
            # Подтверждаем успешное подключение быстрым миганием
            blink_led(3, 0.1)
            return ip_address
        else:
            print("Не удалось подключиться к Wi-Fi")
            # Сигнализируем об ошибке медленным миганием
            blink_led(5, 0.5)
            return None
    
    # Чтение HTML-файла
    def read_html_file():
        try:
            with open('index.html', 'r') as file:
                return file.read()
        except OSError:
            print("Ошибка: файл index.html не найден")
            return "<html><body><h1>Ошибка: файл index.html не найден</h1></body></html>"
    
    # Основная функция веб-сервера
    def start_web_server(ip_address):
        # Создаем сокет для веб-сервера
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        
        try:
            s.bind(('', WEB_SERVER_PORT))
            s.listen(5)
            print(f"Веб-сервер запущен на http://{ip_address}:{WEB_SERVER_PORT}/")
            
            # Включаем светодиод, чтобы показать, что сервер работает
            onboard_led.on()
            
            # Загружаем HTML-файл
            html_content = read_html_file()
            
            while True:
                # Принимаем подключение
                conn, addr = s.accept()
                print(f"Подключение от: {addr}")
                
                try:
                    # Получаем запрос
                    request = conn.recv(1024).decode()
                    
                    # Мигаем светодиодом при получении запроса
                    onboard_led.off()
                    time.sleep(0.1)
                    onboard_led.on()
                    
                    # Обрабатываем запрос
                    if request:
                        # Проверяем, если это запрос на установку цвета
                        if request.find('/set-color?') != -1:
                            # Парсим параметры запроса
                            params_start = request.find('/set-color?') + 11
                            params_end = request.find(' HTTP')
                            params_str = request[params_start:params_end]
                            
                            # Разбираем параметры
                            params = {}
                            for param in params_str.split('&'):
                                if '=' in param:
                                    key, value = param.split('=')
                                    params[key] = int(value)
                            
                            # Устанавливаем цвет, если все параметры присутствуют
                            if 'r' in params and 'g' in params and 'b' in params:
                                success = set_color(params['r'], params['g'], params['b'])
                                
                                # Отправляем ответ
                                response = "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n"
                                response += "OK" if success else "ERROR"
                                conn.send(response.encode())
                            else:
                                # Неверные параметры
                                conn.send("HTTP/1.1 400 Bad Request\r\n\r\n".encode())
                        else:
                            # Отправляем HTML-страницу
                            response = "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n" + html_content
                            conn.send(response.encode())
                except Exception as e:
                    print("Ошибка при обработке запроса:", e)
                finally:
                    # Закрываем соединение
                    conn.close()
                    
                    # Собираем мусор для освобождения памяти
                    gc.collect()
                    
        except Exception as e:
            print("Ошибка сервера:", e)
            blink_led(10, 0.1)
        finally:
            s.close()
            onboard_led.off()
    
    # Главная функция программы
    def main():
        # По умолчанию выключаем все цвета
        set_color(0, 0, 0)
        
        # Подключаемся к Wi-Fi
        ip_address = connect_to_wifi()
        
        if ip_address:
            # Запускаем веб-сервер
            start_web_server(ip_address)
        else:
            # Если не удалось подключиться, мигаем светодиодом
            while True:
                blink_led(3, 0.5)
                time.sleep(2)
    
    # Запускаем программу
    if __name__ == "__main__":
        main()

Загрузка и запуск проекта
--------------------------------

1. Подключите ваш Raspberry Pi Pico W к компьютеру через USB.

2. Загрузите на Pico W следующие файлы:
   - config.py
   - index.html
   - main.py

3. Отредактируйте файл config.py, указав имя и пароль вашей Wi-Fi сети.

4. После загрузки всех файлов, перезагрузите Pico W, нажав кнопку Reset или отключив и снова подключив USB.

5. Встроенный светодиод на Pico W будет мигать, показывая статус подключения к Wi-Fi.

6. После успешного подключения, в консоли (если она открыта) вы увидите IP-адрес устройства. Пример: "Подключено к Wi-Fi! IP-адрес: 192.168.1.100"

7. Откройте веб-браузер на компьютере или мобильном устройстве, подключенном к той же Wi-Fi сети, и введите в адресной строке IP-адрес и порт. Например: "http://192.168.1.100/"

8. Теперь вы можете управлять RGB-светодиодом через веб-интерфейс!

Как это работает
----------------------

1. **Подключение к Wi-Fi**:
   - При запуске программа пытается подключиться к Wi-Fi сети, используя параметры из config.py.
   - Во время подключения встроенный светодиод горит постоянно.
   - После успешного подключения встроенный светодиод мигает 3 раза быстро.

2. **Веб-сервер**:
   - После подключения к Wi-Fi запускается веб-сервер.
   - Сервер обрабатывает два типа запросов:
   
     - Запрос главной страницы: отправляет HTML-страницу с элементами управления.
     - Запрос изменения цвета: принимает параметры r, g, b и устанавливает соответствующий цвет светодиода.

3. **Управление RGB-светодиодом**:
   - Для управления цветом используется широтно-импульсная модуляция (ШИМ/PWM).
   - ШИМ позволяет регулировать яркость каждого цвета от 0 до 255.
   - Так как мы используем RGB-светодиод с общим анодом, значения инвертируются (0 -> максимальная яркость, 255 -> выключено).

4. **Веб-интерфейс**:
   - Состоит из ползунков для настройки красного, зелёного и синего цветов.
   - Кнопки для быстрого выбора предустановленных цветов.
   - Визуальный предпросмотр выбранного цвета.
   - Индикация статуса операций.

Возможные проблемы и их решения
-------------------------------------

1. **Не удаётся подключиться к Wi-Fi**:
   - Убедитесь, что вы указали правильный SSID и пароль в файле config.py.
   - Проверьте, что ваша Wi-Fi сеть работает в диапазоне 2.4 ГГц (Pico W не поддерживает 5 ГГц).
   - Перезагрузите Pico W и ваш Wi-Fi роутер.

2. **Цвета отображаются неправильно**:
   - Возможно, вы используете RGB-светодиод с общим катодом вместо анода. В этом случае нужно изменить функцию value_to_duty в main.py, убрав инверсию значения:
   
   .. code-block:: python
   
       def value_to_duty(value):
           # Для RGB-светодиода с общим катодом без инверсии
           return int(value * 257)

3. **Веб-интерфейс не отображается**:
   - Проверьте, что ваш компьютер или мобильное устройство подключены к той же Wi-Fi сети, что и Pico W.
   - Убедитесь, что вы правильно ввели IP-адрес в браузере.
   - Попробуйте перезагрузить Pico W.

4. **Ошибка "Файл index.html не найден"**:
   - Убедитесь, что вы загрузили все необходимые файлы на Pico W, включая index.html.
   - Проверьте, что файл имеет правильное имя (с учётом регистра).

Расширение проекта
------------------------

1. **Добавление анимации**:
   - Можно добавить функцию для плавного перехода между цветами.
   - Создать различные световые эффекты (например, радуга, пульсация).

2. **Интеграция с датчиками**:
   - Подключить датчик движения для автоматического включения светодиода.
   - Использовать датчик освещённости для автоматической регулировки яркости.

3. **Дополнительные элементы управления**:
   - Добавить цветовое колесо для выбора цвета.
   - Добавить таймер для автоматического выключения.

Заключение
--------------------------------

В этом уроке мы научились управлять RGB-светодиодом через веб-интерфейс с помощью Raspberry Pi Pico W. Мы использовали ШИМ для регулировки яркости каждого цвета, создали простой и интуитивно понятный веб-интерфейс и настроили Wi-Fi подключение.

Этот проект демонстрирует основы IoT (Интернета вещей) и может служить отправной точкой для более сложных проектов умного освещения и домашней автоматизации.

.. note::
    
    Не забудьте экспериментировать с разными цветами и попробовать расширить функциональность проекта!

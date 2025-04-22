Визуализация температуры из OpenWeatherMap на адресной ленте WS2812
=====================================================================

Введение
-----------------------------------------

В этом уроке мы создадим интерактивный индикатор температуры, используя адресную RGB-ленту WS2812 и Raspberry Pi Pico W. Наше устройство будет получать данные о текущей температуре с сервиса OpenWeatherMap и отображать её с помощью различных цветов и анимаций на кольце из 12 светодиодов. Такой проект не только выглядит эффектно, но и служит наглядным индикатором погодных условий.

Необходимые компоненты
-----------------------------------------

- Raspberry Pi Pico W
- Адресная RGB-лента WS2812B (кольцо из 12 светодиодов)
- Провода для подключения
- Источник питания 5В (если используется только кольцо из 12 светодиодов, можно питать от Pico W)
- Резистор 470 Ом (опционально, для защиты первого светодиода)
- Конденсатор 1000 мкФ (опционально, для сглаживания скачков напряжения)
- OLED-дисплей SSD1306 (опционально, для отображения дополнительной информации)

Схема подключения
-----------------------------------------

Подключение адресной ленты WS2812B к Raspberry Pi Pico W:

- VCC ленты к VBUS (5В) Pico W (PIN 40)
- GND ленты к GND Pico W (PIN 38)
- DIN (вход данных) ленты к GP28 (PIN 34) Pico W через резистор 470 Ом

.. note::
    
    Если вы используете более длинную ленту или несколько колец, рекомендуется подключить внешний источник питания 5В напрямую к ленте, соединив только GND с Pico W для общего заземления.

Структура проекта
-----------------------------------------

Наш проект будет состоять из следующих файлов:
- config.py - конфигурационный файл с настройками Wi-Fi и OpenWeatherMap
- neopixel.py - библиотека для работы с адресной лентой WS2812B
- main.py - основной файл программы

Код проекта
-----------------------------------------

1. Сначала создадим файл конфигурации (config.py):

.. code-block:: python

    # Конфигурационный файл для проекта визуализации температуры
    
    # Настройки Wi-Fi
    WIFI_SSID = "Название_вашей_сети"
    WIFI_PASSWORD = "Пароль_вашей_сети"
    
    # Настройки OpenWeatherMap
    OWM_API_KEY = "ваш_api_ключ_openweathermap"
    CITY_NAME = "Moscow"  # Название города на английском
    
    # Настройки обновления
    UPDATE_INTERVAL = 10 * 60  # Интервал обновления данных в секундах (10 минут)
    
    # Настройки светодиодной ленты
    LED_COUNT = 12  # Количество светодиодов в кольце
    LED_PIN = 28     # Номер GPIO пина для управления лентой
    LED_BRIGHTNESS = 0.3  # Яркость от 0.0 до 1.0
    
    # Диапазоны температур для цветового кодирования (в градусах Цельсия)
    TEMP_RANGES = [
        (-30, -20, (0, 0, 255)),     # Синий: очень холодно
        (-20, -10, (0, 128, 255)),   # Голубой: холодно
        (-10, 0, (0, 255, 255)),     # Циан: прохладно
        (0, 10, (0, 255, 128)),      # Бирюзовый: свежо
        (10, 20, (0, 255, 0)),       # Зеленый: комфортно
        (20, 25, (128, 255, 0)),     # Желто-зеленый: тепло
        (25, 30, (255, 255, 0)),     # Желтый: очень тепло
        (30, 35, (255, 128, 0)),     # Оранжевый: жарко
        (35, 50, (255, 0, 0))        # Красный: очень жарко
    ]

2. Теперь нам понадобится библиотека neopixel.py для управления адресной лентой. Её можно загрузить из официального репозитория micropython-lib или создать самостоятельно по документации. Основной файл программы (main.py):

.. code-block:: python

    import network
    import time
    import urequests as requests
    import json
    import machine
    from machine import Pin
    import neopixel
    import random
    import gc
    
    # Импортируем настройки из config.py
    from config import (WIFI_SSID, WIFI_PASSWORD, OWM_API_KEY, CITY_NAME, 
                        UPDATE_INTERVAL, LED_COUNT, LED_PIN, LED_BRIGHTNESS, 
                        TEMP_RANGES)
    
    # Настройка пина для встроенного светодиода
    led = Pin("LED", Pin.OUT)
    
    # Инициализация адресной ленты WS2812B
    pixels = neopixel.NeoPixel(Pin(LED_PIN), LED_COUNT)
    
    # Установка начальной яркости
    brightness = LED_BRIGHTNESS
    
    # Переменные для хранения текущих данных о погоде
    current_temp = None
    current_weather_id = None
    current_color = (0, 0, 0)
    
    # Функция для мигания встроенным светодиодом
    def blink_led(times=1, delay=0.2):
        for _ in range(times):
            led.on()
            time.sleep(delay)
            led.off()
            time.sleep(delay)
    
    # Функция для подключения к Wi-Fi
    def connect_to_wifi():
        # Настраиваем Wi-Fi интерфейс
        wlan = network.WLAN(network.STA_IF)
        wlan.active(True)
        
        # Если уже подключены - отключаемся
        if wlan.isconnected():
            wlan.disconnect()
            time.sleep(1)
        
        print(f"Подключение к Wi-Fi сети {WIFI_SSID}...")
        
        # Установка всех светодиодов в синий цвет для индикации подключения
        set_all_pixels((0, 0, 32))
        
        # Пытаемся подключиться к Wi-Fi
        wlan.connect(WIFI_SSID, WIFI_PASSWORD)
        
        # Ждем подключения с таймаутом
        max_wait = 20
        dot_position = 0
        
        while max_wait > 0:
            if wlan.isconnected():
                break
            
            # Вращаем синюю точку по кругу для индикации процесса подключения
            set_all_pixels((0, 0, 8))
            pixels[dot_position] = adjust_brightness((0, 0, 255))
            pixels.write()
            
            dot_position = (dot_position + 1) % LED_COUNT
            
            # Мигаем встроенным светодиодом
            blink_led(1, 0.1)
            
            max_wait -= 1
            print("Ожидание подключения...")
            time.sleep(0.5)
        
        # Проверяем результат подключения
        if wlan.isconnected():
            ip_address = wlan.ifconfig()[0]
            print(f"Подключено к Wi-Fi! IP-адрес: {ip_address}")
            
            # Мигаем зеленым для индикации успешного подключения
            for _ in range(3):
                set_all_pixels((0, 32, 0))
                pixels.write()
                time.sleep(0.2)
                set_all_pixels((0, 0, 0))
                pixels.write()
                time.sleep(0.2)
            
            return True
        else:
            print("Не удалось подключиться к Wi-Fi")
            
            # Мигаем красным для индикации ошибки
            for _ in range(3):
                set_all_pixels((32, 0, 0))
                pixels.write()
                time.sleep(0.5)
                set_all_pixels((0, 0, 0))
                pixels.write()
                time.sleep(0.5)
            
            return False
    
    # Функция для получения данных о погоде
    def get_weather_data():
        global current_temp, current_weather_id
        
        # API URL для текущей погоды
        url = f"https://api.openweathermap.org/data/2.5/weather?q={CITY_NAME}&appid={OWM_API_KEY}&units=metric"
        
        try:
            # Делаем запрос к API
            print(f"Запрос данных о погоде для города {CITY_NAME}...")
            
            # Пульсация белым цветом во время запроса данных
            pulse_color((32, 32, 32), 3)
            
            response = requests.get(url)
            
            # Проверяем статус ответа
            if response.status_code == 200:
                # Парсим JSON-ответ
                weather_data = response.json()
                response.close()
                
                # Извлекаем температуру и ID погодных условий
                temp = weather_data['main']['temp']
                weather_id = weather_data['weather'][0]['id']
                
                # Сохраняем текущие данные
                current_temp = temp
                current_weather_id = weather_id
                
                print(f"Получены данные о погоде для {CITY_NAME}:")
                print(f"Температура: {temp}°C")
                print(f"ID погоды: {weather_id}")
                
                return temp, weather_id
            else:
                print(f"Ошибка запроса: статус {response.status_code}")
                response.close()
                return None, None
        except Exception as e:
            print(f"Ошибка при получении данных о погоде: {e}")
            return None, None
    
    # Функция для настройки яркости цвета
    def adjust_brightness(color):
        r, g, b = color
        return (int(r * brightness), int(g * brightness), int(b * brightness))
    
    # Функция для установки всех пикселей в один цвет
    def set_all_pixels(color):
        adjusted_color = adjust_brightness(color)
        for i in range(LED_COUNT):
            pixels[i] = adjusted_color
        pixels.write()
    
    # Функция для определения цвета по температуре
    def get_temp_color(temp):
        # Если температура ниже самого низкого диапазона
        if temp < TEMP_RANGES[0][0]:
            return TEMP_RANGES[0][2]
        
        # Если температура выше самого высокого диапазона
        if temp > TEMP_RANGES[-1][1]:
            return TEMP_RANGES[-1][2]
        
        # Находим подходящий диапазон
        for low, high, color in TEMP_RANGES:
            if low <= temp < high:
                return color
        
        # По умолчанию возвращаем белый
        return (255, 255, 255)
    
    # Функция для плавного перехода между цветами
    def fade_to_color(new_color, steps=30, delay=0.03):
        global current_color
        
        # Получаем текущие значения цвета
        r1, g1, b1 = current_color
        r2, g2, b2 = new_color
        
        # Постепенно меняем цвет
        for i in range(steps + 1):
            ratio = i / steps
            r = int(r1 + (r2 - r1) * ratio)
            g = int(g1 + (g2 - g1) * ratio)
            b = int(b1 + (b2 - b1) * ratio)
            
            set_all_pixels((r, g, b))
            time.sleep(delay)
        
        # Обновляем текущий цвет
        current_color = new_color
    
    # Функция для пульсации определенным цветом
    def pulse_color(color, times=1, steps=15, delay=0.02):
        for _ in range(times):
            # Нарастание яркости
            for i in range(steps):
                ratio = i / steps
                r = int(color[0] * ratio)
                g = int(color[1] * ratio)
                b = int(color[2] * ratio)
                set_all_pixels((r, g, b))
                time.sleep(delay)
            
            # Затухание яркости
            for i in range(steps, -1, -1):
                ratio = i / steps
                r = int(color[0] * ratio)
                g = int(color[1] * ratio)
                b = int(color[2] * ratio)
                set_all_pixels((r, g, b))
                time.sleep(delay)
    
    # Функция для создания эффекта дождя (для дождливой погоды)
    def rain_animation(base_color, drop_color, duration=5, drops_per_sec=3):
        end_time = time.time() + duration
        
        while time.time() < end_time:
            # Устанавливаем базовый цвет
            set_all_pixels(base_color)
            
            # Создаем "капли" случайно по кольцу
            for _ in range(2):  # Несколько капель одновременно
                drop_pos = random.randint(0, LED_COUNT - 1)
                pixels[drop_pos] = adjust_brightness(drop_color)
            
            pixels.write()
            time.sleep(1/drops_per_sec)
    
    # Функция для создания эффекта снега (для снежной погоды)
    def snow_animation(duration=5, snowflakes_per_sec=2):
        end_time = time.time() + duration
        
        while time.time() < end_time:
            # Темно-синий фон
            set_all_pixels((0, 0, 32))
            
            # Создаем "снежинки" случайно по кольцу
            for _ in range(3):  # Несколько снежинок одновременно
                snow_pos = random.randint(0, LED_COUNT - 1)
                pixels[snow_pos] = adjust_brightness((200, 200, 255))
            
            pixels.write()
            time.sleep(1/snowflakes_per_sec)
    
    # Функция для создания эффекта молнии (для грозы)
    def thunder_animation(duration=5, flash_count=3):
        end_time = time.time() + duration
        dark_color = (20, 20, 40)  # Темно-синий для облаков
        flash_color = (180, 180, 255)  # Бело-голубая молния
        
        while time.time() < end_time:
            # Устанавливаем темный цвет для облаков
            set_all_pixels(dark_color)
            pixels.write()
            
            # Случайная задержка между вспышками
            time.sleep(random.uniform(0.5, 1.5))
            
            # Вспышка молнии
            if random.random() < 0.7:  # 70% шанс молнии
                for _ in range(random.randint(1, 3)):  # 1-3 вспышки подряд
                    set_all_pixels(flash_color)
                    pixels.write()
                    time.sleep(random.uniform(0.05, 0.15))
                    set_all_pixels(dark_color)
                    pixels.write()
                    time.sleep(random.uniform(0.05, 0.2))
    
    # Функция для создания эффекта облачности
    def cloud_animation(duration=5):
        end_time = time.time() + duration
        base_color = (100, 100, 100)  # Серый для облаков
        
        while time.time() < end_time:
            # Плавное изменение яркости для имитации движения облаков
            for i in range(30):
                ratio = (1 + 0.3 * (0.5 + 0.5 * (i / 30)))
                r = int(base_color[0] * ratio)
                g = int(base_color[1] * ratio)
                b = int(base_color[2] * ratio)
                
                if r > 255: r = 255
                if g > 255: g = 255
                if b > 255: b = 255
                
                set_all_pixels((r, g, b))
                time.sleep(0.1)
    
    # Функция для создания эффекта солнечного света
    def sun_animation(duration=5):
        end_time = time.time() + duration
        
        # Желтый базовый цвет для солнца
        base_color = (255, 180, 0)
        
        while time.time() < end_time:
            # Имитация солнечных лучей, движущихся по кольцу
            for start_pos in range(LED_COUNT):
                # Сначала заполняем все приглушенным желтым
                for i in range(LED_COUNT):
                    pixels[i] = adjust_brightness((64, 45, 0))
                
                # Теперь добавляем яркие лучи
                for offset in range(4):  # 4 луча
                    pos = (start_pos + offset * (LED_COUNT // 4)) % LED_COUNT
                    pixels[pos] = adjust_brightness(base_color)
                
                pixels.write()
                time.sleep(0.1)
    
    # Функция для выбора анимации на основе погодных условий
    def show_weather_animation(weather_id):
        # Группы ID погодных условий:
        # 2xx: Гроза
        # 3xx: Морось
        # 5xx: Дождь
        # 6xx: Снег
        # 7xx: Атмосферные явления (туман и т.д.)
        # 800: Ясно
        # 80x: Облачно
        
        # Определяем тип анимации по ID погоды
        if 200 <= weather_id < 300:  # Гроза
            print("Анимация: Гроза")
            thunder_animation(duration=10)
        elif 300 <= weather_id < 400 or 500 <= weather_id < 600:  # Дождь или морось
            print("Анимация: Дождь")
            blue_shade = (0, 0, 128)
            drop_color = (0, 0, 255)
            rain_animation(blue_shade, drop_color, duration=10)
        elif 600 <= weather_id < 700:  # Снег
            print("Анимация: Снег")
            snow_animation(duration=10)
        elif 700 <= weather_id < 800:  # Туман, дымка и т.д.
            print("Анимация: Облачность/Туман")
            cloud_animation(duration=10)
        elif weather_id == 800:  # Ясно
            print("Анимация: Солнечно")
            sun_animation(duration=10)
        elif 801 <= weather_id < 900:  # Облачно
            print("Анимация: Переменная облачность")
            cloud_animation(duration=10)
        else:
            # По умолчанию просто отображаем цвет по температуре
            temp_color = get_temp_color(current_temp)
            fade_to_color(temp_color)
    
    # Функция для отображения температуры на светодиодном кольце
    def display_temperature(temp):
        # Получаем цвет для текущей температуры
        temp_color = get_temp_color(temp)
        
        print(f"Отображение температуры: {temp}°C, цвет: {temp_color}")
        
        # Плавный переход к новому цвету
        fade_to_color(temp_color)
        
        # Количество активных светодиодов зависит от температуры
        # Отображаем от 1 до 12 светодиодов в зависимости от температуры
        # Например, -20°C = 1 светодиод, +40°C = 12 светодиодов
        min_temp = -20
        max_temp = 40
        temp_range = max_temp - min_temp
        
        # Ограничиваем температуру в заданном диапазоне
        limited_temp = max(min_temp, min(temp, max_temp))
        
        # Вычисляем количество активных светодиодов
        active_leds = int(((limited_temp - min_temp) / temp_range) * LED_COUNT)
        active_leds = max(1, min(active_leds, LED_COUNT))  # Минимум 1, максимум LED_COUNT
        
        print(f"Активных светодиодов: {active_leds} из {LED_COUNT}")
        
        # Отображаем активные светодиоды
        for i in range(LED_COUNT):
            if i < active_leds:
                pixels[i] = adjust_brightness(temp_color)
            else:
                pixels[i] = (0, 0, 0)
        
        pixels.write()
        time.sleep(2)  # Показываем уровень температуры 2 секунды
        
        # Возвращаемся к общему цвету
        for i in range(LED_COUNT):
            pixels[i] = adjust_brightness(temp_color)
        
        pixels.write()
    
    # Главная функция программы
    def main():
        global current_temp, current_weather_id
        
        # Очищаем ленту при запуске
        set_all_pixels((0, 0, 0))
        
        # Приветственная анимация
        pulse_color((32, 0, 32), 2)  # Пульсация пурпурным
        
        # Подключаемся к Wi-Fi
        if not connect_to_wifi():
            # Если не удалось подключиться, мигаем красным и перезагружаемся
            for _ in range(3):
                set_all_pixels((32, 0, 0))
                time.sleep(0.5)
                set_all_pixels((0, 0, 0))
                time.sleep(0.5)
            
            # Перезагружаем устройство
            machine.reset()
        
        # Основной цикл программы
        last_update_time = 0
        
        while True:
            current_time = time.time()
            
            # Проверяем, нужно ли обновить данные о погоде
            if current_time - last_update_time >= UPDATE_INTERVAL:
                # Получаем новые данные о погоде
                temp, weather_id = get_weather_data()
                
                if temp is not None and weather_id is not None:
                    # Отображаем температуру на светодиодном кольце
                    display_temperature(temp)
                    
                    # Показываем анимацию в зависимости от погодных условий
                    show_weather_animation(weather_id)
                    
                    last_update_time = current_time
                else:
                    # Если не удалось получить данные, мигаем желтым
                    for _ in range(3):
                        set_all_pixels((32, 32, 0))
                        time.sleep(0.3)
                        set_all_pixels((0, 0, 0))
                        time.sleep(0.3)
                    
                    # Пробуем снова через минуту
                    last_update_time = current_time - UPDATE_INTERVAL + 60
            
            # Отображаем основной цвет, соответствующий текущей температуре
            if current_temp is not None:
                temp_color = get_temp_color(current_temp)
                set_all_pixels(temp_color)
            
            # Мигаем встроенным светодиодом для индикации работы
            blink_led(1, 0.1)
            
            # Освобождаем память
            gc.collect()
            
            # Ждем перед следующей проверкой
            time.sleep(10)
    
    # Запускаем программу
    if __name__ == "__main__":
        main()

Возможные проблемы и их решения
-----------------------------------------

1. **Светодиоды не включаются или работают некорректно**:
   - Проверьте правильность подключения проводов (VCC, GND, DIN).
   - Убедитесь, что вы используете правильный GPIO пин в конфигурации.
   - Проверьте напряжение питания - для WS2812B требуется 5В.
   - Попробуйте уменьшить яркость в config.py - возможно, ваш источник питания не обеспечивает достаточной мощности.

2. **Ошибка при подключении к Wi-Fi**:
   - Убедитесь, что вы указали правильный SSID и пароль в файле config.py.
   - Проверьте, что ваша Wi-Fi сеть работает в диапазоне 2.4 ГГц (Pico W не поддерживает 5 ГГц).
   - Перезагрузите Pico W и ваш Wi-Fi роутер.

3. **Ошибка при получении данных о погоде**:
   - Проверьте правильность API-ключа OpenWeatherMap.
   - Убедитесь, что название города указано корректно на английском языке.
   - Проверьте подключение к интернету.
   - Проверьте, не превышен ли лимит запросов к API (для бесплатного плана это 1000 запросов в день).

4. **Память переполняется**:
   - Если программа работает некоторое время, а затем зависает, возможно, происходит утечка памяти.
   - Добавьте больше вызовов gc.collect() в различные части кода.
   - Упростите анимации или уменьшите длительность их выполнения.

5. **Цветовая индикация не соответствует ожиданиям**:
   - Настройте диапазоны температур в TEMP_RANGES в файле config.py в соответствии с климатом вашего региона.
   - Убедитесь, что значения LED_BRIGHTNESS не слишком низкие или высокие.

Загрузка и запуск проекта
-----------------------------------------

1. Прежде всего вам нужно зарегистрироваться на OpenWeatherMap и получить бесплатный API-ключ.

2. Скачайте библиотеку neopixel.py для работы с адресными светодиодами WS2812B, если она отсутствует в вашей прошивке MicroPython.

3. Подключите Raspberry Pi Pico W к компьютеру через USB.

4. Создайте и загрузите на Pico W следующие файлы:
   - config.py (с вашими настройками Wi-Fi, API-ключом и настройками светодиодного кольца)
   - main.py (с основным кодом программы)
   - neopixel.py (библиотека для управления светодиодами)

5. Отредактируйте файл config.py, указав:
   - Имя и пароль вашей Wi-Fi сети
   - Ваш API-ключ OpenWeatherMap
   - Название вашего города на английском языке
   - При необходимости настройте яркость светодиодов и другие параметры

6. Убедитесь, что WS2812B-кольцо правильно подключено к Pico W согласно схеме.

7. Запустите программу, нажав кнопку Run в Thonny или перезагрузив Pico W.

8. После успешного запуска вы увидите приветственную анимацию, затем индикацию подключения к Wi-Fi, и наконец, отображение температуры и погодных условий с помощью цветов и анимаций.

Как это работает
-----------------------------------------

1. **Подключение к Wi-Fi**:
   - Программа инициализирует Wi-Fi модуль и подключается к заданной сети.
   - Во время подключения на светодиодном кольце отображается синяя вращающаяся точка.
   - При успешном подключении кольцо мигнет зеленым, при ошибке - красным.

2. **Получение данных о погоде**:
   - После подключения к Wi-Fi, программа отправляет HTTP-запрос к API OpenWeatherMap.
   - Из ответа извлекаются текущая температура и ID погодных условий.
   - Во время запроса кольцо пульсирует белым цветом.

3. **Отображение температуры**:
   - Для отображения температуры используется цветовое кодирование:

     - Синий - очень холодно (ниже -20°C)
     - Голубой и бирюзовый - холодно (-20°C до 0°C)
     - Зеленый - комфортно (10°C до 20°C)
     - Желтый и оранжевый - тепло (20°C до 30°C)
     - Красный - жарко (выше 30°C)

   - Также количество активных светодиодов показывает температуру в диапазоне от -20°C до +40°C.

4. **Погодные анимации**:
   - В зависимости от ID погодных условий, программа выбирает соответствующую анимацию:
   
     - Гроза (ID 2xx): мигающие вспышки молний на темно-синем фоне
     - Дождь (ID 3xx, 5xx): анимация капель, падающих на синем фоне
     - Снег (ID 6xx): медленно мерцающие белые "снежинки" на синем фоне
     - Туман, дымка (ID 7xx): серая пульсация для имитации тумана
     - Ясно (ID 800): яркая желтая анимация вращающихся лучей солнца
     - Облачно (ID 80x): мягкие переходы серого для имитации облаков

5. **Периодическое обновление**:
   - Программа работает в бесконечном цикле и обновляет данные о погоде с интервалом, указанным в config.py.
   - Между обновлениями кольцо светится цветом, соответствующим текущей температуре, обеспечивая постоянную визуальную обратную связь.
   - Регулярно показываются анимации, соответствующие погодным условиям, чтобы сделать отображение более динамичным и информативным.
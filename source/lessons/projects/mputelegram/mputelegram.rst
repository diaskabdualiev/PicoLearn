MPU6050: уведомление о вибрации/наклоне в Telegram
=====================================================================

Введение
-----------------------------------------

В этом уроке мы создадим систему, которая будет отправлять уведомления в Telegram при обнаружении вибрации или наклона с помощью модуля MPU6050. Такая система может быть полезна для мониторинга состояния различных объектов: оповещения о неавторизованном перемещении ценных предметов, отслеживания наклона конструкций, обнаружения падения или даже создания простой системы охраны транспортных средств. Мы будем использовать Raspberry Pi Pico W для чтения данных с акселерометра и гироскопа, их обработки и отправки уведомлений при превышении заданных порогов.

Необходимые компоненты
-----------------------------------------

- Raspberry Pi Pico W
- Модуль MPU6050 (акселерометр и гироскоп)
- Соединительные провода
- Макетная плата (опционально)
- Доступ к Wi-Fi сети
- Аккаунт в Telegram

О датчике MPU6050
-----------------------------------------

MPU6050 - это комбинированный модуль, включающий трехосевой акселерометр и трехосевой гироскоп. Этот датчик позволяет измерять:

- Ускорение по трем осям (x, y, z) - для определения вибрации, движения и ориентации
- Угловую скорость по трем осям - для определения вращения
- Температуру (встроенный датчик температуры)

Модуль использует интерфейс I2C для связи с микроконтроллером, что требует всего двух линий передачи данных (SDA и SCL), помимо питания (VCC) и заземления (GND).

Схема подключения
-----------------------------------------

Подключение MPU6050 к Raspberry Pi Pico W через I2C:

- VCC модуля MPU6050 к 3.3V (PIN 36) Pico W
- GND модуля MPU6050 к GND (PIN 38) Pico W
- SCL модуля MPU6050 к GP1 (PIN 2) Pico W - это I2C0 SCL
- SDA модуля MPU6050 к GP0 (PIN 1) Pico W - это I2C0 SDA

.. note::
    
    Обратите внимание, что библиотека MPU6050, которую мы используем в этом уроке, настроена на подключение к пинам GPIO 22 (SCL) и GPIO 21 (SDA) для ESP32. Мы изменим эти настройки в коде для работы с Raspberry Pi Pico W.

Структура проекта
-----------------------------------------

Наш проект будет состоять из следующих файлов:
- telegram.py - библиотека для работы с Telegram API
- mpu6050.py - библиотека для работы с датчиком MPU6050
- config.py - конфигурационный файл с настройками
- main.py - основной файл программы

Код проекта
-----------------------------------------

1. Сначала модифицируем библиотеку MPU6050 для работы с Raspberry Pi Pico W (mpu6050.py):

.. code-block:: python

    # Находим в коде библиотеки раздел инициализации I2C и заменяем его на:
    
    # Initializing the I2C method for Raspberry Pi Pico W
    # Pin assignment:
    # SCL -> GP1
    # SDA -> GP0
    self.i2c = SoftI2C(scl=Pin(1), sda=Pin(0), freq=100000)

2. Создадим файл конфигурации (config.py):

.. code-block:: python

    # Конфигурационный файл для проекта MPU6050 с уведомлениями в Telegram
    
    # Настройки Wi-Fi
    WIFI_SSID = "Название_вашей_сети"
    WIFI_PASSWORD = "Пароль_вашей_сети"
    
    # Токен Telegram бота (получен от BotFather)
    BOT_TOKEN = "ваш_токен_от_botfather"
    
    # ID чата для отправки уведомлений
    # Можно получить, отправив боту команду /start и посмотрев ID в логах
    CHAT_ID = 123456789  # Замените на ваш ID
    
    # Настройки MPU6050
    MPU_ADDR = 0x68  # Стандартный адрес I2C для MPU6050
    
    # Пороговые значения для отправки уведомлений
    ACCEL_THRESHOLD = 1.5  # Порог ускорения в g (1g = 9.8 м/с²)
    GYRO_THRESHOLD = 30.0  # Порог угловой скорости в градусах/с
    ANGLE_THRESHOLD = 20.0  # Порог наклона в градусах
    
    # Настройки уведомлений
    COOLDOWN_PERIOD = 30  # Период "охлаждения" между уведомлениями (в секундах)
    NOTIFICATION_ENABLED = True  # Включены ли уведомления по умолчанию
    
    # Настройки мониторинга
    CHECK_INTERVAL = 0.1  # Интервал проверки данных датчика (в секундах)
    SAMPLES_PER_CHECK = 5  # Количество измерений для усреднения
    
    # Текст сообщений
    VIBRATION_ALERT_MESSAGE = "🚨 Обнаружена сильная вибрация!"
    TILT_ALERT_MESSAGE = "⚠️ Обнаружен наклон устройства!"
    SYSTEM_STARTED_MESSAGE = "✅ Система мониторинга вибрации и наклона запущена."

3. Теперь создадим основной файл программы (main.py):

.. code-block:: python

    import time
    import machine
    import uasyncio as asyncio
    from machine import Pin, SoftI2C
    import math
    import gc
    
    # Импортируем библиотеки
    from telegram import TelegramBot
    from mpu6050 import MPU6050
    
    # Импортируем настройки из config.py
    from config import (WIFI_SSID, WIFI_PASSWORD, BOT_TOKEN, CHAT_ID, MPU_ADDR,
                       ACCEL_THRESHOLD, GYRO_THRESHOLD, ANGLE_THRESHOLD,
                       COOLDOWN_PERIOD, NOTIFICATION_ENABLED, CHECK_INTERVAL,
                       SAMPLES_PER_CHECK, VIBRATION_ALERT_MESSAGE,
                       TILT_ALERT_MESSAGE, SYSTEM_STARTED_MESSAGE)
    
    # Настройка пина для встроенного светодиода
    led = Pin("LED", Pin.OUT)
    
    # Переменные для отслеживания состояния
    last_notification_time = 0
    notifications_enabled = NOTIFICATION_ENABLED
    mpu = None  # Будет инициализирован позже
    bot = None  # Будет инициализирован позже
    
    # Базовые значения для калибровки
    base_accel = {"x": 0, "y": 0, "z": 0}
    base_angle = {"x": 0, "y": 0}
    
    # Счетчики событий
    vibration_count = 0
    tilt_count = 0
    
    # Функция для мигания светодиодом
    def blink_led(times=1, delay=0.2):
        for _ in range(times):
            led.on()
            time.sleep(delay)
            led.off()
            time.sleep(delay)
    
    # Функция для инициализации MPU6050
    def init_mpu6050():
        global mpu
        try:
            # Инициализация MPU6050
            mpu = MPU6050(addr=MPU_ADDR)
            
            # Настраиваем диапазоны измерения
            mpu.set_accel_range(mpu._ACC_RNG_2G)  # ±2g
            mpu.set_gyro_range(mpu._GYR_RNG_250DEG)  # ±250°/s
            
            print("MPU6050 инициализирован успешно")
            
            # Проверяем чтение данных
            accel_data = mpu.read_accel_data(True)  # В единицах g
            gyro_data = mpu.read_gyro_data()  # В градусах/с
            temp = mpu.read_temperature()  # В градусах Цельсия
            
            print(f"Акселерометр: X={accel_data['x']:.2f}g, Y={accel_data['y']:.2f}g, Z={accel_data['z']:.2f}g")
            print(f"Гироскоп: X={gyro_data['x']:.2f}°/s, Y={gyro_data['y']:.2f}°/s, Z={gyro_data['z']:.2f}°/s")
            print(f"Температура: {temp:.1f}°C")
            
            return True
        except Exception as e:
            print(f"Ошибка инициализации MPU6050: {e}")
            return False
    
    # Функция для калибровки датчика
    def calibrate_mpu6050():
        global base_accel, base_angle
        
        print("Калибровка MPU6050...")
        
        # Среднее значение из нескольких измерений
        samples = 20
        sum_accel = {"x": 0, "y": 0, "z": 0}
        sum_angle = {"x": 0, "y": 0}
        
        for _ in range(samples):
            accel_data = mpu.read_accel_data(True)  # В единицах g
            angle_data = mpu.read_angle()  # В радианах
            
            # Преобразуем радианы в градусы
            angle_deg = {
                "x": math.degrees(angle_data["x"]),
                "y": math.degrees(angle_data["y"])
            }
            
            sum_accel["x"] += accel_data["x"]
            sum_accel["y"] += accel_data["y"]
            sum_accel["z"] += accel_data["z"]
            
            sum_angle["x"] += angle_deg["x"]
            sum_angle["y"] += angle_deg["y"]
            
            time.sleep(0.05)
        
        # Вычисляем средние значения
        base_accel["x"] = sum_accel["x"] / samples
        base_accel["y"] = sum_accel["y"] / samples
        base_accel["z"] = sum_accel["z"] / samples
        
        base_angle["x"] = sum_angle["x"] / samples
        base_angle["y"] = sum_angle["y"] / samples
        
        print(f"Базовое ускорение: X={base_accel['x']:.2f}g, Y={base_accel['y']:.2f}g, Z={base_accel['z']:.2f}g")
        print(f"Базовый угол: X={base_angle['x']:.2f}°, Y={base_angle['y']:.2f}°")
        
        return True
    
    # Функция для проверки вибрации и наклона
    def check_motion():
        global vibration_count, tilt_count
        
        # Считываем текущие значения
        accel_data = mpu.read_accel_data(True)  # В единицах g
        gyro_data = mpu.read_gyro_data()  # В градусах/с
        angle_data = mpu.read_angle()  # В радианах
        
        # Преобразуем радианы в градусы
        angle_deg = {
            "x": math.degrees(angle_data["x"]),
            "y": math.degrees(angle_data["y"])
        }
        
        # Вычисляем разницу ускорения от базовых значений
        accel_diff = {
            "x": abs(accel_data["x"] - base_accel["x"]),
            "y": abs(accel_data["y"] - base_accel["y"]),
            "z": abs(accel_data["z"] - base_accel["z"])
        }
        
        # Вычисляем разницу угла от базовых значений
        angle_diff = {
            "x": abs(angle_deg["x"] - base_angle["x"]),
            "y": abs(angle_deg["y"] - base_angle["y"])
        }
        
        # Вычисляем максимальную разницу ускорения по всем осям
        max_accel_diff = max(accel_diff["x"], accel_diff["y"], accel_diff["z"])
        
        # Вычисляем максимальную абсолютную угловую скорость по всем осям
        max_gyro = max(abs(gyro_data["x"]), abs(gyro_data["y"]), abs(gyro_data["z"]))
        
        # Вычисляем максимальную разницу угла по осям X и Y
        max_angle_diff = max(angle_diff["x"], angle_diff["y"])
        
        # Определяем, превышены ли пороги
        vibration_detected = max_accel_diff > ACCEL_THRESHOLD or max_gyro > GYRO_THRESHOLD
        tilt_detected = max_angle_diff > ANGLE_THRESHOLD
        
        # Обновляем счетчики событий
        if vibration_detected:
            vibration_count += 1
        
        if tilt_detected:
            tilt_count += 1
        
        return {
            "vibration": vibration_detected,
            "tilt": tilt_detected,
            "accel_diff": max_accel_diff,
            "gyro_max": max_gyro,
            "angle_diff": max_angle_diff
        }
    
    # Функция для проверки, нужно ли отправлять уведомление
    def should_notify():
        global last_notification_time
        
        # Если уведомления отключены, возвращаем False
        if not notifications_enabled:
            return False
        
        # Проверяем, прошло ли достаточно времени с момента последнего уведомления
        if time.time() - last_notification_time < COOLDOWN_PERIOD:
            return False
        
        return True
    
    # Функция для отправки уведомления о событии
    def send_event_notification(event_type, data):
        global last_notification_time
        
        if bot is None:
            print("Бот не инициализирован")
            return
        
        current_time = time.time()
        local_time = time.localtime(current_time)
        
        # Форматируем время
        time_str = "{:02d}:{:02d}:{:02d}".format(local_time[3], local_time[4], local_time[5])
        date_str = "{:02d}/{:02d}/{:04d}".format(local_time[2], local_time[1], local_time[0])
        
        # Создаем сообщение в зависимости от типа события
        if event_type == "vibration":
            message = f"{VIBRATION_ALERT_MESSAGE}\n"
            message += f"⏰ Время: {time_str}\n"
            message += f"📅 Дата: {date_str}\n\n"
            message += f"📊 Данные датчика:\n"
            message += f"📏 Ускорение: {data['accel_diff']:.2f}g (порог: {ACCEL_THRESHOLD}g)\n"
            message += f"🔄 Угловая скорость: {data['gyro_max']:.2f}°/с (порог: {GYRO_THRESHOLD}°/с)\n"
            message += f"📈 Всего обнаружено вибраций: {vibration_count}"
        elif event_type == "tilt":
            message = f"{TILT_ALERT_MESSAGE}\n"
            message += f"⏰ Время: {time_str}\n"
            message += f"📅 Дата: {date_str}\n\n"
            message += f"📊 Данные датчика:\n"
            message += f"📐 Угол наклона: {data['angle_diff']:.2f}° (порог: {ANGLE_THRESHOLD}°)\n"
            message += f"📈 Всего обнаружено наклонов: {tilt_count}"
        else:
            message = f"⚠️ Неизвестное событие: {event_type}"
        
        # Отправляем сообщение в Telegram
        bot.send(CHAT_ID, message)
        print(f"Отправлено уведомление о {event_type} в {time_str}")
        
        # Обновляем время последнего уведомления
        last_notification_time = current_time
    
    # Обработчик сообщений для бота
    def message_handler(bot, msg_type, chat_name, sender_name, chat_id, text, entry):
        global notifications_enabled, base_accel, base_angle
        
        print(f"Получено сообщение от {sender_name}: {text}")
        
        # Обработка команды /start
        if text == "/start":
            welcome_message = f"Привет, {sender_name}! 👋\n"
            welcome_message += "Я бот для уведомлений о вибрации и наклоне с датчика MPU6050.\n"
            welcome_message += "Я буду отправлять вам уведомления при обнаружении движения.\n\n"
            welcome_message += "Доступные команды:\n"
            welcome_message += "/start - Показать это сообщение\n"
            welcome_message += "/status - Показать текущий статус системы\n"
            welcome_message += "/enable - Включить уведомления\n"
            welcome_message += "/disable - Отключить уведомления\n"
            welcome_message += "/calibrate - Калибровать датчик\n"
            welcome_message += "/check - Проверить текущие показания датчика"
            
            bot.send(chat_id, welcome_message)
            
            # Также сохраняем ID чата для отправки уведомлений
            if chat_id != CHAT_ID:
                print(f"Получен новый chat_id: {chat_id}. Обновите CHAT_ID в config.py для получения уведомлений.")
        
        # Обработка команды /status
        elif text == "/status":
            status = "включены ✅" if notifications_enabled else "отключены ❌"
            cooldown_remaining = max(0, COOLDOWN_PERIOD - (time.time() - last_notification_time))
            
            status_message = f"📊 Статус системы обнаружения движения:\n\n"
            status_message += f"🔔 Уведомления: {status}\n"
            status_message += f"⏱️ Период охлаждения: {COOLDOWN_PERIOD} сек\n"
            
            if cooldown_remaining > 0:
                status_message += f"⏳ Осталось до следующего уведомления: {int(cooldown_remaining)} сек\n\n"
            else:
                status_message += "✅ Готов к отправке следующего уведомления\n\n"
            
            status_message += f"🚨 Пороговые значения:\n"
            status_message += f"📏 Ускорение: {ACCEL_THRESHOLD}g\n"
            status_message += f"🔄 Угловая скорость: {GYRO_THRESHOLD}°/с\n"
            status_message += f"📐 Угол наклона: {ANGLE_THRESHOLD}°\n\n"
            
            status_message += f"📈 Статистика:\n"
            status_message += f"💥 Обнаружено вибраций: {vibration_count}\n"
            status_message += f"↗️ Обнаружено наклонов: {tilt_count}"
            
            bot.send(chat_id, status_message)
        
        # Обработка команды /enable
        elif text == "/enable":
            notifications_enabled = True
            bot.send(chat_id, "✅ Уведомления включены!")
            print("Уведомления включены пользователем")
        
        # Обработка команды /disable
        elif text == "/disable":
            notifications_enabled = False
            bot.send(chat_id, "❌ Уведомления отключены!")
            print("Уведомления отключены пользователем")
        
        # Обработка команды /calibrate
        elif text == "/calibrate":
            bot.send(chat_id, "🔄 Запуск калибровки датчика...\nПожалуйста, убедитесь, что устройство находится в стабильном положении.")
            
            # Запускаем калибровку
            if calibrate_mpu6050():
                bot.send(chat_id, "✅ Калибровка успешно завершена!")
            else:
                bot.send(chat_id, "❌ Ошибка при калибровке. Проверьте подключение датчика.")
        
        # Обработка команды /check
        elif text == "/check":
            try:
                # Считываем текущие значения
                accel_data = mpu.read_accel_data(True)  # В единицах g
                gyro_data = mpu.read_gyro_data()  # В градусах/с
                angle_data = mpu.read_angle()  # В радианах
                
                # Преобразуем радианы в градусы
                angle_deg = {
                    "x": math.degrees(angle_data["x"]),
                    "y": math.degrees(angle_data["y"])
                }
                
                # Вычисляем абсолютное ускорение
                accel_abs = math.sqrt(accel_data["x"]**2 + accel_data["y"]**2 + accel_data["z"]**2)
                
                # Формируем сообщение с данными
                check_message = f"📊 Текущие показания датчика MPU6050:\n\n"
                check_message += f"📏 Акселерометр (g):\n"
                check_message += f"  X: {accel_data['x']:.2f}\n"
                check_message += f"  Y: {accel_data['y']:.2f}\n"
                check_message += f"  Z: {accel_data['z']:.2f}\n"
                check_message += f"  Абс: {accel_abs:.2f}\n\n"
                
                check_message += f"🔄 Гироскоп (°/с):\n"
                check_message += f"  X: {gyro_data['x']:.2f}\n"
                check_message += f"  Y: {gyro_data['y']:.2f}\n"
                check_message += f"  Z: {gyro_data['z']:.2f}\n\n"
                
                check_message += f"📐 Углы наклона (°):\n"
                check_message += f"  X: {angle_deg['x']:.2f}\n"
                check_message += f"  Y: {angle_deg['y']:.2f}\n\n"
                
                check_message += f"🌡️ Температура: {mpu.read_temperature():.1f}°C"
                
                bot.send(chat_id, check_message)
            except Exception as e:
                bot.send(chat_id, f"❌ Ошибка при чтении данных: {str(e)}")
        
        # Неизвестная команда
        else:
            bot.send(chat_id, "❓ Неизвестная команда. Отправьте /start для просмотра доступных команд.")
    
    # Асинхронная функция для мониторинга движения
    async def monitor_motion():
        print("Запуск мониторинга движения...")
        
        while True:
            try:
                motion_detected = False
                vibration_detected = False
                tilt_detected = False
                
                # Проверяем несколько раз для усреднения
                avg_data = {
                    "accel_diff": 0,
                    "gyro_max": 0,
                    "angle_diff": 0
                }
                
                for _ in range(SAMPLES_PER_CHECK):
                    data = check_motion()
                    
                    avg_data["accel_diff"] += data["accel_diff"] / SAMPLES_PER_CHECK
                    avg_data["gyro_max"] += data["gyro_max"] / SAMPLES_PER_CHECK
                    avg_data["angle_diff"] += data["angle_diff"] / SAMPLES_PER_CHECK
                    
                    vibration_detected = vibration_detected or data["vibration"]
                    tilt_detected = tilt_detected or data["tilt"]
                    
                    await asyncio.sleep(0.01)  # Небольшая пауза между измерениями
                
                # Включаем светодиод при обнаружении движения
                if vibration_detected or tilt_detected:
                    led.on()
                    motion_detected = True
                else:
                    led.off()
                
                # Отправляем уведомления, если нужно
                if should_notify():
                    if vibration_detected:
                        send_event_notification("vibration", avg_data)
                    elif tilt_detected:
                        send_event_notification("tilt", avg_data)
                
                # Ждем до следующей проверки
                await asyncio.sleep(CHECK_INTERVAL)
            
            except Exception as e:
                print(f"Ошибка при мониторинге: {e}")
                await asyncio.sleep(1)
    
    # Асинхронная функция для запуска бота
    async def run_bot():
        global bot
        
        print("Запуск Telegram-бота...")
        bot = TelegramBot(BOT_TOKEN, message_handler)
        
        # Отправляем сообщение о запуске системы
        bot.send(CHAT_ID, SYSTEM_STARTED_MESSAGE)
        
        # Запускаем основной цикл бота
        try:
            print("Бот запущен! Ожидание команд...")
            await bot.run()
        except Exception as e:
            print(f"Ошибка в работе бота: {e}")
            # Мигаем светодиодом при ошибке
            for _ in range(5):
                blink_led(3, 0.1)
                time.sleep(0.5)
    
    # Главная функция программы
    async def main():
        # Импортируем gc для отслеживания памяти
        import gc
        gc.collect()
        
        print("Инициализация...")
        
        # Индикация запуска с помощью светодиода
        blink_led(3, 0.2)
        
        # Инициализируем MPU6050
        if not init_mpu6050():
            print("Не удалось инициализировать датчик MPU6050!")
            # Индикация ошибки
            while True:
                blink_led(2, 0.1)
                await asyncio.sleep(1)
        
        # Калибруем датчик
        calibrate_mpu6050()
        
        # Подключаемся к Wi-Fi
        print(f"Подключение к Wi-Fi: {WIFI_SSID}...")
        
        temp_bot = TelegramBot(BOT_TOKEN, message_handler)
        temp_bot.connect_wifi(WIFI_SSID, WIFI_PASSWORD)
        
        # Мигаем светодиодом после успешного подключения
        blink_led(5, 0.1)
        
        print("Wi-Fi подключен!")
        
        # Запускаем бота в отдельной задаче
        bot_task = asyncio.create_task(run_bot())
        
        # Запускаем мониторинг движения в отдельной задаче
        monitor_task = asyncio.create_task(monitor_motion())
        
        # Запускаем периодическую очистку памяти
        while True:
            gc.collect()
            await asyncio.sleep(60)  # Очищаем память раз в минуту
    
    # Запускаем программу с помощью асинхронного цикла событий
    if __name__ == "__main__":
        try:
            asyncio.run(main())
        except Exception as e:
            print(f"Критическая ошибка: {e}")
            machine.reset()  # Перезагрузка при критической ошибке

Загрузка и запуск проекта
-----------------------------------------

1. Убедитесь, что на вашем Raspberry Pi Pico W установлен MicroPython с поддержкой Wi-Fi.

2. Скопируйте файлы библиотек на ваш Pico W:
   - telegram.py (библиотека для работы с Telegram API)
   - mpu6050.py (библиотека для работы с датчиком MPU6050)

3. Создайте и загрузите на Pico W файлы:
   - config.py (с вашими настройками Wi-Fi, токеном бота и параметрами датчика)
   - main.py (с кодом бота и обработкой данных с датчика)

4. Отредактируйте файл config.py, указав:
   - Имя и пароль вашей Wi-Fi сети
   - Токен вашего Telegram-бота, полученный от BotFather
   - ID чата для отправки уведомлений
   - При необходимости настройте пороговые значения для обнаружения движения

5. Убедитесь, что в библиотеке mpu6050.py правильно указаны пины I2C для Raspberry Pi Pico W:
   - SCL: GP1 (PIN 2)
   - SDA: GP0 (PIN 1)

6. Подключите модуль MPU6050 к Pico W согласно схеме подключения.

7. Запустите программу, нажав кнопку Run в Thonny или перезагрузив Pico W.

8. После запуска и успешной инициализации датчика система отправит сообщение о готовности в указанный чат в Telegram.

9. Отправьте боту команду `/start`, чтобы увидеть список доступных команд.

10. При обнаружении вибрации или наклона вы будете получать уведомления в Telegram.

Как это работает
-----------------------------------------

1. **Инициализация и калибровка**:
   - Система инициализирует датчик MPU6050 и настраивает его диапазоны измерения.
   - Производится калибровка датчика в состоянии покоя для определения базовых значений ускорения и углов наклона.
   - Калибровку можно повторить в любой момент с помощью команды `/calibrate`.

2. **Мониторинг движения**:
   - Система периодически опрашивает датчик MPU6050 и анализирует данные акселерометра, гироскопа и углов наклона.
   - Для повышения точности производится несколько измерений с последующим усреднением.
   - Полученные значения сравниваются с пороговыми, определенными в config.py.

3. **Обнаружение событий**:
   - Система определяет два типа событий:

     - **Вибрация**: превышение порогов ускорения или угловой скорости
     - **Наклон**: превышение порога угла наклона

   - При обнаружении события включается встроенный светодиод для визуальной индикации.

4. **Управление уведомлениями**:
   - Функция `should_notify` проверяет, нужно ли отправлять уведомление, учитывая период "охлаждения" между уведомлениями.
   - Если все условия выполнены, отправляется уведомление в Telegram с подробной информацией о событии.

5. **Управление через Telegram**:
   - Бот поддерживает несколько команд для управления системой:
   
     - `/start` - Приветствие и список команд
     - `/status` - Текущее состояние системы
     - `/enable` и `/disable` - Включение/отключение уведомлений
     - `/calibrate` - Повторная калибровка датчика
     - `/check` - Проверка текущих показаний датчика

Настройка чувствительности
-----------------------------------------

Для оптимальной работы системы в разных условиях может потребоваться настройка пороговых значений в файле config.py:

1. **ACCEL_THRESHOLD** (порог ускорения, в единицах g):
   - Меньшие значения (0.1-0.5g) - для обнаружения легкой вибрации или небольших движений
   - Средние значения (0.5-1.5g) - для обнаружения умеренных движений и вибрации
   - Большие значения (2.0g и выше) - только для сильных ударов или резких движений

2. **GYRO_THRESHOLD** (порог угловой скорости, в градусах/с):
   - Меньшие значения (5-15°/с) - для обнаружения медленных поворотов
   - Средние значения (20-50°/с) - для обнаружения умеренных вращательных движений
   - Большие значения (100°/с и выше) - только для очень быстрых вращений

3. **ANGLE_THRESHOLD** (порог угла наклона, в градусах):
   - Меньшие значения (5-10°) - для обнаружения небольших наклонов
   - Средние значения (15-30°) - для обнаружения значительных наклонов
   - Большие значения (45° и выше) - только для очень больших наклонов

Также можно настроить другие параметры:

- **COOLDOWN_PERIOD** - период между уведомлениями (в секундах)
- **CHECK_INTERVAL** - интервал проверки данных датчика (в секундах)
- **SAMPLES_PER_CHECK** - количество измерений для усреднения

Возможные проблемы и их решения
-----------------------------------------

1. **Ошибка инициализации датчика MPU6050**:
   - Проверьте правильность подключения проводов (VCC, GND, SCL, SDA)
   - Убедитесь, что напряжение питания составляет 3.3В
   - Проверьте, что в коде указан правильный I2C адрес датчика (0x68 или 0x69)
   - Попробуйте использовать внешние подтягивающие резисторы на линиях SCL и SDA (4.7kΩ к VCC)

2. **Частые ложные срабатывания**:
   - Увеличьте пороговые значения в config.py
   - Увеличьте количество измерений для усреднения (SAMPLES_PER_CHECK)
   - Проведите повторную калибровку датчика в стабильном положении
   - Установите устройство на устойчивую поверхность, изолированную от вибраций

3. **Отсутствие уведомлений при движении**:
   - Уменьшите пороговые значения в config.py
   - Проверьте, включены ли уведомления (команда `/status`)
   - Убедитесь, что датчик правильно инициализирован и калиброван

4. **Проблемы с подключением к Wi-Fi**:
   - Убедитесь, что SSID и пароль указаны правильно
   - Проверьте, что ваша Wi-Fi сеть работает в диапазоне 2.4 ГГц (Pico W не поддерживает 5 ГГц)
   - Расположите Pico W ближе к роутеру

5. **Проблемы с Telegram-ботом**:
   - Проверьте правильность токена бота
   - Убедитесь, что ID чата указан правильно
   - Проверьте подключение к интернету

Расширение проекта
-----------------------------------------

1. **Добавление дополнительных типов уведомлений**:
   - Обнаружение свободного падения (все оси акселерометра близки к нулю)
   - Обнаружение ударов (резкие пики ускорения)
   - Определение направления движения или наклона

2. **Интеграция с другими датчиками**:
   - Добавьте GPS-модуль для отправки координат при обнаружении движения
   - Подключите датчик звука для обнаружения шума
   - Добавьте камеру для съемки фото при обнаружении движения

3. **Расширенная настройка через Telegram**:
   - Добавьте команды для изменения пороговых значений через бот
   - Реализуйте настройку периода "охлаждения"
   - Добавьте различные режимы работы (высокая чувствительность, низкая чувствительность)

4. **Энергосбережение для автономной работы**:
   - Реализуйте режимы глубокого сна для экономии энергии
   - Добавьте поддержку питания от батареи с контролем заряда
   - Оптимизируйте код для минимального энергопотребления

5. **Расширенная аналитика**:
   - Сохраняйте историю событий в файл
   - Добавьте анализ частотных характеристик вибрации
   - Реализуйте определение типов движения с помощью простых алгоритмов машинного обучения

Заключение
-----------------------------------------

В этом уроке мы создали систему уведомлений о вибрации и наклоне на основе модуля MPU6050 и Raspberry Pi Pico W. Такая система может быть использована для мониторинга состояния различных объектов, обеспечения безопасности или в качестве компонента более сложных проектов "умного дома".

Использование Telegram в качестве интерфейса управления и канала уведомлений делает систему универсальной и доступной с любого устройства, где установлен мессенджер. При этом вы можете не только получать уведомления, но и контролировать систему, проверять текущие показания датчика и настраивать его работу.

Благодаря гибким настройкам чувствительности и возможности периодической калибровки, система может быть адаптирована для различных условий использования - от мониторинга тонких вибраций до обнаружения значительных перемещений и наклонов.

.. note::
    
    Для наиболее точных измерений рекомендуется размещать датчик MPU6050 на плоской горизонтальной поверхности и выполнять калибровку после каждого изменения положения устройства. Также важно минимизировать влияние внешних факторов, таких как вибрация от работающих механизмов или электромагнитные помехи от других устройств.
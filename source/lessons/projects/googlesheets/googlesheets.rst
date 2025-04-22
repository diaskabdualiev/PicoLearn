Google Sheets: хранение данных о температуре
=====================================================================

Введение
-----------------------------------------

В этом уроке мы научимся отправлять данные с датчика температуры, влажности и давления BME280 в Google Sheets, используя Raspberry Pi Pico W. Такая система позволит вам удалённо мониторить и записывать показания окружающей среды в таблицу, которую можно просматривать из любой точки мира и анализировать данные с помощью встроенных инструментов Google Sheets.

Сохранение данных в облачном хранилище имеет множество преимуществ по сравнению с локальным хранением:
- Доступ к данным откуда угодно
- Автоматическое резервное копирование
- Возможность создания графиков и визуализаций
- Совместный доступ с другими пользователями
- Интеграция с другими сервисами Google

Необходимые компоненты
-----------------------------------------

- Raspberry Pi Pico W
- Датчик BME280 
- Соединительные провода
- Доступ к Wi-Fi сети
- Аккаунт Google
- Модуль MicroGoogleSheet для MicroPython (будет объяснено ниже)

Схема подключения
-----------------------------------------

Подключение датчика BME280 к Raspberry Pi Pico W через I2C:

- VCC датчика BME280 к 3.3V Pico W
- GND датчика BME280 к GND Pico W
- SCL датчика BME280 к GP5 Pico W
- SDA датчика BME280 к GP4 Pico W

.. note::
    
    В этом уроке мы используем пины GP4 и GP5 для I2C соединения, но вы можете использовать другие пины, поддерживающие I2C, изменив соответствующий код.

Настройка Google Sheets и скрипта Apps Script
----------------------------------------------------------------------------------

Перед началом программирования Pico W нам нужно настроить Google Sheets и создать скрипт для приема данных:

1. **Создание таблицы Google Sheets**:
   - Откройте Google Drive и создайте новую таблицу (Sheets)
   - Назовите её, например, "BME280 Data Log"
   - В первой строке создайте заголовки: "Дата", "Время", "Температура (°C)", "Давление (hPa)", "Влажность (%)"

2. **Настройка скрипта Google Apps Script**:
   - В открытой таблице перейдите в меню "Расширения" > "Apps Script"
   - Удалите весь код в редакторе и вставьте следующий:

.. code-block:: javascript

    function doGet(e) {
        var ss = SpreadsheetApp.getActiveSpreadsheet();
        var sheet = ss.getActiveSheet();
        
        var result = {};
        
        try {
        // Получаем параметры
        if (e.parameter.row && e.parameter.data) {
            var row = parseInt(e.parameter.row);
            var data = JSON.parse(e.parameter.data);
            
            // Записываем данные
            for (var i = 0; i < data.length; i++) {
            sheet.getRange(row, i+1).setValue(data[i]);
            }
            
            result.status = "success";
            result.message = "Data recorded successfully";
        } else if (e.parameter.getrow) {
            // Получаем значение ячейки
            var getrow = parseInt(e.parameter.getrow);
            var getcol = parseInt(e.parameter.getcol || 1);
            var value = sheet.getRange(getrow, getcol).getValue();
            
            result.status = "success";
            result.value = value;
        } else {
            result.status = "error";
            result.message = "Missing required parameters";
        }
        } catch (error) {
        result.status = "error";
        result.message = error.toString();
        }
        
        return ContentService.createTextOutput(JSON.stringify(result))
        .setMimeType(ContentService.MimeType.JSON);
    }


3. **Сохранение и развертывание скрипта**:
   - Сохраните скрипт (File > Save)
   - Нажмите "Deploy" > "New deployment"
   - Выберите тип "Web app"
   - Настройте:

     - Execute as: "Me"
     - Who has access: "Anyone"

   - Нажмите "Deploy"
   - Скопируйте "Deployment ID" - он понадобится нам для программы

Файловая структура проекта
-----------------------------------------

Наш проект будет состоять из следующих файлов:
- main.py - основной файл программы
- config.py - конфигурационные параметры 
- bme280_float.py - библиотека для работы с датчиком BME280
- ggsheet.py - библиотека для работы с Google Sheets

Код проекта и его объяснение
-----------------------------------------

1. Сначала создадим файл конфигурации (config.py):

.. code-block:: python

    # Конфигурационный файл для проекта BME280 с Google Sheets
    
    # Настройки Wi-Fi
    wifi_config = {
        'ssid': 'Название_вашей_сети',
        'password': 'Пароль_вашей_сети'
    }
    
    # Настройки Google Sheets
    google_sheet_url = "https://docs.google.com/spreadsheets/d/ВАША_ТАБЛИЦА_ID/edit"
    google_sheet_name = "Sheet1"  # Имя листа в таблице
    google_app_deployment_id = "ВАШИЙ_DEPLOYMENT_ID"  # ID развертывания скрипта
    
    # Параметры программы
    send_interval = 300  # Интервал отправки данных в секундах (по умолчанию 5 минут)

2. Библиотека для работы с Google Sheets (ggsheet.py):

.. code-block:: python

    import urequests as requests
    import json
    import time

    class MicroGoogleSheet:
        def __init__(self, spreadsheetURL, sheetName):
            self.spreadsheetURL = spreadsheetURL
            self.sheetName = sheetName
            self.deploymentID = None
            
        def set_DeploymentID(self, deploymentID):
            self.deploymentID = deploymentID
            
        def _buildURL(self, params):
            baseURL = f"https://script.google.com/macros/s/{self.deploymentID}/exec"
            query = '&'.join([f"{k}={v}" for k, v in params.items()])
            return f"{baseURL}?{query}"
            
        def appendRow(self, row, data):
            if not self.deploymentID:
                raise ValueError("DeploymentID not set. Use set_DeploymentID() method.")
                
            params = {
                'row': row,
                'data': json.dumps(data)
            }
            
            url = self._buildURL(params)
            
            try:
                response = requests.get(url)
                result = json.loads(response.text)
                response.close()
                
                if result.get('status') == 'success':
                    return True
                else:
                    print(f"Error: {result.get('message', 'Unknown error')}")
                    return False
            except Exception as e:
                print(f"Error sending data: {e}")
                return False
                
        def getCell(self, row, col=1):
            if not self.deploymentID:
                raise ValueError("DeploymentID not set. Use set_DeploymentID() method.")
                
            params = {
                'getrow': row,
                'getcol': col
            }
            
            url = self._buildURL(params)
            
            try:
                response = requests.get(url)
                result = json.loads(response.text)
                response.close()
                
                if result.get('status') == 'success':
                    return result.get('value', '')
                else:
                    print(f"Error: {result.get('message', 'Unknown error')}")
                    return None
            except Exception as e:
                print(f"Error getting cell: {e}")
                return None

3. Основной код программы (main.py):

Теперь разберем основной код по частям:

**Импорт библиотек и настройка**

.. code-block:: python

    import time
    from machine import Pin, I2C
    import bme280_float as bme280
    from ggsheet import MicroGoogleSheet
    import config
    import network

В этом блоке мы импортируем все необходимые библиотеки:
- `time` - для управления временем и паузами
- `machine.Pin`, `machine.I2C` - для работы с GPIO и I2C интерфейсом
- `bme280_float` - библиотека для работы с датчиком BME280
- `ggsheet.MicroGoogleSheet` - наша библиотека для взаимодействия с Google Sheets
- `config` - файл с конфигурационными параметрами
- `network` - для управления Wi-Fi соединением

**Функция подключения к Wi-Fi**

.. code-block:: python

    def connect_wifi():
        sta_if = network.WLAN(network.STA_IF)
        sta_if.active(True)
        if not sta_if.isconnected():
            print("Подключение к WiFi:", config.wifi_config['ssid'])
            sta_if.connect(config.wifi_config['ssid'], config.wifi_config['password'])
            while not sta_if.isconnected():
                pass
        print("Подключение к WiFi успешно!")
        print("IP-адрес:", sta_if.ifconfig()[0])

Функция `connect_wifi()`:
1. Создает объект WLAN в режиме клиента (STA_IF)
2. Активирует Wi-Fi модуль
3. Если еще не подключен, пытается подключиться с заданными SSID и паролем
4. Ждет успешного подключения
5. Выводит информацию о подключении, включая присвоенный IP-адрес

**Инициализация аппаратной части**

.. code-block:: python

    # Инициализация I2C с пинами 4 и 5
    i2c = I2C(0, sda=Pin(4), scl=Pin(5))

    # Инициализация BME280 с правильным адресом
    bme = bme280.BME280(i2c=i2c, address=0x76)

    # Google Sheets данные
    GOOGLE_SHEET_URL = "https://docs.google.com/spreadsheets/d/1PhZPDGMI8yiTPmwPs6A185CWVtj03DB9CBn5alx4AZA/edit?gid=0#gid=0"
    GOOGLE_SHEET_NAME = "Sheet1"
    GOOGLE_APP_DEPLOYMENT_ID = "AKfycbymALx2RnjBws57Vv-b714gVTaMucoRx0zb1j7tTqWeevMoE4b6u5U7gj_8WkBQPON-"

Здесь мы:
1. Инициализируем I2C интерфейс на пинах GP4 (SDA) и GP5 (SCL)
2. Создаем объект датчика BME280 с адресом 0x76 (стандартный адрес)
3. Определяем константы для работы с Google Sheets (в рабочем коде их лучше брать из файла конфигурации)

**Функция чтения данных с датчика**

.. code-block:: python

    def read_sensor_data():
        values = bme.values
        print("Исходные значения с датчика:", values)
        
        if not values or len(values) < 3:
            print("Ошибка: неверный формат данных с датчика")
            return None
        
        temperature = values[0]
        pressure = values[1]
        humidity = values[2]
        
        # Обработка строковых значений
        try:
            # Удаляем единицы измерения и обрабатываем разные возможные форматы
            temperature = temperature.replace("C", "").replace("°C", "").replace(",", ".").strip()
            pressure = pressure.replace("hPa", "").replace("гПа", "").replace(",", ".").strip()
            humidity = humidity.replace("%", "").replace(",", ".").strip()
            
            print(f"После обработки: температура={temperature}, давление={pressure}, влажность={humidity}")
            
            # Преобразуем строки в числа
            temperature = float(temperature)
            pressure = float(pressure)
            humidity = float(humidity)
            
            return {
                "temperature": temperature,
                "pressure": pressure,
                "humidity": humidity
            }
        except (ValueError, TypeError) as e:
            print(f"Ошибка преобразования значений в числа: {e}")
            return None

Функция `read_sensor_data()`:
1. Получает значения с датчика BME280 через свойство `values`
2. Проверяет, что мы получили данные в правильном формате
3. Извлекает отдельные значения температуры, давления и влажности
4. Очищает строковые значения от единиц измерения (°C, hPa, %)
5. Заменяет запятые на точки (для корректного преобразования в числа)
6. Преобразует строки в числа с плавающей точкой
7. Возвращает словарь с обработанными значениями или None в случае ошибки

**Функция отправки данных в Google Sheets**

.. code-block:: python

    def send_to_google_sheets():
        # Подключаемся к WiFi
        connect_wifi()
        
        # Получаем данные с датчика
        sensor_data = read_sensor_data()
        if not sensor_data:
            print("Не удалось получить данные с датчика")
            return False
        
        # Текущая дата и время
        timestamp = time.localtime()
        date_str = f"{timestamp[0]}-{timestamp[1]:02d}-{timestamp[2]:02d}"
        time_str = f"{timestamp[3]:02d}:{timestamp[4]:02d}:{timestamp[5]:02d}"
        
        # Преобразуем числа в строки с запятой вместо точки
        temp_str = str(sensor_data['temperature']).replace('.', ',')
        pressure_str = str(sensor_data['pressure']).replace('.', ',')
        humidity_str = str(sensor_data['humidity']).replace('.', ',')
        
        print(f"Данные для отправки: Дата: {date_str}, Время: {time_str}")
        print(f"Температура: {temp_str} °C")
        print(f"Давление: {pressure_str} hPa")
        print(f"Влажность: {humidity_str} %")
        
        try:
            # Создаем экземпляр для работы с Google Sheets
            ggsheet = MicroGoogleSheet(GOOGLE_SHEET_URL, GOOGLE_SHEET_NAME)
            ggsheet.set_DeploymentID(GOOGLE_APP_DEPLOYMENT_ID)
            
            # Получаем последний номер строки в таблице
            row = 1
            while True:
                try:
                    cell_value = ggsheet.getCell(row, 1)
                    if not cell_value:
                        break
                    row += 1
                    # Ограничиваем поиск
                    if row > 1000:
                        row = 1  # Если таблица слишком большая, начинаем с начала
                        break
                except:
                    # Если возникла ошибка при получении ячейки, значит мы достигли конца таблицы
                    break
            
            print(f"Запись будет добавлена в строку {row}")
            
            # Формируем данные для записи с запятыми вместо точек
            data_row = [
                date_str,
                time_str,
                temp_str,
                pressure_str,
                humidity_str
            ]
            
            # Записываем данные в таблицу
            ggsheet.appendRow(row, data_row)
            print("Данные успешно отправлены в Google Sheets!")
            return True
        
        except Exception as e:
            print(f"Ошибка отправки данных в Google Sheets: {e}")
            return False

Функция `send_to_google_sheets()`:
1. Подключается к Wi-Fi (если еще не подключен)
2. Получает данные с датчика через функцию `read_sensor_data()`
3. Формирует строки даты и времени из текущего времени
4. Преобразует числа в строки, заменяя точки на запятые (для правильного отображения в Google Sheets)
5. Создает экземпляр MicroGoogleSheet с URL таблицы и именем листа
6. Устанавливает Deployment ID для скрипта
7. Находит первую пустую строку в таблице, последовательно проверяя значения в первом столбце
8. Формирует массив данных для записи (дата, время, температура, давление, влажность)
9. Отправляет данные в таблицу и возвращает результат операции

**Основной цикл программы**

.. code-block:: python

    def main():
        print("Запуск программы отправки данных с BME280 в Google Sheets...")
        
        # Настраиваемые параметры
        send_interval = 200  # Интервал отправки данных в секундах
        max_retries = 3     # Максимальное количество попыток при ошибке
        
        # Чтение настроек из файла config, если они есть
        if hasattr(config, 'send_interval'):
            send_interval = config.send_interval
        
        print(f"Интервал отправки данных: {send_interval} секунд")
        
        try:
            while True:
                print("\n--- Отправка данных ---")
                
                # Попытки отправки с повторами при ошибке
                success = False
                retries = 0
                
                while not success and retries < max_retries:
                    if retries > 0:
                        print(f"Повторная попытка {retries}/{max_retries}...")
                        # Небольшая задержка перед повторной попыткой
                        time.sleep(5)
                    
                    success = send_to_google_sheets()
                    retries += 1
                    
                    if success:
                        break
                
                if not success:
                    print(f"Не удалось отправить данные после {max_retries} попыток")
                
                print(f"Следующая отправка через {send_interval} секунд...")
                time.sleep(send_interval)
        
        except KeyboardInterrupt:
            print("Программа остановлена пользователем")
        except Exception as e:
            print(f"Критическая ошибка: {e}")
            # Перезапуск программы после короткой задержки
            print("Перезапуск программы через 10 секунд...")
            time.sleep(10)
            main()  # Рекурсивный вызов для перезапуска

    if __name__ == "__main__":
        main()

Функция `main()`:
1. Устанавливает настройки интервала отправки и максимального количества повторов
2. Проверяет наличие пользовательских настроек в файле config.py
3. Запускает бесконечный цикл для периодической отправки данных
4. При каждой итерации:

   - Пытается отправить данные с использованием функции `send_to_google_sheets()`
   - Если отправка не удалась, повторяет попытку до `max_retries` раз
   - Ожидает указанный интервал времени перед следующей отправкой

5. Обрабатывает исключения:

   - KeyboardInterrupt - для корректного завершения программы
   - Другие исключения - перезапускает программу после задержки

Загрузка и запуск проекта
-----------------------------------------

1. Убедитесь, что на вашем Raspberry Pi Pico W установлен MicroPython с поддержкой Wi-Fi.

2. Скопируйте библиотеки на ваш Pico W:
   - bme280_float.py - библиотека для работы с датчиком BME280
   - ggsheet.py - наша библиотека для работы с Google Sheets

3. Создайте и загрузите на Pico W файлы:
   - config.py (с вашими настройками Wi-Fi и Google Sheets)
   - main.py (основной код программы)

4. Отредактируйте файл config.py, указав:
   - Имя и пароль вашей Wi-Fi сети
   - URL вашей таблицы Google Sheets
   - Deployment ID вашего Apps Script
   - Желаемый интервал отправки данных

5. Перезагрузите Pico W, и программа начнет автоматически отправлять данные в вашу таблицу Google Sheets с указанным интервалом.

Анализ данных и создание графиков в Google Sheets
----------------------------------------------------------------------------------

После того, как ваша система начнет отправлять данные в Google Sheets, вы можете использовать встроенные инструменты для их анализа и визуализации:

1. **Создание графика температуры**:
   - Выделите столбцы с датой/временем и температурой
   - Выберите Insert > Chart (Вставка > Диаграмма)
   - Выберите Line chart (Линейный график)
   - Настройте параметры отображения (заголовок, легенду, оси)

2. **Создание графика влажности и давления**:
   - Аналогично создайте графики для столбцов влажности и давления
   - Вы можете создать комбинированный график с несколькими рядами данных

3. **Добавление скользящего среднего**:
   - Для сглаживания колебаний добавьте столбец со скользящим средним
   - Используйте формулу: =AVERAGE(C$2:C2) (где C - столбец с температурой)
   - Добавьте этот ряд данных на график

4. **Анализ данных**:
   - Найдите минимальные/максимальные значения с помощью функций MIN() и MAX()
   - Вычислите среднее значение за день/неделю с помощью AVERAGEIF()
   - Создайте сводную таблицу для анализа по дням недели или часам суток

5. **Настройка оповещений**:
   - В Google Sheets можно настроить правила условного форматирования
   - Выделите ячейки, которые хотите мониторить
   - Format > Conditional formatting > Custom formula
   - Используйте формулу типа =C2>30 для выделения высоких температур

Пример анализа данных с помощью формул
-----------------------------------------

Вот несколько полезных формул для анализа ваших данных в Google Sheets:

1. **Средняя температура за день**:
   ```
   =AVERAGEIF(A:A, "2023-11-15", C:C)
   ```

2. **Максимальная и минимальная температура за все время**:
   ```
   =MAX(C:C)
   =MIN(C:C)
   ```

3. **Амплитуда температуры за день**:
   ```
   =MAXIFS(C:C, A:A, "2023-11-15") - MINIFS(C:C, A:A, "2023-11-15")
   ```

4. **Среднее давление по часам суток**:
   ```
   =AVERAGEIFS(D:D, B:B, ">09:00", B:B, "<10:00")
   ```

Возможные проблемы и их решения
-----------------------------------------

1. **Ошибка подключения к Wi-Fi**:
   - Проверьте правильность SSID и пароля
   - Убедитесь, что ваша Wi-Fi сеть работает в диапазоне 2.4 ГГц (Pico W не поддерживает 5 ГГц)
   - Расположите Pico W ближе к роутеру

2. **Ошибка чтения данных с BME280**:
   - Проверьте подключение датчика (питание, I2C)
   - Убедитесь, что используется правильный I2C адрес (0x76 или 0x77)
   - Проверьте правильность инициализации I2C с пинами GP4 и GP5

3. **Ошибка отправки данных в Google Sheets**:
   - Проверьте URL таблицы и Deployment ID
   - Убедитесь, что Apps Script развернут с доступом "Anyone"
   - Проверьте, что ваша таблица не превышает лимиты API Google (100 запросов в минуту)

4. **Проблемы с форматом данных**:
   - Если в таблице отображаются неправильные значения, проверьте обработку данных в функции read_sensor_data()
   - Для корректного отображения десятичных чисел в Google Sheets замените точки на запятые

5. **Высокое энергопотребление**:
   - Увеличьте send_interval для более редкой отправки данных
   - Реализуйте режим сна между измерениями с помощью machine.deepsleep()

Расширение проекта
-----------------------------------------

1. **Добавление других датчиков**:
   - Подключите дополнительные датчики (освещенности, CO2, качества воздуха)
   - Расширьте код для чтения и отправки большего количества параметров

2. **Создание панели мониторинга**:
   - Используйте Google Data Studio для создания интерактивной панели
   - Подключите вашу таблицу в качестве источника данных

3. **Настройка оповещений**:
   - Добавьте код для отправки сообщений в Telegram при экстремальных значениях
   - Используйте IFTTT для создания автоматических оповещений на основе данных Google Sheets

4. **Энергосбережение для автономной работы**:
   - Реализуйте режимы глубокого сна между измерениями
   - Добавьте поддержку питания от батареи с контролем заряда

5. **Анализ тенденций**:
   - Добавьте в таблицу формулы для прогнозирования будущих значений
   - Используйте FORECAST функции Google Sheets для анализа тенденций

Заключение
-----------------------------------------

В этом уроке мы создали систему для сбора данных о температуре, влажности и давлении с датчика BME280 и их автоматической отправки в Google Sheets. Такая система может использоваться для мониторинга параметров окружающей среды в вашем доме, офисе, теплице или любом другом месте, где важно отслеживать климатические условия.

Благодаря хранению данных в Google Sheets, вы получаете доступ к мощным инструментам анализа, визуализации и совместного использования данных. Вы можете создавать графики для наблюдения за изменениями параметров во времени, настраивать оповещения о критических значениях и делиться информацией с другими пользователями.

Этот проект демонстрирует, как недорогие и компактные устройства, такие как Raspberry Pi Pico W, могут быть использованы для создания полноценных IoT-решений, интегрированных с облачными сервисами.

.. note::
    
    Для надежной работы в течение длительного времени рекомендуется добавить механизмы обработки ошибок и автоматического восстановления подключения, как показано в нашем коде. Также стоит учитывать лимиты API Google (100 запросов в минуту) при настройке интервала отправки данных.
Загрузка библиотек на Pico
===================================

В некоторых проектах вам могут потребоваться дополнительные библиотеки. Поэтому сначала загрузим эти библиотеки на Raspberry Pi Pico W, чтобы потом можно было сразу запустить нужный код.

#. Загрузите соответствующий код по ссылке ниже:

   * :download:`Набор SunFounder Kepler <https://github.com/sunfounder/kepler-kit/archive/refs/heads/main.zip>`

#. Откройте Thonny IDE, подключите Pico к компьютеру с помощью кабеля micro USB и нажмите на надпись «MicroPython (Raspberry Pi Pico).COMXX» в правом нижнем углу.

   .. image:: images/sec_inter1.webp

#. В верхнем меню перейдите в **View** -> **Files**.

   .. image:: images/th_files1.webp

#. Переключитесь в папку, в которую вы скачали `пакет кода <https://github.com/sunfounder/kepler-kit/archive/refs/heads/main.zip>`_, а затем зайдите в каталог ``kepler-kit-main/libs``.

   .. image:: images/th_path1.webp

#. Выделите все файлы или папки в папке ``libs/``, щёлкните правой кнопкой мыши и выберите **Upload to**. Загрузка займёт некоторое время.

   .. image:: images/th_upload1.webp

#. Теперь в разделе ``Raspberry Pi Pico`` вы увидите файлы, которые только что загрузили.

   .. image:: images/th_done1.webp
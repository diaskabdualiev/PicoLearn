Установка MicroPython на ваш Pico
==========================================

Теперь перейдём к установке MicroPython на Raspberry Pi Pico. Thonny IDE предлагает очень удобный способ сделать это «в один клик».

.. note::
   Если вы не хотите обновлять Thonny, вы можете воспользоваться официальным `MicroPython <https://micropython.org/>`_ от Raspberry Pi. Для этого перетащите файл ``rp2_pico_xxxx.uf2`` на Pico (который определяется как USB-накопитель), и он автоматически прошьётся.


#. Откройте Thonny IDE.

   .. image:: images/set_pico1.webp

#. Зажмите кнопку **BOOTSEL** и подсоедините Pico к компьютеру через кабель micro USB. Отпустите **BOOTSEL** после того, как Pico определится как накопитель **RPI-RP2**.

   .. image:: images/bootsel_onboard1.webp

#. В правом нижнем углу Thonny нажмите кнопку выбора интерпретатора и выберите пункт **Install Micropython**.

   .. note::
      Если в вашем Thonny нет такой опции, обновите его до последней версии.

   .. image:: images/set_pico2.webp

#. В поле **Target volume** автоматически отобразится том Pico, который вы только что подключили, а в списке **Micropython variant** выберите **Raspberry Pi.Pico W/Pico WH**.

   .. image:: images/set_pico3.webp

#. Нажмите **Install**, дождитесь завершения установки и закройте это окно.

   .. image:: images/set_pico4.webp


Поздравляем! Теперь ваш Raspberry Pi Pico готов к работе.
# HDMI_adapter_for_MCU_on_FPGA
Подобие простейшей видеокарты с HDMI выходом на FPGA

Устройство представляет из себя видеоадаптер с разрешением 640х480 и выводом изображения через HDMI. Способ построения изображения - тайловый, благодаря чему количество необходимой видеопамяти для такого разрешения экрана удалось сократить до 16Кб. Весь экран разбит на плитки (или тайлы, знакоместа) размером 8х8 пикселей, получается 60 строк по 80 знакомест (всего - 4800 знакомест). Изображение цветное, есть палитра из 16 цветов, но с ограничением - два цвета на знакоместо. 
Карта памяти адаптера имеет следующий вид:

* 0x0000 - 0x12BF: 4800 байт - номера тайлов для каждого знакомемта
* 0x12C0 - 0x257F: 4800 байт - атрибут цвета каждого знакоместа
* 0x2580 - 0x27FF: 640 байт - не используется
* 0x2800 - 0x2FFF: 2048 байт - первый банк с тайлами
* 0x3000 - 0x37FF: 2048 байт - второй банк с тайлами
* 0x3800 - 0x3FFF: 2048 байт - третий банк с тайлами

Память номеров тайлов и атрибутов тайлов идет непрерывно стык в стык, банки тайлов начинаются через небольшой промежуток для выравнивания начала первого банка изображения тайлов.

При запуске FPGA - видеопамять инициализируется следующим образом: в память номеров записаны произвольные биты, сгенерированные случайно, в память каждого атрибута записаны одинаковые значения цвета фона и цвета знака, в первый банк изображений тайлов записан знакогенератор ASCII (первые 128 байт с латиницей, вторые - с кирилицей). Остальные два банка - пустые, могут использоваться для изображений тайлов пользователя. Таким образом при запуске на экране появляется набор из произвольных знаков каждого знакоместа с фиксированными цветами фона и знаков (серебристый на белом фоне). Файл с разрешением *MIF есть в проекте в папке "rom".

Одномоментно на экран может проецироваться только один банк тайлов.

При построении изображения видеоконтроллер по номеру пикселя в строке и номеру строки вычисляет номер знакоместа (от 0 до 4799) и читает из видеопамяти три байта: 1 - номер тайла для данного знакоместа, 2 - значение атрибута цвета для данного знакоместа, 3 - байт одной строки изображения тайла для данного знакоместа. Защёлкивает байт изображения тайла в сдвиговый регистр и выводит изображене в соответствии с цветами, заданными байтом атрибута. Цвета формируются так: старший полубайт (4 бита) задают цвет символа, младший полубайт (4 бита) задаёт цвет фона знакоместа. Все цвета берутся из палитры. Цвета алитры при желании можно задать любые RGB888, но не более 16. При желании можно уменьшить количестви бит атрибута фона (вплоть до нуля), соответственно увеличить количество битов атрибута знака, таким образом можно сделать фиксированный цвет фона (например, чёрный) и увеличит палитру цветов знака до 256.

Для изменения изображения на экране необходимо: 1 - записать в память необходимые номера знакомест, 2 - записать необходимые атрибуты цвета знакомест. При необходимости - записать в память пользовательские изображения тайлов (желательно не использовать для этого первый банк, так как там хранится знакогенератор).

Управление видеоконтроллером происходит по параллельной 8-битной шине:

* AD[7:0] - мультиплексированная шина адреса/данных
* CMD[2:0] - номер команды
* STROBE - стробирующий импульс

Команды CMD[2:0]:

* 000 - запись во временный регистр данных
* 001 - запись младшего байта адреса во временный регистр адреса
* 010 - запись старшего байта адреса во временный регистр адреса
* 011 - запись данных в видеопамять из временного регистра данных по адресу из временного регистра адреса
* 100 - выбор номера банка изображений тайлов (1-3) ис линии данных (AD[1:0])

**Стробирующий импульс имеет положительную полярность _|``|_**

В основе адаптера находится FPGA (в моём случае - Altera Cyclone4 EP4CE6, накупил в своё время по-дешёвке несколько штук б/у, но рабочих на Ali), в качестве видеопамяти используется встроенная в FPGA блочная память. Частота задающего кварцевого генератора - 50MHz, используется один PLL.

**Специально для этого проекта сделал отладочную плату на FPGA с HDMI разъёмом:**

![Markdown](https://github.com/AndrejChoo/HDMI_adapter_for_MCU_on_FPGA/blob/main/Images/ep4ce6_boarg.jpg)

Для управления видеоадаптера использовал отладочную плату Black pill с stm32f411ceu на борту. Хотя можно использовать и что-то более медленное, типа arduino (nano/uno). Для stm32 написалнебольшой  демонстрационный проект (см. в папке "Firmware") с выводом текста, в котором есть библиотека с минимальным набором функций для работы с адаптером. 
Пример вывода текста на экран:

![Markdown](https://github.com/AndrejChoo/HDMI_adapter_for_MCU_on_FPGA/blob/main/Images/hello.jpg)

С растровыми изображениями всё намного сложнее. Из-за ограничений, обусловленных экономией видеопамяти и ускорением работы контроллёра, вывести просто так растровое изображение, даже монохромное невозможно. Можно только его составлять из набора тайлов, будь то - псевдографика или набор маленьких рисунков как у NES. Но, используя некоторые хитрости, можно вывести что-то похожее на растр, уменьшив разрешение экрана в 4 раза. Если поделить каждое знакоместо на 4 части (на 4 кубика 4х4 пикселя), то получится всего 16 комбинаций изображения данного знакоместа. Таким образом можно вывести монохромное изображение в разрешении 160х120 пикселей. Выглядит, конечно, как японский кроссворд, особенно на 2К мониторе:

![Markdown](https://github.com/AndrejChoo/HDMI_adapter_for_MCU_on_FPGA/blob/main/Images/tiger.jpg)

Специально для конвертации *bmp файла в тайловое изображение написал небольшую утилиту. Она конвертирует монохромный (1bit) *bmp файл разрешением 160х120 пикселей в массив байтов (номеров тайлов). Сам набор из 16 тайлов есть в демонстрационном проекте на stm32 (необходимо раскомментировать в файле "main.c"). В данном примере набор из 16 тайлов записывается в CHAR BANK2.

Нормальные тайловые изображения пока не пробовал рисовать, так как не имею такого опыта. 

В перспективе, есть мысль попробовать добавить спрайты. Это значительно усложнит проект, но будет очень интересно и полезно для саморазвития.

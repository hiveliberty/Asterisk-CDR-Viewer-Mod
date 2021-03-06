
===============================================
 Системные требования
===============================================

- PHP 5.2 и выше
- База данных MySQL или другая, которая поддерживается PDO (PostgreSQL...)
- Веб-сервер (Nginx, Apache, Lighttpd...)


===============================================
 Форматы записей разговоров
===============================================

Используется HTML5 плеер для воспроизведения записей разговоров. 
Доступные форматы аудио: MP3, WAV, OGG, AAC...

Проверенные форматы аудио. Проверялось в Internet Explorer 11, Chrome 56, Firefox 52:
- MP3. Поддерживаются все браузеры
- WAV. Не работает в Internet Explorer
- OGG. Не работает в Internet Explorer

Формат GSM не поддерживается для воспроизведения ни одним браузером.


===============================================
 Проверка обновлений
===============================================

Начиная с версии 2.3 доступна проверка обновлений. Чтобы проверить доступность новой версии, нужно щелкнуть на стрелочку в самом низу веб-интерфейса (подвал), рядом с надписью "Asterisk CDR Viewer Mod v[версия]".
Проверяется только доступность новой версии, но автоматического обновления не происходит. Обновлять необходимо вручную.
Если доступна новая версия, то будут отображены: текущая версия, новая версия, изменения в последнем релизе.


===============================================
 Редактирование базы данных
===============================================

С базой данных MySQL удобнее всего работать через PHPMYADMIN. Официальный сайт: https://www.phpmyadmin.net/
Если у вас используется не база данных MySQL (а например PostgreSQL), или PHPMYADMIN не установлен, то следует использовать ADMINER. Официальный сайт: https://www.adminer.org/


===============================================
 Создание таблицы в базе
===============================================

Имя файла записи разговора будет храниться в базе MySQL (можно также выбрать, например, PostgreSQL).

Для MySQL можно использовать файл импорта "mysql_cdr.sql", можно найти в папке docs. Если импортировать этот файл в базу, то в базе будет создана таблица "cdr" со всем необходимыми для Asterisk полями.
Имя колонки для файла записи звонка будет "filename". Также будут созданы необходимые индексы и триггер, о котором можно прочитать ниже.
В дополнение ко всему, файл импорта создаст новые колонки в базе для Asterisk 12+.
Если имя таблицы "cdr" (или имя колонки для файла записи звонка) не устраивает, после импорта сами сможете переименовать. Например, с помощью PHPMYADMIN или ADMINER.

Если НЕ использовали файл импорта в базу
==
Допустим мы настроили Asterisk для работы с базой и уже создали таблицу, например "cdr". Теперь нам необходимо добавить с нашу таблицу новую колонку, например "filename", в которой будет
имя файла с записью, удобнее это сделать через PHPMYADMIN или ADMINER (смотреть здесь: https://www.adminer.org). Название колонки можно задать в конфиге. / `filename` varchar(255) DEFAULT 'none' /

Более подробно о том, как настроить Asterisk для работы с MySQL можно найти в интернете.


===============================================
 Настройка Asterisk
===============================================

Для того, чтобы Asterisk смог взаимодействовать с новыми столбцами в таблице, необходимо в файле cdr_mysql.conf создать их алиасы.
Добавим в конец этого файла ( секция [columns] ) строчки:
alias realdst => realdst
alias remoteip => remoteip
alias start => calldate
alias название_столбца => название_столбца

Вместо "название_столбца" вставьте название столбца, в котором хранится название записи звонка, например "filename".
Алиас "remoteip" нужен для записи IP адреса клиента Asterisk. Это НЕОБЯЗАТЕЛЬНО.


Все изменения производим в extensions.ael, либо в extensions.conf. В зависимости от того, в какой файле у нас написан диалплан.

Если название столбца, в котором хранится название записи звонка у вас отличается от "filename", то необходимо внести соответствующие изменения в диалплан.
Необходимо изменить строку "Set(CDR(filename)=${fname}.mp3);", на "Set(CDR(название_столбца)=${fname}.mp3);"

==
 Для extensions.ael, extensions.conf
==

В "globals" добавим пару переменных:
===
// Если 0, запись разговоров отключена
// Если 1, запись разговоров включена с одновременной конвертацией в MP3
// Если 2, запись разговоров включена и выполняется запись в формат WAV. Преобразование в MP3 формат должно быть выполнено скриптом "proc_records.sh"
RECORDING=1;
// Путь к папке с записями разговоров
DIR_RECORDS=/var/calls/;
===	

Добавим макрос.
Сразу уточним, что в этом макросе, если RECORDING=1 запись прямо во время разговора конвертируется в MP3. т.е. существует некоторая нагрузка на сервер.
Если же RECORDING=2, то нагрузка на сервер минимальная, т.к. запись выполняется в родной формат Asterisk - WAV. Конвертирование в MP3 должно быть выполнено
с помощью скрипта "proc_records.sh", который можно найти в папке docs. В скрипте написаны подробные комментарии по его настройке

==
 Для extensions.ael
==

// MixMonitor
macro recording(calling,called) {
	if ("${RECORDING}" = "1") {
		Set(fname=${UNIQUEID}-${STRFTIME(${EPOCH},,%Y-%m-%d-%H_%M)}-${calling}-${called});
		Set(monopt=nice -n 19 /usr/bin/lame -b 32  --silent "${DIR_RECORDS}${fname}.wav"  "${DIR_RECORDS}${fname}.mp3" && rm -f "${DIR_RECORDS}${fname}.wav" && chmod o+r "${DIR_RECORDS}${fname}.mp3");
		Set(CDR(filename)=${fname}.mp3);
		Set(CDR(realdst)=${called});
		Set(CDR(remoteip)=${CHANNEL(recvip)});
		MixMonitor(${DIR_RECORDS}${fname}.wav,b,${monopt});
   } else if ("${RECORDING}" = "2") {
 		Set(fname=${UNIQUEID}-${STRFTIME(${EPOCH},,%Y-%m-%d-%H_%M)}-${calling}-${called});
		Set(CDR(filename)=${fname}.wav);
		Set(CDR(realdst)=${called});
		Set(CDR(remoteip)=${CHANNEL(recvip)});
		MixMonitor(${DIR_RECORDS}${fname}.wav,b);  
   }
   return;
};

Пример вызова макроса:
context internal {
	_X. => {
		&recording(${CALLERID(num)},${EXTEN});
		Dial(SIP/${EXTEN},60);
		Hangup();
	};
};

==
 Для extensions.conf
==

; MixMonitor
[macro-recording]
exten => s,1,GoToIf($["${RECORDING}" = "1"]?mp3:no)
exten => s,n,GoToIf($["${RECORDING}" = "2"]?wav:no)
exten => s,n(mp3),Set(fname=${UNIQUEID}-${STRFTIME(${EPOCH},,%Y-%m-%d-%H_%M)}-${ARG1}-${ARG2});
exten => s,n,Set(monopt=nice -n 19 /usr/bin/lame -b 32  --silent "${DIR_RECORDS}${fname}.wav"  "${DIR_RECORDS}${fname}.mp3" && rm -f "${DIR_RECORDS}${fname}.wav" && chmod o+r "${DIR_RECORDS}${fname}.mp3");
exten => s,n,Set(CDR(filename)=${fname}.mp3);
exten => s,n,Set(CDR(realdst)=${ARG2});
exten => s,n,Set(CDR(remoteip)=${CHANNEL(recvip)});
exten => s,n,MixMonitor(${DIR_RECORDS}${fname}.wav,b,${monopt});
exten => s,n,Goto(no);
exten => s,n(wav),Set(fname=${UNIQUEID}-${STRFTIME(${EPOCH},,%Y-%m-%d-%H_%M)}-${ARG1}-${ARG2});
exten => s,n,Set(CDR(filename)=${fname}.wav);
exten => s,n,Set(CDR(realdst)=${ARG2});
exten => s,n,MixMonitor(${DIR_RECORDS}${fname}.wav,b);
exten => s,n,Goto(no);
exten => s,n(no),Verbose(Exit record);

Пример вызова макроса:
[internal]
exten => _X.,1,Macro(recording,${CALLERID(num)},${EXTEN})
exten => _X.,n,Dial(SIP/${EXTEN},60)
exten => _X.,n,Hangup()

====
 Дополнительно (необязательно). Если НЕ использовали файл импорта в базу "cdr_mysql.sql"
====

В Asterisk если используется макрос, то звонок совершается с экстеншеном s. Чтобы Номер назначения был действительным, а не s или ~~s~~, то сделаем следующее:
Через PHPMYADMIN или ADMINER (смотреть здесь: https://www.adminer.org). В таблицу нужно добавить новое поле "realdst" с типом "varchar" и размером "80". Теперь нужно добавить триггер на таблицу.
Для этого зайдем в Триггеры - Добавить триггер. Назначаем имя триггеру, остальное оставляем без изменений. В поле "Определение" вставляем текст ниже (то, что начинается на // - не вставлять):
//-- Начало --//
BEGIN
	IF ((NEW.dst = 's' OR NEW.dst = '~~s~~') AND NEW.realdst != '') THEN 
		SET NEW.dst = NEW.realdst;
	END IF;
END
//-- / Конец --//

Для того, чтобы в поле "realdst" записывался правильный Номер назначения, нужно отредактировать диалплан. Макрос выше в редактировании не нуждается.
В используемом у вас макросе необходимо добавить строчку, это только пример. Задайте правильные имена параметров (${number}, ${ARG1}).
==
 Для extensions.ael
==
Set(CDR(realdst)=${number});

==
 Для extensions.conf
==
exten => s,n,Set(CDR(realdst)=${ARG1});


===============================================
 Настройка папки со звонками
===============================================

Записи разговоров будут складываться в папку "/var/calls/" из примера выше.

Есть два варианта хранения файлов записей.
1. Все записи разговоров хранятся в одной папке.
2. Записи разговоров должны распределяться по папкам в соответствии с датой.

Также есть возможность настройки "отложенной конвертации записей разговоров".
Когда днем выполняется запись в формат WAV, а ночью необходимо по CRON запустить скрипт для преобразования файлов из WAV в MP3.
"Отложенную конвертацию записей разговоров" и распределение по папкам в соответствии с датой можно использовать вместе, а можно что-то одно.

За распределение файлов записей по папкам, преобразование файлов из WAV в MP3 отвечает скрипт "proc_records.sh" из папки docs.

==
 Для 2 варианта
==

Каждый день в 00.01 часов записи из папки "/var/calls/" по CRON должны распределяться по дате
в соответствующие папки.

Для распределения файлов по папкам в соответствии с датой нужно использовать скрипт "proc_records.sh" из папки docs.

Формат хранения записей (пример):
1. /var/calls/2014/2014-09/2014-09-29
2. /var/calls/2014/09/29

Настройки скрипта "proc_records.sh" для соответствующего формата:
1.
===
DIR_DST="/var/calls/$Y/$YM/$YMD/"
MOVE_BY_DATE=true
===

2.
===
DIR_DST="/var/calls/$Y/$M/$D/"
MOVE_BY_DATE=true
===

Настройки скрипта "proc_records.sh" для преобразования файлов из WAV в MP3:
--
За включение преобразования файлов из WAV в MP3 в скрипте отвечает переменная "CONV_TO_MP3". Необходимо установить ее значение в true или false.
Также можно настроить уровень вложенности поиска WAV файлов для их преобразования в переменной "DEPTH". Примеры значений для переменной
1 - это все файлы в папке /var/calls/
2 - это все файлы в /var/calls/, /var/calls/2017/ ...
3 - это все файлы в /var/calls/, /var/calls/2017/, /var/calls/2017/04 ...
...

==
 Для вариантов, когда Asterisk сам распределяет записи по папкам в соответствии с датой.
==

Если у вас Asterisk сам распределяет записи звонков по папкам в соответствии с датой, тогда необходимости запуска скрипта по CRON нет.
Если только у вас не настроена "отложенная конвертация записей разговоров"

Возможные форматы хранения записей:
1. /var/calls/2014/2014-09/2014-09-29
2. /var/calls/2014/09/29


===============================================
 Удаление старых записей звонков
===============================================

Для удаления старых записей звонков нужно использовать скрипт "proc_records.sh" из папки docs. В нем есть подробные комментарии по настройке.
Можно удалять только старые записи звонков, но будут оставаться пустые папки. Можно включить удаление пустых папок, тогда все пустые папки в директории,
которая задана в "DIR_SOURCE" будут удалены.

Чтобы все работало, для начала нужно определиться с форматом хранения записей звонков и правильно зададь значение переменной "DEPTH".
Если все файлы хранятся в одной папке, то значение переменной можно установить в "DEPTH=1". Если записи звонков распределяются по папкам в соответствии с датой, то
значение переменной можно установить в "DEPTH=3". Более подробно смотреть комментарии в скрипте.

Чтобы включить удаление старых записей звонков, в скрипте следует задать:
===
CLEAN_OLD=true
===

В переменной "CLEAN_OLD_AFTER" задается количество дней хранения файлов записей звонков. Например: Если задано "CLEAN_OLD_AFTER=365", то все файлы, старше 365 дней будут удалены.
Берется время изменения файла, расширение файла при удалении НЕ учитывается. Т.е. если в папке, например, есть "*.txt" файл старше этого срока, он также будет удален.

В переменной "CLEAN_OLD_EMPTYDIR" можно включить удаление пустых папок. Чтобы это включить, в скрипте следует задать:
===
CLEAN_OLD_EMPTYDIR=true
===

===============================================
 Настройка
===============================================

Все настройки прописаны в файле "inc/config/config.php" с подробными комментариями, тут не должно возникнуть сложностей.

Можно использовать "пользовательский" конфиг. Это значит, что будет использоваться альтернативный конфиг файл с настройками.
Для этого нужно создать еще один файл с конфигом, например: "inc/config/config-anotherconfig.php". Формат имени файла конфига: config-[уникальное_имя_конфига].php
Чтобы использовать созданный "пользовательский" конфиг - config-anotherconfig.php, нужно к адресу сайта с установленным скриптом (Например: http://example.com/cdr/index.php) добавить параметр ?config=anotherconfig.
Получается адрес: http://example.com/cdr/index.php?config=anotherconfig

===

Кратко:
1. Скачать ZIP архив с GitHub или выполнить git clone https://github.com/prog-it/Asterisk-CDR-Viewer-Mod.git
2. Распаковать или Перенести файлы в нужную папку на сервере
3. Настроить параметры в "inc/config/config.php"
4. Почти готово. Если необходим доступ только для определенных пользователей, то необходимо создать файл .htpasswd

htpasswd -c /path/to/.htpasswd admin

Пример конфига для Nginx:
===
location /path/to/script {
	auth_basic "CDR Viewer Mod";
	auth_basic_user_file /path/to/.htpasswd;
}
===

Пример конфига для Apache:
===
<Location "/path/to/script">
	AuthName "CDR Viewer Mod"
	AuthType Basic
	AuthUserFile /path/to/.htpasswd
	AuthGroupFile /dev/null
	require valid-user
</Location>
===

5. Прописать в конфиге скрипта имена пользователей в виде массива, которым разрешен доступ
===
'admins' => array(
	'admin1',
	'admin2',
	'admin3',
),
===

Если массив пустой, то разрешен доступ всем пользователям
===
'admins' => array(

),
===


===============================================
 Настройка тарифов на звонки
===============================================

Тарифы на звонки задаются в файле "my_callrates.csv" (inc/plugins/my_callrates.csv). Путь к этому файлу можно изменить в конфиге.

Формат задания тарифы в CSV файле:
Код_региона,стоимость_минуты[,Направление,Тип_тарификации,Доп_тариф]

-- То, что в фигурных скобках - необязательно

Пример для Мегафона, с поминутной тарификацией, стоимость первой минуты 90 коп., после первой минуты 10 коп (доп. тариф).
===
8922,0.90,Мегафон,m,0.10
===

Типы тарификации:
	s		- посекундно (нет доп. тарифа)
	m		- поминутно (есть доп. тариф)
	c		- за весь звонок (нет доп. тарифа)
	1m+s	- посекундно со 2 минуты (есть доп. тариф)
	30s+s	- посекундно после 30 секунды
	30s+6s	- округление интервалами до 6 сек. по истечении первых 30 сек. разговора (Skype Connect)

===============================================

Остальное можно прочитать с файле "Старый Readme.txt"	
	
	

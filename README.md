# Создание системы общего доступа к логам OS Linux на базе Rsyslog

Хорошо настроенная система логирования - очень ценный актив компании. Правда, чаще всего, даже важные системы логируются только с помощью встроенного журнала логов на "стоковых" настройках, так как большинство системных администратор считают приемлимым и удобным разбирать логи на уровне каждого отдельного хоста, затрачивая на эту простую операцию уйму времени. Это очень архаичный подход, так как он позволяет работать с логами на одном Хосте. Современные приложения все чаще и чаще имеют множество связанных логов распределенными между Хостами, что затрудняет сбор информации, однако не делает задачу невыполнимой.

Для решение этой задачи можно использовать инструмент под названием Rsyslog. Настройка центрального репозитория для сбора логов позволяет вам собрать все необходимое в одном месте, а с помощью знакомого визуализатора (Splunk, Grafana, ELK Stack, в примере рассматривается LogAnalyzer) легко и непринужденно наблюдать за экраном преждупреждений в реальном времени по всей организации или же настроить фильтры для отслеживания конкретных или специфических проблем между целевыми Хостами.

# Расставляем приоритеты
- Сервис-Ориентированная Архитектура (SOA) - приоритет отдаётся приложению и в случае деградации системы логирования происходит потеря логов, однако приложение продолжает работать (PROD среда)
- Антипаттерны SOA - полнота логов в ущерб работы приложения (DEV, TEST, PREPROD среды)

# Unix: погружение в Rsyslog для понимания предмета
При первом знакомстве с конфигурационным файлом Rsyslog'a может показаться что он достаточно сложен и не во всем очевиден, но если научиться его готовить - жить с ним и управлять становится достаточно легко. Rsyslog довольно гибкий и мощный а его конфигурационный файл такой какой он есть... 
Ключевыми моментами, которые слудет помнить, является то, что глобальные директивы среды (размер очереди, протоколы, бекенды, и т.д.) должны распологаться в начале файла. Все системные сообщения проходят его сверху вниз, пока не встретят требуемое правило роутинга. Если системное сообщение встретило правило роутинга которому оно удволетворяет - значит это правило будет выполнено, и если правило не отменит системное сообщене после завершения выполнение, то это сообщение пойдет дальше по файлу в поисках еще совпадений до тех пор, пока какое либо правило не отменит его, либо пока файл конфигурации не будет прочтен до конца.

# Доставка информации по TCP или UDP
Стоит не забывать про #Правило-1: Сеть ненадёжна
В окружении полноценной SOA (SOA, англ. service-oriented architecture, Сервис-Ориентированная Архитектура) и сетевого адаптера прендазначенного для оповещений по сети, то UDP может быть вполне себе неплохоим вариантом. С другой стороны, если вы используете виртуальные среды или blade решения, то доставка по TCP может сильно упростить задачу. Возможно, вам даже стоит использовать смешенный вариант.

# Чем проще винтик, тем сложнее он ломается
Говорить с системой надо на одном языке и понимать как происходит уровень логирования, поэтому сделайте это просто:
```
<facility>.<level>
```
facility - абстракция, назовём её "Слой", ниже по тексту станет понятно зачем нужно
level - уровень лога ошибки - DEBUG,INFO,CRITICAL etc

Широкое применение rsyslog в Enterprise дало ему несколько интересных конфигураций назовём их "Роли". Это таже часть приложения rsyslog, но с кастомными конфигурацими для кажой из объявленных ролей.

Роли Rsyslog:
- Клиенты
- Трансляторы
- Агрегаторы (размещение под капот Анализаторов)

Rsyslog достаточно гибкий в своей конфигурации, что позволяет использоват каждую роль не только как она обозначена, но и совмещенно с любой другой. Например, роль Транслятора помимо своей конфигурации может явялеться также и ролью Клиента, как и Агрегатор может быть не только Агрегатором и т.д.

# Unix: подготовка системы

syslog и rsyslog взаимо исключающие вещи, поэтому убедитесь, что syslog удален.
```
yum install rsyslog
```

В зависимости от верcии rsyslog(не поленитесь взять свежую версию) вы можете отредактировать `/etc/sysconfig/rsyslog` и указать опции для запуска
```
SYSLOGD_OPTIONS="-c 5"
```

Установите logrotate. Не будьте лентяями и не давайте своим лог файлам пухнуть на хостах, что негативно сказывается на стабильности и производительности системы. Управлйте ими с помощью logrotate. Несмотря на центральный репозиторий хранения/управления логами, для каждого хоста не менее важно иметь собственную область ведения логов (обычно это  `/var/log`), если центральный репозиторий не будет доступен (#Правило-1: Сеть ненадёжна). Также обратите внимание, что не все программное обеспечение использует syslog как протокол, некоторые умельцы пишут сразу в файлы журнала. Настройки logroate для ежедневной ротации логов и сохраняя все локальные логи на 30 дней.
```
yum install logrotate
``` 

# Unix: настройка клиента rsyslog на Хосте
Вы конечно попытайтесь, но скорее всего у вас не получится испльзовать один и тот же файл конфигурации для всех хостов. Банально потому что разные особености rsyslog настраиваются например для почтовых серверов и они будет совсем иными в случае LDAP сервера и унифицировать их достаточно проблемно. Так или иначе процентом 80% ваших хостов смогут использовать большую часть конфигурационного файла. А остальные будут иметь свои уникальные настройки. Смеритесь. :)

# Краткое описание конфигурации клиента:
- Поскольку в инфраструктуре множество виртуальных хостов, целесообразно использовать доставку по TCP для настройки клиента.
- Для снижения нагрузки на принимающую сторону(убрать "белый шум") в самом начале перечисляется набор правил, фильтрующий множество "неважных" сообщений syslog'a по критерию <facility>
- Затем следует набор правил, которые пытаются сохранить стандартное собщение syslog, записав его в локальные файлы логов. Идея здесь кроется в том, что если rsyslog не был бы установлен, то мы сохраняем стандартное поведение хоста. Данное поведение целесообразно  применять ко всем Хостам не зависимо от роли.
- Для особо "шумящих" приложений можно применить тонкую настройку. К примеру, приложение, которое логируется локально, однако отправлять в очередь rsyslog нет необходимости, что бы не забивать сеть и в конечном итоге Агрегатор.
- Далее стоит рассмотреть размер буфера очереди сообщений (в примере 5Мб), по заполнению которого сообщения syslog будут отброшены. ОСТОРОЖНО: очередь rsyslog никогда не дожна заполняться при нормальных операциях, однако если она заполняется и вы не начинаете отбрасывать сообщения syslog, тогда некоторые процессы могут зависать, так как их доступ к syslog заблокирован и вы можете оказаться в ситуации, когда вы не сможете зайти на хост, потому что rsyslog заблокировал процес входа в систему, в тот момент когда он попытался залогировать сообщение в `/var/log/secure`
- Наконец, любое сообщение syslog отправляется на сервер Транслятор rsyslog. В примере Клиент rsyslog настраивается для отправки своих сообщений syslog на ретрансляционный узел rsyslog, который живет в одной с ним сети. В примере нарочно не указан FQDN для имени хоста релея, дабы иметь возможность переиспользовать конфигурацию на других хостах, но это предполагает, что все имена FQDN-узлов ретрансляции rsyslog начинаются как `logs.<Domain.name>`.

**Типичная конфигурация клиента rsyslog.conf (обычно /etc/rsyslog.conf)**
```
#########################################
# Rsyslog conf file for CentOS/RHEL CLIENT role
#
# This setup:
# - logs to local files
# - logs to relay host
########################################

# NOTE: Please include the following option to /etc/sysconfig/rsyslog
# SYSLOGD_OPTIONS="-c 5"

# Turn on basic functionality
$ModLoad immark.so      # provides --MARK-- message capability
$MarkMessagePeriod 3600 # mark messages appear every 60 Minutes
$ModLoad imuxsock.so    # Unix sockets
$ModLoad imklog.so      # kernel logging

# Use TCP for network logging
$ModLoad imtcp

# Define the default template
$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat

# Note messages are applied to all rules sequentially unless explicitly discarded

# Messages to discard
:syslogfacility-text, isequal, "news" ~
:syslogfacility-text, isequal, "lpr"  ~
:syslogfacility-text, isequal, "uucp" ~
:syslogfacility-text, isequal, "crit" ~

# Log all kernel messages to the console.
# Logging much else clutters up the screen.
#kern.*                                        /dev/console

# Log to local files
$ActionQueueType              Direct      # Log directly to file
mail.*                                        /var/log/maillog
cron.*                                        /var/log/cron
authpriv,auth.*                               /var/log/secure
local7.*                                      /var/log/boot.log
*.info;mail,authpriv,auth,cron.none           /var/log/messages

# Prevent scheduled SCOM entries from cluttering the central log stores.
:msg, contains, "opt/microsoft/scx" ~

# Log to the central logging server. Note @@=TCP, @=UDP
$ActionQueueType              FixedArray  # Use a fixed block of memory.
$ActionQueuesize              10000       # Reserve 5Mb memory, each queue element is 512b
$ActionQueueDiscardMark       9500        # If the queue looks like filling, start discarding to not block ssh/login/etc.
$ActionQueueDiscardSeverity   0           # When in discarding mode discard everything.
$ActionQueueTimeoutEnqueue    0           # When in discarding mode do not enable throttling.
*.* @@logs:10514

# The end
```

# Unix: Настройка rsyslog как хоста Ретрансляции (relay)
Чаще всего эта конфигурация используется ввиде отдельного сервера в одной сети в Клиентами, где дисковая подсистема этого сервера играет роль дополнительного, резервного буфера. Его задача собрать все rsyslog сообщения локальной сети и переправить их на Агрегатора.

Особености настройки:
 - Увеличенный размер буффера до 3Гб 
 - Слушается оба протокола TCP\UDP, т.к. большинство устройст в сети, например роутеры и принтеры чаще всего используют UDP протокол для отправки syslog сообщений. Так что если вы хотите логировать все сообщения вам просто придется включить UDP протокол.
 - Далее фильтрация неважных сообщений по признаку facility
 - Отправка очищенного трафика сообщений на хост коллектора
 - Учитывая #Правило-1, сохранение всех syslog сообщения по стандартным путям журналов, на тот случай, если rsyslog не был установлен, то система выглядит в точности как будто так и было для этого хоста. О.о 

**Типичная конфигурация хоста Ретрансляции (relay)**
 ```
#########################################
# Rsyslog conf file for CentOS/RHEL RELAY role
#
# This setup:
# - logs localhost messages to local files
# - logs to collector host
########################################

# Turn on basic functionality
$ModLoad immark.so      # provides --MARK-- message capability
$ModLoad imuxsock.so    # Unix sockets
$ModLoad imklog.so      # kernel logging

# Buffer for when upstream server is unavailable
$WorkDirectory /usr/local/etc/rsyslog/work  # default location for work (spool) files
$ActionQueueType LinkedList                 # use asynchronous processing
$ActionQueueFileName srvfwd                 # set file name, also enables disk mode
$ActionQueueMaxDiskSpace 3G                 # Set disk queue limit
$ActionResumeRetryCount -1                  # infinite retries on insert failure
$ActionQueueSaveOnShutdown on               # save in-memory data if rsyslog shuts down

# Enable the receiving logging interface for smart devices
$ModLoad imtcp          # TCP interface
$InputTCPServerRun 10514
# Enable the receiving logging interface for dumb devices
$ModLoad imudp          # UDP interface
$UDPServerRun 514

# Define the default template
$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat

# Note messages are applied to all rules sequentially unless explicitly discarded

# Messages to discard
:syslogfacility-text, isequal, "news" ~
:syslogfacility-text, isequal, "lpr"  ~
:syslogfacility-text, isequal, "uucp" ~

# Log to the central logging server. Note @@=TCP, @=UDP
*.* @@logs.corp.mycompany.com:10514

# Log to local files if local message otherwise discard
if $hostname != $$myhostname and $hostname != 'localhost' then ~
$ActionQueueType Direct                       # Log directly to file
mail.*                                        /var/log/maillog
cron.*                                        /var/log/cron
authpriv,auth.*                               /var/log/secure
local7.*                                      /var/log/boot.log
*.info;mail,authpriv,auth,cron.none           /var/log/messages

# The end
 ```

# Unix настраиваем Аггрегатор (aka collector)
Чаще всего rsyslog collector живет где-то в секьюрном месте и по центру от всех хостов. Его задача собирать все сообщения со всех сетей предприятия (релеев, ретрансляторов) и хранить их централизованно. Коллектор может также быть сконфигурирован и как релей для локальной сети в которой он находится, так же как и клиентом для себя самого.

Конфигурация коллектора:
- В самом начале плагин баз данных (понадобится для LogAnalizer'a) 
- Размер буфера 3Гб
- Роль Хоста в качестве Транслятора - включить UDP 
- Фильтрация по критерию facility
- Запись только очень важных сообщений syslog в глобальную файловую зону
- \*Запись сообщений в центральную базу данных (в примере MySQL)
- Фокус с стандартным поведением syslog сообщений типа как будто бы и не было тут никогда rsyslog'a (как и в предыдущих конфигурациях)
- Обязательно настроить logrotate для ротирования логов стандартного поведения дабы предупредить их распухание. Глубина хранения данных 7 месяцев.

\* для записи сообщений в базу требуется установить плагин:
```
yum install rsyslog-mysql
```

**Типичная конфигурация коллектора rsyslog.conf (обычно `/etc/rsyslog.conf`)**
```
#########################################
# Rsyslog conf file for CentOS COLLECTOR role
#
# This setup:
# - logs localhost messages to local files
# - logs to central repository (database and global files)
########################################

# Turn on basic functionality
$ModLoad immark.so      # provides --MARK-- message capability
$ModLoad imuxsock.so    # Unix sockets
$ModLoad imklog.so      # kernel logging

# Allow logging to database
$ModLoad ommysql
# Buffer DB transactions
$WorkDirectory /usr/local/etc/rsyslog/work  # default location for work (spool) files
$ActionQueueType LinkedList                 # use asynchronous processing
$ActionQueueFileName dbq                    # set file name, also enables disk mode
$ActionQueueMaxDiskSpace 3G                 # Set disk queue limit
$ActionResumeRetryCount -1                  # infinite retries on insert failure
$ActionQueueSaveOnShutdown on               # save in-memory data if rsyslog shuts down

# Enable the receiving logging interface for smart devices
$ModLoad imtcp          # TCP interface
$InputTCPServerRun 10514
# Enable the receiving logging interface for dumb devices
$ModLoad imudp          # UDP interface
$UDPServerRun 514

# Define the default template
$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat

# Note messages are applied to all rules sequentially unless explicitly discarded

# Messages to discard
:syslogfacility-text, isequal, "news" ~
:syslogfacility-text, isequal, "lpr"  ~
:syslogfacility-text, isequal, "uucp" ~

# Log to global files
mail.*                                        /var/log/global/maillog
cron.*                                        /var/log/global/cron
authpriv,auth.*                               /var/log/global/secure
*.info;mail,authpriv,auth,cron.none           /var/log/global/messages

# Log all messages to a database server,db,user,pw
*.* :ommysql:sql-server1.corp.mycompany.com,Syslog,rsyslog,some_secret_database_password;

# Log to local files if local message otherwise discard
if $hostname != $$myhostname and $hostname != 'localhost' then ~
$ActionQueueType Direct                       # Log directly to file
mail.*                                        /var/log/maillog
cron.*                                        /var/log/cron
authpriv,auth.*                               /var/log/secure
local7.*                                      /var/log/boot.log
*.info;mail,authpriv,auth,cron.none           /var/log/messages

# The end
```


#Unix: Создание базы данных репозитория

Плагин rsyslog-mysql содержит инструкцию для создания чистой базы данных для rsyslog, которая также совместима с LogAnalizer'ом.

Для того, что бы база не распухла, можно использовать скрипт **mysql_trim_syslog_table.sh**
```
#!/bin/sh
#
# BSD License for mysql_trim_syslog_table.sh
# Copyright (c) 2013, Arthur Gouros
# All rights reserved.
# 
# Redistribution and use in source and binary forms, with or without 
# modification, are permitted provided that the following conditions are met:
# 
# - Redistributions of source code must retain the above copyright notice, 
#   this list of conditions and the following disclaimer.
# - Redistributions in binary form must reproduce the above copyright notice, 
#   this list of conditions and the following disclaimer in the documentation 
#   and/or other materials provided with the distribution.
# - Neither the name of Arthur Gouros nor the names of its contributors 
#   may be used to endorse or promote products derived from this software 
#   without specific prior written permission.
# 
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE 
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE 
# LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR 
# CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF 
# SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS 
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN 
# CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) 
# ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE 
# POSSIBILITY OF SUCH DAMAGE.
#
#
# Trim the size of the syslog table to a specified epoch.
#
#
# Author: Arthur Gouros 13/12/2011


# Globals
DAYS_TO_KEEP=61
DB_HOST="sql-server1.corp.mycompany.com"
DB_USERNAME="rsyslog"
DB_PASSWORD="some_secret_database_password"
SYSLOG_DATABASE="Syslog"
SYSLOG_TABLE="SystemEvents"

MYSQL="/usr/bin/mysql"
MYSQL_SCRIPT="/var/tmp/mysql_syslog_script.sql"
LOGFILE="/var/tmp/mysql_syslog_script.log"

get_mysql_date_epoch()
{
  # Generate a MySQL compatible date that is in the past.
  # FreeBSD
  # date -v-${DAYS_TO_KEEP}d '+%Y-%m-%d 00:00:00'
  # Linux
  date --date="${DAYS_TO_KEEP} days ago" '+%Y-%m-%d 00:00:00'
}

trim_syslog_table()
{
  # Create SQL query
  echo "DELETE FROM SystemEvents WHERE ReceivedAt < \"${1}\"" > ${MYSQL_SCRIPT}
  # Execute SQL query
  ${MYSQL} --skip-reconnect -h ${DB_HOST} -u ${DB_USERNAME} -p${DB_PASSWORD} ${SYSLOG_DATABASE} < ${MYSQL_SCRIPT} > ${LOGFILE} 2>&1
}

###################
# Main
###################
mysql_date_epoch="`get_mysql_date_epoch`"
trim_syslog_table "${mysql_date_epoch}"

exit 0
```

# Unix: Устанавливаем LogAnalizator
LogAnalizator - это простой веб приложение, которе позволяет вам наблюдать за всеми вашими логами в базеданных в реальном времени а также фильтровать их в зависимости от потребностей. LogAnalizator очень просто поставить и настроить. скачайте последную версию с официального сайта и следуйте инструкциям для настройки веб сервеа и подключения к SQL базе данных.

Отредактируйте config.php и добавьте туда что-то подобное:

```
$CFG['DefaultSourceID'] = 'Source1';

$CFG['Sources']['Source1']['ID'] = 'Source1';
$CFG['Sources']['Source1']['Name'] = 'rsyslog';
$CFG['Sources']['Source1']['ViewID'] = 'SYSLOG';
$CFG['Sources']['Source1']['SourceType'] = SOURCE_PDO;
$CFG['Sources']['Source1']['DBTableType'] = 'monitorware';
$CFG['Sources']['Source1']['DBType'] = DB_MYSQL;
$CFG['Sources']['Source1']['DBServer'] = 'sql-server1.corp.mycompany.com';
$CFG['Sources']['Source1']['DBName'] = 'Syslog';
$CFG['Sources']['Source1']['DBUser'] = 'loganalyzer';
$CFG['Sources']['Source1']['DBPassword'] = 'some_secret_password_with_readonly_access_to_SystemEvents_table';
$CFG['Sources']['Source1']['DBTableName'] = 'SystemEvents';
$CFG['Sources']['Source1']['DBEnableRowCounting'] = false;

```

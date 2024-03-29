В своих постах в этом году я обсуждал рефлекторную реакцию на различные типы ожидания, а в этом посте я собираюсь продолжить тему статистики ожидания и обсудить ожидание `PAGEIOLATCH_XX`. Я говорю «подождите», но на самом деле есть несколько видов `PAGEIOLATCH`ожидания, которые я обозначил XX в конце. Наиболее распространенные примеры:

- `PAGEIOLATCH_SH` – (SHare) ожидание загрузки страницы файла данных с диска в пул буферов, чтобы можно было прочитать ее содержимое.
- `PAGEIOLATCH_EX`или `PAGEIOLATCH_UP` – (EXclusive или UPdate) ожидание загрузки страницы файла данных с диска в буферный пул, чтобы можно было изменить ее содержимое.

Из них, безусловно, наиболее распространенным типом является `PAGEIOLATCH_SH`.

Когда этот тип ожидания является наиболее распространенным на сервере, рефлекторная реакция заключается в том, что в подсистеме ввода-вывода должна быть проблема, и поэтому расследование должно быть сосредоточено на этом.

Первое, что нужно сделать, это сравнить `PAGEIOLATCH_SH`количество и продолжительность ожидания с вашим базовым уровнем. Если объем ожиданий более или менее одинаков, но продолжительность каждого ожидания чтения стала намного больше, то меня беспокоит проблема с подсистемой ввода-вывода, например:

- Неправильная конфигурация/неисправность на уровне подсистемы ввода-вывода.
- Сетевая задержка
- Другая рабочая нагрузка ввода-вывода, вызывающая конкуренцию с нашей рабочей нагрузкой
- Конфигурация репликации/зеркалирования синхронной подсистемы ввода/вывода

По моему опыту, закономерность часто такова, что количество `PAGEIOLATCH_SH`ожиданий значительно увеличилось по сравнению с базовым (нормальным) значением, а продолжительность ожидания также увеличилась (т. е. время чтения ввода-вывода увеличилось), потому что большое количество операций чтения перегружает подсистему ввода-вывода. Это не проблема подсистемы ввода-вывода — SQL Server управляет большим количеством операций ввода-вывода, чем должно быть. Теперь необходимо переключиться на SQL Server, чтобы определить причину дополнительных операций ввода-вывода.

#### Причины большого количества операций чтения ввода-вывода

SQL Server имеет два типа чтения: логический ввод-вывод и физический ввод-вывод. Когда компоненту «Методы доступа» механизма хранения требуется доступ к странице, он запрашивает у пула буферов указатель на страницу в памяти (называемую логическим вводом-выводом), и пул буферов проверяет свои метаданные, чтобы убедиться, что эта страница уже в памяти.

Если страница находится в памяти, буферный пул передает методу доступа указатель, а ввод-вывод остается логическим вводом-выводом. Если страницы нет в памяти, буферный пул выдает «настоящий» ввод-вывод (называемый физическим вводом-выводом), и поток должен ждать его завершения, что приводит к ожиданию `PAGEIOLATCH_XX`. Как только ввод-вывод завершается и указатель становится доступным, поток получает уведомление и может продолжать работу.

В идеальном мире вся ваша рабочая нагрузка уместилась бы в памяти, и поэтому, как только пул буферов «разогреется» и удержит всю рабочую нагрузку, больше не требуется чтения, только запись обновленных данных. Однако это не идеальный мир, и у большинства из вас нет такой роскоши, поэтому некоторые чтения неизбежны. Пока количество чтений остается на уровне вашего базового уровня, проблем нет.

Когда внезапно и неожиданно требуется большое количество операций чтения, это признак значительного изменения либо рабочей нагрузки, либо объема памяти буферного пула, доступного для хранения копий страниц в памяти, либо того и другого.

Вот некоторые возможные основные причины (не исчерпывающий список):

- Внешняя нагрузка на память Windows на SQL Server, заставляющая диспетчер памяти уменьшать размер пула буферов.
- Запланируйте раздувание кеша, что приведет к заимствованию дополнительной памяти из пула буферов.
- План запроса, выполняющий сканирование таблицы/кластеризованного индекса (вместо поиска по индексу) из-за:
    - увеличение объема рабочей нагрузки
    - проблема сниффинга параметров
    - требуемый некластеризованный индекс, который был удален или изменен
    - неявное преобразование

Один шаблон для поиска, который предполагает, что сканирование таблицы/кластеризованного индекса является причиной, также видит большое количество `CXPACKET`ожиданий вместе с `PAGEIOLATCH_SH`ожиданиями. Это распространенный шаблон, указывающий на выполнение больших параллельных сканирований таблицы/кластеризованного индекса.

Во всех случаях вы можете посмотреть, какой план запроса вызывает ожидание, `PAGEIOLATCH_SH`используя `sys.dm_os_waiting_tasks`DMV и другие, и вы можете получить код для этого в моем блоге [здесь](https://www.sqlskills.com/blogs/paul/advanced-performance-troubleshooting-waits-latches-spinlocks/) . Если у вас есть сторонний инструмент мониторинга, он может помочь вам определить виновника, не пачкая рук.

#### Пример рабочего процесса с SQL Sentry и Plan Explorer

В простом (очевидно надуманном) примере предположим, что я нахожусь в клиентской системе, использую [набор инструментов SQL Sentry,](https://sentryone.com/platform) и вижу всплеск ожидания ввода-вывода в панели мониторинга [SQL Sentry](https://sentryone.com/platform/sql-server-performance-monitoring) , как показано ниже:

![[pa-ws.png]]

Я решил исследовать, щелкнув правой кнопкой мыши выбранный интервал времени во время всплеска, а затем перейдя к представлению Top SQL, которое покажет мне самые дорогие выполненные запросы:

![[PR_KJ_DashboardInvestigate.png]]

В этом представлении я могу увидеть, какие длительные запросы или запросы с большим объемом операций ввода-вывода выполнялись в момент возникновения всплеска, а затем выбрать детализацию их планов запросов (в данном случае имеется только один длительный запрос, который длился почти минуту):

![[PR_KJ_TopSQLQuery.png]]

Если я смотрю на план в клиенте SQL Sentry или открываю его в [SQL Sentry Plan Explorer](https://sentryone.com/plan-explorer) , я сразу вижу множество проблем. Количество чтений, необходимых для возврата 7 строк, кажется слишком большим, дельта между оценочными и фактическими строками велика, и план показывает, что сканирование индекса происходит там, где я ожидал поиска:

![[PR_KJ_PlanWarnings.png]]

Причина всего этого выделена в предупреждении оператора `SELECT`: **Это неявное преобразование!**

Неявные преобразования — это коварная проблема, вызванная несоответствием между типом данных предиката поиска и типом данных искомого столбца или вычислением, выполняемым для столбца таблицы, а не для предиката поиска. В любом случае SQL Server не может использовать поиск по индексу для столбца таблицы и вместо этого должен использовать сканирование.

Это может произойти в, казалось бы, невинном коде, и типичным примером является использование вычисления даты. Если у вас есть таблица, в которой хранится возраст клиентов, и вы хотите выполнить расчет, чтобы узнать, сколько из них на сегодняшний день старше 21 года, вы можете написать такой код:

```sql
WHERE DATEADD (YEAR, 21, [MyTable].[BirthDate]) <= @today;
```

С этим кодом вычисление выполняется в столбце таблицы, поэтому поиск по индексу не может использоваться, что приводит к недоступному для поиска выражению (технически известному как выражение, не допускающее SARG) и сканированию таблицы/кластеризованного индекса. Это можно решить, перенеся вычисление на другую сторону оператора:

```sql
WHERE [MyTable].[BirthDate] <= DATEADD (YEAR, -21, @today);
```

С точки зрения того, когда для простого сравнения столбцов требуется преобразование типа данных, которое может вызвать неявное преобразование, мой коллега Джонатан Кехайяс написал отличный [пост в блоге](https://www.sqlskills.com/blogs/jonathan/implicit-conversions-that-cause-index-scans/) , в котором сравниваются все комбинации типов данных и отмечается, когда потребуется неявное преобразование.

#### Краткое содержание

Не попадайтесь в ловушку, думая, что чрезмерное `PAGEIOLATCH_XX`ожидание вызвано подсистемой ввода-вывода. По моему опыту, они обычно вызваны чем-то, связанным с SQL Server, и именно с этого я бы начал устранение неполадок.

Что касается общей статистики ожидания, вы можете найти дополнительную информацию об ее использовании для устранения неполадок с производительностью в:

- Серия сообщений в моем блоге SQLskills, начиная со [статистики ожидания, или, пожалуйста, скажите мне, где это болит](https://www.sqlskills.com/blogs/paul/wait-statistics-or-please-tell-me-where-it-hurts/)
- Моя библиотека типов ожидания и классов защелки [здесь](https://www.sqlskills.com/help/waits/)
- Мой интерактивный учебный курс Pluralsight SQL Server: [устранение неполадок производительности с помощью статистики ожидания](http://www.pluralsight.com/training/Courses/TableOfContents/sqlserver-waits)
- [Часовой SQL](https://sentryone.com/platform/sql-server-performance-monitoring)

В следующей статье этой серии я расскажу о другом типе ожидания, который часто вызывает рефлекторные реакции. А пока удачного устранения неполадок!
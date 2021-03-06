Как удалить OSD
---------------

Для примера будем удалять `osd.42`.

#. ``ceph osd out osd.42``. Эта команда заставит Ceph перенести все данные с
   этого диска на другие диски без даже вре́менного понижения количества реплик.
#. Мониторить ``ceph osd safe-to-destroy``.
#. На ноде: ``sudo systemctl stop ceph-osd@42``.
#. ``ceph osd purge osd.42``.

Дальнейшие операцию производятся на ноде под правами root:

#. Посмотреть и запомнить вывод ``lsblk -f``. Пригодится далее для ``wipefs``.
#. Посмотреть и запомнить ``readlink -f /var/lib/ceph/osd/ceph-42/*``
   (Пригодится для удаления журнального раздела если он выносной).
#. ``umount /var/lib/ceph/osd/ceph-42``.
#. ``rmdir /var/lib/ceph/osd/ceph-42``.
#. ``wipefs -a /dev/{data-disk-partition(s)}``. см. сохранённый вывод ``lsblk``.
#. ``wipefs -a /dev/{data-disk}``. см. сохранённый вывод ``lsblk``.
#. Если выносной журнал/бд: ``fdisk /dev/{journal-disk}``, удалить
   соответствующий раздел. Современный fdisk умеет работать с GPT.
   какой именно раздел -- см. сохранённый вывод ``readlink``.
#. ``partprobe /dev/{journal-disk}``. fdisk не умеет говорить ядру о применении
   измененной таблицы разделов если диск используется (например, под другие
   журналы/бд на этом же диске. Эта тулза из комплекта parted.
#. Перед извлечением диска физически на лету выполнить:
   ``echo 1 > /sys/block/{data-disk}/device/delete``.


.. include:: upgrade-to-luminous.rst


CephFS
------

Хранить образы виртуалок на CephFS -- полный маразм.

Типичные крутилки/инструкции
----------------------------

* Минимизация влияния бекфиллов и рекавери на IO

  #. Дефолтные настройки параметров МЕНЯЮТСЯ в разных версиях Ceph, поэтому если
     интересное для вас значение совпадает с дефолтом -- пропишите его явно.
  #. Замедляя бекфиллы и рекавери вы увеличиваете риск потери данных, потому что
     во время этих процессов может умереть OSD.
  #. Чем больше значение приоритета тем больше Ceph уделяет времени на выполнение
     данной операции. В команде nice  и в управлении дисковыми приоритетами в
     Linux -- наоборот (чем больше число -- тем меньше времени уделяется).
  #. TODO: там есть ещё параметры ограничивающие по LoadAverage...

  .. code::

     osd_max_backfills = 1

     osd_recovery_max_active = 1
     osd_recovery_threads = 1
     osd_recovery_op_priority = 1
     osd_client_op_priority = 63

     osd_scrub_priority = 1
     osd_scrub_begin_hour = 1
     osd_scrub_end_hour = 7
     osd_scrub_during_recovery = false

* Reweight by utilisation + новые ребалансер в Люминоусе

.. include:: bluestore.rst

Как работает
------------
* Tiering vs bcache vs dm-cache + инструкции по дмкешу.
* почему дедупликация крайне затруднена в архитектуре Ceph
*
  .. _filestore_wal:

  В filestore всё полностью пишется в журнал. WAL используется как
  writeback-cache по сути. Один write в rados превращается в два сисколла write
  -- один в журнал (с синком) и один в основное хранилище. Основное хранилище фсинкается
  время от времени. Запись в журнал линейная, а в основное хранилище рандомная. При записи
  в хранилище поможет параллельность которую может диск (например, NCQ). При записи в журнал
  параллельность не используется, поэтому диск под журнал для файлстора надо бенчить именно
  так: wal_bench_.

* при выносе журнала или БД на отдельный диск теряется возможность перевставлять диски в
  другой нод. При старте ОСД (бай дефолт есть параметр) обновляет себя в крушмапе.
* При потере журнала вседиски на него зааттаченные превращаются в труху
* При потере данных всех мониторов теряется весь кластер.
* Нужно использовать именно три реплики потому что если две - то при скраб ерроре не понятно
  кому верить
* запись и чтение делается исключительно с мастера в актинг сете. При записи данные
  отправляются на мастер осд а он по кластер-сети  отправляет параллельно на два слейва.
  on_safe-коллбэк клиента вызывается когда данные записались в WAL на всех репликах.
  Должидания прописывания в основное хранилище в принципе нет. Есть коллбэк когда данные
  есть в памяти на всех трёх репликах.
* бкеш врёт относительно ротейшионал и цеф использует не те настройки. Бкеш writeback
  (кеширование записи) не нужен потому что с файлстором это делается через WAL, а с
  блюстором есть опция по записи даже больших запросов в БД которую нужно вынести на ССД.
  С чтением тоже не нужен потому что:

  #. виртуалки с рбд и так не плохо кешируют то что уже читали

  #. запись потребляет в 3 раза больше иопсов (размер пула=3). а на самом деле ещё больше по
     причине двойной записи и даже ещё больше если это файлстор. Чтение требует один-в-один.
     поэтому цеф на чтение хорош.

  #. Нормальный кеш делает через тиеринг в цефе (но это не точно).

* Описание цифр в ceph -s. откуда берутся цифры и что они означают.
* Как посчитать реальную вместимость кластера. мин. загруженность осд.
* сколько должен давать кластер иопсов и мегабайтов в секунду на чтение и на запись.
  какие паттерны использования и параллельность.
* ceph-deploy требует GPT. Размер журнала и БД надо выставлять.
* Инструкцию по перемещению журнала на другое место для файлстора. и факт что это невозможно для блюстора.
* понимание, что ИО одного и того же обжекта (или 4-мб-блока в рбд) никак не распараллеливается магически.
  и оно будет равно иопсам журнала или осн. хранилища.
* почему мелкие объекты плохо в радосе и большие тоже плохо.
* почему при убирании диска надо сначала сделать цеф осд аут, а не просто вырубить диск.
* для более быстрой перезагрузки используйте kexec. systemctl kexec. однако с кривым железом может
  не работать (сетёвки и рейды/хба).
* https://habrahabr.ru/post/313644/
* почему size=3 min_size=1 (size 3/1) моежт привести к проблемам.
* Каждая пг устанавливает свой кворум таким образом много
* ссылка на калькулятор количества ПГ. почему много пг плохо и мало пг тоже плохо.

  * http://ceph.com/pgcalc

  * если мало - то неравномерность, потенциально не все осд могут быть заюзаны.

  * если много - юсадж памяти, перегрузка сети


.. include:: bench.rst


Мониторинг
----------

* два вида экспортеров под прометеус
* мониторить температуры, свап, иопсы (латенси) дисков

Сеть
----

* что бек сеть надо точно 10 гигабит. привести расчёты.
* Отключить оффлоадинг (и как проверить помогло ли) - меряем RTT внутри TCP.
* джамбофреймы могут помочь но не особо. сложности со свичами обычно.
* мониторить состояние линка. оно иногда самопроизвольно падает с гигабита на 100 мегабит.
* Тюнинг TCP/IP - отключать контрак

Диски
-----

* Не имеет никакого смысла использовать рэйды как хранилище для Ceph. Здесь
  имеется в виду какой-либо способ программного или аппаратного объединения
  дисков в один виртуальный. Потенциальные проблемы:

  * Опасность обмана команд по сбросу кеша. Например, включенный Writeback на
    аппартаном RAID без BBU.

  * Программный RAID (mdadm, зеркало) ПОВРЕЖДАЕТ данные при записи в режиме
    O_DIRECT если в процессе записи страница меняется в параллельном потоке.
    В этом случае ПОДТВЕРЖДЁННЫЕ данные будут различаться в половинках
    зеркального рэйда. При следующем (scrub?) рэйда будут проблемы.
    TODO: Нужен proof.

  * Программные рэйды не защищают от сбоя питания -- да, разумеется вышестоящие
    FS/БД должны быть готовы к повреждению неподтверждённых данных, но при
    проверке (scrub?) различие данных на репликах приведёт к проблемам.

  * Во время смерти диска RAID находится в состоянии degraded пока не добавят
    новый диск. Либо нужен spare-диск который в случае с Ceph глупо не
    использовать. Degraded RAID внезапно для Ceph будет давать худшие
    характеристики пока не восстановится. RAID не знает какие данные нужны а
    какие -- нет, поэтому процесс восстановления реплик -- долгий --
    синхронизирует мусор либо нули.

  * Для RAID нужны диски одинакового размера. Для Ceph это не требуется.

  * Аппаратные рэйды нужно отдельно мониторить и администрировать.

  * Зеркало не нужно потому что Ceph сам сделает столько реплик сколько
    требуется. Страйпинг не нужен потому что повышение производительности
    делается другими способами (с помощью SSD). Raid 5,6 в случае дегрейда
    причиняет боль.

  * В общем и целом, Ceph можно рассматривать как огромный распределённый RAID.
    Зачем делать RAID состоящий из RAID не понятно.

* Акустик, хпа, паверсейвинг, настроить автотесты по смарту.
* отдискардить ссд перед использованием.
* fstrim -v -a (filestore on ssd), blkdiscard on LVM/Partition.
* мониторить смарт
* как бенчить - ссд и разного рода коммерческий обман. деградация изза недискарда - надо дать
  продыхнуть, некоторое количество быстрых ячеек и тиринг на них. суперкапазиторы. перегрев ссд и тхроттлинг
* бенчмаркинг несколько дисков одновременно ибо контроллеры.
* на ссд обновлять прошивки критично важно. ещё про блеклисты в ядрах насчёт багов.
* дискард на них медленный, поэтому лучше оставить продискарденную область и этого достаточно.
* жеоательно не ставить одинаковые диски с одинаковым юсаджем - ибо умрут скорее всего одновременно
  ибо нагрузка примерно одинаковая.
* Диск шедулеры
* имхо магнитные сас-диски не нужны. их возможности не будут задействованы для получения преимущества
  перед сата. Сата 12 гбит для магнитных дисков не нужен. Для магнитных (7200 оборотов)
  даже сата2 (3 гбит ~ 300 МБ/сек) хватит.
* убедиться что диски подключены как сата6.
* чего ожидать от бенчмаркинга. реальная таблица с реальными моделями.
* при бенчмаркинге ссд может оказаться что уперлись в контроллер а не в диск.
* NCQ, 7200 и 180 IOPS. 32 для большей возможности выбора в диске. Также как посчитать теоретические иопсы.

Процессоры и память
-------------------

* Для Ceph-нодов требуется (а не просто рекомендуется) память с ECC. Сбой в памяти
  мастер-OSD в acting set приведёт к необнаруживаемому повреждению данных
  даже если это BlueStore со своим CRC32c. Данные могут повредиться до
  подсчёта CRC32 и распространиться по slave-OSD.

  Немного близкая тема и про клиентов. Если данные испорчены до подсчёта
  CRC32 в рамках протокола мессенджера Ceph, то они будут повреждены и это
  не обнаружится.

* CPU Governor & powersave mode. Отличная статья в арче:
  https://wiki.archlinux.org/index.php/CPU_frequency_scaling

* CRC32 аппаратное в блюсторе (и в месенджере не с блюстором?)
* Гипертрединг не нужен. Потому что это просто доп-набор регистров.
  В Ceph нет CPU-bound задач. Есть CRC32, но оно реализуется через спец команду
  в sse4.3, а такой блок ЕМНИП один на ядро. Однако, при сжатии в блюсторе может иметь
  значение.
* ramspeed = ramsmp
* cpuburn
* i7z, powertop
* cpupower frequency-info, how to set governor (+permanently)
* grub + nopti + performance + luacode + meltdown

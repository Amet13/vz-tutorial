# Руководство по созданию и управлению контейнерами и виртуальными машинами на базе Virtuozzo

## <a name='toc'></a>Содержание
1. [Введение в виртуализацию](#intro)
  - [Эмуляция оборудования](#emulation)
  - [Полная виртуализация](#full-virt)
  - [Паравиртуализация](#paravirt)
  - [Контейнерная виртуализация (виртуализация уровня ОС)](#cont-virt)
  - [Virtuozzo — объединение технологий виртуализации уровня ОС и полной виртуализации](#vz7)
2. [Краткая история проекта Virtuozzo](#history)
3. [Установка и подготовительные действия](#install)
  - [Установка Virtuozzo с помощью ISO-образа (bare-metal installation)](#bare-metal)
  - [Установка Virtuozzo на заранее установленный дистрибутив](#exists-distro)
  - [Подготовительные действия](#prepare)
4. [Управление шаблонами гостевых ОС и приложений контейнеров](#templates-ct)
  - [Шаблоны гостевых ОС](#guest-os)
  - [Шаблоны приложений](#app-templ)
5. [Создание и настройка контейнеров](#containers)
  - [Конфигурационные файлы](#configs)
  - [Создание контейнера](#create-ct)
  - [Настройка контейнера](#setup-ct)
  - [Запуск и вход](#run-enter)
6. [Управление контейнерами](#management-ct)
  - [Управление состоянием контейнера](#status-ct)
  - [Переустановка контейнера](#reinstall-ct)
  - [Клонирование контейнера](#clone-ct)
  - [Запуск команд в контейнере с хост-ноды](#run-commands)
  - [Расширенная информация о контейнерах](#extra-info)
7. [Управление ресурсами контейнеров](#resources-ct)
  - [Дисковые квоты](#quota)
  - [Процессор](#cpu)
  - [Операции ввода/вывода](#io)
  - [Память](#memory)
  - [Мониторинг ресурсов](#monitoring)
8. [Миграция контейнеров](#migration-ct)
9. [Проброс устройств в контейнеры](#forward-dev-ct)
  - [TUN/TAP](#tun-tap)
  - [FUSE](#fuse)
10. [Работа с виртуальными машинами](#vmachines)
  - [Создание и запуск ВМ](#create-vm)
  - [Дополнения гостевой ОС](#guest-tools)
11. [Планы Virtuozzo](#roadmap)
12. [Ссылки](#links)
13. [TODO](#todo)
14. [Лицензия](#license)

## [⬆](#toc) <a name='intro'></a>Введение в виртуализацию
Виртуализация — предоставление наборов вычислительных ресурсов или их логического объединения, абстрагированное от аппаратной реализации, и обеспечивающее изоляцию вычислительных процессов.

Виртуализацию можно использовать в:
* консолидации серверов (позволяет мигрировать с физических серверов на виртуальные, тем самым увеличивается коэффициент использования аппаратуры, что позволяет существенно сэкономить на аппаратуре, электроэнергии и обслуживании)
* разработке и тестировании приложений (возможность одновременно запускать несколько различных ОС, это удобно при разработке кроссплатформенного ПО, таким образом значительно повышается качество, скорость разработки и тестирования приложений)
* бизнесе (использование виртуализации в бизнесе растет с каждым днем и постоянно находятся новые способы применения этой технологии, например, возможность безболезненно сделать снапшот)
* организации виртуальных рабочих станций (так называемых "тонких клиентов")

*Общая схема взаимодействия виртуализации с аппаратурой и программным обеспечением*
![Общая схема взаимодействия виртуализации с аппаратурой и программным обеспечением](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/virt-scheme.png)

Понятие виртуализации можно условно разделить на две категории:
* виртуализация платформ, продуктом этого вида виртуализации являются виртуальные машины
* виртуализация ресурсов преследует целью комбинирование или упрощение представления аппаратных ресурсов для пользователя и получение неких пользовательских абстракций оборудования, пространств имен, сетей

Взаимодействие приложений и операционной системы (ОС) с аппаратным обеспечением осуществляется через абстрагированный слой виртуализации.

Существует несколько подходов организации виртуализации:
* эмуляция оборудования (QEMU, Bochs, Dynamips)
* полная виртуализация (KVM, HyperV, VirtualBox)
* паравиртуализация (Xen, L4, Trango)
* виртуализация уровня ОС (LXC, Virtuozzo, Jails, Solaris Zones)

### <a name='emulation'></a>Эмуляция оборудования
Эмуляция аппаратных средств является одним из самых сложных методов виртуализации.
Главной проблемой при эмуляции аппаратных средств является низкая скорость работы, в связи с тем, что каждая команда моделируется на основных аппаратных средствах.

В процессе эмуляции оборудования используется механизм динамической трансляции, то есть каждая из инструкций эмулируемой платформы заменяется на заранее подготовленный фрагмент инструкций физического процессора.

Эмуляция позволяет использовать виртуализированные аппаратные средства еще до выхода реальных.
Например, управление неизмененной ОС, предназначенной для PowerPC на системе с ARM процессором.

*Эмуляция оборудования моделирует аппаратные средства*
![Схема эмуляции оборудования](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/emulation.png)

### <a name='full-virt'></a>Полная виртуализация
Полная виртуализация использует гипервизор, который осуществляет связь между гостевой ОС и аппаратными средствами физического сервера.
В связи с тем, что вся работа с гостевой операционной системой проходит через гипервизор, скорость работы данного типа виртуализации ниже чем в случае прямого взаимодействия с аппаратурой.
Основным преимуществом является то, что в ОС не вносятся никакие изменения, единственное ограничение — операционная система должна поддерживать основные аппаратные средства.

*Полная виртуализация использует гипервизор*
![Схема полной виртуализации](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/full-virt.png)

Полная виртуализация возможна исключительно при условии правильной комбинации оборудования и программного обеспечения.

### <a name='paravirt'></a>Паравиртуализация
Паравиртуализация имеет некоторые сходства с полной виртуализацией.
Этот метод использует гипервизор для разделения доступа к основным аппаратным средствам, но объединяет код, касающийся виртуализации, в непосредственно операционную систему, поэтому недостатком метода является то, что гостевая ОС должна быть изменена для гипервизора.
Но паравиртуализация существенно быстрее полной виртуализации, скорость работы виртуальной машины приближена к скорости реальной, это осуществляется за счет отсутствия эмуляции аппаратуры и учета существования гипервизора при выполнении системных вызовов в коде ядра.
Вместо привилегированных операций совершаются гипервызовы обращения ядра гостевой ОС к гипервизору с просьбой о выполнении операции.

*Паравиртуализация разделяет процесс с гостевой ОС*
![Схема паравиртуализации](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/paravirt.png)

В паравиртуальном режиме (PV) оборудование не эмулируется, и гостевая операционная система должна быть специальным образом модифицирована для работы в таком окружении.
Начиная с версии 3.0, ядро Linux поддерживает запуск в паравиртуальном режиме без перекомпиляции со сторонними патчами.
Преимущество режима паравиртуализации состоит в том, что он не требует поддержки аппаратной виртуализации со стороны процессора, а также не тратит вычислительные ресурсы для эмуляции оборудования на шине PCI.

Режим аппаратной виртуализации (HVM), который появился в Xen, начиная с версии 3.0 гипервизора требует поддержки со стороны оборудования.
В этом режиме для эмуляции виртуальных устройств используется QEMU, который весьма медлителен несмотря на паравиртуальные драйвера.
Однако со временем поддержка аппаратной виртуализации в оборудовании получила настолько широкое распространение, что используется даже в современных процессорах лэптопов.

### <a name='cont-virt'></a>Контейнерная виртуализация (виртуализация уровня ОС)
Виртуализация уровня операционной системы отличается от других.
Она использует технику, при которой сервера виртуализируются непосредственно над ОС.
Недостатком метода является то, что поддерживается одна единственная операционная система на физическом сервере, которая изолирует контейнеры друг от друга.
Преимуществом виртуализации уровня ОС является "родная" производительность.

*Виртуализация уровня ОС изолирует серверы*
![Схема виртуализации уровня ОС](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/cont-virt.png)

Виртуализация уровня ОС — метод виртуализации, при котором ядро операционной системы поддерживает несколько изолированных экземпляров пространства пользователя (контейнеров) вместо одного.
С точки зрения пользователя эти экземпляры полностью идентичны реальному серверу.
Для систем на базе UNIX эта технология может рассматриваться как улучшенная реализация механизма chroot.
Ядро обеспечивает полную изолированность контейнеров, поэтому программы из разных контейнеров не могут воздействовать друг на друга.

### <a name='vz7'></a>Virtuozzo — объединение технологий виртуализации уровня ОС и полной виртуализации
Virtuozzo позволяет создавать множество защищенных, изолированных друг от друга контейнеров на одном узле.
Помимо этого разрабатываются возможности по созданию виртуальных машин на базе QEMU/KVM.
Управление контейнерами и виртуальными машинами происходит с помощью специализированных утилит.

*Архитектура Virtuozzo 7*
![Архитектура Virtuozzo 7](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vz7-architect.png)

Каждый контейнер ведет себя так же, как автономный сервер и имеет собственные файлы, процессы, сеть (IP адреса, правила маршрутизации).
В отличие от KVM или Xen, Virtuozzo использует одно ядро, которое является общим для всех виртуальных сред.

Контейнеры можно разделить на две составляющие:
* ядро (namespaces, cgroups, CRIU, ploop, vcmmd...)
* пользовательские утилиты (prlctl, vzctl, vzpkg, vzlist, vzdump...)

Namespaces — пространства имен.
Это механизм ядра, который позволяет изолировать процессы друг от друга. Изоляция может быть выполнена в шести контекстах (пространствах имен):
* mount — предоставляет процессам собственную иерархию файловой системы и изолирует ее от других таких же иерархий по аналогии с chroot
* PID — изолирует идентификаторы процессов (PID) одного пространства имен от процессов с такими же идентификаторами другого пространства
* network — предоставляет отдельным процессам логически изолированный от других стек сетевых протоколов, сетевой интерфейс, IP-адрес, таблицу маршрутизации, ARP и прочие реквизиты
* IPC — обеспечивает разделяемую память и взаимодействие между процессами
* UTS — изоляция идентификаторов узла, таких как имя хоста (hostname) и домена (domain)
* user — позволяет иметь один и тот же набор пользователей и групп в рамках разных пространств имен, в каждом контейнере могут быть свой
root и любые другие пользователи и группы

CGroups (Control Groups) — позволяет ограничить аппаратные ресурсы некоторого набора процессов.
Под аппаратными ресурсами подразумеваются: процессорное время, память, дисковая и сетевая подсистемы.
Набор или группа процессов могут быть определены различными критериями.
Например, это может быть целая иерархия процессов, получающая все лимиты родительского процесса.
Кроме этого возможен подсчет расходуемых группой ресурсов, заморозка (freezing) групп, создание контрольных точек (checkpointing) и их перезагрузка.
Для управления этим полезным механизмом существует специальная библиотека libcgroups, в состав которой входят такие утилиты, как cgcreate, cgexec и некоторые другие.

CRIU (Checkpoint/Restore In Userspace) — обеспечивает создание контрольной точки для произвольного приложения, а также возобновления работы приложения с этой точки.
Основной целью CRIU является поддержка миграции контейнеров.
Уже поддерживаются такие объекты как процессы, память приложений, открытые файлы, конвейеры, IPC сокеты, TCP/IP и UDP сокеты, таймеры, сигналы, терминалы, файловые дескрипторы.
В разработке также находится миграция TCP соединений.

VCMM (Virtuozzo containers memory management) — сервис управления механизмом memory cgroups в пространстве пользователя.
Менеджер памяти 4 поколения управляет memory cgroups, который присутствует в ванильном ядре, поэтому не требует сторонних патчей со стороны Virtuozzo.

Проведенные тестирования показывают, что OpenVZ (Virtuozzo) является одним из наиболее актуальных решений на рынке виртуализации, так как показывает внушительные результаты в различных тестированиях.

*График времени отклика системы*
![Время отклика системы](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/response-time.png)

На графике времени отклика системы можно наблюдать результаты трех тестов — с нагрузкой на систему и виртуальную машину, без нагрузки, нагрузкой только на ВМ.
Во всех тестах OpenVZ показал результаты наименьшего времени отклика, в то время, когда ESXi и Hyper-V показывают оверхед 700-3000%, когда у OpenVZ всего 1-3%.

*График пропускной способности сети*
![Пропускная способность сети](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/network.png)

На втором графике — результаты тестирования пропускной способности сети.
На графике можно наблюдать, что OpenVZ обеспечивает практическую нативную пропускную способность 10Gb сети (9.7Gbit/sec отправка и 9.87Gbit/sec прием).

## [⬆](#toc) <a name='history'></a>Краткая история проекта Virtuozzo
В 1999 году возникла идея создания Linux-контейнеров, а уже в 2002 году компания SWsoft представила первый релиз коммерческой версии Virtuozzo. В том же 2002 году появились первые клиенты в Кремниевой долине.

В 2004 году выпуск Virtuozzo для Windows.
В 2005 году было принято решение о разделении Virtuozzo на два отдельных проекта, свободный OpenVZ (под лицензией GNU GPL) и проприетарный Virtuozzo.

В 2006 году OpenVZ стал доступен для Debian Linux, переход к ядру RHEL 4.
В 2007 году портирован на RHEL 5.

В 2011 году появилась идея создания проекта CRIU, OpenVZ портирован на RHEL 6.
В 2012 году стала доступна CRIU v0.1.

В конце 2014 года компания Odin анонсировала открытие кодовой базы Parallels Cloud Server и объединение ее с открытой OpenVZ.

В апреле 2015 года был открыт репозиторий с ядром RHEL 7 (3.10), в мае были открыты исходные коды пользовательских утилит, а в июне выложены тестовые сборки ISO-образов и RPM-пакеты.

## [⬆](#toc) <a name='install'></a>Установка и подготовительные действия
Существует два способа установки Virtuozzo:
* с помощью ISO-образа дистрибутива
* с помощью установки пакетов и ядра на заранее установленный дистрибутив

Установка Virtuozzo с помощью PXE (Preboot Execution Environment) подробно описана в [документации](https://docs.openvz.org/virtuozzo_7_installation_using_pxe_guide.webhelp/).

### <a name='bare-metal'></a>Установка Virtuozzo с помощью ISO-образа (bare-metal installation)
Дистрибутив Virtuozzo основан на операционной системе [CloudLinux](https://www.cloudlinux.com/) с патчами для ядра RHEL 7, утилитами управления и модифицированным установщиком.
Рекомендуется именно этот способ установки Virtuozzo.

Текущая последняя версия ISO-образа доступна по адресу: https://download.openvz.org/virtuozzo/releases/7.0-beta3/x86_64/iso/

После записи дистрибутива на носитель, можно приступать к установке Virtuozzo.
Для этого необходимо загрузиться с носителя.

*Экран установки Virtuozzo после загрузки с носителя*
![Экран установки Virtuozzo](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vz-install/install-vz.png)

Установка Virtuozzo ничем не отличается от установки обычного Linux-дистрибутива.
Установщик Anaconda предложит установить дату и время, раскладку клавиатуры, языковые параметры.
Также необходимо будет произвести разметку диска и настроить сеть.
По умолчанию включен kdump, который позволяет в будущем выяснить причины сбоев в ядре, поэтому рекомендуется его не отключать.

*Экран установки параметров системы*
![Настройки системы](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vz-install/anaconda.png)

*Пример разметки для 20GB диска*
![Разметка диска](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vz-install/partitioning.png)

Необходимо для раздела `/` выделить не менее 8GB доступного дискового пространства.
Размер раздела `swap` равен примерно половине объема оперативной памяти.
Все остальное дисковое пространство выделяется под раздел `/vz` с файловой системой ext4.

*Настройки сетевого интерфейса и имени хоста*
![Настройки сети](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vz-install/network.png)

Также необходимо задать пароль пользователя `root` и создать локального пользователя, например `vzuser`.

*Установка пароля суперпользователя и создание локального пользователя*
![Настройки пользователей](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vz-install/user.png)

После установки необходимо перезагрузиться.

На этом установка Virtuozzo с помощью ISO-образа завершена.

*Меню загрузчика Grub после установки Virtuozzo*
![Grub](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vz-install/grub.png)

Первый вход в систему осуществляется от пользователя `vzuser`, по SSH.

Пример получения прав суперпользователя на сервере:
```
amet13@mint-17 ~ $ ssh vzuser@192.168.0.150
vzuser@192.168.0.150's password: пароль_пользователя_vzuser
[vzuser@virtuozzo ~]$ su -
Password: пароль_пользователя_root
[root@virtuozzo ~]#
```

### <a name='exists-distro'></a>Установка Virtuozzo на заранее установленный дистрибутив
Поддерживаемые дистрибутивы:
* CloudLinux 7
* CentOS 7
* Scientific Linux 7
* прочие дистрибутивы, основанные на RHEL 7

Установка пакетов на примере дистрибутива CentOS 7.
Virtuozzo 7 beta3.

Пакет `virtuozzo-release` содержит метаинформацию и yum-репозитории, необходимые для установки пакетов:
```
[root@virtuozzo ~]# yum localinstall https://download.openvz.org/virtuozzo/releases/7.0-beta3/x86_64/os/Packages/v/virtuozzo-release-7.0.0-20.vz7.x86_64.rpm
```

Установка необходимых RPM-пакетов:
```
[root@virtuozzo ~]# yum install prlctl prl-disp-service vzkernel
```

В качестве зависимостей также установятся такие пакеты как `criu`, `libvirt`, `lvm2`, `nfs-utils`, `quota`, `vcmmd`, `vzctl`, `vztt` и другие.

По окончании установки пакетов необходимо перезагрузиться:
```
[root@virtuozzo ~]# reboot
```

В меню загрузчика должен появиться новый пункт `Virtuozzo 7`.
Необходимо загрузиться с этого ядра.

*Меню загрузчика Grub после установки Virtuozzo*
![Grub](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vz-install/grub-vz.png)

### <a name='prepare'></a>Подготовительные действия
На сервере важно всегда обновлять программное обеспечение, так как в новых версиях не только могут добавлять новые возможности, но и исправлять уязвимости.
Указанная ниже команда обновляет все существующие в системе пакеты:
```
[root@virtuozzo ~]# yum update
```
Для сервера важно, чтобы было установлено правильное время.
Чтобы синхронизировать время с интернетом необходимо настроить сервер `ntp`.

Если во время установки ОС, не была установлена корректная временная зона, то можно это сделать позже:
```
[root@virtuozzo ~]# timedatectl set-timezone Europe/Moscow
```
По умолчанию сервер `ntp` уже установлен в системе, поэтому достаточно только добавить сервис в автозагрузку:
```
[root@virtuozzo ~]# systemctl start ntpd
[root@virtuozzo ~]# systemctl enable ntpd
```

## [⬆](#toc) <a name='templates-ct'></a>Управление шаблонами гостевых ОС и приложений контейнеров
### <a name='guest-os'></a>Шаблоны гостевых ОС
Шаблоны гостевых ОС используются для создания контейнеров.

Просмотр списка уже имеющихся локальных шаблонов гостевых ОС:
```
[root@virtuozzo ~]# vzpkg list -O --with-summary
centos-5-x86                       :Centos 5 (for ix86) Virtuozzo Template
centos-6-x86_64                    :Centos 6 (for AMD64/Intel EM64T) Virtuozzo Template
centos-7-x86_64                    :Centos 7 (for AMD64/Intel EM64T) Virtuozzo Template
```

Доступные удаленно шаблоны:
```
[root@virtuozzo ~]# vzpkg list --available --with-summary
debian-8.0-x86_64
fedora-22-x86_64
ubuntu-14.04-x86_64
ubuntu-14.10-x86_64
ubuntu-15.04-x86_64
```

Установка всех доступных шаблонов:
```
[root@virtuozzo ~]# vzpkg list --available --with-summary | xargs vzpkg install template
```
Альтернативный вариант установки шаблонов:
```
[root@virtuozzo ~]# yum install debian-8.0-x86_64-ez fedora-22-x86_64-ez ubuntu-14.04-x86_64-ez ubuntu-14.10-x86_64-ez ubuntu-15.04-x86_64-ez
```

После этого можно увидеть список доступных локально шаблонов гостевых ОС:
```
[root@virtuozzo ~]# vzpkg list -O --with-summary
fedora-22-x86_64                   :Fedora 22 (for AMD64/Intel EM64T) Virtuozzo Template
ubuntu-14.10-x86_64                :Ubuntu 14.10 (for AMD64/Intel EM64T) Virtuozzo Template
ubuntu-14.04-x86_64                :Ubuntu 14.04 (for AMD64/Intel EM64T) Virtuozzo Template
ubuntu-15.04-x86_64                :Ubuntu 15.04 (for AMD64/Intel EM64T) Virtuozzo Template
debian-8.0-x86_64                  :Debian 8.0 (for AMD64/Intel EM64T) Virtuozzo Template
debian-8.0-x86_64-minimal          :Debian 8.0 minimal (for AMD64/Intel EM64T) Virtuozzo Template
centos-5-x86                       :Centos 5 (for ix86) Virtuozzo Template
centos-7-x86_64                    :Centos 7 (for AMD64/Intel EM64T) Virtuozzo Template
centos-6-x86_64                    :Centos 6 (for AMD64/Intel EM64T) Virtuozzo Template
```

Установка и обновление кэша шаблона:
```
[root@virtuozzo ~]# vzpkg create cache ubuntu-14.04-x86_64
[root@virtuozzo ~]# vzpkg update cache ubuntu-14.04-x86_64
```

Если не указывать имя шаблона, то установка или обновление кэша произойдет для всех имеющихся шаблонов.

Просмотр даты последнего обновления кэша:
```
[root@virtuozzo ~]# vzpkg list -O
fedora-22-x86_64                   2015-10-19 01:24:25
ubuntu-14.10-x86_64                2015-10-19 01:34:18
ubuntu-14.04-x86_64                2015-10-19 01:50:54
ubuntu-15.04-x86_64                2015-10-19 02:10:18
debian-8.0-x86_64                  2015-10-19 01:00:28
debian-8.0-x86_64-minimal          2015-10-19 01:20:22
centos-5-x86                       2015-10-19 00:37:27
centos-7-x86_64                    2015-10-19 00:48:16
centos-6-x86_64                    2015-10-19 00:43:44
```

### <a name='app-templ'></a>Шаблоны приложений
Существует возможность установки шаблонов приложений для контейнеров.
Основное отличие между шаблонами гостевых ОС и шаблонами приложений в том, что шаблоны гостевых ОС используются для создания контейнеров, а шаблоны приложений, обеспечивают дополнительное ПО для уже имеющихся контейнеров.

Просмотр списка шаблонов приложений для `centos-7-x86_64`:
```
[root@virtuozzo ~]# vzpkg list centos-7-x86_64
centos-7-x86_64                    2016-02-09 17:01:05
centos-7-x86_64      mod_ssl       
centos-7-x86_64      jsdk          
centos-7-x86_64      cyrus-imap    
centos-7-x86_64      jre           
centos-7-x86_64      docker        
centos-7-x86_64      mailman       
centos-7-x86_64      devel         
centos-7-x86_64      mysql         
centos-7-x86_64      php           
centos-7-x86_64      spamassassin  
centos-7-x86_64      vzftpd        
centos-7-x86_64      postgresql    
centos-7-x86_64      tomcat        
```

Пример установки шаблона приложений `tomcat` и `jre`:
```
[root@virtuozzo ~]# vzpkg list ct5
centos-7-x86_64                    2016-02-09 17:00:57
[root@virtuozzo ~]# vzpkg install ct5 tomcat jre
[root@virtuozzo ~]# prlctl exec ct5 systemctl start tomcat
[root@virtuozzo ~]# prlctl exec ct5 systemctl status tomcat | grep Active
   Active: active (running) since Tue 2016-02-09 19:56:43 MSK; 41s ago
```

После установки можно проверить список установленных шаблонов для контейнера:
```
[root@virtuozzo ~]# vzpkg list ct5
centos-7-x86_64                    2016-02-09 17:00:57
centos-7-x86_64      tomcat        2016-02-09 19:56:03
centos-7-x86_64      jre           2016-02-09 20:03:50
```

Удаление шаблона приложения из контейнера:
```
[root@virtuozzo ~]# vzpkg remove ct5 tomcat
Removed:
 tomcat                 noarch    0:7.0.54-2.el7_1
 tomcat-admin-webapps   noarch    0:7.0.54-2.el7_1
 tomcat-webapps         noarch    0:7.0.54-2.el7_1
 tomcat-lib             noarch    0:7.0.54-2.el7_1
 tomcat-el-2.2-api      noarch    0:7.0.54-2.el7_1

[root@virtuozzo ~]# vzpkg list ct5
centos-7-x86_64                    2016-02-09 17:00:57
centos-7-x86_64      jre           2016-02-09 20:03:50
```

## [⬆](#toc) <a name='containers'></a>Создание и настройка контейнеров
### <a name='configs'></a>Конфигурационные файлы
В старых версиях OpenVZ основным идентификатором контейнера является CTID, который вручную указывался при создании контейнера.
Сейчас в этом нет необходимости, на смену CTID пришел UUID, который создается автоматически.
Однако существует возможность указания UUID вручную.

Каждый контейнер имеет свой конфигурационный файл, который хранится в каталоге `/etc/vz/conf/`.
Именуются конфиги по UUID контейнера.
Например, для контейнера с `UUID = {3d32522a-80af-4773-b9fa-ea4915dee4b3}`, конфиг будет называться `3d32522a-80af-4773-b9fa-ea4915dee4b3.conf`.

При создании контейнера можно использовать типовую конфигурацию.
Типовые файлы конфигураций находятся в том же каталоге `/etc/vz/conf/`:
```
[root@virtuozzo ~]# ls /etc/vz/conf/ | grep sample
ve-basic.conf-sample
ve-confixx.conf-sample
ve-vswap.1024MB.conf-sample
ve-vswap.2048MB.conf-sample
ve-vswap.256MB.conf-sample
ve-vswap.512MB.conf-sample
ve-vswap.plesk.conf-sample
vps.vzpkgtools.conf-sample
```

В этих конфигурационных файлах описаны контрольные параметры ресурсов, выделенное дисковое пространство, оперативная память и т.д.
Например, при использовании конфига `ve-vswap.512MB.conf-sample`, создается контейнер с дисковым пространством 10GB, оперативной памятью 512MB и swap 512MB:
```
[root@virtuozzo ~]# grep "DISKSPACE\|PHYSPAGES\|SWAPPAGES\|DISKINODES" /etc/vz/conf/ve-vswap.512MB.conf-sample
PHYSPAGES="131072:131072"
SWAPPAGES="131072"
DISKSPACE="10485760:10485760"
DISKINODES="655360:655360"
```

Это удобно, так как существует возможность создавать свои конфигурационные файлы для различных вариаций контейнеров.
Создадим свой конфигурационный файл, на базе уже существующего `vswap.512MB`.
Исправим в нем только значения `PHYSPAGES`, `SWAPPAGES`, `DISKSPACE`, `DISKINODES`:
```
[root@virtuozzo ~]# cp /etc/vz/conf/ve-vswap.512MB.conf-sample /etc/vz/conf/ve-vswap.1GB.conf-sample
[root@virtuozzo ~]# vim /etc/vz/conf/ve-vswap.1GB.conf-sample
PHYSPAGES="262144:262144"
SWAPPAGES="262144"
DISKSPACE="20971520:20971520"
DISKINODES="1310720:1310720"
```
Таким образом, при использовании этого конфигурационного файла, будет создаваться контейнер, которому будет доступно 20GB выделенного дискового пространства, 1GB оперативной памяти и 1GB swap.

Установка конфигурационного файла шаблона на примере `vswap.1GB` (контейнер должен быть создан):
```
[root@virtuozzo ~]# prlctl set ct1 --applyconfig vswap.1GB
The CT has been successfully configured.
```

### <a name='create-ct'></a>Создание контейнера
В качестве параметра к идентификатору контейнера может использоваться любое имя:
```
[root@virtuozzo ~]# CT=ct1
[root@virtuozzo ~]# prlctl create $CT --ostemplate debian-8.0-x86_64 --vmtype=ct
Creating the Virtuozzo Container...
The Container has been successfully created.
```

Таким образом был создан контейнер с именем `ct1` на базе шаблона `debian-8.0-x86_64`.

Теперь можно просмотреть список имеющихся в системе контейнеров:
```
[root@virtuozzo ~]# prlctl list -a
UUID                                    STATUS       IP_ADDR         T  NAME
{3d32522a-80af-4773-b9fa-ea4915dee4b3}  stopped      -               CT ct1
```

Если же при создании контейнера не указывать желаемый шаблон, то Virtuozzo будет использовать шаблон по умолчанию.
Конфигурационный файл, в котором указаны директивы по умолчанию `/etc/vz/vz.conf`.
По умолчанию, используется шаблон `centos-6` и конфигурационный файл `basic`:
```
[root@virtuozzo ~]# grep "CONFIGFILE\|DEF_OSTEMPLATE" /etc/vz/vz.conf
CONFIGFILE="basic"
DEF_OSTEMPLATE=".centos-7"
```

Если планируется создание большого количества однотипных контейнеров, основываясь на одном и том же конфиге, то значения можно исправить на нужные.

### <a name='setup-ct'></a>Настройка контейнера
Контейнер создан, его можно запускать.
Но перед первым запуском необходимо установить его IP адреса, hostname, указать DNS сервера и задать пароль суперпользователя.

Добавление IP адресов:
```
[root@virtuozzo ~]# prlctl set ct1 --ipadd 192.168.0.161/24
[root@virtuozzo ~]# prlctl set ct1 --ipadd fe80::20c:29ff:fe01:fb08
```

Установка DNS серверов и hostname:
```
[root@virtuozzo ~]# prlctl set ct1 --nameserver 192.168.0.1,192.168.0.2
[root@virtuozzo ~]# prlctl set ct1 --hostname ct1.virtuozzo.localhost
```

Установка поискового домена (если требуется):
```
[root@virtuozzo ~]# prlctl set ct1 --searchdomain 192.168.0.1
```

Установка пароля суперпользователя:
```
[root@virtuozzo ~]# prlctl set ct1 --userpasswd root:eVjfsDkTE63s5Nw
```

Сгенерировать пароль можно штатными средствами Linux:
```
[root@virtuozzo ~]# cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 15 | head -1
eVjfsDkTE63s5Nw
```

Или воспользоваться утилитой `pwgen`:
```
[root@virtuozzo ~]# yum localinstall http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
[root@virtuozzo ~]# yum install pwgen
[root@virtuozzo ~]# pwgen -s 15 1
eVjfsDkTE63s5Nw
```

Пароль будет установлен в контейнер, в файл `/etc/shadow` и не будет сохранен в конфигурационный файл контейнера.
Если же пароль будет утерян или забыт, то можно будет просто задать новый.

Для запуска контейнера при старте хост-ноды добавляем:
```
[root@virtuozzo ~]# prlctl set ct1 --onboot yes
```

### <a name='run-enter'></a>Запуск и вход
После настроек нового контейнера. его можно запустить:
```
[root@virtuozzo ~]# prlctl start ct1
Starting the CT...
The CT has been successfully started.
```

Проверяем сетевые интерфейсы внутри гостевой ОС:
```
[root@virtuozzo ~]# prlctl exec ct1 ifconfig | grep "lo\|venet" -A 1
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
--
venet0    Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
          inet addr:127.0.0.1  P-t-P:127.0.0.1  Bcast:0.0.0.0  Mask:255.255.255.255
--
venet0:0  Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
          inet addr:192.168.0.161  P-t-P:192.168.0.161  Bcast:192.168.0.255  Mask:255.255.255.0
```

Также проверим корректность hostname:
```
[root@virtuozzo ~]# prlctl exec ct1 hostname
ct1.virtuozzo.localhost
```

Проверяем доступность контейнера в сети и корректность пароля суперпользователя:
```
[root@virtuozzo ~]# ssh root@192.168.0.161
root@192.168.0.161's password: eVjfsDkTE63s5Nw
root@ct1:~#
```

Вход в контейнер напрямую с хост-ноды:
```
[root@virtuozzo ~]# prlctl enter ct1
entered into CT
root@ct1:/#
```

Выход из контейнера:
```
root@ct1:/# exit
logout
[root@virtuozzo ~]#
```

## [⬆](#toc) <a name='management-ct'></a>Управление контейнерами
### <a name='status-ct'></a>Управление состоянием контейнера
Статус контейнера:
```
[root@virtuozzo ~]# prlctl status ct1
CT ct1 exist running
[root@virtuozzo ~]# prlctl status ct2
CT ct2 exist stopped
```

По выводу команды можно наблюдать, что контейнер `ct1` существует и запущен, а контейнер `ct2` существует и остановлен.

Остановка контейнера:
```
[root@virtuozzo ~]# prlctl stop ct1
Stopping the CT...
The CT has been successfully stopped.
```

Иногда нужно выключить контейнер как можно быстрее, например если контейнер был подвержен взлому.
Для того чтобы срочно выключить контейнер, нужно использовать ключ `--kill`:
```
[root@virtuozzo ~]# prlctl stop ct1 --kill
Stopping the CT...
The CT has been forcibly stopped
```

Перезапуск контейнера:
```
[root@virtuozzo ~]# prlctl restart ct1
Restarting the CT...
The CT has been successfully restarted.
```

Приостановка контейнера сохраняет текущее состояние контейнера в файл, позволяя восстановить контейнер в то же состояние, в котором он был приостановлен, это может быть полезно, к примеру если перезагружается хост-нода и нужно сохранить состояние процессов в контейнере.

Команда `suspend` приостанавливает контейнера, а `resume` — восстанавливает:
```
[root@virtuozzo ~]# prlctl suspend ct1
Suspending the CT...
The CT has been successfully suspended.
[root@virtuozzo ~]# prlctl status ct1
CT ct1 exist suspended
[root@virtuozzo ~]# prlctl resume ct1
Resuming the CT...
The CT has been successfully resumed.
```

Для удаления контейнера существует команда `delete`, перед удалением, контейнер нужно сначала остановить:
```
[root@virtuozzo ~]# prlctl stop
Stopping the CT...
The CT has been successfully stopped.
[root@virtuozzo ~]# prlctl delete ct1
Removing the CT...
The CT has been successfully removed.
```

Команда выполняет удаление частной области сервера и переименовывает файл конфигурации, дописывая к нему `.destroyed`.

### <a name='reinstall-ct'></a>Переустановка контейнера
Для переустановки ОС в контейнере, существует команда `vzctl reinstall`.

Прежде чем переустанавливать контейнер его нужно сначала остановить:
```
[root@virtuozzo ~]# prlctl stop ct1
Stopping the CT...
The CT has been successfully stopped.
```

Переустановка и старт контейнера:
```
[root@virtuozzo ~]# vzctl reinstall ct1
...
Container was successfully reinstalled
[root@virtuozzo ~]# prlctl start ct1
Starting the CT...
The CT has been successfully started.
```

По умолчанию, `vzctl reinstall` без дополнительных параметров, сохраняет все файлы (частную область) прошлого контейнера  в каталог `/old` нового контейнера.
Для того, чтобы не копировать частную область предыдущего контейнера, необходимо использовать ключ `--skipbackup`:
```
[root@virtuozzo ~]# vzctl reinstall ct1 --skipbackup
```

### <a name='clone-ct'></a>Клонирование контейнера
Virtuozzo позволяет клонировать контейнеры:
```
[root@virtuozzo ~]# prlctl clone ct1 --name ct2
Clone the ct1 CT to CT ct2...
The CT has been successfully cloned.
[root@virtuozzo ~]# prlctl list -a
UUID                                    STATUS       IP_ADDR         T  NAME
{3d32522a-80af-4773-b9fa-ea4915dee4b3}  running      192.168.0.161   CT ct1
{54bc2ba6-b040-469e-9fda-b0eabda822d4}  stopped      192.168.0.161   CT ct2
```

При клонировании контейнера необходимо помнить о смене IP адреса, иначе при попытке запуска будет наблюдаться ошибка:
```
[root@virtuozzo ~]# prlctl start ct2
Starting the CT...
Failed to start the CT: PRL_ERR_VZCTL_OPERATION_FAILED
Unable to add ip 192.168.0.161: Address already in use
Failed to start the Container
```

Сначала нужно удалить старые IP адреса:
```
[root@virtuozzo ~]# prlctl set ct2 --ipdel 192.168.0.161/24
[root@virtuozzo ~]# prlctl set ct2 --ipdel fe80::20c:29ff:fe01:fb08
```

Затем добавить новые:
```
[root@virtuozzo ~]# prlctl set ct2 --ipadd 192.168.0.162/24
[root@virtuozzo ~]# prlctl set ct2 --ipadd fe80::20c:29ff:fe01:fb09
```
Смена hostname:
```
[root@virtuozzo ~]# prlctl set ct2 --hostname ct2.virtuozzo.localhost
The CT has been successfully configured.
```

После этого контейнер можно запустить:
```
[root@virtuozzo ~]# prlctl start ct2
Starting the CT...
The CT has been successfully started.
```

### <a name='run-commands'></a>Запуск команд в контейнере с хост-ноды
Пример запуска команды в контейнере:
```
[root@virtuozzo ~]# prlctl exec ct1 cat /etc/issue
Debian GNU/Linux 8 \n \l
```

Иногда бывает нужно выполнить команду на нескольких контейнерах.
Для этого можно использовать команду:
```
[root@virtuozzo ~]# CMD="cat /etc/issue"
[root@virtuozzo ~]# for i in `prlctl list -o name -H`; do echo "CT $i"; prlctl exec $i $CMD; done
CT ct1
Debian GNU/Linux 8 \n \l

CT ct2
Debian GNU/Linux 8 \n \l
```

### <a name='extra-info'></a>Расширенная информация о контейнерах
Подробная информация о контейнере:
```
[root@virtuozzo ~]# prlctl list -i ct4
INFO
ID: {22c418d7-948b-456e-9d84-d59ab5ead661}
EnvID: 22c418d7-948b-456e-9d84-d59ab5ead661
Name: ct4
Description:
Type: CT
State: running
OS: centos7
Template: no
Uptime: 00:00:00 (since 2016-02-09 17:04:41)
Home: /vz/private/22c418d7-948b-456e-9d84-d59ab5ead661
Owner: root
Effective owner: owner
GuestTools: state=possibly_installed
Autostart: on
Autostop: suspend
Autocompact: on
Undo disks: off
Boot order:
EFI boot: off
Allow select boot device: off
External boot device:
Remote display: mode=off address=0.0.0.0
Remote display state: stopped
Hardware:
  cpu cpus=unlimited VT-x accl=high mode=32 cpuunits=1000 ioprio=4
  memory 512Mb
  video 0Mb 3d acceleration=highest vertical sync=yes
  memory_guarantee auto
  hdd0 (+) image='/vz/private/22c418d7-948b-456e-9d84-d59ab5ead661/root.hdd' type='expanded' 10240Mb mnt=/
  venet0 (+) type='routed' ips='192.168.0.164/255.255.255.0 FE80:0:0:0:20C:29FF:FE01:FB10/64 '
Host Shared Folders: (-)
Features:
Encrypted: no
Faster virtual machine: on
Adaptive hypervisor: off
Disabled Windows logo: on
Auto compress virtual disks: on
Nested virtualization: off
PMU virtualization: off
Offline management: (-)
Hostname: ct4.virtuozzo.localhost
DNS Servers: 192.168.0.1 192.168.0.2
Search Domains: 192.168.0.1
```

Существует также возможность просмотра дополнительной информации о контейнерах:
```
[root@virtuozzo ~]# prlctl list -o type,status,name,hostname,dist,ip
T  STATUS       NAME                             HOSTNAME                         DIST            IP_ADDR
CT running      ct2                           ct2.virtuozzo.localhost       debian          192.168.0.162 FE80:0:0:0:20C:29FF:FE01:FB09
CT running      ct1                            ct1.virtuozzo.localhost        debian          192.168.0.161 FE80:0:0:0:20C:29FF:FE01:FB08
```

Список всех доступных полей:
```
[root@virtuozzo ~]# prlctl list -L
uuid                 UUID
envid                ENVID
status               STATUS
name                 NAME
dist                 DIST
owner                OWNER
system-flags         SYSTEM_FLAGS
description          DESCRIPTION
numproc              NPROC
ip                   IP_ADDR
ip_configured        IP_ADDR
hostname             HOSTNAME
netif                NETIF
mac                  MAC
features             FEATURES
location             LOCATION
iolimit              IOLIMIT
netdev               NETDEV
type                 T
ha_enable            HA_ENABLE
ha_prio              HA_PRIO
-                    -
```

## [⬆](#toc) <a name='resources-ct'></a>Управление ресурсами контейнеров
Доступные контейнеру ресурсы контролируются с помощью набора параметров управления ресурсами.
Все эти параметры можно редактировать в файлах шаблонов, в каталоге `/etc/vz/conf/`.
Их можно установить вручную, редактируя соответствующие конфиги или используя утилиты Virtuozzo.

Параметры контроля ресурсов условно разделяют на группы:
* дисковые квоты
* процессор
* операции ввода/вывода
* память
* сеть

### <a name='quota'></a>Дисковые квоты
Администратор сервера Virtuozzo может устанавливать дисковые квоты, в терминах дискового пространства и количества inodes, число которых примерно равно количеству файлов.
Это первый уровень дисковой квоты.
В дополнение к этому, администратор может использовать обычные утилиты внутри окружения, для настроек стандартных дисковых квот UNIX для пользователей и групп.

Для использования дисковых квот, соответствующая директива должна присутствовать в конфигурационном файле Virtuozzo:
```
[root@virtuozzo ~]# grep DISK_QUOTA /etc/vz/vz.conf
DISK_QUOTA=yes
```

Основные параметры:
* `DISKSPACE` — общий размер дискового пространства (задается в Kb)
* `DISKINODES` — общее число дисковых inodes
* `QUOTATIME` — время (в секундах) на которое контейнер может превысить значение soft предела

Параметры записываются в конфигурационный файл в виде:
```
COMMAND="softlimit:hardlimit"
```
где:
* `COMMAND` — команда (`DISKSPACE` или `DISKINODES`)
* `softlimit` — значение которое превышать нежелательно, после пересечения этого предела наступает grace период, по истечении которого, дисковое пространство или inodes прекратят свое существование
* `hardlimit` — значение которое превысить нельзя

Например, запись:
```
DISKSPACE="19922944:20971520"
DISKINODES="1300000:1310720"
QUOTATIME="600"
```
означает, что задается `softlimit` для дискового пространства равным ~19G и `hardlimit` равный 20G, то же самое с inodes 1300000 и 1310720 соответственно.

Если размер занятого дискового пространства или inodes будет выше `softlimit`, то в течении 600 сек (10 мин), в случае не освобождения дискового пространства или inodes, они прекратят свое существование.

Аналогично, можно установить эти параметры с помощью `vzctl`:
```
[root@virtuozzo ~]# vzctl set ct1 --diskspace 5G:6G --save
Resize the image /vz/private/3d32522a-80af-4773-b9fa-ea4915dee4b3/root.hdd to 6291456K
dumpe2fs 1.42.9 (28-Dec-2013)
[root@virtuozzo ~]# vzctl set ct1 --diskinodes 10000:110000 --save
```

### <a name='cpu'></a>Процессор
Планировщик процессора в Virtuozzo также двухуровневый.
На первом уровне планировщик решает, какому контейнеру дать квант процессорного времени, базируясь на значении параметра `CPUUNITS` для контейнера.
На втором уровне стандартный планировщик GNU/Linux решает, какому процессу в выбранном контейнере дать квант времени, базируясь на стандартных приоритетах процесса.

Основными параметрами управления CPU являются:
* `CPUUNITS` — гарантируемое минимальное количество времени процессора, которое получит соответствующий контейнер
* `CPUMASK` — привязка контейнера к конкретным процессорам, по умолчанию нагрузка распределяется на все процессоры
* `CPULIMIT` — верхний лимит процессорного времени в процентах
* `CPUS` — количество используемых процессорных ядер контейнером
* `NODEMASK` — привязка ядер NUMA-систем к контейнеру

Параметр `CPUUNITS` указывает процессорное время доступное для контейнера.
По умолчанию для каждого контейнера это значение равно 1000.
То есть, если для контейнера `ct1` установить значение 2000, а для контейнера `ct2` оставить значение 1000, то при равных условиях контейнер `ct1` получит ровно в два раза больше процессорного времени.
```
[root@virtuozzo ~]# prlctl set ct1 --cpuunits 2000
set cpuunits 2000
```

Если система многопроцессорная, то установка параметра `CPUMASK` может пригодиться для привязки контейнера к конкретным процессорам.
В случае восьмипроцессорной системы можно привязать контейнер к процессорам 0-3, 6, 7:
```
[root@virtuozzo ~]# prlctl set ct1 --cpumask 0-3,6,7
set cpu mask 0-3,6,7
```

Параметр `CPULIMIT` указывает общий верхний лимит процессорного времени для всех ядер процессора:
```
[root@virtuozzo ~]# prlctl set ct1 --cpulimit 15
set cpulimit 15%
```
Для одноядерного процессора верхний лимит будет равен 100%, для двухядерного 200% и так далее.

Существует также возможность задания `CPULIMIT` в абсолютных значениях (MHz):
```
[root@virtuozzo ~]# prlctl set ct1 --cpulimit 600m
set cpulimit 600Mhz
```

В параметре `CPUS` задается число доступных для контейнера процессорных ядер.
Контейнер по умолчанию получает в использование все процессорные ядра:
```
[root@virtuozzo ~]# CPUINFO="grep processor /proc/cpuinfo"
[root@virtuozzo ~]# prlctl exec ct1 $CPUINFO
processor	: 0
processor	: 1
processor	: 2
processor	: 3
```

Установим для контейнера лимит в 2 процессорных ядра:
```
[root@virtuozzo ~]# prlctl set ct1 --cpus 2
set cpus(4): 2
[root@virtuozzo ~]# prlctl exec ct1 $CPUINFO
processor	: 0
processor	: 1
```

Для систем архитектуры NUMA существует возможность привязки контейнера к процессорам NUMA-нод:
```
[root@virtuozzo ~]# vzctl set ct1 --nodemask 0 --save
```

Аналогично все параметры можно вручную прописать в конфигурационный файл контейнера:
```
CPUUNITS="2000"
CPUMASK="0-3,6,7"
CPULIMIT="15"
CPULIMIT_MHZ="600"
CPUS="2"
NODEMASK="0"
```

Утилиты контроля ресурсов процессора, гарантируют любому контейнеру количество времени центрального процессора, которое собственно и получает этот контейнер.
При этом контейнер может потреблять больше времени, чем определено этой величиной, если нет другого конкурирующего с ним за время CPU сервера.

### <a name='io'></a>Операции ввода/вывода
В Virtuozzo существует возможность управления дисковыми операциями ввода/вывода.
Можно устанавливать значения таких параметров как:
* `IOPRIO`
* `IOLIMIT`
* `IOPSLIMIT`

Параметр `IOPRIO` указывает приоритет операция ввода вывода для контейнера.
По умолчанию для всех контейнеров установлен равный приоритет (значение 4).

Изменение значения параметра можно регулировать от 0 (максимальный приоритет) до 7:
```
[root@virtuozzo ~]# prlctl set ct1 --ioprio 6
set ioprio 6
```

Параметр `IOLIMIT` позволяет ограничивать пропускную способность операций ввода/вывода.
По умолчанию параметр имеет значение 0, то есть отсутствие лимитов.

Установка значения в MB/s:
```
[root@virtuozzo ~]# prlctl set ct1 --iolimit 20
Set up iolimit: 20971520
```

Существует возможность указания префиксов метрических значений:
* `G` — гигабайт
* `M` — мегабайт
* `K` — килобайт
* `B` — байт

Максимальная пропускная способность дисковых операций ввода/вывода составляет 2GB/s.

Помимо ограничения пропускной способности операций ввода/вывода, существует возможность ограничения количества операций ввода/вывода в секунду.

Параметр `IOPSLIMT` позволяет установить численное значение операций ввода/вывода в секунду, например 300:
```
[root@virtuozzo ~]# prlctl set ct1 --iopslimit 300
set IOPS limit 300
```

По умолчанию значение этого параметра равно 0, что означает отсутствие лимитов.

Параметры можно указать вручную в конфигурационном файле контейнера:
```
IOPRIO="6"
IOLIMIT="20971520"
IOPSLIMIT="300"
```

Проверка ограничения пропускной способностей операций ввода/вывода на примере `IOLIMIT`.

Значение `IOLIMIT` равно 0:
```
root@ct1:/# dd if=/dev/zero of=test bs=1048576 count=10
10+0 records in
10+0 records out
10485760 bytes (10 MB) copied, 0.210523 s, 49.8 MB/s
```

Значение `IOLIMIT` равно 500K:
```
root@ct1:/# dd if=/dev/zero of=test bs=1048576 count=10
10+0 records in
10+0 records out
10485760 bytes (10 MB) copied, 17.4388 s, 601 kB/s
```

### <a name='memory'></a>Память
В Virtuozzo используется управление памятью четвертого поколения с помощью VCMM.
В прошлом же использовалось управление памятью с помощью:
* VSwap (третье поколение)
* SLM (второе поколение)
* User Beancounters (первое поколение)

С пользовательской стороны управление памятью с помощью VSwap и VCMM ничем не отличаются, однако с точки зрения реализации, VCMM уже находится в ванильном ядре и не требует патчей со стороны разработчиков Virtuozzo.

Ограничение физической памяти и swap задаются в конфигурационном файле контейнера параметрами `PHYSPAGES` и `SWAPPAGES`.
Значения устанавливаются в блоках, например:
```
PHYSPAGES="262144:262144"
SWAPPAGES="262144:262144"
```
равняются значениям в 1024MB (262144 блок / 256 = 1024MB).

С помощью `prlctl` значения параметров можно указывать в метрической системе:
```
[root@virtuozzo ~]# prlctl set ct1 --memsize 1G --swappages 1G
Set the memsize parameter to 1024Mb.
Set swappages 262144
```

Overcommiting — возможность использования большего числа ресурсов, чем выдано контейнеру.

Значение `VM_OVERCOMMIT` указывает число, во сколько раз больше памяти сможет использовать контейнер в случае необходимости.
По умолчанию значение `VM_OVERCOMMIT` равно 1.5.
То есть для контейнера установлено, с 1024MB оперативной памяти и 1024MB swap, суммарно доступно 2048MB памяти, в случае необходимости контейнер сможет использовать (2048MB * 1.5 = 3072MB) памяти.

Для изменения значения достаточно прописать параметр в конфигурационный файл контейнера и перезапустить его:
```
VM_OVERCOMMIT="2"
```

Также возможна установка параметра с помощью `vzctl`:
```
[root@virtuozzo ~]# vzctl set ct1 --vm_overcommit 2 --save
```

При использовании значения 2 для ранее упомянутого контейнера с 2048MB памяти, будет доступно (2048MB * 2 = 4096MB) памяти.
Естественно, если если эти ресурсы доступны на хост-ноде.

### <a name='monitoring'></a>Мониторинг ресурсов
С помощью утилиты `vznetstat` можно увидеть входящий и исходящий трафик (в байтах и пакетах) для всех контейнеров:
```
[root@virtuozzo ~]# vznetstat
UUID                                 Net.Class     Input(bytes) Input(pkts)        Output(bytes) Output(pkts)
0                                    0                244486        3024              1567749        2491
54bc2ba6-b040-469e-9fda-b0eabda822d4 0                     0           0                    0           0
4730cba8-deed-4168-9f9e-34373e618026 0                     0           0                    0           0
3d32522a-80af-4773-b9fa-ea4915dee4b3 0               2925512       49396             49398885       49254
```

Для конкретного контейнера можно воспользоваться ключом `-v`:
```
[root@virtuozzo ~]# vznetstat -v 3d32522a-80af-4773-b9fa-ea4915dee4b3
UUID                                 Net.Class     Input(bytes) Input(pkts)        Output(bytes) Output(pkts)
3d32522a-80af-4773-b9fa-ea4915dee4b3 0               2925512       49396             49398885       49254
```

Утилита `vzstat` позволяет узнать информацию по нагрузке на контейнер, занятым ресурсам и состоянии сети:
```
[root@virtuozzo ~]# vzstat -p 3d32522a-80af-4773-b9fa-ea4915dee4b3 -t
loadavg		0 0 0
CTNum		3
procs		289 1 288 0 0 0 0
CPU		16 0 2 3 95
sched latency	372 9
Mem		989 360 0
Mem latency	1 0
  ZONE0 (DMA): size 15MB, act 4MB, inact 4MB, free 4MB (0/0/1)
  ZONE1 (DMA32): size 1007MB, act 243MB, inact 274MB, free 355MB (43/54/64)
  Mem lat (ms): A0 1, K0 0, U0 1, K1 0, U1 0
  Slab pages: 62MB/62MB (ino 22MB, de 0MB, bh 1MB, pb 0MB)
Swap		952 952 0.000 0.000
Net stats	0.382 5949 5.542 5820
if br0 stats	0.171 2975 2.771 2910
if lo stats	0.000 0 0.000 0
if virbr1-nic stats	0.000 0 0.000 0
if enp0s3 stats	0.211 2975 2.771 2910
if virbr1 stats	0.000 0 0.000 0
Disks stats	0.000 0.000

    CTID ST   %VM    %KM        PROC     CPU     SOCK FCNT MLAT IP
```

`vzpid` позволяет узнать к какому контейнеру принадлежит процесс, это может быть полезно при просмотре списка процессов с хост-ноды и поиска "процесса-грузчика":
```
[root@virtuozzo ~]# top
top - 20:43:26 up 33 min,  1 user,  load average: 0.00, 0.01, 0.05
Tasks: 178 total,   1 running, 176 sleeping,   1 stopped,   0 zombie
%Cpu(s):  1.7 us,  3.6 sy,  0.0 ni, 88.5 id,  0.0 wa,  0.0 hi,  6.2 si,  0.0 st
KiB Mem :  1013704 total,   382912 free,   138656 used,   492136 buff/cache
KiB Swap:   975868 total,   975868 free,        0 used.   688028 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
   5625 33        20   0  364432   6232   1284 S  26.2  0.6   0:03.20 apache2
...
[root@virtuozzo ~]# vzpid 5625
Pid	VEID	Name
5625	3d32522a-80af-4773-b9fa-ea4915dee4b3	apache2
```

Утилита `vzps` аналогична утилите `ps`, она позволяет вывести список процессов и их состояние для конкретного контейнера:
```
[root@virtuozzo ~]# vzps aufx -E 3d32522a-80af-4773-b9fa-ea4915dee4b3
    USER     PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
       0    2432  0.0  0.0      0     0 ?        S    20:10   0:00 [kthreadd/3d3252]
       0    2433  0.0  0.0      0     0 ?        S    20:10   0:00  \_ [khelper]
       0    2420  0.0  0.3  28168  3136 ?        Ss   20:10   0:00 init -z
     101    3088  0.0  0.1  26168  1448 ?        Ss   20:10   0:00  \_ /lib/systemd/systemd-networkd
       0    3117  0.0  0.1  28856  1620 ?        Ss   20:10   0:00  \_ /lib/systemd/systemd-journald
       0    3135  0.0  0.1  38916  1624 ?        Ss   20:10   0:00  \_ /lib/systemd/systemd-udevd
       0    3376  0.0  0.3  55156  3128 ?        Ss   20:10   0:00  \_ /usr/sbin/sshd -D
     102    3380  0.0  0.1  25732  1092 ?        Ss   20:10   0:00  \_ /lib/systemd/systemd-resolved
       0    3382  0.0  0.1  25884  1120 ?        Ss   20:10   0:00  \_ /usr/sbin/cron -f
       0    3388  0.0  0.1 182848  1884 ?        Ssl  20:10   0:00  \_ /usr/sbin/rsyslogd -n
       0    3433  0.0  0.0  12648   840 ?        Ss+  20:10   0:00  \_ /sbin/agetty --noclear tty2 linux
       0    3434  0.0  0.0  12648   840 ?        Ss+  20:10   0:00  \_ /sbin/agetty --noclear --keep-baud console 115200 38400 9600 linux
       0    3508  0.0  0.0  20200   956 ?        Ss   20:10   0:00  \_ /usr/sbin/xinetd -pidfile /run/xinetd.pid -stayalive -inetd_compat -inetd_ipv6
       0    3617  0.0  0.1  65452  1164 ?        Ss   20:10   0:00  \_ /usr/sbin/saslauthd -a pam -c -m /var/run/saslauthd -n 2
       0    3625  0.0  0.0  65452   836 ?        S    20:10   0:00  |   \_ /usr/sbin/saslauthd -a pam -c -m /var/run/saslauthd -n 2
       0    3755  0.0  0.2  73496  2724 ?        Ss   20:10   0:00  \_ /usr/sbin/apache2 -k start
      33    5747  0.2  0.5 363364  5300 ?        Sl   20:46   0:00  |   \_ /usr/sbin/apache2 -k start
       0    4074  0.0  0.2  36144  2388 ?        Ss   20:10   0:00  \_ /usr/lib/postfix/master
     105    4081  0.0  0.2  38208  2316 ?        S    20:10   0:00      \_ pickup -l -t unix -u -c
     105    4082  0.0  0.2  38256  2336 ?        S    20:10   0:00      \_ qmgr -l -t unix -u

```

Для утилиты `top` также существует аналог `vztop`.
Пример просмотра списка процессов отсортированных по нагрузке на процессор для контейнера `ct1`:
```
[root@virtuozzo ~]# vztop -E 3d32522a-80af-4773-b9fa-ea4915dee4b3 -o %CPU -b
vztop - 21:13:45 up  1:03,  1 user,  load average: 0.01, 0.04, 0.32
Tasks:  20 total,   0 running,  20 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.7 sy,  0.0 ni, 98.5 id,  0.1 wa,  0.0 hi,  0.5 si,  0.0 st
KiB Mem :  1013704 total,   378752 free,   136600 used,   498352 buff/cache
KiB Swap:   975868 total,   975868 free,        0 used.   691636 avail Mem

                                    CTID     PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    3d32522a-80af-4773-b9fa-ea4915dee4b3    5747 33        20   0  364424   6304   1280 S  40.0  0.6   0:03.37 apache2
    3d32522a-80af-4773-b9fa-ea4915dee4b3    2420 root      20   0   28168   3136   1908 S   0.0  0.3   0:00.32 systemd
    3d32522a-80af-4773-b9fa-ea4915dee4b3    2432 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kthreadd/3d3252
    3d32522a-80af-4773-b9fa-ea4915dee4b3    2433 root      20   0       0      0      0 S   0.0  0.0   0:00.00 khelper
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3088 101       20   0   26168   1448   1204 S   0.0  0.1   0:00.06 systemd-network
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3117 root      20   0   28856   1620   1356 S   0.0  0.2   0:00.12 systemd-journal
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3135 root      20   0   38916   1624   1132 S   0.0  0.2   0:00.04 systemd-udevd
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3376 root      20   0   55156   3128   2460 S   0.0  0.3   0:00.01 sshd
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3380 102       20   0   25732   1092    896 S   0.0  0.1   0:00.00 systemd-resolve
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3382 root      20   0   25884   1120    900 S   0.0  0.1   0:00.01 cron
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3388 root      20   0  182848   1884   1412 S   0.0  0.2   0:00.02 rsyslogd
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3433 root      20   0   12648    840    692 S   0.0  0.1   0:00.00 agetty
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3434 root      20   0   12648    840    692 S   0.0  0.1   0:00.00 agetty
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3508 root      20   0   20200    956    756 S   0.0  0.1   0:00.00 xinetd
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3617 root      20   0   65452   1164    328 S   0.0  0.1   0:00.00 saslauthd
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3625 root      20   0   65452    836      0 S   0.0  0.1   0:00.00 saslauthd
    3d32522a-80af-4773-b9fa-ea4915dee4b3    3755 root      20   0   73496   2724   1512 S   0.0  0.3   0:00.58 apache2
    3d32522a-80af-4773-b9fa-ea4915dee4b3    4074 root      20   0   36144   2388   1848 S   0.0  0.2   0:00.06 master
    3d32522a-80af-4773-b9fa-ea4915dee4b3    4081 105       20   0   38208   2316   1776 S   0.0  0.2   0:00.04 pickup
    3d32522a-80af-4773-b9fa-ea4915dee4b3    4082 105       20   0   38256   2336   1792 S   0.0  0.2   0:00.02 qmgr
```

## [⬆](#toc) <a name='migration-ct'></a>Миграция контейнеров
В текущей версии Virtuozzo пока недоступна возможность онлайн-миграции контейнеров без их отключения.

Пример оффлайн миграции контейнера с хост-ноды `vz-source` на `vz-dest` (192.168.0.180).

Устанавливаем на `vz-source` и `vz-dest` последние версии `vzmigrate` и `rsync`:
```
[root@vz-source ~]# yum install vzmigrate rsync
```

Создаем и копируем SSH-ключ с `vz-source` на `vz-dest` для беспарольной аутентификации:
```
[root@vz-source ~]# cd /root && ssh-keygen
[root@vz-source ~]# ssh-copy-id root@192.168.0.180
```

Останавливаем контейнер перед миграцией:
```
[root@vz-source ~]# prlctl stop ct3
Stopping the CT...
The CT has been successfully stopped.
```

Запускаем миграцию в `screen`:
```
[root@vz-source ~]# screen -S migrate-dest
[root@vz-source ~]# vzmigrate 192.168.0.180 ct3
Connection to destination node (192.168.0.180) is successfully established
Moving/copying CT 4730cba8-deed-4168-9f9e-34373e618026 -> CT 4730cba8-deed-4168-9f9e-34373e618026, [], [] ...
locking 4730cba8-deed-4168-9f9e-34373e618026
Checking bindmounts
Check cluster ID
Checking keep dir for private area copy
Checking technologies
Checking templates for CT
Checking IP addresses on destination node
Check target CT name: ct3
Checking RATE parameters in config
Checking ploop format 2
copy CT private /vz/private/4730cba8-deed-4168-9f9e-34373e618026
Successfully completed
```

Проверяем на `vz-dest` наличие только что смигрированного контейнера, если он смигрирован, то запускаем его:
```
[root@vz-dest ~]# prlctl list ct3
UUID                                    STATUS       IP_ADDR         T  NAME
{4730cba8-deed-4168-9f9e-34373e618026}  stopped      192.168.0.163   CT ct3
[root@vz-dest ~]# prlctl start ct3
Starting the CT...
The CT has been successfully started.
```

## [⬆](#toc) <a name='forward-dev-ct'></a>Проброс устройств в контейнеры
### <a name='tun-tap'></a>TUN/TAP
Технология VPN позволяет устанавливать безопасное сетевое соединение между компьютерами.
Для того чтобы VPN работала в контейнере, необходимо разрешить использование TUN/TAP устройств для контейнера.

*Схема работы Virtual Private Network*
![VPN](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vpn.png)

По умолчанию модуль TUN уже загружен в ядро, проверить это можно командой `lsmod`:
```
[root@virtuozzo ~]# lsmod | grep tun
tun                    27183  1
```

Если все-таки модуль отключен, то включить его можно командой `modprobe`:
```
[root@virtuozzo ~]# modprobe tun
```

Проброс модуля TUN в контейнер:
```
[root@virtuozzo ~]# vzctl set ct3 --devnodes net/tun:rw --save
Setting devices
Create /etc/tmpfiles.d/device-tun.conf
[root@virtuozzo ~]# prlctl exec ct3 ls -l /dev/net/tun
crw------- 1 root root 10, 200 Feb 10 13:12 /dev/net/tun
```

На этом настройка TUN окончена.
Далее необходимо установить ПО для работы с VPN.
Например одну из программ:
* [OpenVPN](http://openvpn.net)
* [tinc](http://tinc-vpn.org)
* [VTun](http://vtun.sourceforge.net)

### <a name='fuse'></a>FUSE
FUSE (Filesystem in Userspace) — модуль Linux-ядра, позволяющий создавать виртуальные файловые системы.
FUSE может пригодиться, например при монтировании Яндекс.Диска или других виртуальных файловых систем.

Для того, чтобы для контейнеров был доступен FUSE, его необходимо включить на хост-ноде:
```
[root@virtuozzo ~]# modprobe fuse
[root@virtuozzo ~]# lsmod | grep fuse
fuse                  106371  0
```

Также необходимо добавить модуль в автозагрузку, чтобы он подгружался автоматически при рестарте хост-ноды:
```
[root@virtuozzo ~]# echo fuse >> /etc/modules-load.d/vz.conf
```

Включение FUSE для контейнера:
```
[root@virtuozzo ~]# vzctl set ct3 --devnodes fuse:rw --save
Setting devices
Create /etc/tmpfiles.d/device-fuse.conf
[root@virtuozzo ~]# prlctl exec ct3 ls -l /dev/fuse
crw------- 1 root root 10, 229 Feb 10 13:42 /dev/fuse
```

Пример подключения Яндекс.Диска в контейнере:
```
[root@virtuozzo ~]# prlctl exec ct3 yum install fuse davfs2
[root@virtuozzo ~]# prlctl exec ct3 mount -t davfs https://webdav.yandex.ru /mnt/
Please enter the username to authenticate with server
https://webdav.yandex.ru or hit enter for none.
  Username: username
Please enter the password to authenticate user username with server
https://webdav.yandex.ru or hit enter for none.
  Password:  pass
```

## [⬆](#toc) <a name='vmachines'></a>Работа с виртуальными машинами
Помимо создания контейнеров, Virtuozzo 7 поддерживает создание и управление виртуальными машинами на базе QEMU/KVM.
Утилита `prlctl` имеет возможно создавать и управлять виртуальными машинами, помимо этого также доступно управление ВМ с помощью libvirt.

### <a name='create-vm'></a>Создание и запуск ВМ
Создание виртуальной машины практически ничем не отличается от создания контейнера:
```
[root@virtuozzo ~]# prlctl create vm1 --distribution rhel7 --vmtype vm
Creating the virtual machine...
Generate the VM configuration for rhel.
The VM has been successfully created.
```

Параметр `--distribution` указывает на семейство ОС или дистрибутив для оптимизации виртуального окружения.
Список всех официально поддерживаемых ОС:
```
[root@virtuozzo ~]# prlctl create vm1 -d list
The following values are allowed:
win-2000        	win-xp          	win-2003        	win-vista       
win-2008        	win-7           	win-8           	win-2012        
win-8.1         	win             	rhel            	rhel7           
suse            	debian          	fedora-core     	fc              
xandros         	ubuntu          	mandriva        	mandrake        
centos          	centos7         	psbm            	redhat          
opensuse        	linux-2.4       	linux-2.6       	linux           
mageia          	mint            	freebsd-4       	freebsd-5       
freebsd-6       	freebsd-7       	freebsd-8       	freebsd         
```

Для каждой виртуальной машины в каталоге `/vz/vmprivate/` создается собственная директория с именем `ВМ.pvm`:
```
[root@virtuozzo ~]# ls /vz/vmprivate/vm1.pvm/
config.pvs  config.pvs.backup  harddisk.hdd
```

У каждой ВМ имеется как минимум два файла:
* файл конфигурации `config.pvs`
* виртуальный жесткий диск `harddisk.hdd`

В файле конфигурации описываются параметры виртуальной машины в XML-формате.

В свою очередь виртуальный жесткий диск может быть двух типов:
* `plain` — диск с фиксированным размером
* `expanded` — диск с изменяемым размером

По умолчанию при создании виртуальной машины, создается expanded-диск с размером 65G.

Просмотр только что созданной ВМ:
```
[root@virtuozzo ~]# prlctl list -a
UUID                                    STATUS       IP_ADDR         T  NAME
{6fe60288-fe50-49fe-a68d-7a8330837358}  stopped      -               CT ct1
{2cdb07fd-a68a-4279-81c1-3d269460c2f7}  stopped      -               CT ct2
{485372f0-2ae3-4bfe-aa55-e556c37fea9f}  stopped      -               VM vm1
```

По аналогии с контейнерами установим необходимые параметры для виртуальной машины:
```
[root@virtuozzo ~]# prlctl set vm1 --device-set net0 --ipadd 192.168.0.180/24
[root@virtuozzo ~]# prlctl set vm1 --device-set net0 --ipadd FE80:0:0:0:20C:29FF:FE01:FB07
[root@virtuozzo ~]# prlctl set vm1 --nameserver 192.168.0.1,192.168.0.2
[root@virtuozzo ~]# prlctl set vm1 --onboot yes
[root@virtuozzo ~]# prlctl set vm1 --memsize 1024
[root@virtuozzo ~]# prlctl set vm1 --videosize 64
[root@virtuozzo ~]# prlctl set vm1 --cpus 2
[root@virtuozzo ~]# prlctl set vm1 --cpuunits 1000
[root@virtuozzo ~]# prlctl set vm1 --cpulimit 1024m
[root@virtuozzo ~]# prlctl set vm1 --cpumask 0-1
[root@virtuozzo ~]# prlctl set vm1 --ioprio 6
[root@virtuozzo ~]# prlctl set vm1 --iolimit 0
[root@virtuozzo ~]# prlctl set vm1 --iopslimit 0
```

Параметры виртуальной машины установлены, ее можно запускать, однако для установки гостевой ОС необходим образ ОС.
Для хранения образов ОС можно создать каталог и централизованно хранить все там:
```
[root@virtuozzo ~]# mkdir /vz/vmprivate/images/
[root@virtuozzo ~]# ls /vz/vmprivate/images/ -1
CentOS-7-x86_64-Minimal-1503-01.iso
9200.16384.WIN8_RTM.120725-1247_X64FRE_SERVER_EVAL_RU-RU-HRM_SSS_X64FREE_RU-RU_DV5.ISO
```

Ознакомительные образы Windows Server можно найти по адресу: https://www.microsoft.com/ru-ru/evalcenter/evaluate-windows-server-2012

Установка ВМ с образа `CentOS-7-x86_64-Minimal-1503-01.iso`:
```
[root@virtuozzo ~]# prlctl set vm1 --device-set cdrom1 --image "/vz/vmprivate/images/CentOS-7-x86_64-Minimal-1503-01.iso" --iface scsi --position 1
```

Изменение размера диска до 8G:
```
[root@virtuozzo ~]# prl_disk_tool resize --hdd /vz/vmprivate/vm1.pvm/harddisk.hdd --size 8G
```

Просмотр конфигурации виртуальной машины перед ее запуском:
```
[root@virtuozzo ~]# prlctl list vm1 -i | grep Hardware -A9
Hardware:
  cpu cpus=2 VT-x accl=high mode=32 cpuunits=1000 cpulimit=1024Mhz ioprio=6 iolimit='0' mask=0-1
  memory 1024Mb
  video 64Mb 3d acceleration=highest vertical sync=yes
  memory_guarantee auto
  hdd0 (+) scsi:0 image='/vz/vmprivate/vm1.pvm/harddisk.hdd' type='expanded' 8192Mb subtype=virtio-scsi
  cdrom0 (+) ide:0 image='/vz/vmprivate/vm1.pvm/cloud-config.iso'
  cdrom1 (+) scsi:1 image='/vz/vmprivate/images/CentOS-7-x86_64-Minimal-1503-01.iso' subtype=virtio-scsi
  usb (+)
  net0 (+) dev='vme4292dc5f' network='Bridged' mac=001C4292DC5F card=virtio ips='192.168.0.180/255.255.255.0 FE80:0:0:0:20C:29FF:FE01:FB07/64 '
```

Подключение VNC для ВМ:
```
[root@virtuozzo ~]# prlctl set vm1 --vnc-mode manual --vnc-port 5901 --vnc-passwd Oiwaiqud
Configure VNC: Remote display: mode=manual port=5901
```

Для каждой виртуальной машины должен быть установлен уникальный порт для VNC.

Запуск виртуальной машины:
```
[root@virtuozzo ~]# prlctl start vm1
Starting the VM...
The VM has been successfully started.
```

Теперь к ней можно подключиться по VNC:
```
user@localhost ~ $ sudo apt-get install xvnc4viewer
user@localhost ~ $ xvnc4viewer 192.168.0.150:5901
Password: Oiwaiqud
```

*Подключенная VNC-сессия к виртуальной машине*
![VNC](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/vnc.png)

Далее следует обычная установка ОС в виртуальную машину.
По окончании установки необходимо перезагрузиться.

*Установленная гостевая ОС*
![Установленная гостевая ОС](https://raw.githubusercontent.com/Amet13/virtuozzo-tutorial/master/images/installed-os.png)

После установки ОС, можно соединиться к ней по SSH:
```
user@localhost ~ $ ssh root@192.168.0.180
root@192.168.0.180's password: eihaixahghath7A
[root@vm1 ~]#
```

### <a name='guest-tools'></a>Дополнения гостевой ОС
Virtuozzo поддеживает Virtuozzo Guest Tools (дополнения гостевой ОС), которые позволяют выполнять некоторые операции в ВМ такие как:
* запуск команд в ВМ с помощью `prlctl exec`
* установка паролей для пользователей с помощью `prlctl set --userpasswd`
* управление сетевыми настройками в ВМ

Установка дополнений для `vm1` с хост-ноды:
```
[root@virtuozzo ~]# prlctl installtools vm1
Installing...
The Parallels tools have been successfully installed.
```

Далее необходимо войти в ВМ, например по SSH и запустить скрипт установки дополнений:
```
[root@virtuozzo ~]# ssh root@192.168.0.180
root@192.168.0.180's password: eihaixahghath7A
[root@vm1 ~]# mount /dev/cdrom /mnt/
mount: /dev/sr0 is write-protected, mounting read-only
[root@vm1 ~]# bash /mnt/install
Preparing...                          ################################# [100%]
Updating / installing...
   1:qemu-guest-agent-vz-2.5.0-19.el7 ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:prl_nettool-7.0.1-3.vz7          ################################# [100%]
Preparing...                          ################################# [100%]
Updating / installing...
   1:vz-guest-udev-7.0.0-2            ################################# [100%]
Done!
```

Проверка корректности установки дополнений с хост-ноды:
```
[root@virtuozzo ~]# prlctl exec vm1 uname -a
Linux vm1.tld 3.10.0-229.el7.x86_64 #1 SMP Fri Mar 6 11:36:42 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
[root@virtuozzo ~]# prlctl set vm1 --userpasswd testuser:iel9cophoo2Aisa
Authentication tokens updated successfully.
[root@virtuozzo ~]# prlctl exec vm1 id testuser
uid=1000(testuser) gid=1000(testuser) groups=1000(testuser)
```

В гостевой Windows установка дополнений сводится к трем пунктам:
* установка драйвера, который находится в `<CD_root>/vioserial/<Win_version>/amd64/vioser.inf`
* запуск `prl_nettool_<Win_arch>.msi` и `qemu-ga-<Win_arch>.msi`
* проверка работоспособности сервиса `qemu-ga.exe`

## [⬆](#toc) <a name='roadmap'></a>Планы Virtuozzo
* 2016 — Virtuozzo 7 Technical Preview 2 — Containers.
Цель: живая миграция контейнеров с использованием CRIU
* 2016 — Virtuozzo 7 Technical Preview 2 — Virtual machines
* 2016 — Virtuozzo RC
* 2016 — Virtuozzo RTM

## [⬆](#toc) <a name='links'></a>Ссылки
* https://docs.openvz.org
* https://src.openvz.org/projects
* https://wiki.openvz.org/Main_Page
* https://lists.openvz.org/mailman/listinfo/devel
* https://bugs.openvz.org/secure/Dashboard.jspa

## [⬆](#toc) <a name='todo'></a>TODO
* Создание шаблона приложения для автоматического создания контейнера (https://bugs.openvz.org/browse/OVZ-6682)
* Создание шаблона гостевой ОС на основе vztt/vzmktmpl
* Шаблоны для виртуальных машин, добавление устройств, команды управления, CPU hotplug, оптимизация памяти с KSM
* Проброс устройств (nfs/pptp/usb/vlan) (http://habrahabr.ru/post/210460/)
* Онлайн-миграции (все еще недоступно в текущей версии)
* Управление сетью в Virtuozzo (veth/vlan/шейпинг)
* Снапшоты и клонирование шаблонов
* Бекапы

## [⬆](#toc) <a name='license'></a>Лицензия
![CC BY-SA 4.0](https://licensebuttons.net/l/by-sa/4.0/88x31.png)

[Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/deed.ru)

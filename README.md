Лабораторная работа №2 — Аутентификация с помощью LDAP

Цель работы
Научиться настраивать LDAP-сервер и подключать к нему LDAP-клиентов

LDAP (Lightweight Directory Access Protocol — легковесный протокол доступа к каталогам) —  это протокол для хранения и получения данных из каталога с иерархической структурой.
LDAP не является протоколом аутентификации или авторизации 

С увеличением числа серверов затрудняется управление пользователями на этих сервере. LDAP решает задачу централизованного управления доступом. 
С помощью LDAP можно синхронизировать:
UID пользователей
Группы (GID)
Домашние каталоги
Общие настройки для хостов 
И т. д. 

LDAP работает на следующих портах: 
389/TCP — без TLS/SSL
636/TCP — с TLS/SSL

Основные компоненты LDAP



Атрибуты — пара «ключ-значение». Пример атрибута: mail: admin@example.com
Записи (entry) — набор атрибутов под именем, используемый для описания чего-либо

Пример записи:
dn: sn=Ivanov, ou=people, dc=digitalocean,dc=com
objectclass: person
sn: Ivanov
cn: Ivan Ivanov

Data Information Tree (DIT) — организационная структура, где каждая запись имеет ровно одну родительскую запись и под ней может находиться любое количество дочерних записей. Запись верхнего уровня — исключение

На основе LDAP построено много решений, например: Microsoft Active Directory, OpenLDAP, FreeIPA и т. д.

В данной работе будет рассмотрена установка и настройка FreeIPA. FreeIPA — это готовое решение, включающее в себе:
Сервер LDAP на базе Novell 389 DS c предустановленными схемами
Сервер Kerberos
Предустановленный BIND с хранилищем зон в LDAP
Web-интерфейс управления

1) Установка FreeIPA сервера

Для начала нам необходимо настроить FreeIPA-сервер.
Для этого потребуется ВМ с Linux CentOS 7/8, к которой нужно подключиться по SSH и перейти в root-пользователя: sudo -i 

Начнём настройку FreeIPA-сервера: 
Установим часовой пояс: timedatectl set-timezone Europe/Moscow
Установим утилиту chrony: yum install -y chrony
Запустим chrony и добавим его в автозагрузку: systemctl enable chronyd --now
Если требуется, поменяем имя нашего сервера: hostnamectl set-hostname <имя сервера>
Выключим Firewall: systemctl stop firewalld
Отключим автозапуск Firewalld: systemctl disable firewalld
Остановим Selinux: setenforce 0
Поменяем в файле /etc/selinux/config, параметр Selinux на disabled
vi /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of these three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

Для дальнейшей настройки FreeIPA нам потребуется, чтобы DNS-сервер хранил запись о нашем LDAP-сервере. В рамках данной лабораторной работы мы не будем настраивать отдельный DNS-сервер и просто добавим запись в файл /etc/hosts
vi /etc/hosts

127.0.0.1   localhost localhost.localdomain 
127.0.1.1 ipa.testdomain.lan ipa
192.168.57.10 ipa.testdomain.lan ipa


Установим модуль DL1: yum install -y @idm:DL1
Установим FreeIPA-сервер: yum install -y ipa-server

Запустим скрипт установки: ipa-server-install
Далее, потребуется указать параметры нашего LDAP-сервера, после ввода каждого параметра нажимаем Enter, если нас устраивает параметр, указанный в квадратных скобках, то можно сразу нажимать Enter:

Do you want to configure integrated DNS (BIND)? [no]: no
Server host name [ipa.testdomain.lan]: <Нажимаем Enter>
Please confirm the domain name [testdomain.lan]: <Нажимаем Enter>
Please provide a realm name [TESTDOMAIN.LAN]: <Нажимаем Enter>
Directory Manager password: <Указываем пароль минимум 8 символов>
Password (confirm): <Дублируем указанный пароль>
IPA admin password: <Указываем пароль минимум 8 символов>
Password (confirm): <Дублируем указанный пароль>
NetBIOS domain name [TESTDOMAIN]: <Нажимем Enter>
Do you want to configure chrony with NTP server or pool address? [no]: no
The IPA Master Server will be configured with:
Hostname:       ipa.testdomain.lan
IP address(es): 192.168.57.10
Domain name:    testdomain.lan
Realm name:     TESTDOMAIN.LAN

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=TESTDOMAIN.LAN
Subject base: O=TESTDOMAIN.LAN
Chaining:     self-signed
Проверяем параметры, если всё устраивает, то нажимаем yes
Continue to configure the system with these values? [no]: yes


Далее начнётся процесс установки. Процесс установки занимает примерно 10-15 минут (иногда время может быть другим). Если мастер успешно выполнит настройку FreeIPA то в конце мы получим сообщение: 
The ipa-server-install command was successful

При вводе параметров установки мы вводили 2 пароля:
Directory Manager password — это пароль администратора сервера каталогов, У этого пользователя есть полный доступ к каталогу.
IPA admin password — пароль от пользователя FreeIPA admin

После успешной установки FreeIPA, проверим, что сервер Kerberos может выдать нам билет: 
[root@ipa ~]# kinit admin
Password for admin@TESTDOMAIN.LAN:  #Указываем Directory Manager password
[root@ipa ~]# klist           #Запросим список билетов Kerberos
Ticket cache: KCM:0
Default principal: admin@TESTDOMAIN.LAN

Valid starting     Expires            Service principal
08/02/22 18:18:25  08/03/22 17:32:39  krbtgt/TESTDOMAIN.LAN@TESTDOMAIN.LAN
[root@ipa ~]# 

Для удаление полученного билета воспользуемся командой: kdestroy

Мы можем зайти в Web-интерфейс нашего FreeIPA-сервера, для этого на нашей хостой машине нужно прописать следующую строку в файле Hosts:
192.168.57.10 ipa.testdomain.lan

В Unix-based системах файл хост находится по адресу /etc/hosts, в Windows — c:\Windows\System32\Drivers\etc\hosts. Для добавления строки потребуются права администратора.

После добавления DNS-записи откроем c нашей хост-машины веб-страницу:



Откроется окно управления FreeIPA-сервером. В имени пользователя укажем admin, в пароле укажем наш IPA admin password и нажмём войти. 



Откроется веб-консоль упрвления FreeIPA. Данные во FreeIPA можно вносить как через веб-консоль, так и средствами коммандной строки.

На этом установка и настройка FreeIPA-сервера завершена.

2) Установка FreeIPA клиента


$ vim /etc/hosts 
10.0.0.11 ipa.testdomain.lan ipa

$ yum install -y  freeipa-client
$ ipa-client-install --mkhomedir --domain=IVAN-GB.RU --server=ipa.testdomain.lan --no-ntp -p admin -w <password>

...

The ipa-client-install command was successful
Задание
Установить FreeIPA-сервер на ВМ1
Установить FreeIPA-клиент на ВМ2
Создать в панеле управления FreeIPA пользователя 
Зайти на ВМ2 за созданного пользователя на FreeIPA-сервере по SSH. Аутентификация должна осуществляться с проверкой LDAP, в случае если пользователь в системе отсутствует/заходит впервые, то система должна предложить сгенерировать новый пароль. 



Рекомендуемые источники
Статья о LDAP - https://ru.wikipedia.org/wiki/LDAP
Статья о настройке FreeIPA - https://www.dmosk.ru/miniinstruktions.php?mini=freeipa-centos
FreeIPA wiki - https://www.freeipa.org/page/Wiki_TODO
Статья «Chapter 13. Preparing the system for IdM client installation» - https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/installing_identity_management/preparing-the-system-for-ipa-client-installation_installing-identity-management
Статья «про LDAP по-русски» - https://pro-ldap.ru/ 

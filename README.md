# karma

SRE/DevOps
Test Cases
Задачи
1. Nginx раздаёт видео в mp4 по byte-range. Нужно написать конфиг,
который ограничивает скорость потока в 8 Мбит. Видео доступно по
временной ссылке в течение 1 недели. По истечении ссылка
должна возвращать код 410.


server {
	listen 80;
		server_name secure-link-test;
		root /var/www;
		location / {
		#8mbit/sec ~ 1MB/sec
			limit_rate 1m;
			secure_link $arg_md5,$arg_expires;
			secure_link_md5 "$secure_link_expires$uri secret";

			if ($secure_link = "") { return 403; }
			if ($secure_link = "0") { return 410; }
		}
}

date -d "2024-10-31 23:59" +%s
1730433540

echo -n '1730433540/test.mp4 secret' | openssl md5 -binary | openssl base64 | tr +/ -_ | tr -d =
al8UgK_4nDsFyeSiJ0NE7w

curl -I 'http://secure-link-test/test.mp4?md5=al8UgK_4nDsFyeSiJ0NE7w&expires=1730433540'
HTTP/1.1 200 OK


date -d "2024-10-21 23:59" +%s
1729569540

echo -n '1729569540/test.mp4 secret' | openssl md5 -binary | openssl base64 | tr +/ -_ | tr -d =
xmyGb9qdUznfhjyO9bib4w

curl -I 'http://secure-link-test/test.mp4?md5=xmyGb9qdUznfhjyO9bib4w&expires=1729569540'
HTTP/1.1 410 Gone

2. На сервере с Nginx высокая нагрузка из-за неверного конфига. Мы
изменили конфиг и пришли к выводу, что нужно раскатать новый
конфиг и перезапустить все процессы Nginx с новым конфигом.
Серверов 120 штук. Напишите команду.

- для этого следует использовать инструмент автоматизации управления инфраструктурой - Ansible мой выбор.
В inventory должны быть предварительно заведены соответствующие сервера.
Раскатываемый конфиг должен быть использован как шаблон в соответствующей ansible role.

- bash
for server in $(cat server_list.txt); do rsync --rsync-path="sudo rsync" -avz ~/nginx_test.conf user@$server:/etc/nginx/nginx_test.conf; ssh user@$server 'sudo nginx -t && sudo systemctl reload nginx || true'; done

3. Идёт большой поток запросов на Nginx приводящий к ошибкам
410/403. Нужно кикнуть айпишники, которые генерируют больше 60
запросов с ошибками за 3 минуты в бан на 45 минут. Решите любым
удобным инструментом.

- самое простое c помощью fail2ban:

[Definition]
failregex = <HOST>.*\"(GET|HEAD).*" (403|410)

[nginx-ban]
enabled  = true
filter   = nginx-ban
action   = iptables[name=nginx-ban, port=http, protocol=tcp]
logpath  = /var/log/nginx/access.log
maxretry = 10
findtime = 180
bantime  = 270
backend = auto

- самое правильное с помощью OpenResty и общим Redis для всех проектов

4. На некоторых из 120 серверов отвалился mount point. Нужно
перемонтировать. Перемонтирование может не сработать с
первого раза.

- аналогично 2, для этого следует использовать ansible
если нет, то например
for server in $(cat server_list.txt);  do ssh user@$line "mount -o remount /mnt/test || true"; done

где server_list.txt список серверов

5. MySQL работает в master-slave конфигурации. Каждый день
репликация останавливается. В статусе видим:
Last_SQL_Errno: 1205
Last_SQL_Error: Lock wait timeout exceeded; try restarting transaction
Какие могут быть причины?

- на slave не хватает памяти провести транзакцию
- на slave не хватает времени провести транзакцию

6. На сервере 4 винта: sda, sdb, sdc, sdd. Каждый разбит на 4 раздела.
1-й в md0 raid1 /boot
2-й в md1 raid0 swap
3-й в md2 raid6 /
4-й в xfs и смонтирован в /mnt/sd?4
Винт sdd вышел из строя, его заменили на новый. Включите его в
систему (набор команд) без перезагрузки сервера.

условие для raid6 минимум 4 диска - выполняется

sfdisk -d /dev/sda | sfdisk /dev/sdd
mdadm --manage /dev/md0 --add /dev/sdd1
mdadm --manage /dev/md1 --add /dev/sdd2
mdadm --manage /dev/md2 --add /dev/sdd3
mkfs.xfs /dev/sdd4 & mount /dev/sdd4 /mnt/sdd4

7. Нужно скопировать директорию с большим количеством мелких
файлов с сервера A на сервер B. Ключи ssh к обоим находятся
только у вас на компьютере. Вход только по ключам. Написать
команду в одну строку.

scp -r -o StrictHostKeyChecking=no user@192.168.100.102:~/test/ user@192.168.100.103:~/

Задачи со «звёздочкой»
8. На веб-приложение выросла нагрузка, количество запросов
увеличилось незначительно, но сервера уже не хватает. Выяснить
причину.

по какому-то из устанавливаемых параметров достигли установленного предела. например макс число коннектов для mysql

9. Тоже самое, что №1 но видео лежит в mp4, а отдаётся в HLS.
Напишите конфиг.

- не осилил

10. На MySQL выросла нагрузка, сервера не хватает. Выяснить причину.

если включен slow log(или другой иструмент диагностики например PMM) то смотрим туда, если нет то командой show full processlist смотрим текущие запросы и ищем долго выполняющийся

11. Каждый день рано утром прилетает алерт о недоступности вашего
высоконагруженного веб-сервера. Через ~5 минут прилетает
сообщение, что ошибки больше нет. Разберитесь в чём дело.

5 минут скорее всего минимальный квант мониторинга для указаного алерта.
"рано утром" скорее всего указывает на то что вряд-ли мы имеем дело с моментально возросшей нагрузкой.
наоборот скорее всего на раннее утро приходятся регулярные служебные события
итого сначала всегда ищем подтверждение события
если подтверждается то в данном случае смотрим регулярно происходящие события:
- на самом сервере: кроны, бэкапы
- на зависимых серверах: балансировщики, базы данных

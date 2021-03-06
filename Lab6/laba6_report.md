# Цель работы

Развить навыки администрирования ОС Linux. Получить первое практическое знакомство с технологией SELinux. Проверить работу SELinux на практике совместно с веб-сервером Apache.

# Подготовка лабораторного стенда

1.	Установим/обновим (за суперпользователя) веб-сервер Apache с помощью команды yum install httpd (Рис. 1)
   ![рис.1.](11.png)

2.	В конфигурационном файле /etc/httpd/httpd.conf зададим параметр ServerName: ServerName test.ru чтобы при запуске веб-сервера не выдавались лишние сообщения об ошибках, не относящихся к лабораторной работе.


3.	Также необходимо проследить, чтобы пакетный фильтр был отключен или в своей рабочей конфигурации позволял подключаться к 80-му и 81-му портам протокола tcp. Добавим разрешающие правила с помощью команд: 
	iptables -I INPUT -p tcp --dport 80 -j ACCEPT
	iptables -I INPUT -p tcp --dport 81 -j ACCEPT
	iptables -I OUTPUT -p tcp --sport 80 -j ACCEPT
	iptables -I OUTPUT -p tcp --sport 81 -j ACCEPT

   ![рис.2.](22.png)

# Выполнение лабораторной работы

1.	Войдем в систему с полученными учётными данными и убедимся, что SELinux работает в режиме enforcing политики targeted с помощью команд getenforce и sestatus

   ![рис.33.](33.png)

2. 	Обратимся к веб-серверу, запущенному на нашем компьютере, и убедимся, что последний работает: service httpd status (Рис. 4)
	Найдем веб-сервер Apache в списке процессов, определим его контекст безопасности, используем команду ps auxZ | grep httpd
	В нашем случае контекст безопасности unconfined_u:system_r:httpd_t
   ![рис.4.](44.png)

3. 	Посмотрим текущее состояние переключателей SELinux для Apache с помощью команды sestatus –b | grep httpd (Рис. 5)
	Многие из переключателей находятся в положении «off».
   ![рис.5.](55.png)

4. 	Посмотрим статистику по политике с помощью команды seinfo, также определим множество пользователей, ролей и типов.
	Пользователей: 9, ролей: 12, типов: 3920.
	Определим тип файлов и поддиректорий, находящихся в директории /var/www с помощью команды ls -lZ /var/www (Рис. 6)
   ![рис.6.](66.png)

5. Определим тип файлов, находящихся в директории /var/www/html с помощью команды ls –lZ /var/www/html (Рис. 7)

   ![рис.7.](77.png)

6. Определим круг пользователей, которым разрешено создание файлов в директории /var/www/html. 
	Видно, что только суперпользователь может создать файл в данной директории. (Рис. 8)
   ![рис.8.](88.png)
   
7. 	В следствие этого создадим от имени суперпользователя html-файл /var/www/html/test.html (Рис. 9) следующего содержания: 
	<html>
	<body>test</body>
	</html>
   ![рис.9.](99.png)
 
 
8. Проверим контекст созданного файла. (Рис. 10)
	Контекст, присваиваемый по умолчанию вновь созданным файлам в директории /var/www/html: unconfined_u:object_r:httpd_sys_content_t

   ![рис.10.](1010.png)

9. Обратимся к файлу через веб-сервер, введя в браузере firefox адрес http://127.0.0.1/test.html (Рис. 11)
	Изучим справку man httpd_selinux и выясним, какие контексты файлов определены для httpd и сопоставим их с типом файла test.html. Проверим контекст файла командой ls –Z /var/www/html/test.html
	Т.к. по умолчанию пользователи CentOS являются свободными (unconfined) от типа, созданному нами файлу test.html был сопоставлен SELinux, пользователь unconfined_u. Это первая часть контекста. Далее политика ролевого разделения доступа RBAC используется процессами, но не файлами, поэтому роли не имеют никакого значения для файлов. Роль object_r используется по умолчанию для файлов на «постоянных» носителях и на сетевых файловых системах. Тип httpd_sys_content_t позволяет процессу httpd получить доступ к файлу. Благодаря наличию последнего типа мы получили доступ к файлу при обращении к нему через браузер.
   ![рис.11.](1111.png)
   
10. Изменим контекст файла /var/www/html/test.html с httpd_sys_content_t на другой, к которому процесс httpd не должен иметь доступа, в нашем случае, на samba_share_t  (Рис. 12):
	chcon –t samba_share_t /var/www/html/test.html
	ls –Z /var/www/html/test.html

   ![рис.12.](1212.png)

11. Попробуем еще раз получить доступ к файлу через веб-сервер, введя в браузере firefox адрес http://127.0.0.1/test.html
	Мы получили сообщение об ошибке. 
	Проанализируем ситуацию, просмотрев log-файлы веб-сервера Apache, системный log-файл и audit.log при условии уже запущенных процессов setroubleshootd и audtd. (Рис. 13)
	Исходя из log-файлов, мы можем заметить, что проблема в измененном контексте на шаге 13, т.к. процесс httpd не имеет доступа на samba_share_t. В системе оказались запущены процессы setroubleshootd и audtd, поэтому ошибки, связанные с измененным контекстом, также есть в файле /var/log/audit/audit.log.
   ![рис.13.](1313.png)
   
12. Попробуем запустить веб-сервер Apache на прослушивание TCP-порта 81 (а не 80, как рекомендует IANA и прописано в /etc/services), заменив в файле /etc/httpd/conf/httpd.conf строчку Listen 80 на Listen 81.
	Перезапустим веб-сервер Apache и попробуем обратиться к файлу через веб-сервер, введя в браузере firefox адрес http://127.0.0.1/test.html (Рис. 14)
	Из того, что при запуске файла через браузер появилась ошибка, можно сделать предположение, что в списках портов, работающих с веб-сервером Apache, отсутствует порт 81.
   ![рис.14.](1414.png) 
   
13. Подтвердим свои догадки, проанализировав log-файлы: tail –n1  /var/log/messages и просмотрев файлы /var/log/http/error_log, /var/log/http/access_log и /var/log/audit/audit.log (Рис. 15)
	Во всех log-файлах появились записи, кроме /var/log/messages.
   ![рис.15.](1515.png)
   
14. Выполним команду semanage port –a –t http_port_t –p tcp 81
	После этого проверим список портов командой semanage port –l | grep http_port_t 
	Убедились, что порт 81 присутствует в списке. (Рис. 16)
   ![рис.16.](1616.png)    
   
15. Вернем контекст httpd_sys_content_t к файлу /var/www/html/test.html (Рис. 17):
	chcon –t httpd_sys_content_t /var/www/html/test.html
	После этого вновь попробуем получить доступ к файлу через веб-сервер, введя в браузере firefox адрес http://127.0.0.1:81/test.html
	Увидели слово содержимое файла - слово «test».

   ![рис.17.](1717.png)   
   
16. Исправим обратно конфигурационный файл apache, вернув Listen 80. 
	Удалим привязку http_port_t к 81 порту: semanage port –d –t http_port_t –p tcp 81. Данную команду выполнить невозможно на моей версии CentOS, поэтому получаем ошибку. 
	Удалим файл /var/www/html/test.html: rm /var/www/html/test.html (Рис. 18)
   ![рис.18.](1818.png)
   
# Выводы

Я развил навыки администрирования ОС Linux. Получил первое практическое знакомство с технологией SELinux. Проверил работу SELinux на практике совместно с веб-сервером Apache.

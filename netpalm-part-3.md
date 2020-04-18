## Введение в NetPalm — Часть 3(В работе)

В [первой части](http://blog.wimwauters.com/networkprogrammability/2020-04-14_netpalm_introduction_part1/) мы сосредоточились на получении конфигурации с устройств с помощью NetPalm. Во [второй части](https://blog.wimwauters.com/networkprogrammability/2020-04-15_netpalm_introduction_part2/) мы сосредоточились на изменении конфигурации.

А сейчас мы рассмотрим более продвинутые возможности NetPalm:
1. использование шаблонов Jinja2 для изменения конфигурации;
2. использование сервисных шаблонов.

### 3.1 Использование шаблонов Jinja2

NetPalm позволяет изменять конфигурацию на устройствах, используя шаблоны Jinja2 в сочетании с NAPALM и Netmiko.

В этой статье мы добавим несколько loopback интерфейсов на нашем устройстве, но с использованием NetPalm (с использованием NAPALM или Netmiko).

Сначала создадим шаблон Jinja2. Создадим файл `loopback.j2` и поместим его в папку `netpalm/backend/plugins/jinja2_templates`. 
Будем использовать следующий шаблон:

ВНИМАНИЕ тут накладочка с отображением на github.io
чтобы шаблон был верным надо удалить все `\`
```
\{\% for interface in interfaces \%\}
   interface \{\{interface.name\}\}
   description \{\{interface.description\}\}
   ip address \{\{interface.ipv4_addr\}\} \{\{interface.ipv4_mask\}\}
\{\% endfor \%\}
```
решил дополнительно сделать скрин всё же:
![imagej1](/images/part3/j1.png)

NetPalm предоставляет возможность работать с помощью шаблона Jinja2 (см. [здесь](https://documenter.getpostman.com/view/2391814/SzYbxcQx?version=latest#85056012-e546-41d7-b832-19e9285823f7) для Netmiko и [здесь](https://documenter.getpostman.com/view/2391814/SzYbxcQx?version=latest#44fdde62-c21c-417b-95d4-54fa2640d135) для NAPALM.

Пример:
![image1](/images/part3/1.png)

По сути, аргумент `template` указывает на шаблон, который мы сохранили в папке `netpalm/backend/plugins/jinja2_templates`, в нашем случае это будет `loopback` (не loopback.j2). Далее в аргументах нам нужно передать переменные для нашего шаблона Jinja2. В нашем случае мы хотим предоставить следующую информацию:
```
{
  "interfaces": [
    {
      "name": "Loopback3001",
      "description": "Description for Loopback 3001",
      "ipv4_addr": "200.200.200.230",
      "ipv4_mask": "255.255.255.255"
    },
    {
      "name": "Loopback3002",
      "description": "Description for Loopback 3002",
      "ipv4_addr": "200.200.200.231",
      "ipv4_mask": "255.255.255.255"
    }
  ]
}
```

Пример полного REST API вызова:
![image2](/images/part3/2.png)

И просмотр статуса задачи:
![image3](/images/part3/3.png)

Далее, давайте проверим, правильно всё ли верно отработало. Для этого мы будем использовать Netmiko `getconfig` API ([этот вариант](https://documenter.getpostman.com/view/2391814/SzYbxcQx?version=latest#4d45bf0a-f408-47e0-8dba-c0b26529591c)).
![image4](/images/part3/4.png)

Два loopback интерфейса были успешно добавлены:
![image5](/images/part3/5.png)

Или проверим через SSH:
```
csr1000v-1#show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       10.10.20.48     YES other  up                    up
GigabitEthernet2       unassigned      YES NVRAM  administratively down down
GigabitEthernet3       unassigned      YES NVRAM  administratively down down
Loopback76             unassigned      YES unset  up                    up
Loopback100            unassigned      YES unset  up                    up
Loopback1000           unassigned      YES unset  up                    up
Loopback2000           10.0.0.40       YES other  up                    up
Loopback3001           200.200.200.230 YES manual up                    up
Loopback3002           200.200.200.231 YES manual up                    up
```

### Использование сервисных шаблонов

NetPalm поддерживает работу с сервисными шаблонами. Сервисные шаблоны позволяют работать на уровне сервиса, а не на уровне отдельных устройств. Звучит довольно абстрактно, не так ли?

Представьте, что у вас есть десятки устройств, на которые вы хотите добавить дополнительные интерфейсы (или добавить интерфейсы к определенным VLAN, или создать trunk'овые порты и пр.). В первой и второй статьях мы видели, как этого добиться, используя `getconfig`/`setconfig` на каждом устройстве. Эти действия могут быть сложными и вызывать ошибки, если вам нужно повторить это для нескольких десятков устройств, возможно, устройств с разными операционными системами или от нескольких производителей.

Сервисные шаблоны позволят вам определить один шаблон, который может использоваться на нескольких устройствах. Сервисные шаблоны поддерживают всю логику Jinja. Другими словами, они позволяют запускать операторы IF/ELSE внутри отдельной операции (извлекать, создавать, удалять). Преимущество здесь в том, что нам нужно определить такой шаблон только один раз и он будет работать на устройствах для которых написан.

#### Сервисные шаблоны: обзор кода

Созданный сервисный шаблон необходимо хранить в папке `netpalm > backend > plugins > service_templates`. NetPalm поставляется с примерами шаблонов для начала работы. Для этой заметки я использую свой собственный шаблон сервиса (`loopback.j2`), хотя и очень вдохновлен существующими примерами.

Сервисный файл определяет несколько CRUD операций. Раздел для получения конфигурации с вашего устройства (RETRIEVE), добавления конфигурации (CREATE) и удаление конфигурации с вашего устройства (DELETE).

#### Раздел получения информации

Давайте сначала посмотрим на самое простое, что есть извлечение. Как видите, это часть шаблона Jinja, которому нам нужно передать переменные, такие как хост, имя пользователя и пароль. Также обратите внимание, что для получения интерфейсов мы будем использовать `netmiko`.

```
      {
        "operation": "retrieve",
        "payload": {
          "library": "netmiko",
          "connection_args": {
            "device_type": "cisco_xe",
            "host": "{{host}}",
            "username": "{{username}}",
            "password": "{{password}}",
            "port": "8181"
          },
        "command": "show ip interface brief"
        }
      }
```

Мы будем передавать в эти переменные через json-body как часть REST вызова. Ниже приведен пример:
```
{
	"operation": "retrieve",
	"args":{
		"hosts":["ios-xe-mgmt-latest.cisco.com"],
		"username":"developer",
		"password":"C1sco12345"
		
	}
}
```

Обладая этой информацией, NetPalm знает, что вы хотите получить информацию сервисным шаблоном, и выполнит команду, указанную вами в разделе `retrieve` сервисного шаблона.

Попробуем. Переходим в POSTMAN и делаем GET-запрос по адресу http://localhost:9000/service/loopback. Здесь слово loopback относится к имени шаблона обслуживания (но без расширения j2).
![image6](/images/part3/6.png)

В ответе есть ID задачи, который мы можем использовать для запроса `http://localhost:9000/task/id`. При этом будет показан список интерфейсов на нашем устройстве.
![image7](/images/part3/7.png)

#### Раздел добавления конфигурации

Cоздание\удаление разделов в целом аналогично вышеизложенному. Быстро переходим к делу.

Ниже приведена часть `create` сервисного шаблона.
```
      {
        "operation": "create",
        "payload": {
          "library": "napalm",
          "connection_args": {
            "device_type": "cisco_ios",
            "host": "{{host}}",
            "username": "{{username}}",
            "password": "{{password}}",
            "optional_args": {"port": 8181}
        },
        "j2config": {
    	    "template":"loopback_create",
    	    "args":{
              "interfaces": [
                {
                  "name": "Loopback250",
                  "description": "Description for Loopback 250",
                  "ipv4_addr": "200.200.200.20",
                  "ipv4_mask": "255.255.255.255"
                },
                {
                  "name": "Loopback251",
                  "description": "Description for Loopback 251",
                  "ipv4_addr": "200.200.200.21",
                  "ipv4_mask": "255.255.255.255"
                }
              ]
          }
    	  }
```

Выглядит довольно похоже на пример получения конфигурации. Тем не менее, здесь мы также передаем шаблон Jinja. Как это работает? NetPalm должен знать, что вы хотите создать на вашем устройстве. В нашем маленьком примере мы добавим два интерфейса, используя `napalm`.

Далее вам нужно будет создать файл в папке `netpalm > backend > plugins > jinja2_templates`. Мы назовем файл `loopback_create.j2` и он будет выглядеть следующим образом:
```
\{\% for interface in interfaces \%\}
   interface \{\{interface.name\}\}
   description \{\{interface.description\}\}
   ip address \{\{interface.ipv4_addr\}\} \{\{interface.ipv4_mask\}\}
\{\% endfor \%\}
```
![imagej2](/images/part3/j2.png)


Это шаблон Jinja2, на самом деле тот же самый, который мы использовали в первом разделе статьи. Нам нужно будет передать данные в шаблон, и это именно то, что делает для нас `j2config` часть секции `create`. Как видно из приведенного выше фрагмента шаблона сервиса, мы предоставляем список из двух интерфейсов.

Давайте попробуем. Вызов POSTMAN REST не меняется, за исключением того, что по понятным причинам мы указываем операцию `create`.
![image8](/images/part3/8.png)

Результат:
![image9](/images/part3/9.png)

Тоже самое через SSH:

![image10](/images/part3/10.png)

#### Удаления конфигурации раздел

С точки зрения логики секция удаления очень похожа на секцию создания. Также здесь нам нужно создать шаблон Jinja2 для удаления интерфейсов loopback, а затем обратиться к этому шаблону в разделе delete сервисного шаблона.

Сначала создайте файл `loopback_remove.j2` в папке `netpalm > backend > plugins > jinja2_templates`, следующим содержанием:
```
\{\% for interface in interfaces \%\}
   no interface \{\{interface.name\}\}
\{\% endfor \%\}
```
![imagej3](/images/part3/j3.png)

В приведенном выше фрагменте обратите внимание на слово `no`, означающее, что мы хотим удалить интерфейс с нашего устройства. Также обратите внимание, что нам нужно знать только имя интерфейса.

Далее рассмотрим операцию удаления:
```
     "operation": "delete",
        "payload": {
          "library": "napalm",
          "connection_args": {
            "device_type": "cisco_ios",
            "host": "{{host}}",
            "username": "{{username}}",
            "password": "{{password}}",
            "optional_args": {"port": 8181}
          },
        "j2config": {
    	    "template":"loopback_remove",
    	    "args":{
    		    "interfaces": [
                {
                  "name": "Loopback250"
                },
                {
                  "name": "Loopback251"
                }
              ]
    	    }
        }
      }
```

Очень похоже на раздел `create`, но здесь мы имеем в виду шаблон `loopback_remove` и предоставляем только имя интерфейса, который мы хотим удалить.

Так как мы имеем дело с функцией delete, то нам нужно указать операцию `delete` в JSON-body.
![image11](/images/part3/11.png)

Посмотрим результат в селекте POSTMAN'a:
![image12](/images/part3/12.png)

И в SSH:
```
csr1000v-1#show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       10.10.20.48     YES other  up                    up
GigabitEthernet2       10.255.255.2    YES other  administratively down down
GigabitEthernet3       10.255.255.2    YES other  up                    up
```

Весь сервисный шаблон - [тут](https://github.com/wiwa1978/blog-hugo-netlify-code/blob/master/Netpalm_Introduction/netpalm/backend/plugins/service_templates/loopback.j2).

Пока все, ребята. Надеюсь, вы начали видеть преимущества сервисных шаблонов NetPalm.

## Спасибо.


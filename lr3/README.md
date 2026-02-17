
# Отчет по лабораторной работе №3
## Тема: Ansible + Caddy

### Ход работы

### Подготовка файлов конфигурации

ansible.cfg

```cfg
[defaults]
host_key_checking = false
inventory = inventory/hosts
```

inventory/hosts

```
[my_servers]
local_server ansible_host=104.28.196.109 ansible_user=quwiier
```

### Создаём файлы inventory/hosts и ansible.cfg

### Создаём виртуальное окружение

![alt text](<imgs/Screenshot_83.png>)

### В виртуальном окружении устанавливаем absible с помощью pip

![alt text](<imgs/Screenshot_84.png>)

### Проверяем, что ansible видит хост

![alt text](<imgs/Screenshot_85.png>)

![alt text](<imgs/Screenshot_86.png>)

### Создаём файл test.txt

![alt text](<imgs/Screenshot_87.png>)
![alt text](<imgs/Screenshot_88.png>)

### Инициализируем caddy
![alt text](<imgs/Screenshot_89.png>)

### Прописываем roles/caddy_deploy/tasks/main.yml и ./caddy_deploy.yml

### Запускаем проверяем, что всё работает
![alt text](<imgs/Screenshot_90.png>)

### Регестрируем домен

![alt text](<imgs/domen.jpg>)

### Создаём template используя Jinja

### Прописываем vars для использования в шаблонах

### Добавляем обработку шаблонов и перезагрузку в таски

### Вновь запускаем плейбук

![alt text](<imgs/Screenshot_91.png>)

### Убеждаемся, что всё работает

![alt text](<imgs/Screenshot_92.png>)

### Ответы на вопросы

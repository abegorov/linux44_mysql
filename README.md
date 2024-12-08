# Репликация MySQL

## Задание

В материалах приложена ссылка на дамп базы `bet.dmp`. Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы: `bookmaker`, `competition`, `market`, `odds`, `outcome`. Настроить **GTID** репликацию.

## Реализация

Задание сделано на **rockylinux/9** версии **v9.5-20241118.0**. Для автоматизации процесса написан **Ansible Playbook** [playbook.yml](playbook.yml) который последовательно запускает следующие роли:

## Запуск

Необходимо скачать **VagrantBox** для **rockylinux/9** версии **v9.5-20241118.0** и добавить его в **Vagrant** под именем **rockylinux/9/v9.5-20241118.0**. Сделать это можно командами:

```shell
curl -OL https://dl.rockylinux.org/pub/rocky/9.5/images/x86_64/Rocky-9-Vagrant-Vbox-9.5-20241118.0.x86_64.box
vagrant box add Rocky-9-Vagrant-Vbox-9.5-20241118.0.x86_64.box --name "rockylinux/9/v9.5-20241118.0"
rm Rocky-9-Vagrant-Vbox-9.5-20241118.0.x86_64.box
```

Для того, чтобы **vagrant 2.3.7** работал с **VirtualBox 7.1.0** необходимо добавить эту версию в **driver_map** в файле **/usr/share/vagrant/gems/gems/vagrant-2.3.7/plugins/providers/virtualbox/driver/meta.rb**:

```ruby
          driver_map   = {
            "4.0" => Version_4_0,
            "4.1" => Version_4_1,
            "4.2" => Version_4_2,
            "4.3" => Version_4_3,
            "5.0" => Version_5_0,
            "5.1" => Version_5_1,
            "5.2" => Version_5_2,
            "6.0" => Version_6_0,
            "6.1" => Version_6_1,
            "7.0" => Version_7_0,
            "7.1" => Version_7_0,
          }
```

После этого нужно сделать **vagrant up**.

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.3.7**
- **VirtualBox 7.1.4_SUSE r165100**
- **Ansible 2.18.1**
- **Python 3.11.11**
- **Jinja2 3.1.4**

## Проверка

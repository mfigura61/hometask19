# OTUS ДЗ 19 PXE
-----------------------------------------------------------------------
### Домашнее задание

    Настройка PXE сервера для автоматической установки
Цель: Отрабатываем навыки установки и настройки DHCP, TFTP, PXE загрузчика и автоматической загрузки
1. Следуя шагам из документа https://docs.centos.org/en-US/8-docs/advanced-install/assembly_preparing-for-a-network-install установить и настроить загрузку по сети для дистрибутива CentOS8
В качестве шаблона воспользуйтесь репозиторием https://github.com/nixuser/virtlab/tree/main/centos_pxe
2. Поменять установку из репозитория NFS на установку из репозитория HTTP
3. Настройить автоматическую установку для созданного kickstart файла (*) Файл загружается по HTTP
* 4. автоматизировать процесс установки Cobbler cледуя шагам из документа https://cobbler.github.io/quickstart/

#### Что было сделано.

1.Внесем изменния в Вагрантфайл. Как, спустя несколько часов поисков проблемы, выяснилось, чтобы инсталляция успешно пошла надо , чтобы виртуалки имели более 2 гб озу.

``` vb.memory = "4096"
```
2. Дальнейшие изменения вносим в файл-скрипт конфигурации виртуалок. Нам необходимо переделать сетевую установку с nfs на http. Для этого переделаем пути к файлам и задействуем в системе небольшой http сервер.Заодно закомментим все, что связано с nfs.

Сразу на старте скриптом установим его с другими пакетами.
``` yum -y install python3-twisted
```

Основные изменения :
```
append initrd=images/CentOS-8.2/initrd.img ip=enp0s3:dhcp inst.repo=http://10.0.0.20:8080
LABEL linux-auto
  menu label ^Auto install system
  kernel images/CentOS-8.2/vmlinuz
  append initrd=images/CentOS-8.2/initrd.img ip=enp0s3:dhcp inst.ks=http://10.0.0.20:8080/ks.cfg inst.repo=http://10.0.0.20:8080
```
Чтобы было проще, все нужные файлы я сложил в одну папку и отдал ее по http.

```
mkdir pxe_common
cp /home/vagrant/cfg/ks.cfg /home/vagrant/pxe_common/
rsync -a /mnt/centos8-autoinstall/ /home/vagrant/pxe_common/
twistd  web --path /home/vagrant/pxe_common/

```
Инсталляция вручную запускается, автоматическая тоже, что и требовалось. оба файла модифицированного стенда во вложении.




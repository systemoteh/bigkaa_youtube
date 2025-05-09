# Среда разработки на базе kubernetes

Для разработки достаточно сделать однонодовый кластер kubernetes. Где он будет располагаться не важно. В моём случае это будет виртуальная машина 8CPU 16Gb RAM.

## Планируемые приложения

Все необходимые для разработки приложения планируется размешать к kubernetes.

В качестве дистрибутива kubernetes будет использоваться k3s.

Планируемые приложения:

- Gitlab.
- Gitlab runner.
- Harbor.
- Контейнер с инструментами разработчика.
  - Доступ по ssh.
  - Домашняя директория пользователя смонтирована как внешний том.
  - Установленные инструменты разработчика (компиляторы, доп утилиты) в домашнюю директорию пользователя.
- База данных PostgreSQL без кластера.
- Minio.
- ArgoCD.

## Операционная система

В качестве OS для виртуальной машины используется Rocky Linux 9.

Приложения устанавливает по своему желанию. Непосредственно на этой виртуальной машине разработчики работать не будут.

Обязательным к установке является DNS сервер. Он будет использоваться для поддержки внутренних доменов типа local и т.п.

### DNS сервер

```shell
sudo su

dnf install bind bind-utils -y
```

В файле `/etc/named.conf` вносим следующие изменения:

```text
options {
        listen-on port 53 { any; };
        recursion yes;
        dnssec-validation no;
};        
zone "systemoteh.ru" IN {
        type master;
        file "systemoteh.ru";
};        
```

Файл `/var/named/systemoteh.ru`:

```zone
$TTL 1D
@       IN SOA  @ k3s.systemoteh.ru. (
                2025021200      ; serial
                1D      ; refresh
                1H      ; retry
                1W      ; expire
                3H )    ; minimum
        NS      dev
dev     A       192.168.1.30 ; это ip виртуальной машины с kubernetes
gitlab      IN CNAME dev
registry    IN CNAME dev
pg          IN CNAME dev
minio       IN CNAME dev
argocd-dev  IN CNAME dev
postgre     IN CNAME dev
```

Применяем конфигурационные файлы:

```shell
named-checkconf
named-checkzone systemoteh.ru /var/named/systemoteh.ru
systemctl start named
systemctl enable named
```

Заменяем IP адрес DNS в клиенте.

```shell
sed -i -e 's/^dns=.*/dns=192.168.1.30\;/' /etc/NetworkManager/system-connections/enp0s3.nmconnection
systemctl restart NetworkManager
```

Проверяем работоспособность клиента DNS:

```shell
cat /etc/resolv.conf
dig mail.ru
dig dev.systemoteh.ru
```

В файл /etc/hosts добавить записи
```shell
192.168.1.30 pg.systemoteh.ru
192.168.1.30 argocd-dev.systemoteh.ru
192.168.1.30 registry.systemoteh.ru
192.168.1.30 minio.systemoteh.ru
192.168.1.30 gitlab.systemoteh.ru
```

## k3s

k3s будем ставить в варианте с одной нодой. Мы не предусматриваем дальнейшего расширения
кластера. Если будем использовать много нод, придётся возиться с сетевой файловой системой, которую желательно вынести на отдельный сервер. В общем, многонодовый кластер для разработки - это отдельная песня для команды разработчиков.

Вместо встроенного ingress controller на базе traefik, будем использовать более привычный на базе nginx.

```shell
curl -sfL https://get.k3s.io | sh -s - server --default-local-storage-path "/var/k3s/storage" --data-dir "/var/k3s/data" --disable=traefik
```

```shell
service k3s status
kubectl version
# if kubectl doesn't work
vi ~/.bashrc
# add new alias
alias kubectl='/usr/local/bin/kubectl'
# save the file and restart current session
kubectl version
kubectl get pods -A
```

Не забудьте скопировать файл `/etc/rancher/k3s/k3s.yaml` на машины, где вы планируете
обращаться к кластеру k3s.

Кластер установлен.

Необходимо добавить forward DNS и ip-записи в ConfigMap coredns -n kube-system
```yaml
.:53 {
    forward . 8.8.8.8 1.1.1.1  # Публичные DNS-серверы
        # ... остальные настройки
    hosts /etc/coredns/NodeHosts {
      ttl 60
      reload 15s
      192.168.1.30 gitlab.systemoteh.ru
      192.168.1.30 argocd-dev.systemoteh.ru
      192.168.1.30 registry.systemoteh.ru
      192.168.1.30 minio.systemoteh.ru
      fallthrough
  }
        # ... остальные настройки
}
    

```

Перезапуск пода coredns
```shell
kubectl -n kube-system get pods | grep coredns

kubectl -n kube-system delete pod coredns-ccb96694c-mshgk
```

## Установка приложений

[Установка приложений](k3s-application.md).

## Контейнер разработчика

[Контейнер разработчика](ws.md).

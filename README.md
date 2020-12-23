# chudinanton_platform
chudinanton Platform repository
<details>
<summary> <b>ДЗ №4 - kubernetes-network (Сетевая подсистема Kubernetes )</b></summary>

- [x] Основное ДЗ

- [x] Дополнительно ДЗ №1

- [x] Дополнительно ДЗ №2

- [x] Дополнительно ДЗ №3

### <b>Основное задание 1 - ClusterIP </b>

<pre>
Удобны в тех случаях, когда:
- Нам не надо подключаться к конкретному поду сервиса
- Нас устраивается случайное расределение подключений между подами
- Нам нужна стабильная точка подключения к сервису, независимая от
подов, нод и DNS-имен

Например:
- Подключения клиентов к кластеру БД (multi-read) или хранилищу
- Простейшая (не совсем, use IPVS, Luke) балансировка нагрузки внутри
кластера
</pre>

### <b>Основное задание 2 - Включение IPVS </b>
<pre>
Что такое IPVS
IPVS (IP Virtual Server) реализует балансировку нагрузки на транспортном уровне, обычно называемую коммутацией LAN уровня 4, как часть ядра Linux.

IPVS работает на хосте и действует как балансировщик нагрузки перед кластером реальных серверов. IPVS может направлять запросы для служб на основе TCP и UDP к реальным серверам и заставлять службы реальных серверов отображаться как виртуальные службы на одном IP-адресе.

IPVS против IPTABLES
Режим IPVS был представлен в Kubernetes v1.8, идет бета-версия в v1.9 и GA в v1.11. Режим IPTABLES был добавлен в v1.1 и стал рабочим режимом по умолчанию, начиная с v1.2. И IPVS, и IPTABLES основаны на netfilter. Различия между режимами IPVS и IPTABLES заключаются в следующем:

IPVS обеспечивает лучшую масштабируемость и производительность для больших кластеров.

IPVS поддерживает более сложные алгоритмы балансировки нагрузки, чем IPTABLES (наименьшая нагрузка, наименьшее количество соединений, местонахождение, взвешенное значение и т. Д.).

IPVS поддерживает проверку работоспособности сервера, повторные попытки подключения и т. Д.
</pre>

Полезная команда в toolbox внутри ВМ
<pre>
ipvsadm --list -n
</pre>

### <b>Основное задание 3 - Установка MetalLB</b>

<pre>
MetalLB позволяет запустить внутри кластера L4-балансировщик,
который будет принимать извне запросы к сервисам и раскидывать их
между подами.
</pre>
Не ясно что лучше использовать после лекции. IPVS или MetalLB.
В ДЗ видно что они работают вместе.

- Cделали Load balancer web-svc-lb для web с опубликованным 80-м портом
<pre>
#kubectl describe svc web-svc-lb                                                                      
Name:                     web-svc-lb
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=web
Type:                     LoadBalancer
IP:                       10.107.131.31
LoadBalancer Ingress:     172.17.255.1
Port:                     <unset>  80/TCP
TargetPort:               8000/TCP
NodePort:                 <unset>  32546/TCP
Endpoints:                172.17.0.6:8000,172.17.0.7:8000,172.17.0.8:8000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
</pre>
Присвоен ip <b>172.17.255.1</b>
- Добавили маршрут к сети в миникубе
- Проверил работоспособность 

### <b>Дополнительное задание №1 - DNS через MetalLB</b> 
- Сделаны два манифеста coredns-tcp-svc-lb.yaml и coredns-udp-svc-lb.yaml где создаются сервисы со статичным ip <b>172.17.255.10</b> в ns kube-system с kube-dns

Документация по шарингу сервиса на одном ip:

https://metallb.universe.tf/usage/#ip-address-sharing

Необходимо использовать 
<pre>
annotations:
    metallb.universe.tf/allow-shared-ip
</pre>

<pre>
# nslookup kubernetes.default.svc.cluster.local 172.17.255.10
Server:         172.17.255.10
Address:        172.17.255.10:53


Name:   kubernetes.default.svc.cluster.local
Address: 10.96.0.1
</pre>


### <b>Основное задание 4 - Cоздаем "коробочный" Ingress контроллер ingress-nginx и proxy сервиса</b>
- Cоздание ingress-nginx
<pre>
#kubectl get all -n ingress-nginx                                                                 
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-clx7t        0/1     Completed   0          26m
pod/ingress-nginx-admission-patch-lgnhp         0/1     Completed   0          26m
pod/ingress-nginx-controller-5dbd9649d4-wqc4z   1/1     Running     0          26m

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
service/ingress-nginx                        LoadBalancer   10.98.9.95      172.17.255.2   80:32076/TCP,443:30751/TCP   25m
service/ingress-nginx-controller             NodePort       10.99.138.241   <none>         80:31905/TCP,443:31198/TCP   26m
service/ingress-nginx-controller-admission   ClusterIP      10.97.13.85     <none>         443/TCP                      26m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           26m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-5dbd9649d4   1         1         1       26m

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           7s         26m
job.batch/ingress-nginx-admission-patch    1/1           8s         26m
</pre>

Очень слабо освещено cоздание ingress-nginx

Для ingress-nginx присвоен ip <b>172.17.255.2</b>

- <b>Подключение приложение Web к Ingress (Создание Headless-сервиса)</b>
<pre>
Ingress-контроллер не требует ClusterIP для балансировки
трафика
Список узлов для балансировки заполняется из ресурса Endpoints
нужного сервиса (это нужно для "интеллектуальной" балансировки,
привязки сессий и т.п.)
Поэтому мы можем использовать headless-сервис для нашего вебприложения.
</pre>
Смотрим на различия в ClusterIP для опубликованного headless-сервиса:
<pre>
#kubectl get service/web-svc-cip                                                                    
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web-svc-cip   ClusterIP   10.107.67.253   <none>        80/TCP    58m
#kubectl get service/web-svc                                                                         
NAME      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
web-svc   ClusterIP   None         <none>        80/TCP    52s
</pre>

Довольно путанные названия сервисов в ДЗ. Лучше было для наглядности назвать web-svc-with-cip и web-svc-without-cip

- <b>Создание правил Ingress, настройка ingress-прокси</b>
<pre>
networking.k8s.io/v1beta1 уже устарела о чем выводится предупреждение
Warning: networking.k8s.io/v1beta1 Ingress is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
Лучше сразу делать на networking.k8s.io/v1 чуть изменив манифест
</pre>
Вновь придумано путанное имя web-ingress

Просмотр созданного правила:
<pre>
#kubectl describe ingress/web
Name:             web
Namespace:        default
Address:          192.168.153.4
Default backend:  default-http-backend:80 (error: endpoints "default-http-backend" not found)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /web   web-svc:8000 (172.17.0.6:8000,172.17.0.7:8000,172.17.0.8:8000)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason  Age                    From                      Message
  ----    ------  ----                   ----                      -------
  Normal  Sync    8m40s (x2 over 9m25s)  nginx-ingress-controller  Scheduled for sync
</pre>
Заметим что есть ошибка (error: endpoints "default-http-backend" not found)

### <b>Дополнительное задание №2 - Ingress для Dashboard</b>

Install

To deploy Dashboard, execute following command:
<pre>
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
</pre>
Создан dashboard-ingress.yaml методом гугления на 443 порту. Иначе не будет работать авторизация.

Используем аннотацию:
<pre>
nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
</pre>

Cервис становится доступным по следующей ссылке:
<pre>
http://172.17.255.2/dashboard/
</pre>
Авторизация отдельный квест :)
<pre>
kubectl -n kube-system get secret
kubectl -n kube-system describe secret deployment-controller-token-dtsgl
</pre>

### <b>Дополнительное задание №3 - Canary для Ingress</b>
Что нам нужно сделать?
- Создаем Deployment <b>prod</b> с нужным нам приложением.
- Создаем Headless-сервис svc-prod с app <b>prod</b> с соответсвующими портами
- Делаем Ingress правило для svc-prod
Проверяем:
<pre>
curl http://172.17.255.2/production
</pre>

- Создаем Deployment <b>canary</b> с нужным нам приложением и replicas: 1  на то она и canary
- Создаем Headless-сервис svc-canary с app <b>canary</b> с соответсвующими портами
- Создаем Ingress правило для svc-canary используя в аннотации canary-by-header и canary-by-header-value:

<pre>
  annotations:
    ... 
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "canary" 
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
    ...
</pre>
др. МАН:

https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary

БАГА: нужно прописать на хостовую машину с которой идет проверка hostname  в hosts:
<pre>
172.17.255.2 nginx-ingress.local
</pre>

Поверяем:
<pre>
#for i in $(seq 1 10); do curl -s -H "canary: true" http://nginx-ingress.local/production | grep "HOSTNAME="; done
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'

Если используем веса:
...
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "50"
....

#for i in $(seq 1 10); do curl -s http://nginx-ingress.local/production | grep "HOSTNAME="; done 
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='prod-7978c597c-8qsfc'
export HOSTNAME='prod-7978c597c-8qsfc'
export HOSTNAME='prod-7978c597c-7zg2m'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='canary-7ccb57755d-dtg4z'
export HOSTNAME='prod-7978c597c-7zg2m'
export HOSTNAME='prod-7978c597c-vhksb'
</pre>
</details>
<details>
<summary>
<b>ДЗ №3 - kubernetes-security (Безопасность и управление доступом 
)</b></summary>

- [x] Основное ДЗ

- [x] Дополнительно ДЗ №1

- [x] Дополнительно ДЗ №2

- [x] Дополнительно ДЗ №3

### <b>Основное задание 1</b>
- Создать Service Account bob, дать ему роль admin в рамках всего кластера
- Создать Service Account dave без доступа к кластеру

### <b>Основное задание 2</b>
- Создать Namespace prometheus
- Создать Service Account carol в этом Namespace
- Дать всем Service Account в Namespace prometheus возможность делать get, list, watch в отношении Pods всего кластера

### <b>Основное задание 3</b>
- Создать Namespace dev
- Создать Service Account jane в Namespace dev
- Дать jane роль admin в рамках Namespace dev
- Создать Service Account ken в Namespace dev
- Дать ken роль view в рамках Namespace dev
</details>
<details>
<summary>
<b>ДЗ №2 - kubernetes-controllers (Механика запуска и взаимодействия контейнеров в Kubernetes )</b></summary>

- [x] Основное ДЗ

- [x] Дополнительно ДЗ №1

- [x] Дополнительно ДЗ №2

- [x] Дополнительно ДЗ №3

### <b>Основное задание</b>

- Установка kind и поднятие кластера
- Изучение ReplicaSet контроллера
- Разворот frontend в ReplicaSet
<pre>
Контроллер ReplicaSet не позволяет проводить обновление pod'ов при изменении манифеста.
</pre>
- Изучение контроллера Deployment
- Сборка образов paymentservice
- Разворот paymentservice с помощью Deployment
- Обновление Deployment paymentservice и Rollback
- Изучение readinessProbe

### <b>Дополнительное задание №1</b>
- Написание двух стратегий обновления Deployment: Аналог blue-green и Reverse Rolling Update
### <b>Дополнительное задание №2</b>
- Изучение DaemonSet на примере Node Exporter и разворот на worker нодах c namespace monitoring
<pre>
Сдул манифест отсюда и вычистил все непонятное и лишнее на текущем этапе:
https://github.com/prometheus-operator/kube-prometheus/blob/master/manifests/node-exporter-daemonset.yaml
Образ взят отсюда, не стал собирать свой:
https://hub.docker.com/r/prom/node-exporter
</pre>
### <b>Дополнительное задание №3</b>
- Развернул Node Exporter на мастерах используя соответствующий допуск самому поду:
<pre>
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: "Exists"
        effect: NoSchedule
</pre>

Результат:
<pre>
# kubectl get daemonsets,pods -n monitoring                                                             
NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/node-exporter   6         6         6       6            6           <none>          101s

NAME                      READY   STATUS    RESTARTS   AGE
pod/node-exporter-2wdsh   1/1     Running   0          101s
pod/node-exporter-4drt9   1/1     Running   0          101s
pod/node-exporter-5kp8f   1/1     Running   0          101s
pod/node-exporter-blq6x   1/1     Running   0          101s
pod/node-exporter-mq8x6   1/1     Running   0          101s
pod/node-exporter-q6d2x   1/1     Running   0          101s
</pre>
</details>
<details>
<summary>
<b>ДЗ №1 kubernetes-intro (Знакомство с Kubernetes, основные понятия и архитектура )</b></summary>

- [x] Основное ДЗ

- [x] Дополнительно ДЗ №1

### <b>Основное задание</b>

- Подготовка репозитория для выполнения ДЗ. (travis + служебные файлы)

Настройка локального окружения:
- Установка kubectl
- Установка Minikube & Запуск Minikube
- Dashboard & k9s
- Изучение задания по автовостановлению системны подов после удаления.

Ответ:
<pre>
kube-scheduler, kube-controller-manager, kube-apiserver, etcd стартует благодаря kubelet, они описаны в манифестах /etc/kubernetes/manifests
Где в частности есть и проверки состояния livenessProbe

Документация:
https://kubernetes.io/docs/concepts/workloads/pods/#static-pods

coredns запущен как Deployments c ReplicaSet
Документация:
https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

kube-proxy запусскается как DaemonSet и перезапускается/запускатеся автоматически на каждой ноде.
Документация:
https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
</pre>
- Генерация Dockerfile из nginx:latest + загрузка Docker Hub
- Написание web-pod.yaml манифеста для запуска Pod web из созданного Dockerfile лежащего на Docker Hub
- Ознакомление с базовым траблшутингом для Pod
- Добавление Init контейнера генерирующий страницу index.html.
- Подключение Volumes
- Запуск приложения

Проверка работы приложения:
<pre>
kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
</pre>

- Установка и настройка kube-forwarder для удобства
- Генерация frontend Dockerfile загрузка на Docker Hub
- Изучение ad-hoc режима

### <b>Дополнительное задание Hipster Shop frontend</b>
- cd ..
- В результате, после применения исправленного манифеста pod
frontend должен находиться в статусе Running
</details>

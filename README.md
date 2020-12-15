# chudinanton_platform
chudinanton Platform repository
## ДЗ №3
 - [x] Основное ДЗ

### Основное задание 1
- Создать Service Account bob, дать ему роль admin в рамках всего кластера
- Создать Service Account dave без доступа к кластеру

### Основное задание 2
- Создать Namespace prometheus
- Создать Service Account carol в этом Namespace
- Дать всем Service Account в Namespace prometheus возможность делать get, list, watch в отношении Pods всего кластера

### Основное задание 3
- Создать Namespace dev
- Создать Service Account jane в Namespace dev
- Дать jane роль admin в рамках Namespace dev
- Создать Service Account ken в Namespace dev
- Дать ken роль view в рамках Namespace dev

## ДЗ №2
 - [x] Основное ДЗ
 - [x] Задание со *
 - [x] Задание со ** 

### Основное задание

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

### Дополнительное задание №1
- Написание двух стратегий обновления Deployment: Аналог blue-green и Reverse Rolling Update
### Дополнительное задание №2
- Изучение DaemonSet на примере Node Exporter и разворот на worker нодах c namespace monitoring
<pre>
Сдул манифест отсюда и вычистил все непонятное и лишнее на текущем этапе:
https://github.com/prometheus-operator/kube-prometheus/blob/master/manifests/node-exporter-daemonset.yaml
Образ взят отсюда, не стал собирать свой:
https://hub.docker.com/r/prom/node-exporter
</pre>
### Дополнительное задание №3
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

## ДЗ №1
 - [x] Основное ДЗ
 - [x] Задание со *

### Основное задание

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

### Дополнительное задание Hipster Shop frontend
- cd ..
- В результате, после применения исправленного манифеста pod
frontend должен находиться в статусе Running


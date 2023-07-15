# devops-DZ14.4-K8S-AppUpdate
# Домашнее задание к занятию «Обновление приложений»

### Цель задания

Выбрать и настроить стратегию обновления приложения.

### Чеклист готовности к домашнему заданию

1. Кластер K8s.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).
2. [Статья про стратегии обновлений](https://habr.com/ru/companies/flant/articles/471620/).

-----

### Задание 1. Выбрать стратегию обновления приложения и описать ваш выбор

1. Имеется приложение, состоящее из нескольких реплик, которое требуется обновить.
2. Ресурсы, выделенные для приложения, ограничены, и нет возможности их увеличить.
3. Запас по ресурсам в менее загруженный момент времени составляет 20%.
4. Обновление мажорное, новые версии приложения не умеют работать со старыми.
5. Вам нужно объяснить свой выбор стратегии обновления приложения.

### Задание 2. Обновить приложение

1. Создать deployment приложения с контейнерами nginx и multitool. Версию nginx взять 1.19. Количество реплик — 5.
2. Обновить версию nginx в приложении до версии 1.20, сократив время обновления до минимума. Приложение должно быть доступно.
3. Попытаться обновить nginx до версии 1.28, приложение должно оставаться доступным.
4. Откатиться после неудачного обновления.

## Дополнительные задания — со звёздочкой*

Задания дополнительные, необязательные к выполнению, они не повлияют на получение зачёта по домашнему заданию. **Но мы настоятельно рекомендуем вам выполнять все задания со звёздочкой.** Это поможет лучше разобраться в материале.

### Задание 3*. Создать Canary deployment

1. Создать два deployment'а приложения nginx.
2. При помощи разных ConfigMap сделать две версии приложения — веб-страницы.
3. С помощью ingress создать канареечный деплоймент, чтобы можно было часть трафика перебросить на разные версии приложения.

### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

-----

## Ответ



# Задание 1

Если **недоступность** приложения на период развёртывания **допустима**, то из-за ограниченности ресурсов и несовместимости версий можно выбрать вариант обновления `Recreate`.
Старые реплики будут уничтожены, вместо них запущены обновленные. Поскольку обновление мажорное, то части приложения будут обновлены все вместе, это защитит от проблем с совместимостью.
Если же **недоступность** приложения **недопустима**, то выберем стратегию `RollingUpdate`. Будем переходить на новую версию, постепенно удаляя старые поды после того как поднимутся новые. Можно указать лимиты в Deployment в 20% (вместо 25%). Переводим на новые реплики трафик по какому-нибудь признаку.

# Задание 2

1. Создал `deploy1.yml` (версия `Nginx` **1.19**). Запустил развертывание. Проверил статус

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeployment
  labels:
    app: nginx-multitool
  annotations:
    kubernetes.io/change-cause: "Nginx 1.19"
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 85%
      maxUnavailable: 85%
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
          - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
          - containerPort: 8080
        env:
          - name: HTTP_PORT
            value: "8080"
          - name: HTTPS_PORT
            value: "11443"
```

[Ссылка на deploy1.yml](deploy1.yml)

```bash
vk@vkvm:~/14_4$ kubectl apply -f deploy1.yml
deployment.apps/mydeployment created

vk@vkvm:~/14_4$ kubectl get deployments,replicasets -o wide
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS        IMAGES                               SELECTOR
deployment.apps/mydeployment   5/5     5            5           7s    nginx,multitool   nginx:1.19,wbitt/network-multitool   app=nginx-multitool

NAME                                      DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES                               SELECTOR
replicaset.apps/mydeployment-748b869d84   5         5         5       7s    nginx,multitool   nginx:1.19,wbitt/network-multitool   app=nginx-multitool,pod-template-hash=748b869d84
```

2. Создал `deploy2.yml`(изменение - версия `Nginx` **1.20**). Запустил развертывание. Проверил статус

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeployment
  labels:
    app: nginx-multitool
  annotations:
    kubernetes.io/change-cause: "Nginx 1.20"
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 85%
      maxUnavailable: 85%
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
          - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
          - containerPort: 8080
        env:
          - name: HTTP_PORT
            value: "8080"
          - name: HTTPS_PORT
            value: "11443"
```

[Ссылка на deploy2.yml](deploy2.yml)

```bash
vk@vkvm:~/14_4$ kubectl apply -f deploy2.yml
deployment.apps/mydeployment configured
vk@vkvm:~/14_4$ kubectl get deployments,replicasets -o wide
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS        IMAGES                               SELECTOR
deployment.apps/mydeployment   5/5     5            5           45s   nginx,multitool   nginx:1.20,wbitt/network-multitool   app=nginx-multitool

NAME                                      DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES                               SELECTOR
replicaset.apps/mydeployment-6c98cfbb56   5         5         5       6s    nginx,multitool   nginx:1.20,wbitt/network-multitool   app=nginx-multitool,pod-template-hash=6c98cfbb56
replicaset.apps/mydeployment-748b869d84   0         0         0       45s   nginx,multitool   nginx:1.19,wbitt/network-multitool   app=nginx-multitool,pod-template-hash=748b869d84
```

3. Создал `deploy3.yml`(изменение - версия `Nginx` **1.28**). Запустил развертывание. Проверил статус

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeployment
  labels:
    app: nginx-multitool
  annotations:
    kubernetes.io/change-cause: "Nginx 1.28"
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 85%
      maxUnavailable: 85%
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:1.28
        ports:
          - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
          - containerPort: 8080
        env:
          - name: HTTP_PORT
            value: "8080"
          - name: HTTPS_PORT
            value: "11443"
```

[Ссылка на deploy3.yml](deploy3.yml)

```bash
vk@vkvm:~/14_4$ kubectl apply -f deploy3.yml
deployment.apps/mydeployment configured
vk@vkvm:~/14_4$ kubectl get deployments,replicasets -o wide
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS        IMAGES                               SELECTOR
deployment.apps/mydeployment   1/5     5            1           100s   nginx,multitool   nginx:1.28,wbitt/network-multitool   app=nginx-multitool

NAME                                      DESIRED   CURRENT   READY   AGE    CONTAINERS        IMAGES                               SELECTOR
replicaset.apps/mydeployment-6c98cfbb56   1         1         1       61s    nginx,multitool   nginx:1.20,wbitt/network-multitool   app=nginx-multitool,pod-template-hash=6c98cfbb56
replicaset.apps/mydeployment-6f585b5848   5         5         0       18s    nginx,multitool   nginx:1.28,wbitt/network-multitool   app=nginx-multitool,pod-template-hash=6f585b5848
replicaset.apps/mydeployment-748b869d84   0         0         0       100s   nginx,multitool   nginx:1.19,wbitt/network-multitool   app=nginx-multitool,pod-template-hash=748b869d84
```

Видим, что **приложение остается доступным, т.к. часть подов остаётся доступной**. Но что-то пошло не так, разбираемся...

Проверил логи, события

```bash
vk@vkvm:~/14_4$ kubectl get pods
NAME                            READY   STATUS             RESTARTS   AGE
mydeployment-6c98cfbb56-vfmdf   2/2     Running            0          4m3s
mydeployment-6f585b5848-8sh4q   1/2     ImagePullBackOff   0          3m19s
mydeployment-6f585b5848-bplnl   1/2     ImagePullBackOff   0          3m20s
mydeployment-6f585b5848-gnvfj   1/2     ImagePullBackOff   0          3m19s
mydeployment-6f585b5848-lm29z   1/2     ImagePullBackOff   0          3m19s
mydeployment-6f585b5848-lx4kl   1/2     ImagePullBackOff   0          3m19s

vk@vkvm:~/14_4$ kubectl logs mydeployment-6f585b5848-8sh4q
Defaulted container "nginx" out of: nginx, multitool
Error from server (BadRequest): container "nginx" in pod "mydeployment-6f585b5848-8sh4q" is waiting to start: trying and failing to pull image

vk@vkvm:~/14_4$ kubectl get events --field-selector involvedObject.name=mydeployment-6f585b5848-8sh4q
LAST SEEN   TYPE      REASON      OBJECT                              MESSAGE
5m15s       Normal    Scheduled   pod/mydeployment-6f585b5848-8sh4q   Successfully assigned default/mydeployment-6f585b5848-8sh4q to node2
4m23s       Normal    Pulling     pod/mydeployment-6f585b5848-8sh4q   Pulling image "nginx:1.28"
4m21s       Warning   Failed      pod/mydeployment-6f585b5848-8sh4q   Failed to pull image "nginx:1.28": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:1.28": failed to resolve reference "docker.io/library/nginx:1.28": docker.io/library/nginx:1.28: not found
4m21s       Warning   Failed      pod/mydeployment-6f585b5848-8sh4q   Error: ErrImagePull
5m12s       Normal    Pulling     pod/mydeployment-6f585b5848-8sh4q   Pulling image "wbitt/network-multitool"
5m9s        Normal    Pulled      pod/mydeployment-6f585b5848-8sh4q   Successfully pulled image "wbitt/network-multitool" in 1.004181855s (2.81756382s including waiting)
5m9s        Normal    Created     pod/mydeployment-6f585b5848-8sh4q   Created container multitool
5m9s        Normal    Started     pod/mydeployment-6f585b5848-8sh4q   Started container multitool
2s          Normal    BackOff     pod/mydeployment-6f585b5848-8sh4q   Back-off pulling image "nginx:1.28"
3m42s       Warning   Failed      pod/mydeployment-6f585b5848-8sh4q   Error: ImagePullBackOff
```

Посмотрел историю развёртываний

```bash
vk@vkvm:~/14_4$ kubectl rollout history deployment mydeployment
deployment.apps/mydeployment 
REVISION  CHANGE-CAUSE
1         Nginx 1.19
2         Nginx 1.20
3         Nginx 1.28
```

Отменил последнее развёртывание, проверил статус

```bash
vk@vkvm:~/14_4$ kubectl rollout undo deployment mydeployment
deployment.apps/mydeployment rolled back

vk@vkvm:~/14_4$ kubectl get deployments,replicasets -o wide
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS        IMAGES                               SELECTOR
deployment.apps/mydeployment   5/5     5            5           8m56s   nginx,multitool   nginx:1.20,wbitt/network-multitool   app=nginx-multitool

NAME                                      DESIRED   CURRENT   READY   AGE     CONTAINERS        IMAGES                               SELECTOR
replicaset.apps/mydeployment-6c98cfbb56   5         5         5       8m17s   nginx,multitool   nginx:1.20,wbitt/network-multitool   app=nginx-multitool,pod-template-hash=6c98cfbb56
replicaset.apps/mydeployment-6f585b5848   0         0         0       7m34s   nginx,multitool   nginx:1.28,wbitt/network-multitool   app=nginx-multitool,pod-template-hash=6f585b5848
replicaset.apps/mydeployment-748b869d84   0         0         0       8m56s   nginx,multitool   nginx:1.19,wbitt/network-multitool   app=nginx-multitool,pod-template-hash=748b869d84

vk@vkvm:~/14_4$ kubectl rollout history deployment mydeployment
deployment.apps/mydeployment 
REVISION  CHANGE-CAUSE
1         Nginx 1.19
3         Nginx 1.28
4         Nginx 1.20
```

Убедились, что **REVISION Nginx 1.20 поменялся с 2 на 4**

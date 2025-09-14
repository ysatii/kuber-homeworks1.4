# Домашнее задание к занятию «Сетевое взаимодействие в Kubernetes»

### Примерное время выполнения задания

120 минут

### Цель задания

Научиться настраивать доступ к приложениям в Kubernetes:
- Внутри кластера через **Service** (ClusterIP, NodePort).
- Снаружи кластера через **Ingress**.

Это задание поможет вам освоить базовые принципы сетевого взаимодействия в Kubernetes — ключевого навыка для работы с кластерами.
На практике Service и Ingress используются для доступа к приложениям, балансировки нагрузки и маршрутизации трафика. Понимание этих механизмов поможет вам упростить управление сервисами в рабочих окружениях и снизит риски ошибок при развёртывании.

------

## **Подготовка**
### **Чеклист готовности**
- Установлен Kubernetes (MicroK8S, Minikube или другой).
- Установлен `kubectl`.
- Редактор для YAML-файлов (VS Code, Vim и др.).

------

### Инструменты, которые пригодятся для выполнения задания

1. [Инструкция](https://microk8s.io/docs/getting-started) по установке MicroK8S.
2. [Инструкция](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download) по установке Minikube. 
3. [Инструкция](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)по установке kubectl.
4. [Инструкция](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools) по установке VS Code

### Дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/services-networking/service/) Описание Service.
3. [Описание](https://kubernetes.io/docs/concepts/services-networking/ingress/) Ingress.
4. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

## **Задание 1: Настройка Service (ClusterIP и NodePort)**
### **Задача**
Развернуть приложение из двух контейнеров (`nginx` и `multitool`) и обеспечить доступ к ним:
- Внутри кластера через **ClusterIP**.
- Снаружи через **NodePort**.

### **Шаги выполнения**
1. **Создать Deployment** с двумя контейнерами:
   - `nginx` (порт `80`).
   - `multitool` (порт `8080`).
   - Количество реплик: `3`.
2. **Создать Service типа ClusterIP**, который:
   - Открывает `nginx` на порту `9001`.
   - Открывает `multitool` на порту `9002`.
3. **Проверить доступность** изнутри кластера:
```bash
 kubectl run test-pod --image=wbitt/network-multitool --rm -it -- sh
 curl <service-name>:9001 # Проверить nginx
 curl <service-name>:9002 # Проверить multitool
```
4. **Создать Service типа NodePort** для доступа к `nginx` снаружи.
5. **Проверить доступ** с локального компьютера:
```bash
 curl <node-ip>:<node-port>
   ```
 или через браузер.

### **Что сдать на проверку**
- Манифесты:
  - `deployment-multi-container.yaml`
  - `service-clusterip.yaml`
  - `service-nodeport.yaml`
- Скриншоты проверки доступа (`curl` или браузер).
---
## Решение 1
Создадим свое пространство имен netology-1-4
листинг ns.yaml
```
apiVersion: v1
kind: Namespace
metadata:
  name: netology-1-4
```

Выполнение
```
kubectl apply -f ns.yaml
```

Листинг deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: netology-1-4
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
              readOnly: true
      volumes:
        - name: html
          configMap:
            name: web-index
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool-deploy
  namespace: netology-1-4
spec:
  replicas: 3
  selector:
    matchLabels:
      app: multitool
  template:
    metadata:
      labels:
        app: multitool
    spec:
      containers:
        - name: multitool
          image: wbitt/network-multitool:alpine-minimal
          env:
            - name: HTTP_PORT
              value: "8080"
          ports:
            - containerPort: 8080

```

Выполнение 
kubectl apply -f deployment.yaml
 

заставим nginx отвечать нужной нам страницей что бе отличать от стандартного 
Листинг configmap-webindex.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-index
  namespace: netology-1-4
data:
  index.html: |
    <html>
      <body style="font-family:sans-serif">
        <h1>Hello from nginx (one-container-per-pod)</h1>
        <p>This is port 80.</p>
      </body>
    </html>
```

```
kubectl apply -f configmap-webindex.yaml
```

Листинг service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: netology-1-4
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - name: http-nginx
      port: 9001
      targetPort: 80
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: multitool-svc
  namespace: netology-1-4
spec:
  type: ClusterIP
  selector:
    app: multitool
  ports:
    - name: http-multitool
      port: 9002
      targetPort: 8080
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  namespace: netology-1-4
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - name: http-nginx
      port: 80
      targetPort: 80
      nodePort: 30080
      protocol: TCP

```
команда
```
kubectl apply -f service.yaml
```

## проверка внутри кластера. Создадим контейнер и с него будем выполнять проверки
kubectl -n netology-1-4 run test-pod --image=wbitt/network-multitool -it --rm -- sh
# в строке  shell внутри контейнера:
curl nginx-svc:9001
curl multitool-svc:9002

## выйдем из контейнера и произведем проверку, сначала проверим какой ип у ноды
```
minikube ip
```

curl http://192.168.49.2:30080/  

![рисунок 1](https://github.com/ysatii/kuber-homeworks1.4/blob/main/img/img_1.jpg)  
![рисунок 2](https://github.com/ysatii/kuber-homeworks1.4/blob/main/img/img_2.jpg)  


[ссылка на файлы развертыванияи](https://github.com/ysatii/kuber-homeworks1.4/tree/main/k8s)  


---




## **Задание 2: Настройка Ingress**
### **Задача**
Развернуть два приложения (`frontend` и `backend`) и обеспечить доступ к ним через **Ingress** по разным путям.

### **Шаги выполнения**
1. **Развернуть два Deployment**:
   - `frontend` (образ `nginx`).
   - `backend` (образ `wbitt/network-multitool`).
2. **Создать Service** для каждого приложения.
3. **Включить Ingress-контроллер**:
```bash
 microk8s enable ingress
   ```
4. **Создать Ingress**, который:
   - Открывает `frontend` по пути `/`.
   - Открывает `backend` по пути `/api`.
5. **Проверить доступность**:
```bash
 curl <host>/
 curl <host>/api
   ```
 или через браузер.

### **Что сдать на проверку**
- Манифесты:
  - `deployment-frontend.yaml`
  - `deployment-backend.yaml`
  - `service-frontend.yaml`
  - `service-backend.yaml`
  - `ingress.yaml`
- Скриншоты проверки доступа (`curl` или браузер).

---
## Решение
### Создадим пространство имен 
Листинг ns.yaml
```
apiVersion: v1
kind: Namespace
metadata:
  name: netology-1-4
```

команда 
```
microk8s kubectl apply -f ns.yaml
```

### подымим поды
листнг deployment-frontend.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
```

команда
```
microk8s kubectl apply -n netology-1-4 -f deployment-frontend.yaml
```

листинг deployment-backend.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
  namespace: netology-1-4
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: http-echo
          image: hashicorp/http-echo:0.2.3
          args:
            - "-text=backend OK"
            - "-listen=:8000"
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /
              port: 8000
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
```
команда
```
microk8s kubectl apply -n netology-1-4 -f deployment-backend.yaml
```

### подымим сервисы 
листинг service-frontend.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```

команда 
```
microk8s kubectl apply -n netology-1-4 -f service-frontend.yaml
```

листинг  service-backend.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 8000
      targetPort: 8000
      protocol: TCP
```
команда
```
microk8s kubectl apply -n netology-1-4 -f service-backend.yaml
```

подымим ingress контроллер
```
microk8s kubectl apply -n netology-1-4 -f ingress.yaml
```

### проверка подов и сервисов
проверка подов 
microk8s kubectl get pods  -n netology-1-4 -o wide

проверка сервисов
microk8s kubectl get svc -n netology-1-4 -o wide

проверим ингресс
microk8s kubectl get ing -n netology-1-4


 в ingress.yaml указан host: demo.local — добавим его в /etc/hosts



### проверки доступности бэкенда и фронтента
curl -i http://demo.local/
curl -i http://demo.local/api

![рисунок 1](https://github.com/ysatii/kuber-homeworks1.4/blob/main/img/img_3.jpg)  
![рисунок 2](https://github.com/ysatii/kuber-homeworks1.4/blob/main/img/img_4.jpg)  


[ссылка на файлы развертыванияи](https://github.com/ysatii/kuber-homeworks1.4/tree/main/k8s_ingress)  

## Шаблоны манифестов с учебными комментариями
### **1. Deployment (nginx + multitool)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: # ПРИМЕР: "multi-container-app"
spec:
  replicas: # ЗАДАНИЕ: Укажите количество реплик
  selector:
    matchLabels:
      app: # ДОПОЛНИТЕ: Метка для селектора
  template:
    metadata:
      labels:
        app: # ПОВТОРИТЕ метку из selector.matchLabels
    spec:
      containers:
 - name: # ЗАДАНИЕ: Название первого контейнера
        image: nginx
        ports:
 - containerPort: 80
 - name: multitool
        image: wbitt/network-multitool
        ports:
 - containerPort: 8080
        env:
 - name: HTTP_PORT
          value: "8080" # КЛЮЧЕВОЙ МОМЕНТ: Порт должен совпадать с containerPort
```
### **2. Ingress (для frontend и backend)**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: # ЗАДАНИЕ: Придумайте имя, допустим example-ingress
  annotations:  # ВАЖНО: Эта аннотация нужна для rewrite правил
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
 - http:
      paths:
 - path: /
        pathType: Prefix
        backend:
          service:
            name: # УКАЖИТЕ: Имя frontend Service
            port:
              number: 80
 - path: /api # КЛЮЧЕВОЙ ПУТЬ: API endpoint
        pathType: Prefix
        backend:
          service:
            name: # УКАЖИТЕ: Имя backend Service
            port:
              number: 80
```
---

## **Правила приёма работы**
1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

## **Критерии оценивания задания**
1. Зачёт: Все задачи выполнены, манифесты корректны, есть доказательства работы (скриншоты).
2. Доработка (на доработку задание направляется 1 раз): основные задачи выполнены, при этом есть ошибки в манифестах или отсутствуют проверочные скриншоты.
3. Незачёт: работа выполнена не в полном объёме, есть ошибки в манифестах, отсутствуют проверочные скриншоты. Все попытки доработки израсходованы (на доработку работа направляется 1 раз). Этот вид оценки используется крайне редко.

## **Срок выполнения задания**  
1. 5 дней на выполнение задания.
2. 5 дней на доработку задания (в случае направления задания на доработку).

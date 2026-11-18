# NetologyTaskK8S4

# Домашнее задание к занятию «Сетевое взаимодействие в Kubernetes»

## Задание 1. Настройка Service (ClusterIP и NodePort)

### 1. Создать Deployment с двумя контейнерами

Создан Deployment `multi-container-app`, содержащий два контейнера:

- `nginx` — порт `80`;
- `multitool` — порт `8080`.

Количество реплик — `3`.

Для `multitool` порт HTTP был изменён на `8080` с помощью переменной окружения `HTTP_PORT`, чтобы избежать конфликта с nginx.

Проверка Deployment:

```
kubectl get pods
kubectl get deployment multi-container-app
```

Результат: работают три реплики, в каждой по два контейнера:

```
multi-container-app-...   2/2   Running
multi-container-app-...   2/2   Running
multi-container-app-...   2/2   Running
```

Deployment:

```
READY   UP-TO-DATE   AVAILABLE
3/3     3            3
```

<img width="590" height="392" alt="image" src="https://github.com/user-attachments/assets/72d761dd-e9e9-46ca-92e8-4ee2a62af52c" />

[Манифест deployment-multi-container.yaml](deployment-multi-container.yaml)

---

### 2. Создать Service типа ClusterIP

Создан Service `multi-container-svc`.

Он предоставляет:

- nginx через порт `9001`;
- multitool через порт `9002`.

Проверка:

```
kubectl get svc multi-container-svc
kubectl get endpoints multi-container-svc
```

Результат:

```
multi-container-svc   ClusterIP   10.152.183.144   <none>   9001/TCP,9002/TCP
```

Service обнаружил все три реплики приложения.

<img width="637" height="60" alt="image" src="https://github.com/user-attachments/assets/7c99016c-babf-42fe-985e-e38002abbf62" />

[Манифест service-clusterip.yaml](service-clusterip.yaml)

---

### 3. Проверить доступность изнутри кластера

Для проверки был запущен временный Pod:

```
kubectl run test-pod --image=wbitt/network-multitool --rm -it -- sh
```

Проверка nginx:

```
curl multi-container-svc:9001
```

Получен ответ:

```
Welcome to nginx!
```

Проверка multitool:

```
curl multi-container-svc:9002
```

Получен ответ:

```
WBITT Network MultiTool (with NGINX)
```

Это подтверждает доступность обоих контейнеров через ClusterIP Service.

<img width="1101" height="453" alt="image" src="https://github.com/user-attachments/assets/6e20f825-b37b-4168-9635-7c69f542e7e5" />

---

### 4. Создать Service типа NodePort

Создан Service `multi-container-nodeport` для доступа к nginx снаружи кластера.

Проверка:

```
kubectl get svc multi-container-nodeport
```

Результат:

```
multi-container-nodeport   NodePort   10.152.183.86   <none>   80:30080/TCP
```

[Манифест service-nodeport.yaml](service-nodeport.yaml)

---

### 5. Проверить доступ с локального компьютера

IP Kubernetes-ноды:

```
10.0.2.15
```

Проверка:

```
curl http://10.0.2.15:30080
```

Получен ответ nginx:

```
Welcome to nginx!
```

Это подтверждает, что приложение доступно через NodePort с локального компьютера.

<img width="994" height="514" alt="image" src="https://github.com/user-attachments/assets/87a16b3b-94a9-46e0-ba27-da65990b3388" />

---

## Задание 2. Настройка Ingress

### 1. Развернуть два Deployment

Созданы два приложения:

- `frontend` — образ `nginx`;
- `backend` — образ `wbitt/network-multitool`.

Проверка:

```
kubectl get pods
```

Оба приложения успешно запущены:

```
frontend-...   1/1   Running
backend-...    1/1   Running
```

[Манифест frontend](deployment-frontend.yaml)

[Манифест backend](deployment-backend.yaml)

---

### 2. Создать Service для каждого приложения

Созданы:

- `frontend-svc`;
- `backend-svc`.

Проверка:

```
kubectl get svc frontend-svc backend-svc
```

Результат:

```
frontend-svc   ClusterIP   ...   80/TCP
backend-svc    ClusterIP   ...   80/TCP
```

<img width="525" height="243" alt="image" src="https://github.com/user-attachments/assets/98d6ee49-6c27-430d-adae-eacc4043bca3" />

[Манифест service-frontend.yaml](service-frontend.yaml)

[Манифест service-backend.yaml](service-backend.yaml)

---

### 3. Включить Ingress-контроллер

Ingress-контроллер включён командой:

```
microk8s enable ingress
```

В используемой версии MicroK8s был установлен Traefik.

Проверка:

```
kubectl get pods -n ingress
```

Результат:

```
traefik-...   1/1   Running
```

---

### 4. Настроить маршрутизацию через Ingress

Настроены маршруты:

```
/      → frontend-svc
/api   → backend-svc
```

Так как используемый MicroK8s разворачивает Traefik, для маршрута `/api` был применён Middleware `StripPrefix`, который удаляет префикс `/api` перед передачей запроса в backend.

Проверка:

```
kubectl get ingress
kubectl get middleware
```

Результат:

```
backend-ingress    public   *
frontend-ingress   public   *
```

Middleware:

```
strip-api
```

<img width="438" height="153" alt="image" src="https://github.com/user-attachments/assets/a03ae4c5-33d6-4040-9aca-1f602b790d9c" />

[Манифест ingress.yaml](ingress.yaml)

---

### 5. Проверить доступность

Проверка frontend:

```
curl http://10.0.2.15/
```

Получен ответ:

```
Welcome to nginx!
```

Проверка backend:

```
curl http://10.0.2.15/api
```

Получен ответ:

```
WBITT Network MultiTool (with NGINX) - backend-... - HTTP: 80 , HTTPS: 443
```

Это подтверждает корректную маршрутизацию:

```
/      → frontend
/api   → backend
```

<img width="1011" height="453" alt="image" src="https://github.com/user-attachments/assets/8403d4b6-a335-4d9c-9a50-af71a1b22197" />

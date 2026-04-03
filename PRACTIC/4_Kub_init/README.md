# Пара 4 — Kubernetes: установка кластера, первые поды
**Время:** 80 минут
**Тема:** kubeadm, kubectl, первые объекты — понять как устроен кластер изнутри

---

## Цель занятия

К концу пары студент знает как запустить кластер, понимает роль каждого компонента и умеет создавать, проверять и удалять объекты.

---

## Что должно быть сделано к концу пары ✅

- [ ] Проверить состояние кластера (все ноды Ready, все компоненты healthy)
- [ ] Запустить первый под и зайти в него
- [ ] Изучить что внутри пода (hostname, env, filesystem)
- [ ] Создать Pod вручную через YAML
- [ ] Убедиться что kubelet перезапускает упавший контейнер
- [ ] Посмотреть системные поды Control Plane
- [ ] Написать отличие Pod vs Container (1–2 предложения письменно)

---

## Окружение

> Используем уже подготовленный кластер (kubeadm или minikube).
> Если кластера нет — поднять за 5 минут:
```bash
# Вариант А: minikube (рекомендуется для лаб)
minikube start --cpus=2 --memory=4096

# Вариант Б: k3s (легковесный production-ready)
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

---

## Ход работы

### Блок 1 — Состояние кластера (15 мин)

```bash
# Ноды кластера
kubectl get nodes -o wide
kubectl describe node <имя-ноды> | head -50

# Компоненты Control Plane
kubectl get pods -n kube-system
kubectl get componentstatuses  # или: kubectl get cs

# Посмотреть конфиги статических подов Control Plane
ls /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -A5 "- --"

# API ресурсы кластера
kubectl api-resources | head -20

# Версия
kubectl version --short
```

**Вопрос:** Какие поды в `kube-system` всегда должны быть Running?

---

### Блок 2 — Первый Pod (20 мин)

```bash
# Запустить под императивно
kubectl run nginx --image=nginx:alpine --port=80
kubectl get pods
kubectl get pods -o wide   # на какой ноде?

# Следить за жизненным циклом
kubectl get pods -w  # Ctrl+C чтобы выйти

# Зайти внутрь
kubectl exec -it nginx -- sh
  # внутри:
  hostname         # имя пода
  cat /etc/hosts   # IP пода и DNS
  env | grep KUBE  # переменные окружения от K8s
  ps aux           # только наши процессы (PID namespace)
  ip addr          # свой IP (NET namespace)
  exit

# Логи
kubectl logs nginx
kubectl logs nginx -f   # следить в реальном времени

# Описание пода (события + состояние)
kubectl describe pod nginx
```

---

### Блок 3 — Pod через YAML (25 мин)

**Файл `pod.yaml`:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-webserver
  labels:
    app: webserver
    env: lab
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10

  - name: log-sidecar
    image: busybox:latest
    command: ["/bin/sh", "-c", "while true; do echo $(date) >> /var/log/access.log; sleep 5; done"]
    volumeMounts:
    - name: logs
      mountPath: /var/log

  volumes:
  - name: logs
    emptyDir: {}
```

```bash
# Применить
kubectl apply -f pod.yaml
kubectl get pods -w  # ждём Running

# Посмотреть оба контейнера в поде
kubectl get pod my-webserver -o jsonpath='{.spec.containers[*].name}'

# Логи из конкретного контейнера
kubectl logs my-webserver -c log-sidecar

# Войти в конкретный контейнер
kubectl exec -it my-webserver -c nginx -- sh

# Посмотреть YAML запущенного пода (с добавленными defaults)
kubectl get pod my-webserver -o yaml | head -60
```

---

### Блок 4 — Самовосстановление (10 мин)

```bash
# Убить основной процесс nginx — K8s должен перезапустить
kubectl exec my-webserver -c nginx -- kill 1

# Следить что происходит
kubectl get pods -w

# Посмотреть счётчик рестартов
kubectl get pod my-webserver
# RESTARTS должен увеличиться
```

**Вопрос:** Почему Pod не удалился, а перезапустился? Кто за это отвечает?

---

## Что сдать преподавателю

1. `kubectl get nodes` — все Ready
2. `kubectl get pods -n kube-system` — все системные поды Running
3. `kubectl get pods` — два пода (nginx и my-webserver) Running
4. `kubectl get pod my-webserver` — показать RESTARTS > 0 после kill

---

## Частые проблемы

| Проблема | Решение |
|----------|---------|
| Pod в состоянии `ImagePullBackOff` | Нет интернета или неверное имя образа |
| Pod `Pending` — не назначается нода | `kubectl describe pod` → Events → смотреть причину (taintmd, ресурсы) |
| `Error: container "log-sidecar" not found` | Опечатка в имени контейнера в `exec -c` |
| minikube не стартует | `minikube delete && minikube start` — сбросить состояние |

# Пара 7 — Безопасность Kubernetes: RBAC, NetworkPolicy, Falco
**Время:** 80 минут
**Тема:** Принцип минимальных привилегий — кто что может делать в кластере

---

## Цель занятия

К концу пары студент настраивает RBAC так, чтобы приложение имело только нужные права, изолирует сетевой трафик между подами и видит в реальном времени подозрительную активность через Falco.

---

## Что должно быть сделано к концу пары ✅

- [ ] Создать ServiceAccount с ограниченными правами (только read pods)
- [ ] Убедиться что SA не может удалять поды (`kubectl auth can-i`)
- [ ] Создать NetworkPolicy default-deny-all и разрешить только нужный трафик
- [ ] Проверить изоляцию — один под не видит другой
- [ ] (Бонус) Запустить Falco и сгенерировать alert при входе в контейнер

---

## Ход работы

### Блок 1 — RBAC (30 мин)

```bash
# Создать namespace для демо
kubectl create namespace rbac-demo
```

**`rbac.yaml`:**
```yaml
---
# ServiceAccount для нашего приложения
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-reader
  namespace: rbac-demo
---
# Role: только чтение подов в этом namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-demo
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
# Специально НЕ даём: create, delete, update
---
# RoleBinding: связываем SA с Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader-binding
  namespace: rbac-demo
subjects:
- kind: ServiceAccount
  name: app-reader
  namespace: rbac-demo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rbac.yaml

# Проверить права
kubectl auth can-i list pods \
  --namespace rbac-demo \
  --as=system:serviceaccount:rbac-demo:app-reader
# → yes

kubectl auth can-i delete pods \
  --namespace rbac-demo \
  --as=system:serviceaccount:rbac-demo:app-reader
# → no

kubectl auth can-i list pods \
  --namespace default \
  --as=system:serviceaccount:rbac-demo:app-reader
# → no (Role ограничена namespace rbac-demo)
```

**Запустить под от имени ServiceAccount:**

```yaml
# pod-rbac-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test
  namespace: rbac-demo
spec:
  serviceAccountName: app-reader
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["/bin/sh", "-c", "sleep 3600"]
```

```bash
kubectl apply -f pod-rbac-demo.yaml

# Войти в под и попробовать API
kubectl exec -it rbac-test -n rbac-demo -- sh

  # Внутри — kubectl использует SA токен автоматически
  kubectl get pods -n rbac-demo     # ✓ работает
  kubectl delete pod rbac-test -n rbac-demo  # ✗ Forbidden!
  kubectl get pods -n default        # ✗ Forbidden!
  exit
```

---

### Блок 2 — NetworkPolicy (30 мин)

```bash
kubectl create namespace netpol-demo
```

**Запустить тестовые поды:**
```bash
# Frontend
kubectl run frontend -n netpol-demo --image=nginx:alpine --labels=role=frontend
kubectl expose pod frontend -n netpol-demo --port=80 --name=frontend-svc

# Backend
kubectl run backend -n netpol-demo --image=nginx:alpine --labels=role=backend
kubectl expose pod backend -n netpol-demo --port=80 --name=backend-svc

# Database
kubectl run database -n netpol-demo --image=nginx:alpine --labels=role=database
kubectl expose pod database -n netpol-demo --port=80 --name=database-svc
```

**Проверить что ВСЕ могут общаться (до политик):**
```bash
# frontend → backend → OK
kubectl exec frontend -n netpol-demo -- wget -qO- backend-svc
# frontend → database → OK (это плохо!)
kubectl exec frontend -n netpol-demo -- wget -qO- database-svc
```

**`networkpolicies.yaml`:**
```yaml
---
# 1. Default: запретить весь входящий трафик
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: netpol-demo
spec:
  podSelector: {}  # применить ко ВСЕМ подам
  policyTypes:
  - Ingress
  # ingress: []  — пустой список = ничего не разрешаем
---
# 2. Разрешить frontend принимать внешний трафик (от ingress)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Ingress
  ingress:
  - {}  # разрешить всем
---
# 3. Backend принимает только от frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
---
# 4. Database принимает только от backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-from-backend
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      role: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend
```

```bash
kubectl apply -f networkpolicies.yaml

# Проверить изоляцию
kubectl exec frontend -n netpol-demo -- wget -qO- --timeout=3 backend-svc
# ✓ Работает (разрешено)

kubectl exec frontend -n netpol-demo -- wget -qO- --timeout=3 database-svc
# ✗ Timeout! (запрещено — frontend не может напрямую к database)

kubectl exec backend -n netpol-demo -- wget -qO- --timeout=3 database-svc
# ✓ Работает (backend → database разрешено)

# Нарисовать на доске схему разрешённых потоков
kubectl get networkpolicies -n netpol-demo
```

---

### Блок 3 — TLS Сертификаты с OpenSSL (25 мин)

> Выпускаем собственный CA, подписываем сертификат и подключаем его к Ingress

#### 3.1 — Создать собственный CA

```bash
mkdir -p ~/ssl-lab && cd ~/ssl-lab

# Генерировать приватный ключ CA (ECDSA P-256 — быстрее RSA, сильнее по размеру)
openssl genrsa -out ca.key 4096
# Или через ECDSA:
# openssl ecparam -name prime256v1 -genkey -noout -out ca.key

# Самоподписанный корневой сертификат CA (10 лет)
openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 \
  -days 3650 \
  -out ca.crt \
  -subj "/C=RU/ST=Moscow/O=SiriusLab CA/CN=SiriusLab Root CA"

# Посмотреть что получилось
openssl x509 -in ca.crt -noout -text | grep -E "Issuer:|Subject:|Not (Before|After)"
```

#### 3.2 — Создать CSR для веб-сервера

```bash
# Конфигурационный файл с SAN (Subject Alternative Names)
cat > webapp.ext << 'EOF'
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = webapp.local
DNS.2 = www.webapp.local
IP.1 = 127.0.0.1
EOF

# Ключ сервера
openssl genrsa -out webapp.key 2048

# CSR (Certificate Signing Request)
openssl req -new \
  -key webapp.key \
  -out webapp.csr \
  -subj "/C=RU/O=SiriusLab/CN=webapp.local"

# Посмотреть CSR
openssl req -in webapp.csr -noout -text | grep -E "Subject:|DNS:|IP:"
```

#### 3.3 — Подписать сертификат нашим CA

```bash
# Подписать CSR — CA выдаёт сертификат на 1 год
openssl x509 -req \
  -in webapp.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out webapp.crt \
  -days 365 \
  -sha256 \
  -extfile webapp.ext

# Проверить результат
openssl x509 -in webapp.crt -noout -text | grep -A5 "Subject Alternative"
# Должно показать: DNS:webapp.local, IP:127.0.0.1

# Проверить цепочку доверия
openssl verify -CAfile ca.crt webapp.crt
# → webapp.crt: OK
```

#### 3.4 — Подключить сертификат к Kubernetes Ingress

```bash
# Создать TLS Secret в Kubernetes
kubectl create secret tls webapp-tls \
  --cert=webapp.crt \
  --key=webapp.key \
  -n netpol-demo

# Проверить Secret
kubectl get secret webapp-tls -n netpol-demo
kubectl describe secret webapp-tls -n netpol-demo
```

**`ingress-tls.yaml`:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-tls-ingress
  namespace: netpol-demo
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - webapp.local
    secretName: webapp-tls    # наш Secret с сертификатом
  rules:
  - host: webapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

```bash
kubectl apply -f ingress-tls.yaml

# Добавить в /etc/hosts
echo "$(minikube ip) webapp.local" | sudo tee -a /etc/hosts

# Проверить TLS соединение с нашим CA
curl --cacert ca.crt https://webapp.local
# ✓ Соединение установлено с нашим сертификатом!

# Детальная проверка через openssl
openssl s_client -connect webapp.local:443 -CAfile ca.crt -showcerts 2>&1 | \
  grep -E "subject=|issuer=|Verify return code"
# → Verify return code: 0 (ok)
```

#### 3.5 — Декодировать сертификат из K8s Secret

```bash
# Посмотреть что хранится в Secret
kubectl get secret webapp-tls -n netpol-demo \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl x509 -noout -text | \
  grep -E "Subject:|Issuer:|DNS:|IP:|Not "

# Проверить срок действия
kubectl get secret webapp-tls -n netpol-demo \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl x509 -noout -enddate
```

---

### Блок 4 — Falco (Бонус, 15 мин)

> Только если остаётся время и Falco установлен (`helm install falco falcosecurity/falco`)

```bash
# Посмотреть текущие правила Falco
kubectl get pods -n falco
kubectl logs -n falco -l app.kubernetes.io/name=falco -f &

# Сгенерировать Alert: зайти в работающий контейнер
kubectl exec -it frontend -n netpol-demo -- sh
# Falco сгенерирует: "A shell was spawned in a container"
exit

# Ещё один Alert: читать /etc/shadow
kubectl exec frontend -n netpol-demo -- cat /etc/shadow 2>/dev/null || true
# Falco: "Read sensitive file untrusted"
```

---

## Что сдать преподавателю

1. `kubectl auth can-i list pods -n rbac-demo --as=system:serviceaccount:rbac-demo:app-rbash
eader` → **yes**
2. `kubectl auth can-i delete pods -n rbac-demo --as=system:serviceaccount:rbac-demo:app-reader` → **no**
3. `kubectl exec frontend -- wget database-svc` → **timeout** (NetworkPolicy работает)
4. `kubectl exec backend -- wget database-svc` → **200 OK**
5. `openssl verify -CAfile ca.crt webapp.crt` → **webapp.crt: OK**
6. `curl --cacert ca.crt https://webapp.local` → **ответ от nginx** (TLS работает)

---

## Частые проблемы

| Проблема | Решение |
|----------|---------|
| NetworkPolicy не применяется | CNI плагин должен поддерживать NetworkPolicy (Calico, Cilium, Weave). Flannel — НЕ поддерживает! |
| `kubectl exec` в rbac-test даёт Forbidden сразу | Проверить что `serviceAccountName: app-reader` в pod spec |
| `wget` timeout слишком долгий | Добавить `--timeout=3` |
| `openssl verify` — unable to get local issuer | Убедиться что `-CAfile` указывает на тот же `ca.crt`, которым подписывали |
| `curl` — SSL certificate problem | Добавить `--cacert ca.crt` или `-k` для тестирования без проверки |
| Ingress не поднимает HTTPS | Проверить что Secret в том же namespace что Ingress; `kubectl describe ingress` — Events |
| Falco не видит events | Нужен privileged режим для Falco DaemonSet — проверить логи |

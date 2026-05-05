Коротко: **Ingress — це “один об’єкт = все одразу”**, а **HTTPRoute — це частина модульної моделі Gateway API (розділення ролей)**.

---

# 🔑 Ключова різниця

## Ingress (старий підхід)
- Один ресурс описує:
  - host
  - path
  - backend
  - (частково) поведінку контролера через annotations
- Тісно прив’язаний до конкретного контролера

👉 Тобто:
> Dev/ops описує ВСЕ в одному YAML

---

## HTTPRoute + Gateway (новий підхід)

- Gateway API розділяє відповідальності:

### 1. Gateway
- описує **інфраструктуру**
- listeners (ports, TLS, LB)
- аналог LoadBalancer / entry point

### 2. HTTPRoute
- описує **routing logic**
- host, path, backend

👉 Тобто:
> інфраструктура ≠ маршрути

---

# 🧠 Головна ідея

**Ingress:**
[ routing + infra hints ] → один ресурс

**Gateway API:**

Gateway (infra) ← керує platform team
HTTPRoute ← керує app team


---

# 👥 Хто має керувати чим

## 🏗 Gateway → Platform / DevOps / Cluster Admin

Чому:
- відкриває порти (80/443)
- керує TLS сертифікатами
- інтегрується з LoadBalancer / cloud
- може впливати на всю кластерну мережу

👉 Це **shared infrastructure**, ризик високий

---

## 🚀 HTTPRoute → Application teams

Чому:
- лише описує:
  - `host`
  - `path`
  - `service`
- не впливає на інші сервіси (при правильних policy)

👉 Це **application-level routing**

---

# 🔐 Чому це важливо (реальний сенс)

У Ingress:
- будь-який dev може:
  - повісити свій host
  - зламати routing іншого сервісу
  - гратись з annotations (NGINX специфічні)

У Gateway API:
- Gateway контролює:
  - **хто може attach-итись**
- через `allowedRoutes`:

```yaml
allowedRoutes:
  namespaces:
    from: Same
```
або
```
from: Selector
```
👉 Тобто:

чітка multi-tenant ізоляція

# ⚖️ Ingress vs HTTPRoute (Gateway API)

| Критерій | Ingress | HTTPRoute (Gateway API) |
|----------|--------|-------------------------|
| Модель | Моноліт (все в одному ресурсі) | Розділена (Gateway + Route) |
| Розділення відповідальності | ❌ Немає | ✅ Чітке (infra vs routing) |
| Хто керує | Dev + Ops разом | Gateway → Ops, HTTPRoute → Dev |
| Multi-tenancy | ❌ Слабка | ✅ Сильна (allowedRoutes, policies) |
| Контроль доступу | Обмежений | Гнучкий (namespace selector, RBAC) |
| Прив’язка до контролера | Сильна (annotations) | Слабка (стандартизовані CRD) |
| Розширюваність | Через annotations (хаотично) | Через API (чисто, extensible) |
| TLS / Ports | Частково в Ingress | Повністю в Gateway |
| Конфлікти (host/path) | Можливі і неконтрольовані | Контрольовані Gateway |
| Layer 4 (TCP/UDP) | ❌ Немає | ✅ Є (TCPRoute, UDPRoute) |
| Статус | Stable, але legacy | Сучасний стандарт (evolving) |
| Підтримка cloud-native фіч | Обмежена | Розширена |
| Use case | Простий ingress | Enterprise / multi-team |

---

# 🧠 Висновок

- **Ingress** → прості кластери, low governance  
- **Gateway API (HTTPRoute)** → production, multi-team, security-first

## Ingress example 
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-app-simple-chart
  namespace: simple-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: k8s.home
      http:
        paths:
          - path: /test-nginx
            pathType: Prefix
            backend:
              service:
                name: simple-app-simple-chart
                port:
                  number: 80

```
## Gateway API example
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: simple-app-simple-chart
  namespace: simple-app
spec:
  hostnames:
    - k8s.home
  parentRefs:
    - name: my-gateway
      namespace: nginx-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /test-nginx
      backendRefs:
        - name: simple-app-simple-chart
          port: 80
```
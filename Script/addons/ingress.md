
## 1. Podstawy Ingress w Kubernetes

### 1.1. Co to jest Ingress?

* **Ingress** to obiekt API Kubernetes, który definiuje sposób kierowania ruchu sieciowego warstwy aplikacji (L7 HTTP/S) do konkretnych zasobów typu Service wewnątrz klastra.
* Ingress sam w sobie nie robi nic – do jego egzekwowania (enforcement) potrzebny jest działający **Ingress Controller** (kontroler Ingress).
* Aby użyć Ingress, należy:

    1. Zainstalować lub włączyć Ingress Controller (np. NGINX Ingress, Traefik, Contour, Istio Gateways, HAProxy Ingress itp.).
    2. Zdefiniować zasób `Ingress`, gdzie opisujemy reguły host/path, wskazujące serviceName i servicePort.

### 1.2. Różnica między Service a Ingress

* **Service** typu LoadBalancer, NodePort czy ClusterIP działa na warstwie L4 (IP/TCP lub UDP). Nie rozróżnia żądań HTTP według nagłówka Host lub ścieżki (path). Przy LoadBalancer każdy Service otrzymuje swój dedykowany publiczny adres IP („external IP”) lub port.
* **Ingress** działa na warstwie L7 – potrafi odczytać nagłówek HTTP Host, URI i na tej podstawie przekierować ruch do różnych usług. Dzięki temu:

    * Jedna zewnętrzna końcówka (np. pojedynczy LoadBalancer lub pojedynczy NodePort + hostPort) może obsłużyć wiele wirtualnych hostów (wiele domen) lub ścieżek.
    * Możliwe centralne zarządzanie certyfikatami TLS (terminacja TLS).

---

## 2. Wymagania i architektura

### 2.1. Ingress Controller

* **Ingress Controller** to komponent (część ekosystemu CNI), który obserwuje zasoby Ingress w API Kubernetes i tworzy odpowiednie reguły w warstwie L7 (np. konfigurację NGINX, reguły Envoy, zasoby HAProxy, pluginy Traefik).
* Przykładowe implementacje Ingress Controllerów:

    * **NGINX Ingress Controller** (najpopularniejszy, oferuje rozbudowane anotacje)
    * **Traefik** (nowoczesny, wspiera dynamiczne discovery)
    * **Contour** (bazuje na Envoy Proxy)
    * **Istio Gateway** (część Istio Service Mesh)
    * **HAProxy Ingress**
    * **Kong Ingress Controller**
    * **ALB Ingress Controller** (dla AWS)
* Architektura:

    1. Ingress Controller zwykle działa jako Deployment w jednym (lub wielu) podzie (-ach) w namespace `ingress-nginx`, `traefik`, `istio-system` itp.
    2. Do Controller przypisuje się Service typu LoadBalancer lub NodePort, który wystawia „punkt wejścia” na świat zewnętrzny.
    3. Gdy utworzymy Ingress, kontroler reaguje na zmiany w API i przeładunuje własną konfigurację (np. dynamiczne załadowanie nowych reguł w NGINX).
    4. Opcjonalnie możemy mieć wiele instancji Ingress Controller (np. dla różnych klas ruchu), co realizuje się przez zasoby `IngressClass`.

### 2.2. IngressClass i `ingressClassName`

* Od Kubernetes 1.18 wprowadzono zasób **IngressClass**, który pozwala zdefiniować różne klasy Ingress (np. `nginx`, `traefik`, `istio`).

* W specyfikacji Ingress pojawiło się pole `ingressClassName`, umożliwiające przypisanie danego Ingress do konkretnego IngressClass.

* Przykład zasobu IngressClass:

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: IngressClass
  metadata:
    name: nginx
  spec:
    controller: k8s.io/ingress-nginx
    parameters:
      apiGroup: k8s.nginx.org/v1
      kind: NginxIngressControllerConfig
      name: nginx-config
  ```

    * W `controller:` podajemy identyfikator, za który odpowiada dany Ingress Controller (np. dla NGINX: `k8s.io/ingress-nginx`).
    * `parameters` pozwala przekazać konfigurację specyficzną dla implementacji (np. ConfigMap).

* W definicji Ingress:

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: my-ingress
  spec:
    ingressClassName: nginx
    rules:
      # ...
  ```

    * W ten sposób Ingress Controller, którego `controller` odpowiada `nginx`, będzie odbierał ten Ingress i obsługiwał go.

### 2.3. Wersja API

* Od Kubernetes v1.19 zasób **Ingress** jest dostępny w grupie `networking.k8s.io/v1`.
* Poprzednie wersje (1.18) miały `networking.k8s.io/v1beta1`, a starsze – `extensions/v1beta1`.
* W tutorialu będziemy korzystać z **`apiVersion: networking.k8s.io/v1`** (zalecane w nowych klastrach).

---

## 3. Ingress Controller – wybór i instalacja

Przykład: **NGINX Ingress Controller** (wersja Opensource, utrzymywana przez Kubernetes Community)

1. **Dodanie repozytorium Helm**

   ```bash
   helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
   helm repo update
   ```

2. **Instalacja z domyślnymi wartościami**

   ```bash
   helm install ingress-nginx ingress-nginx/ingress-nginx \
     --namespace ingress-nginx \
     --create-namespace
   ```

    * Utworzy deployment `ingress-nginx-controller` (z zwykle jedną repliką, można skalować).
    * Utworzy Service typu `LoadBalancer` (w chmurach publicznych) lub `NodePort` w on-premises (zależnie od konfiguracji).

3. **Sprawdzenie działania**

   ```bash
   kubectl get pods -n ingress-nginx
   kubectl get svc -n ingress-nginx
   ```

    * Service `ingress-nginx-controller` otrzyma zewnętrzny adres IP (typ LoadBalancer).
    * Na tym adresie działa frontend, który odbiera ruch HTTP/S i przekazuje do Ingress Controller.

> **Uwaga**: inne implementacje mają własne instrukcje instalacji. Traefik można wgrać przez Helm Chart `traefik/traefik`, Contour – `projectcontour/contour`, Istio Gateway jest częścią instalacji Istio itp.

---

## 4. Definicja zasobu Ingress (`networking.k8s.io/v1`)

Poniżej omówimy poszczególne pola specyfikacji Ingress.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: production
  annotations:
    # przykładowe anotacje specyficzne dla Ingress Controller:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
        - www.example.com
      secretName: example-tls-secret
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

### 4.1. Pole `metadata.annotations`

* Tutaj wprowadzamy anotacje specyficzne dla wybranego Ingress Controller (np. opcje NGINX, Traefik, Istio).
* Przykładowe anotacje dla **NGINX Ingress Controller**:

    * `nginx.ingress.kubernetes.io/rewrite-target: /` – pozwala przepisywać ścieżki URL (np. kasować prefiks).
    * `nginx.ingress.kubernetes.io/ssl-redirect: "true"` – wymusza przekierowanie HTTP → HTTPS.
    * `nginx.ingress.kubernetes.io/proxy-body-size: "10m"` – zwiększenie maksymalnego rozmiaru ciała żądania.
    * `nginx.ingress.kubernetes.io/auth-type: basic` i `nginx.ingress.kubernetes.io/auth-secret: my-basic-auth` – włączenie prostego uwierzytelniania BasicAuth.
    * `nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/16,192.168.0.0/24"` – ograniczenie dostępu tylko z określonych CIDR.
* Dla **Traefik** używamy np.:

    * `traefik.ingress.kubernetes.io/router.middlewares: default-auth@kubernetescrd`
    * `traefik.ingress.kubernetes.io/redirect-entry-point: https`

### 4.2. Pole `spec.ingressClassName`

* Definiuje, który Ingress Controller będzie obsługiwał ten zasób.
* Wartość powinna odpowiadać `metadata.name` zasobu `IngressClass`.
* Jeśli nie podamy `ingressClassName`, Kubernetes może użyć `default` IngressClass (jeżeli taki jest oznaczony jako `default: true`).

### 4.3. Sekcja `spec.tls`

* Pozwala na konfigurację terminacji ruchu TLS.
* `hosts` – lista domen, dla których certyfikat będzie ważny.
* `secretName` – nazwa Secretu typu `kubernetes.io/tls` zawierającego klucz prywatny (`tls.key`) i certyfikat (`tls.crt`) w Base64.
* Gdy Ingress Controller napotka żądanie HTTPS na wskazane hosty, pobierze `tls.crt` i `tls.key` z Secretu i użyje ich do terminacji TLS.

Przykład Secretu typu TLS:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: LS0tL.....  # Base64 cert
  tls.key: LS0tL.....  # Base64 key
```

### 4.4. Sekcja `spec.rules`

Zawiera listę reguł, które określają sposób mapowania host/ścieżka → Service. Każda reguła wygląda następująco:

```yaml
- host: example.com      # opcjonalne; jeśli brak, Ingress będzie "catch-all"
  http:
    paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: foo-service
            port:
              number: 80
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: bar-service
            port:
              number: 8080
```

* **`host`** (stała domena, np. `example.com` lub `api.example.com`). Jeśli pominąć w regule, oznacza to, że reguła pasuje do wszystkich hostów (catch‐all), jednak większość produkcyjnych konfiguracji używa jawnego hosta.
* **`http.paths`** – lista ścieżek (ścieżka aplikacji, URL), każda z:

    * **`path`** – wzorzec ścieżki (np. `/`, `/api`, `/static`).
    * **`pathType`** – typ dopasowania ścieżki. Możliwe wartości:

        * `Exact` – dopasowanie ściśle tej ścieżki (np. `/foo` dopasuje tylko `/foo`, ale nie `/foo/bar`).
        * `Prefix` – dopasowanie prefiksowe (np. `/foo` dopasuje `/foo`, `/foo/`, `/foo/bar` itd.).
        * `ImplementationSpecific` – sposób dopasowania zależy od implementacji Ingress Controller (może to być np. regex lub prefix). Zależnie od wybranego kontrolera, może być bardziej elastyczne, ale mniej przenośne między różnymi controllerami.
* **`backend.service.name`** oraz `backend.service.port.number` (albo `port.name` w przypadku usługi, która ma nazwane porty).

    * `service.name` – nazwa zasobu Service w tym samym namespace.
    * `service.port.number` – numer portu (lub `port.name` w Service). Niezależnie od tego, czy Service ma typ ClusterIP, NodePort czy LoadBalancer, Ingress przekieruje do tego Service.

> **Uwaga**: starsze wersje Ingress (v1beta1) używały pola `backend.serviceName` i `backend.servicePort` (bez zagnieżdżenia), jednak w `networking.k8s.io/v1` format został zmieniony na powyższy.

---

## 5. Przykłady praktyczne

### 5.1. Prosty routing HTTP

#### 5.1.1. Założenia

* W namespace `production` mamy dwa Deploymenty i odpowiadające im Services:

    * **Service** `frontend-service` (port 80)
    * **Service** `api-service` (port 8080)

#### 5.1.2. Manifesty Service (przykład)

```yaml
# frontend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: production
spec:
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

```yaml
# api-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    app: api
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

#### 5.1.3. Ingress dla obu usług

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
```

**Działanie**:

* Żądanie `http://example.com/` (lub `http://example.com/index.html`) trafi do **frontend-service:80**.
* Żądanie `http://example.com/api/users` zostanie przekierowane do **api-service:8080** (ścieżka `/api/users`); dzięki anotacji `rewrite-target: /` NGINX postawi w komunikacie do `api-service` ścieżkę `/users` (bez prefiksu `/api`).

> **Ważne**: Bez `rewrite-target` Ingress Controller przekaże dokładnie tą samą ścieżkę do backendu (tzn. `/api/users`). Dlatego często stosuje się rewrite, gdy backend oczekuje, że „/” to główna ścieżka aplikacji.

---

### 5.2. Routing host‐based (wiele domen)

Niektóre aplikacje wymagają obsługi wielu domen w jednym Ingress. Można to zrobić, definiując kilka reguł `rules` z różnymi `host`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  namespace: production
spec:
  ingressClassName: nginx
  rules:
    - host: app1.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-service
                port:
                  number: 80
    - host: app2.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2-service
                port:
                  number: 80
```

* `app1.example.com` zostanie skierowany do `app1-service`.
* `app2.example.com` trafi do `app2-service`.

Opcjonalnie można dodać catch‐all (bez pola `host`), by obsługiwać ruch z innych domen:

```yaml
- http:
    paths:
      - path: /legacy
        pathType: Prefix
        backend:
          service:
            name: legacy-service
            port:
              number: 80
```

Wtedy każde żądanie przychodzące na dowolny host, które pasuje do `/legacy`, trafi do `legacy-service`.

---

### 5.3. Routing path‐based w pojedynczym hoście

Jeżeli jedna domena (host) ma wiele mikroserwisów, możemy je rozdzielić według ścieżek.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: single-host-path-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /users(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: users-service
                port:
                  number: 8080
          - path: /products(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: products-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: default-service
                port:
                  number: 80
```

* Pierwszy `path` używa regex‐u (możliwy dzięki `ImplementationSpecific` w NGINX), by złapać `/users`, `/users/` oraz `/users/anything` i przepisywać go do backendu (np. `/anything`).
* Dzięki `rewrite-target: /$2` można wyciąć prefiks `/users` lub `/products`.
* Ostatecznie każdy inny ruch (pasujący do `/`) trafia do `default-service`.

> **Uwaga**: regex‐owe dopasowania nie są częścią specyfikacji Kubernetes Networking `v1` – to funkcja Ingress Controller (NGINX). Inne kontrolery mogą wymagać innych anotacji (np. Traefik używa `PathPrefixStrip: "/users"`).

---

### 5.4. Terminacja TLS

#### 5.4.1. Tworzenie Secretu TLS

1. Wygeneruj parę klucz/certyfikat (np. Let's Encrypt lub samopodpisany w celach testowych):

   ```bash
   openssl req -x509 -nodes -days 365 \
     -newkey rsa:2048 \
     -keyout tls.key \
     -out tls.crt \
     -subj "/CN=example.com/O=Example"
   ```
2. Utwórz Secret:

   ```bash
   kubectl create secret tls example-tls-secret \
     --namespace=production \
     --cert=./tls.crt \
     --key=./tls.key
   ```

#### 5.4.2. Ingress z TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - secure.example.com
      secretName: example-tls-secret
  rules:
    - host: secure.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-service
                port:
                  number: 443
```

* Ingress Controller posłuży się Secretem `example-tls-secret` do terminacji TLS po stronie NGINX (na warstwie kontrolera).
* Ruch do `https://secure.example.com/` jest odszyfrowany, a następnie przekazywany do `secure-service:443` jako HTTP (w zależności od konfiguracji, może to być HTTP lub ponowny TLS do backendu).

> **Uwaga**: Jeśli backend nasłuchuje na HTTP (np. port 80), należy ustawić `service.port.number: 80`; terminacja TLS będzie po stronie Ingress Controller.

---

### 5.5. Zaawansowane funkcje Ingress Controller (NGINX)

#### 5.5.1. Uwierzytelnianie BasicAuth

1. Utwórz Secret typu podstawowego uwierzytelniania:

   ```bash
   printf "user:\$(openssl passwd -stdin -apr1)\n" | kubectl create secret generic basic-auth --from-file=auth -n production
   ```

    * Polecenie `openssl passwd -apr1` wygeneruje hash dla podanego hasła.
    * Otrzymamy Secret z kluczem `auth`, który zawiera plik htpasswd.

2. Anotacje w Ingress:

   ```yaml
   metadata:
     annotations:
       nginx.ingress.kubernetes.io/auth-type: basic
       nginx.ingress.kubernetes.io/auth-secret: basic-auth
       nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
   ```

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: auth-ingress
     namespace: production
     annotations:
       nginx.ingress.kubernetes.io/auth-type: basic
       nginx.ingress.kubernetes.io/auth-secret: basic-auth
       nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
   spec:
     ingressClassName: nginx
     rules:
       - host: secure.example.com
         http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: protected-service
                   port:
                     number: 80
   ```

* Każde żądanie do `secure.example.com` spowoduje, że NGINX wyświetli okno dialogowe BasicAuth i sprawdzi poświadczenia w pliku htpasswd przechowywanym w Secret.

#### 5.5.2. Limitowanie ruchu (Rate Limiting)

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-connections: "1"
    nginx.ingress.kubernetes.io/limit-rpm: "10"
    nginx.ingress.kubernetes.io/limit-rps: "2"
    nginx.ingress.kubernetes.io/limit-whitelist: "127.0.0.1"
```

* `limit-connections: "1"` – maksymalnie jedno aktywne połączenie dla klienta.
* `limit-rps: "2"` – maksymalnie 2 żądania na sekundę.
* `limit-rpm: "10"` – maksymalnie 10 żądań na minutę.
* `limit-whitelist: "127.0.0.1"` – z tego CIDR limity nie będą nakładane.

#### 5.5.3. Redirect HTTP → HTTPS

Jeśli chcemy, by każde żądanie HTTP (port 80) przekierowywało na HTTPS (port 443):

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

* `ssl-redirect: "true"` – włącza przekierowanie HTTP → HTTPS.
* `force-ssl-redirect: "true"` – wymusza przekazanie nagłówka `X-Forwarded-Proto`, co pozwala backendowi wykryć, że żądanie pierwotnie przyszło przez HTTP.

#### 5.5.4. Rewrite i strip path

* Jeśli potrzebujemy usunąć fragment ścieżki przed przekazaniem do backendu, używamy `rewrite-target`.
* Przykład: chcemy, by ruch `/app1/(.*)` był przekazany jako `/$1` do usługi `app1-service`:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /app1/(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: app1-service
                port:
                  number: 80
```

Warto pamiętać, że aby regex zadziałał, `pathType` musi być `ImplementationSpecific`.

---

## 6. Integracja z cert-manager (Let's Encrypt)

Aby zautomatyzować wydawanie i odnawianie certyfikatów TLS od Let's Encrypt, można zainstalować **cert-manager** oraz skonfigurować zasób **Issuer** lub **ClusterIssuer**. Poniżej przykład:

### 6.1. Instalacja cert-manager

```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.12.0/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### 6.2. Tworzenie ClusterIssuer dla Let's Encrypt

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # serwer ACME (Let's Encrypt production)
    server: https://acme-v02.api.letsencrypt.org/directory
    email: twoj-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

* `solvers.http01.ingress.class: nginx` – oznacza, że cert-manager skorzysta z Ingress Controller klasy `nginx` do weryfikacji.

### 6.3. Zasób Certificate korzystający z ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: production
spec:
  secretName: example-com-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - example.com
    - www.example.com
```

* Po zastosowaniu cert-manager automatycznie utworzy Secret `example-com-tls-secret` zawierający certyfikat i klucz.
* Ingress może odwołać się do tego Secretu:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: production
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
        - www.example.com
      secretName: example-com-tls-secret
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

* Gdy cert-manager odświeży certyfikat (np. co 60 dni), Ingress automatycznie korzysta z nowego certyfikatu.

---

## 7. Debugowanie i weryfikacja działania

### 7.1. Sprawdzenie stanu Ingress

```bash
kubectl get ingress -n production
kubectl describe ingress production-ingress -n production
```

* W `kubectl get ingress` zobaczymy kolumny: `HOSTS`, `ADDRESS`, `PORTS`, `AGE`.
* W `kubectl describe ingress` znajdziemy szczegółowe reguły, status, eventy (np. błędy w konfiguracji lub lutowaniu Service).

### 7.2. Weryfikacja Ingress Controller

* Sprawdź status podów Ingress Controller:

  ```bash
  kubectl get pods -n ingress-nginx
  ```
* Upewnij się, że Service LoadBalancer (lub NodePort) Ingress Controller ma odpowiedni External IP:

  ```bash
  kubectl get svc -n ingress-nginx
  ```

    * W kolumnie `EXTERNAL-IP` pojawi się adres, pod którym Ingress jest dostępny.
* Możesz przetestować żądanie:

  ```bash
  curl -H "Host: example.com" http://<INGRESS-EXTERNAL-IP>/
  ```

### 7.3. Sprawdzanie logów Ingress Controller

* Dla NGINX:

  ```bash
  kubectl logs -n ingress-nginx <INGRESS-POD-NAME>
  ```

    * Tutaj zobaczysz, czy Ingress Controller załadował poprawnie reguły, czy występują błędy w konfiguracji NGINX.

* W trakcie debugowania upewnij się, że:

    * Nazwa hosta w nagłówku (`Host`) jest poprawna.
    * Service, do którego kieruje Ingress, działa i zwraca odpowiedź.
    * Porty w `backend.service.port` odpowiadają faktycznym portom serwisu.

### 7.4. Test lokalny (minikube / KIND / k3d)

W środowisku lokalnym (np. Minikube) często musimy dodać w pliku `/etc/hosts` wpis kierujący `example.com` na adres klastra (np. `minikube ip`):

```bash
echo "$(minikube ip) example.com" | sudo tee -a /etc/hosts
```

Następnie:

```bash
curl http://example.com/
curl -k https://example.com/   # "-k" wyłącza weryfikację certyfikatu, jeśli samopodpisany
```

---

## 8. Najlepsze praktyki i uwagi końcowe

1. **Używaj `IngressClass` i `ingressClassName`**

    * W większych klastrach często działają równolegle różni kontrolerzy Ingress (np. `nginx`, `traefik`). Określaj `ingressClassName`, by unikać kolizji.

2. **Trzymaj reguły Ingress w repozytorium w postaci YAML**

    * Powinniśmy mieć jasną strukturę: `ingress/production-frontend-ingress.yaml`, `ingress/staging-api-ingress.yaml` itp.
    * Ułatwia to audyt i rollback.

3. **Unikaj wielkich, skomplikowanych reguł w pojedynczym Ingress**

    * Dla czytelności i łatwiejszej konserwacji lepiej rozdzielić zasoby Ingress, jeśli aplikacja ma wiele niezależnych hostów/ścieżek.

4. **Centralizuj konfigurację TLS przez cert-manager**

    * Pozwala to automatycznie odnawiać certyfikaty i nie martwić się manualnym updaitem Secretu TLS.

5. **Stosuj anotacje rozsądnie**

    * Każdy Ingress Controller posiada własne sety anotacji. Zapoznawaj się z dokumentacją NGINX, Traefik czy Contour, by unikać niezgodności.
    * Np. `nginx.ingress.kubernetes.io/rewrite-target` nie zadziała przy Traefiku – tam używa się `traefik.ingress.kubernetes.io/router.middlewares`.

6. **Sprawdzaj limity wielkości żądań i timeouty**

    * Domyślnie NGINX ma ograniczenie rozmiaru ciała żądania (np. 1 MB). Można je zmienić za pomocą anotacji:

      ```yaml
      nginx.ingress.kubernetes.io/proxy-body-size: "10m"
      ```
    * Timeouty (połączeń, keepalive) modyfikujemy, np.:

      ```yaml
      nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
      ```

7. **Monitoruj logi i metryki**

    * Większość kontrolerów (NGINX, Traefik, Istio) udostępnia metryki Prometheus.
    * Użyj Prometheus + Grafana do monitorowania stanu Ingress Controller (ruch, błędy 5xx, 4xx, zużycie zasobów).

8. **Zabezpiecz dostęp (WAF, rate-limiting, whitelisting)**

    * Jeśli aplikacje wystawione są na publiczny adres, warto dodać zabezpieczenia, np. Web Application Firewall (ModSecurity w NGINX), ograniczenia rate-limiting, whitelisting CIDR.

9. **Regularne testy i CI/CD**

    * W pipeline CI/CD warto weryfikować poprawność manifestów (np. `kubectl apply --dry-run=client/server -f`).
    * Można też użyć narzędzi lintujących (np. `kube-linter`, `kubeval`) do sprawdzania zasad.

10. **Dokumentuj i standaryzuj wewnętrzne naming convention**

    * Konsystencja: `namespace: production`, `ingress: prod-<app>-ingress`, `secret: tls-<app>`, itp.
    * Ułatwia to identyfikację, która aplikacja jest która i skąd pochodzi ruch.


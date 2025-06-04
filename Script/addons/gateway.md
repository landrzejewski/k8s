
### 1.1. Co to jest Kubernetes Gateway API?

Kubernetes Gateway API to zestaw nowych zasobów (CRD: Custom Resource Definitions) i powiązanych mechanizmów, które mają zastąpić lub uzupełnić tradycyjne obiekty Ingress. Główne zalety Gateway API:

* **Większa modułowość**: Rozdziela zasoby odpowiedzialne za klasę gateway (GatewayClass), sam gateway (Gateway) oraz reguły routingu (HTTPRoute, TCPRoute itp.).
* **Lepsza ekspresywność**: Możliwość definiowania zaawansowanych reguł, np. filtrów, przepisów UA, limitów, a także routing na podstawie nagłówków czy fragmentów ścieżki.
* **Rozszerzalność**: Obsługuje różne protokoły (HTTP, HTTPS, TCP, TLSProxy, UDP), co sprawia, że jest bardziej uniwersalne niż standardowe Ingress.
* **Oddzielenie polityki od implementacji**: GatewayClass określa, jakiego kontrolera użyć, a użytkownik definiuje jedynie reguły ruchu.

### 1.2. Wymagania wstępne

1. Klaster Kubernetes (minimum wersja 1.22+ zalecana).
2. Zainstalowane i skonfigurowane narzędzie `kubectl` (potrafi połączyć się z Twoim klastrem).
3. Dostęp do środowiska typu shell (Linux/macOS/WSL), z uprawnieniami pozwalającymi na tworzenie zasobów w klastrze.
4. (Opcjonalnie) zainstalowany Helm (wersja 3+) – ułatwia instalację Contoura.

---

## 2. Instalacja Kubernetes Gateway API (CRD)

Gateway API wymaga zainstalowania odpowiednich CRD (CustomResourceDefinition), aby klaster “rozumiał” nowe zasoby: `GatewayClass`, `Gateway`, `HTTPRoute` i inne.

1. Pobierz najnowszą wersję CRD Gateway API (w chwili pisania tutoriala jest to v1beta2, ale przed instalacją sprawdź [repozytorium gateway-api](https://github.com/kubernetes-sigs/gateway-api/releases) w celu pobrania aktualnej wersji):

   ```bash
   # Przykładowo dla wersji v1beta2:
   kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1beta2/gateway-api-crds.yaml
   ```

2. Sprawdź, czy CRD zostały poprawnie utworzone:

   ```bash
   kubectl get crd | grep gateway.networking.k8s.io
   ```

   Powinieneś zobaczyć listę CRD, np.:

   ```
   gateways.gateway.networking.k8s.io
   gatewayclasses.gateway.networking.k8s.io
   httproutes.gateway.networking.k8s.io
   tcproutes.gateway.networking.k8s.io
   tlsroutes.gateway.networking.k8s.io
   udproutes.gateway.networking.k8s.io
   ```

   Jeśli lista jest pusta lub CRD nie istnieją, sprawdź ewentualne błędy w logach komendy `kubectl apply`.

---

## 3. Instalacja i konfiguracja kontrolera Contour jako implementacji Gateway API

Gateway API to tylko zestaw definicji zasobów – potrzebny jest również kontroler, który te zasoby “wyciągnie” i przekształci w rzeczywiste reguły routingu na poziomie warstwy L7 (np. Envoy, HAProxy, itp.). W tym tutorialu skorzystamy z [Contour](https://projectcontour.io/), który implementuje wsparcie dla Gateway API.

> **Uwaga**: Możesz też wybrać innego kontrolera (np. Istio Gateway, Kong Gateway, Contour czy OSM), ale poniższe kroki dotyczą Contoura.

### 3.1. Przygotowanie przestrzeni nazw (namespace)

Zaleca się utworzyć oddzielny namespace dla komponentów kontrolera:

```bash
kubectl create namespace projectcontour
```

### 3.2. Instalacja Contoura (przez Helm)

Jeżeli masz zainstalowany Helm, łatwo wgrasz Contoura jako chart. Przykładowa procedura:

1. Dodaj repozytorium Helm Contoura:

   ```bash
   helm repo add contour https://projectcontour.io/contour-helm-chart
   helm repo update
   ```

2. Zainstaluj Contoura w wersji wspierającej Gateway API. Warto określić wartości, które włączają implementację Gateway API (domyślnie od wersji Contour 1.19+ jest to wspierane).

   Utwórz plik `contour-values.yaml` z zawartością:

   ```yaml
   # contour-values.yaml
   contour:
     env:
       - name: CONTOUR_GATEWAY_API
         value: "true"
   ```

   Następnie uruchom:

   ```bash
   helm install contour contour/contour \
     --namespace projectcontour \
     --values contour-values.yaml
   ```

3. Sprawdź, czy wszystkie pod’y się uruchomiły:

   ```bash
   kubectl get pods -n projectcontour
   ```

   Powinny pojawić się m.in.:

    * `contour-xxxxx` (deployment)
    * `envoy-xxxxx` (daemonset lub deployment, w zależności od konfiguracji)

   Jeżeli któryś z podów nie startuje poprawnie, sprawdź jego logi:

   ```bash
   kubectl logs -n projectcontour deploy/contour
   kubectl logs -n projectcontour ds/envoy
   ```

### 3.3. Tworzenie serwisu typu LoadBalancer lub NodePort dla Envoy

Dla ruchu zewnętrznego potrzebujesz wystawić Envoy (proxy) na zewnątrz klastra. W zależności od środowiska możesz wybrać:

* **LoadBalancer** (jeśli Twój klaster jest w chmurze, która udostępnia LB, np. AWS/GCP/Azure).
* **NodePort** (jeśli lokalny klaster, np. minikube, kind, bare-metal).

#### 3.3.1. Przykład Service typu LoadBalancer

Jeśli klaster wspiera Service typu LoadBalancer:

```yaml
# envoy-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: envoy-lb
  namespace: projectcontour
spec:
  type: LoadBalancer
  selector:
    app: envoy
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
```

Zastosuj manifest:

```bash
kubectl apply -f envoy-lb.yaml
```

Po chwili sprawdź, czy pojawił się zewnętrzny adres IP:

```bash
kubectl get svc -n projectcontour envoy-lb
```

Powinieneś zobaczyć kolumnę `EXTERNAL-IP`. To będzie adres, pod którym Gateway będzie dostępny spoza klastra.

#### 3.3.2. Przykład Service typu NodePort

Jeśli nie masz LB, użyj NodePort (np. na minikube lub bare-metal):

```yaml
# envoy-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: envoy-nodeport
  namespace: projectcontour
spec:
  type: NodePort
  selector:
    app: envoy
  ports:
    - name: http
      port: 80
      targetPort: 8080
      nodePort: 30080
    - name: https
      port: 443
      targetPort: 8443
      nodePort: 30443
```

```bash
kubectl apply -f envoy-nodeport.yaml
```

W takim wypadku Gateway będzie dostępny na wszystkich węzłach klastra pod portem 30080 (HTTP) i 30443 (HTTPS). W środowisku deweloperskim możesz przetestować:

```bash
kubectl get nodes -o wide
```

i następnie w przeglądarce połączyć się z `http://<NodeIP>:30080`.

---

## 4. Tworzenie GatewayClass i Gateway

Gdy CRD oraz kontroler (Contour) są już zainstalowane, czas zdefiniować klasę Gateway oraz sam obiekt Gateway.

### 4.1. GatewayClass

GatewayClass określa, jaki kontroler będzie obsługiwał definicje Gateway. Dla Contoura nazwa kontrolera to `projectcontour.io/gateway-controller`.

Stwórz manifest `gatewayclass.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta2
kind: GatewayClass
metadata:
  name: contour-gatewayclass
spec:
  controllerName: projectcontour.io/gateway-controller
```

Zastosuj:

```bash
kubectl apply -f gatewayclass.yaml
```

Sprawdź, czy GatewayClass został zaakceptowany:

```bash
kubectl get gatewayclass
```

Powinieneś zobaczyć w kolumnie `CONTROLLER` wartość `projectcontour.io/gateway-controller` oraz w kolumnie `ACCEPTED` – `True`.

### 4.2. Gateway

Obiekt Gateway instancjonuje fizyczny (lub wirtualny) punkt wejścia (Envoy) i definiuje, pod jakim adresem/protokole będzie działał. Zakładamy, że chcesz, aby Gateway nasłuchiwał na wszystkich dostępnych adresach (adresie IP z serwisu LoadBalancer/NodePort).

Przykład dla namespace `default` (możesz wybrać inny namespace, ważne żeby zasoby HTTPRoute były w tym samym namespace lub żeby definiować `GatewayNamespace`/`GatewayName` zgodnie z potrzebami):

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1beta2
kind: Gateway
metadata:
  name: example-gateway
  namespace: default
spec:
  gatewayClassName: contour-gatewayclass
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      # hostname: można pominąć lub wpisać konkretną domenę
      allowedRoutes:
        namespaces:
          from: All
```

Zastosuj manifest:

```bash
kubectl apply -f gateway.yaml
```

Sprawdź stan Gateway:

```bash
kubectl get gateway example-gateway -n default -o wide
```

W kolumnie `ADDRESS` powinieneś zobaczyć adres IP (LoadBalancer) lub wartość `<NodePort>` w zależności od typu serwisu. Kolumna `READY` powinna być `True`. Jeżeli stan to `False` lub `0/1`, sprawdź:

```bash
kubectl describe gateway example-gateway -n default
kubectl logs -n projectcontour deploy/contour
```

Upewnij się, że kontroler Contour zaakceptował ten Gateway (w opisie powinno się pojawić, że listenery zostały skonfigurowane).

---

## 5. Konfiguracja HTTPRoute (routing ruchu)

Teraz, gdy masz działający Gateway, trzeba zdefiniować reguły HTTPRoute, które powiedzą, jak kierować ruch do konkretnych serwisów w klastrze.

Poniżej przykład prostej aplikacji „ku-bare minimalnego serwisu” oraz HTTPRoute.

### 5.1. Utworzenie przykładowej aplikacji

Wyląduje prosty serwis „hello-world” zwracający tekst. W namespace `default`:

1. Manifesta Deployment i Service:

   ```yaml
   # hello-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: hello-deployment
     namespace: default
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: hello
     template:
       metadata:
         labels:
           app: hello
       spec:
         containers:
           - name: hello
             image: hashicorp/http-echo:latest
             args:
               - "-text=Hello from Kubernetes Gateway!"
             ports:
               - containerPort: 5678
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: hello-service
     namespace: default
   spec:
     selector:
       app: hello
     ports:
       - port: 80
         targetPort: 5678
   ```

2. Zastosuj oba zasoby:

   ```bash
   kubectl apply -f hello-deployment.yaml
   ```

Sprawdź, czy pod’y się uruchomiły:

```bash
kubectl get pods -l app=hello -n default
kubectl get svc hello-service -n default
```

### 5.2. Definicja HTTPRoute

Utwórz plik `httproute.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta2
kind: HTTPRoute
metadata:
  name: hello-route
  namespace: default
spec:
  parentRefs:
    - name: example-gateway       # musi wskazywać na utworzony wcześniej Gateway
  hostnames:
    - "example.local"            # możesz tu podać nazwę DNS; dla testów można użyć /etc/hosts
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: "/"           # wszystkie ścieżki
      backendRefs:
        - name: hello-service
          port: 80
```

Zastosuj manifest:

```bash
kubectl apply -f httproute.yaml
```

### 5.3. Weryfikacja reguł

1. Sprawdź, czy HTTPRoute został zaakceptowany przez kontroler:

   ```bash
   kubectl get httproute hello-route -n default -o wide
   ```

   W kolumnie `ACCEPTED` powinna być wartość `True`.

2. Dodaj w lokalnym pliku `/etc/hosts` (lub w DNS) rekord wskazujący na zewnętrzny adres IP/NodeIP Twojego Gateway (z kroku 3.3). Przykład (Linux/macOS):

   ```bash
   sudo nano /etc/hosts
   ```

   Dodaj linię:

   ```
   <EXTERNAL_IP>   example.local
   ```

   Gdzie `<EXTERNAL_IP>` to adres zwrócony przez `kubectl get svc envoy-lb` lub adres węzła i port 30080 (w przypadku NodePort).
   – Jeżeli używasz NodePort, zamiast hosta `example.local` możesz zrobić `curl` bezpośrednio:

   ```bash
   curl http://<NodeIP>:30080/
   ```

3. W przeglądarce (lub za pomocą `curl`) sprawdź:

   ```bash
   curl http://example.local/      
   ```

   Powinieneś otrzymać odpowiedź:

   ```
   Hello from Kubernetes Gateway!
   ```

   Jeżeli widzisz tekst, znaczy to, że ruch HTTP przepłynął przez Gateway (Envoy), został przeadresowany przez Contoura na Service `hello-service`, a dalej do Deployment’a „hello”.

---

## 6. Test działania i debugowanie

### 6.1. Sprawdzanie logów

* **Kontroler Contour**:

  ```bash
  kubectl logs -n projectcontour deploy/contour
  ```
* **Envoy** (jeśli występują problemy z routingiem):

  ```bash
  kubectl logs -n projectcontour ds/envoy
  ```

Upewnij się, że nie ma błędów dotyczących parsowania Gateway/HTTPRoute lub że nie występują konflikty (np. te same hostnames w wielu HTTPRoute do różnych Gateway).

### 6.2. Sprawdzanie statusu zasobów

* **Gateway**:

  ```bash
  kubectl describe gateway example-gateway -n default
  ```

  W sekcji `Conditions` powinieneś zobaczyć stan `Ready: True`. Jeśli jest `False`, fragment `Messages` może wskazać problem z instalacją controllera lub zdefiniowanym listenerem.

* **HTTPRoute**:

  ```bash
  kubectl describe httproute hello-route -n default
  ```

  Upewnij się, że w `Conditions` widnieje `Accepted: True` oraz że `AttachedRoutes` wskazuje Twój gateway. Jeśli zaś w `Conditions` jest `ResolvedRefs: False`, to może oznaczać, że `backendRef` nie istnieje (np. literówka w nazwie Service) lub port jest niepoprawny.

### 6.3. Dodatkowe narzędzia

* `kubectl get gatewayclasses.gateway.networking.k8s.io`
* `kubectl get gateways.gateway.networking.k8s.io`
* `kubectl get httproutes.gateway.networking.k8s.io`

oraz odpowiednio z `-n <namespace> -o yaml` lub `-o wide` dla lepszej diagnostyki.

---

## 7. Wskazówki dotyczące dalszej rozbudowy

1. **Routing zaawansowany**

    * Możesz definiować wiele `hostname` w `HTTPRoute`, np. `["app1.example.com", "app2.example.com"]`.
    * Możesz użyć różnych typów matchowania: `PathPrefix`, `Exact`, a nawet stosować filtry `Header`, `QueryParam`.
    * Przekierowania (HTTPRedirect) – od v1beta2 jest wsparcie dla `HTTPRoute` z akcjami `Redirect`.

2. **SSL/TLS**

    * Aby obsługiwać HTTPS, musisz dostarczyć certyfikat w postaci zasobu `Secret` w namespace, w którym definiujesz `Gateway` (lub wskazać `gateway.namespace` w `TLSCertificateRef`).
    * Przykład listenera HTTPS w `Gateway`:

      ```yaml
      listeners:
        - name: https
          protocol: HTTPS
          port: 443
          tls:
            mode: Terminate
            certificateRefs:
              - name: my-tls-secret
          allowedRoutes:
            namespaces:
              from: All
      ```
    * `my-tls-secret` to Secret typu `kubernetes.io/tls` zawierający `tls.crt` i `tls.key`.

3. **Load Balancery i health checki**

    * Scheduled health checki można skonfigurować w Envoy (Contour przekazuje domyślne konfiguracje), ale w bardziej zaawansowanych przypadkach możesz dostosować parametry health check.

4. **Wiele HTTPRoute i priorytety**

    * Gateway API pozwala na nakładanie wielu reguł – jeśli kilka `HTTPRoute` odwołuje się do tego samego `Gateway`, to kolejność dopasowywania odbywa się w oparciu o priorytet (np. dokładne dopasowanie ścieżki przed prefixem).
    * Możesz także definiować w poszczególne namespace (koniecznie upewnij się, że `allowedRoutes.namespaces.from` ma dobrą wartość: `All`, `Same`, albo lista konkretnych).

5. **Możliwe problemy na środowisku produkcyjnym**

    * Upewnij się, że masz monitoring dla Envoy i Contour, np. Prometheusa, aby sprawdzać opóźnienia, błędy 5xx, itp.
    * Jeśli używasz LoadBalancera w chmurze, pamiętaj o poprawnym skonfigurowaniu health-probec dla portu 8080/8443 (Contour), bo w przeciwnym razie chmurowy LB może uznać backend za niezdrowy.



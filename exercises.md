## 1. Deployment Nginx z niestandardową etykietą i skalowaniem

**Cel:**
Stworzyć Deployment Nginx z etykietami i skalowaniem replik.

**Wymagania:**

* Deployment o nazwie `nginx-deploy`
* Obraz: `nginx:1.21-alpine`
* W spec.template.metadata.labels:

    * `app: web-nginx`
    * `env: staging`
* Początkowo 3 repliki (replicas: 3)
* Po utworzeniu skalować do 5 replik (`kubectl scale deployment nginx-deploy --replicas=5`)

**Wskazówki:**

* Użyć pliku YAML z kind: Deployment.
* Po utworzeniu sprawdzić stan podów: `kubectl get pods -l app=web-nginx`.
* Skalowanie wykonać komendą `kubectl scale`.

---

## 2. Redis Master-Slave z Headless Service

**Cel:**
Skonfigurować klaster Redis z jednym masterem i przynajmniej dwoma podrzędnymi (slave), korzystając z Headless Service dla replikacji przez DNS.

**Wymagania:**

1. Namespace (opcjonalnie domyślny)
2. Deployment `redis-master`:

    * Obraz: `redis:6.2`
    * 1 replika
    * Etykieta w podzie: `app: redis-master`
3. Headless Service `redis-master-headless` (clusterIP: None)

    * Port 6379
    * Selekcja `app: redis-master`
4. Deployment `redis-slave`:

    * Obraz: `redis:6.2`
    * 2 repliki
    * W ARG/ENV: `--slaveof redis-master-headless 6379` (lub analogiczna konfiguracja)
    * Etykieta w podzie: `app: redis-slave`
5. Slaves automatycznie powinny odnajdywać mastera przez DNS (np. `redis-master-headless.default.svc.cluster.local:6379`)

**Wskazówki:**

* Headless Service pozwala na rozwiązywanie rekordów DNS poszczególnych podów mastera.
* W Deploymentach warto użyć initContainer, który poczeka, aż master odpowie na `redis-cli ping`.
* Sprawdzić stan repliki: w podzie-slave `redis-cli info replication | grep role` powinno wskazać “slave”.

---

## 3. Deployment Node.js z Rolling Update i Canary Release

**Cel:**
Wdrożyć prostą aplikację Node.js w wersji v1, a następnie zaktualizować do v2 w trybie Canary (20% nowych podów), a po weryfikacji – do 100%.

**Wymagania:**

1. Deployment `node-app`:

    * Obraz bazowy: `node:14-alpine`
    * W ENV: `APP_VERSION=v1`
    * Port 3000 (serwer HTTP wyświetla wersję z `APP_VERSION`)
    * Liczba replik: 5
2. Właściwa strategia RollingUpdate w spec.strategy (domyślne `RollingUpdate`)
3. Aktualizacja:

    * Zmiana `APP_VERSION` na `v2`
    * Ustawienie `maxSurge: 1`, `maxUnavailable: 0`, żeby najpierw powstał dokładnie 1 nowy pod (20% z 5 = 1)
    * Po zweryfikowaniu canary (logi, brak błędów) Deployment stopniowo wymieni wszystkie repliki na v2

**Wskazówki:**

* Pierwsze wdrożenie z `kubectl apply -f`.
* Aby wypuścić Canary, zaktualizować spec Deploymentu lub użyć `kubectl patch/set image`.
* Monitorować `kubectl rollout status deployment/node-app` i `kubectl get pods -l app=node-app`.

---

## 4. Deployment z Sidecar (nginx + synchronizator) i Service typu LoadBalancer

**Cel:**
Stworzyć Deployment, w którym każdy pod zawiera dwa kontenery: nginx (serwuje stronę) i sidecar (co 30 s generuje/aktualizuje pliki w katalogu współdzielonym). Następnie wystawić aplikację przez Service typu LoadBalancer lub NodePort.

**Wymagania:**

1. Deployment `web-sidecar`:

    * 3 repliki
    * Dwa kontenery:

        * `nginx:1.21` (mount wolumenu EmptyDir pod `/usr/share/nginx/html`)
        * `alpine:3.14` jako sidecar (co 30 s zapisuje plik `/src/index.html`, kopiuje do `/dest/index.html` na tym samym wolumenie)
    * Wolumen: `emptyDir` o nazwie np. `web-content`
    * Etykieta: `app: web-sidecar`
2. Service `web-loadbalancer`:

    * Typ: `LoadBalancer` (w Minikube można też użyć `NodePort`)
    * Selekcja `app: web-sidecar`
    * Port 80 → targetPort 80

**Wskazówki:**

* Sidecar używa `volumeMounts` do tego samego `emptyDir` w `/src` i `/dest`.
* Jeśli nie masz rzeczywistego LoadBalancer, zmień typ na NodePort lub użyj `minikube tunnel`.
* Po wdrożeniu testować: `curl <IP_LB>:80` i obserwować zmieniające się co 30 s daty.

---

## 5. Migracja MySQL 5.7 → 8.0 z zachowaniem danych przy użyciu PersistentVolumeClaim

**Cel:**
Zaktualizować istniejący Deployment MySQL z wersji 5.7 do 8.0, korzystając z tego samego PVC, tak aby dane zostały zachowane i przy minimalnej przerwie w działaniu.

**Wymagania:**

1. PVC `mysql-pv-claim`:

    * `storage: 5Gi`, `accessModes: ReadWriteOnce`
2. Deployment `mysql-v1`:

    * Obraz: `mysql:5.7`
    * Montuje PVC w `/var/lib/mysql`
    * Etykieta: `app: mysql`, `version: v1`
    * Secret `mysql-root-pass` z kluczem `password` (np. `password123`)
3. Nowy Deployment `mysql-v2`:

    * Obraz: `mysql:8.0`
    * Również montuje ten sam PVC w `/var/lib/mysql`
    * Strategia RollingUpdate: `maxUnavailable: 0`, `maxSurge: 1`
    * Etykieta: `app: mysql`, `version: v2`
    * W kontenerze opcjonalnie `postStart` hook uruchamia `mysql_upgrade`

**Wskazówki:**

* Przed migracją sprawdzić, czy w `mysql:8.0` wymagana jest komenda `mysql_upgrade`.
* Nowy Deployment powinien przejąć PVC dopiero, gdy stary zwolni wolumen (rollingUpdate).
* Po wdrożeniu w nowym podzie zweryfikować, czy istniejące tabele/rekordy są nienaruszone.

---

## 6. Deployment HTTPD + Service LoadBalancer z adnotacjami (np. AWS) i timeoutami

**Cel:**
Wdrożyć prosty serwer HTTP (Apache httpd) i wystawić go przez Service typu LoadBalancer wzbogacony o adnotacje określające protokół backendu i timeout.

**Wymagania:**

1. Deployment `httpd-server`:

    * 2 repliki, obraz `httpd:2.4-alpine`
    * Port 80
    * Etykieta: `app: httpd-app`
2. Service `httpd-lb`:

    * Typ: `LoadBalancer`
    * Selekcja: `app: httpd-app`
    * Port 80 → targetPort 80
    * Adnotacje (przykład AWS):

        * `service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"`
        * `service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"`

**Wskazówki:**

* W zależności od chmury użyć odpowiednich adnotacji (GCP, Azure, AWS).
* Po utworzeniu sprawdzić, czy LB otrzymał zewnętrzny IP/hostname (`kubectl get svc httpd-lb`).
* Testować zachowanie podczas długotrwałych połączeń (np. `ab -n 1000 -c 10 http://<LB_IP>/`), by weryfikować timeout.

---

## 7. Deployment Frontend + PodDisruptionBudget (PDB)

**Cel:**
Ustawić PodDisruptionBudget, aby podczas działań utrzymaniowych (np. `kubectl drain`) w klastrze zawsze było dostępne przynajmniej 2 spośród 4 podów frontendu.

**Wymagania:**

1. Deployment `frontend-app`:

    * 4 repliki, obraz `nginx:1.19`
    * Etykieta: `app: frontend-app`
2. PodDisruptionBudget `frontend-pdb`:

    * `minAvailable: 2`
    * `selector`: `app: frontend-app`
3. Test: podczas `kubectl drain <node>` upewnić się, że PDB blokuje usunięcie podów, jeżeli spowodowałoby to spadek poniżej 2 dostępnych replik.

**Wskazówki:**

* PDB typu `policy/v1`, `kind: PodDisruptionBudget`.
* Aby zobaczyć blokadę, zrobić `kubectl drain <node> --ignore-daemonsets` i obserwować komunikat o naruszeniu PDB.
* Po odblokowaniu (np. odczekaniu, aż nowe pody wystartują w innym węźle) drain dokończy się poprawnie.

---

## 8. Deployment Python z ConfigMap i Secret jako wolumeny

**Cel:**
Użyć ConfigMap i Secret, montując je jako pliki w kontenerze, aby aplikacja Python odczytywała konfigurację i hasło z plików.

**Wymagania:**

1. ConfigMap `app-config` zawierająca np. plik `config.json` z przykładową treścią:

   ```json
   {
     "service_url": "https://api.example.com",
     "retry_count": 3
   }
   ```
2. Secret `app-secret` zawierający klucz `db_password` (np. `password123`).
3. Deployment `python-app`:

    * 2 repliki, obraz `python:3.9-alpine`
    * Montowanie ConfigMap w `/etc/config/config.json` (jako plik)
    * Montowanie Secret w `/etc/secret/db_password` (jako plik)
    * Komenda uruchomienia:

      ```bash
      python /usr/src/app/main.py --config /etc/config/config.json --password-file /etc/secret/db_password
      ```
    * Etykieta: `app: python-app`

**Wskazówki:**

* Tworzyć ConfigMap przez `kubectl create configmap app-config --from-file=config.json=./config.json`.
* Tworzyć Secret przez `kubectl create secret generic app-secret --from-literal=db_password=password123`.
* W spec.template.spec Deploymentu definiować dwa wolumeny: jeden typu `configMap`, drugi typu `secret`, a następnie `volumeMounts`.
* Aplikacja (w symulacji) może jedynie wypisać zawartość tych plików, by zweryfikować, że montaż działa.

---

## 9. Service ClusterIP i komunikacja między namespace’ami

**Cel:**
Stworzyć dwie aplikacje w oddzielnych namespace’ach: backend dostępny przez Service typu ClusterIP oraz frontend (Nginx) w drugim namespace’ie, który proxy-passes ruch do backendu przez pełną nazwę DNS.

**Wymagania:**

1. Namespace `backend-ns`:

    * Deployment `backend-app`:

        * 2 repliki, obraz `hashicorp/http-echo:0.2.3`, argument `-text="Hello from backend"`, port 5678
        * Etykieta: `app: backend-app`
    * Service `backend-svc` (ClusterIP):

        * Port 8080 → targetPort 5678
        * Selekcja `app: backend-app`
2. Namespace `frontend-ns`:

    * ConfigMap `nginx-conf` zawierająca plik `nginx.conf`, w którym:

      ```
      events {}
      http {
        server {
          listen 80;
          location / {
            proxy_pass http://backend-svc.backend-ns.svc.cluster.local:8080;
          }
        }
      }
      ```
    * Deployment `frontend-app`:

        * 2 repliki, obraz `nginx:1.21-alpine`
        * Montowanie powyższego ConfigMap jako `/etc/nginx/nginx.conf` (subPath: `nginx.conf`)
        * Etykieta: `app: frontend-app`
    * Service `frontend-svc` (NodePort):

        * Port 80 → targetPort 80
        * NodePort np. 30080
        * Selekcja `app: frontend-app`

**Wskazówki:**

* Utworzyć namespace’y: `kubectl create namespace backend-ns` oraz `kubectl create namespace frontend-ns`.
* W `frontend-ns` utworzyć ConfigMap z plikiem `nginx.conf`.
* W Deploymentach odpowiednio ustawić `namespace` i `labels`.
* Sprawdzić:

    * Z poda w `frontend-ns`: `curl http://backend-svc.backend-ns.svc.cluster.local:8080`
    * Z zewnątrz: `curl http://<NodeIP>:30080` → powinno zwrócić “Hello from backend”.

---

## 10. Horizontal Pod Autoscaler (HPA) na przykładzie “cpu-burner”

**Cel:**
Utworzyć Deployment wykorzystujący obraz generujący obciążenie CPU oraz skonfigurować HPA, który skaluje liczbę podów w zależności od średniego wykorzystania CPU.

**Wymagania:**

1. Deployment `cpu-burner`:

    * Obraz: `vish/stress`
    * Args: `--cpu 1 --timeout 600s` (uruchamia “burn” CPU)
    * Początkowo `replicas: 1`
    * Ustawić zasoby:

        * `resources.requests.cpu: "200m"`
        * `resources.limits.cpu: "500m"`
    * Etykieta: `app: cpu-burner`
2. HorizontalPodAutoscaler `cpu-burner-hpa`:

    * target Deployment: `cpu-burner`
    * `minReplicas: 1`
    * `maxReplicas: 4`
    * target CPU utilization: 50%
3. Weryfikacja:

    * Uruchomić dodatkowe obciążenie w podzie (`kubectl exec -it <pod> -- stress --cpu 2 --timeout 300s`) lub analogiczny sposób
    * Obserwować skalowanie: `kubectl get hpa cpu-burner-hpa --watch`

**Wskazówki:**

* Deployment definiuje sekcję `resources`, żeby HPA mogło zbierać metryki CPU.
* HPA w wersji `autoscaling/v2` (lub `autoscaling/v2beta2` w starszych klastrach).
* Po zakończeniu obciążenia liczba replik powinna wrócić do 1.

---

## 11. NetworkPolicy: Izolacja komunikacji w jednym namespace

**Cel:**
W namespace `secure-ns` pozwolić tylko front-endowi na połączenie do back-endu, a zablokować każdy inny ruch.

**Wymagania:**

1. Namespace `secure-ns`.
2. Deployment `backend-app`:

    * 1 replika, obraz `hashicorp/http-echo:0.2.3`, `-text="Hello Secure Backend"`, port 5678
    * Etykieta: `app: backend-app`
3. Service `backend-svc` (ClusterIP):

    * Port 80 → targetPort 5678
    * Selekcja `app: backend-app`
4. Deployment `frontend-app`:

    * 1 replika, obraz `alpine:3.14`, w pętli co 10 s wykonuje `curl http://backend-svc` i wypisuje wynik
    * Etykieta: `app: frontend-app`
5. Etap 1 (bez polityki): zweryfikować, że `frontend-app` dostaje odpowiedź z `backend-app`.
6. Etap 2: utworzyć NetworkPolicy `allow-frontend-to-backend`, w której:

    * `podSelector`: `app: backend-app`
    * `policyTypes: ["Ingress"]`
    * `ingress.from`: źródło z `podSelector: {app: frontend-app}`
    * `ports`: TCP 80
    * Brak innych reguł oznacza deny-by-default dla pozostałych podów

**Wskazówki:**

* Po utworzeniu NetworkPolicy front-end musi dalej się łączyć, ale żaden inny pod (np. tymczasowy busybox) nie ma dostępu.
* Test: z `frontend-app` i z tymczasowego `busybox` w `secure-ns`.

---

## 12. NetworkPolicy: Izolacja między namespace’ami i zewnętrzne IP

**Cel:**
W namespace `prod-ns` najpierw całkowicie zablokować ruch przychodzący (deny-all), a następnie otworzyć dostęp tylko dla podów z `test-ns` oraz z określonego zakresu IP.

**Wymagania:**

1. Namespace `prod-ns` i `test-ns`.
2. Deployment `prod-app` w `prod-ns`:

    * 1 replika, obraz `nginx:1.21-alpine`, port 80
    * Etykieta: `app: prod-app`
3. Service `prod-svc` (ClusterIP) w `prod-ns`, 80 → 80
4. Deployment `test-client` w `test-ns`:

    * 1 replika, obraz `busybox:1.32`, w pętli co 10 s próbuje `wget http://prod-svc.prod-ns.svc.cluster.local`
    * Etykieta: `app: test-client`
5. Krok 1: w `prod-ns` utworzyć NetworkPolicy `deny-all-ingress`:

    * `podSelector: {}` (wszystkie pody)
    * `policyTypes: ["Ingress"]`
    * `ingress: []` (brak reguł) → deny all
6. Krok 2: w `prod-ns` utworzyć NetworkPolicy `allow-test-and-external`:

    * `podSelector: {app: prod-app}`
    * `policyTypes: ["Ingress"]`
    * `ingress.from`:

        1. Źródło: `namespaceSelector: {matchLabels: {name: test-ns}}` + `podSelector: {app: test-client}`
        2. `ipBlock: {cidr: "203.0.113.0/24"}`
    * `ports: TCP 80`

**Wskazówki:**

* Namespace `test-ns` musi mieć etykietę `name=test-ns`, by `namespaceSelector` zadziałał.
* Polityki są łączone (union) – wystarczy, że któraś polityka pozwoli ruch do pódów `app=prod-app`.
* Testować: przed polityką ruch działa, po deny-all nic nie działa, po allow regułach ruch z `test-client` i z IP z `203.0.113.0/24` działa.

---

## 13. Ingress: Proste routowanie HTTP (ścieżki /v1 i /v2)

**Cel:**
Skonfigurować Ingress, który przekierowuje ruch `/v1` do jednej aplikacji, a `/v2` do drugiej, na jednym hoście.

**Wymagania:**

1. Namespace `web-ns`.
2. Deployment `app-v1` w `web-ns`:

    * 1 replika, obraz `hashicorp/http-echo:0.2.3 -text="App v1"`, port 5678
    * Etykieta: `app: app-v1`
3. Service `app-v1-svc` (ClusterIP): 80 → 5678
4. Deployment `app-v2` w `web-ns`:

    * 1 replika, obraz `hashicorp/http-echo:0.2.3 -text="App v2"`, port 5678
    * Etykieta: `app: app-v2`
5. Service `app-v2-svc` (ClusterIP): 80 → 5678
6. Ingress `web-ingress` w `web-ns`:

    * `annotations: { "kubernetes.io/ingress.class": "nginx" }` (lub inna klasa)
    * `rules`:

        * `host: example.com`

            * `paths`:

                * `/v1` → service `app-v1-svc:80`
                * `/v2` → service `app-v2-svc:80`

**Wskazówki:**

* Upewnić się, że w klastrze jest kontroler Ingress (np. NGINX Ingress).
* Po utworzeniu Ingress dodać do `/etc/hosts` rekord `ADDRESS example.com`.
* Testować:

    * `curl -H "Host: example.com" http://example.com/v1` → “App v1”
    * `curl -H "Host: example.com" http://example.com/v2` → “App v2”.

---

## 14. Ingress z TLS (HTTPS) i wieloma hostami

**Cel:**
Zadeklarować Ingress, który obsługuje dwa różne hosty (example1.com i example2.com) przez HTTPS, używając jednego Secret typu TLS.

**Wymagania:**

1. Namespace `web-ns`.
2. Deployment `app1` w `web-ns`:

    * 1 replika, obraz `hashicorp/http-echo:0.2.3 -text="Hello from app1"`, port 5678
    * Etykieta: `app: app1`
3. Service `app1-svc` (ClusterIP): 80 → 5678
4. Deployment `app2` w `web-ns`:

    * 1 replika, obraz `hashicorp/http-echo:0.2.3 -text="Hello from app2"`, port 5678
    * Etykieta: `app: app2`
5. Service `app2-svc` (ClusterIP): 80 → 5678
6. TLS Secret `tls-secret` w `web-ns`, zawierający klucz `tls.key` i certyfikat `tls.crt` (np. self-signed)
7. Ingress `multi-host-ingress` w `web-ns`:

    * `annotations: { "kubernetes.io/ingress.class": "nginx" }`
    * `tls:`

        * `secretName: tls-secret`
        * `hosts: [ "example1.com", "example2.com" ]`
    * `rules`:

        * host `example1.com`: path `/` → service `app1-svc:80`
        * host `example2.com`: path `/` → service `app2-svc:80`

**Wskazówki:**

* Jeżeli nie masz certyfikatu, wygenerować self-signed with OpenSSL.
* Utworzyć Secret TLS:

  ```
  kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key -n web-ns
  ```
* Po utworzeniu Ingress sprawdzić `kubectl get ingress multi-host-ingress -n web-ns` → `ADDRESS`.
* Dodać do `/etc/hosts`:

  ```
  <ADDRESS> example1.com example2.com
  ```
* Testować:

    * `curl -k https://example1.com/` → “Hello from app1”
    * `curl -k https://example2.com/` → “Hello from app2”
      (flag `-k` przy self-signed cert).


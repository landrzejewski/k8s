## 1. Podstawy działania Network Policies

### 1.1 Co to jest Network Policy?

Network Policy to zasób (obiekt) Kubernetes w ramach API grupy **`networking.k8s.io/v1`**, który definiuje zasady filtrowania ruchu sieciowego. Jest to swego rodzaju firewall na poziomie warstwy 3/4 (IP/TCP/UDP) wewnątrz klastra. Pozwala dokładnie określić:

* które pody mogą komunikować się między sobą (ingress),
* do jakich zewnętrznych adresów pody mogą wysyłać ruch (egress).

### 1.2 Dlaczego warto stosować Network Policies?

1. **Bezpieczeństwo**

    * Minimalizacja powierzchni ataku – blokowanie niepotrzebnego ruchu wewnątrz klastra.
    * Izolacja aplikacji – wrażliwe mikroserwisy nie są niepotrzebnie dostępne dla innych komponentów.

2. **Separacja środowisk**

    * Deweloperskie, testowe i produkcyjne namespace’y mogą być odizolowane od siebie.
    * Oddzielenie ruchu między różnymi zespołami lub modułami.

3. **Wymagania regulacyjne i audytowe**

    * Spełnianie norm, które narzucają kontrolę ruchu sieciowego (np. PCI DSS, HIPAA).

### 1.3 Model działania

* **Domyślne ustawienie klastra**: jeśli w przestrzeni nazw (namespace) nie ma żadnej Network Policy, ruch do/pomiędzy podami jest **dozwolony**.
* **Pierwsza utworzona polityka**: w namespace, w którym zdefiniowano przynajmniej jedną Network Policy, obowiązuje zasada “deny-by-default” dla ruchu w kierunku **ingress** – tzn. wszystkie połączenia do podów są blokowane, chyba że wyraźnie zostanie do nich dozwolony ruch przez politykę.
* **Egress**: dopuszczanie lub odrzucanie ruchu wychodzącego (domyślnie egress jest dozwolony, aż do momentu, gdy pojawi się polityka uwzględniająca `policyTypes: ["Egress"]`).

Mechanizm enforcementu Network Policy jest realizowany przez wtyczki sieciowe (CNI – Container Network Interface), takie jak Calico, Weave, Cilium, Kube-router czy Canal. Jeżeli w klastrze nie ma włączonej wtyczki, która implementuje Network Policies, to zdefiniowane polityki zostaną zignorowane.

---

## 2. Wymagania w klastrze

1. **Plugin CNI wspierający Network Policies**
   W praktyce najczęściej używane są:

    * Calico
    * Cilium
    * Weave Net
    * Kube-router
    * Canal (kombinacja Calico i Flannel)

   Przykładowo, Calico zapewnia pełną obsługę Network Policies zgodną ze specyfikacją Kubernetes i dodatkowo wzbogaconą o funkcje L7 (np. filtrowanie HTTP).

2. **Wersja Kubernetes ≥ 1.9**
   Oficjalne wsparcie dla zasobu NetworkPolicy zostało dodane już w Kubernetes 1.3, a od 1.9 jest to zasadniczo stabilna funkcjonalność v1 API.

3. **Włączona opcja `NetworkPolicy` w konfiguracji klastra (dla niektórych instalatorów)**
   W kubeadm, eks, gke, aks zazwyczaj nie trzeba nic robić – obsługa jest domyślnie włączona, ale należy upewnić się, czy wybrany CNI wspiera Network Policies.

4. **Namespace, w którym definiujemy polityki**
   Polityki działają zawsze w kontekście konkretnego namespace’u: nie można nimi wymusić reguł w innych namespace’ach.

Jeżeli spełnione są powyższe warunki, możemy rozpocząć definiowanie i testowanie Network Policies.

---

## 3. Kluczowe elementy NetworkPolicy

Zasób **NetworkPolicy** składa się z kilku fundamentalnych części:

1. **`podSelector`**

    * Określa, do jakich podów w namespace’u odnosi się dana polityka.
    * Jest to wymagane.
    * Przykładowo:

      ```yaml
      podSelector:
        matchLabels:
          app: frontend
      ```

      albo, aby objąć wszystkie pody:

      ```yaml
      podSelector: {}   # pusty selektor – wszystkie pody w namespace
      ```

2. **`policyTypes`**

    * Określa, czy polityka dotyczy ruchu przychodzącego (**Ingress**), wychodzącego (**Egress**) czy obu.
    * Jeśli pominąć to pole, to w momencie obecności w spec’yficznymi regułami dla ingress/egress Kubernetes sam ustawi domyślnie odpowiedni typ.
    * Przykład:

      ```yaml
      policyTypes:
        - Ingress
        - Egress
      ```
    * Możliwe wartości: `Ingress`, `Egress`.

3. **`ingress`** (opcjonalnie)

    * Lista reguł definiujących, z jakich źródeł i na jakich portach ruch jest dozwolony do wybranych podów.
    * Każda reguła składa się z pól:

        * `from`: definiuje źródła (podSelector, namespaceSelector, ipBlock).
        * `ports`: definiuje porty i protokoły (TCP/UDP).

      Przykład pojedynczej reguły:

      ```yaml
      ingress:
        - from:
            - podSelector:
                matchLabels:
                  role: backend
            - namespaceSelector:
                matchLabels:
                  project: foo
          ports:
            - protocol: TCP
              port: 8080
      ```

4. **`egress`** (opcjonalnie)

    * Analogicznie do ingress, ale dotyczy ruchu wychodzącego z podów objętych `podSelector`.
    * Możemy określić `to` zamiast `from`:

      ```yaml
      egress:
        - to:
            - ipBlock:
                cidr: 10.0.0.0/24
          ports:
            - protocol: TCP
              port: 53    # np. zezwalamy na DNS do serwera w sieci 10.0.0.0/24
      ```

5. **`namespaceSelector`**

    * Pozwala zaznaczyć pody w innych namespace’ach.
    * Przykład:

      ```yaml
      namespaceSelector:
        matchLabels:
          team: devops
      ```
    * Uwaga: jeżeli jednocześnie określimy `podSelector` i `namespaceSelector`, oznacza to „pody w wybranym namespace, które mają dane labelki”.

6. **`ipBlock`**

    * Umożliwia zdefiniowanie zewnętrznych (adresatów poza klastra) zakresów IP.
    * Składa się z pola `cidr` (np. `10.1.0.0/16`) oraz opcjonalnego `except`, czyli listy wykluczonych podsieci.
    * Przykład:

      ```yaml
      - ipBlock:
          cidr: 0.0.0.0/0
          except:
            - 10.1.3.0/24
            - 10.1.4.0/24
      ```

7. **`ports`**

    * W ramach reguły (ingress lub egress) możemy podać listę portów i protokołów, np.

      ```yaml
      ports:
        - protocol: TCP
          port: 443
        - protocol: UDP
          port: 67
      ```

8. **Łączenie reguł**

    * **OR** między elementami w liście: jeśli zdefiniujemy kilka wpisów w `ingress` lub `egress`, ruch jest dozwolony, jeżeli spełnia **przynajmniej jedną** regułę.
    * W obrębie jednej reguły, gdy podamy wiele `from` lub `to`, to również jest traktowane jako OR: wystarczy, że źródło należy do jednego z wymienionych.
    * Wewnątrz warstwy `ports`: wystarczy, że port/protokół pasuje do jednego z wymienionych wpisów.

9. **Scoping**

    * Ważne: wszystkie selektory (pod, namespace) odnoszą się tylko do tego samego namespace, chyba że korzystamy z `namespaceSelector`.
    * Nie możemy w NetworkPolicy odnosić się do resource po nazwie podu bez użycia selektora. Zawsze jest to matchLabel.

---

## 4. Przykładowe scenariusze i manifesty

W tej sekcji pokażemy praktyczne przykłady użycia Network Policies, zaczynając od najbardziej podstawowych.

---

### 4.1. Domyślne odrzucanie ruchu (Deny-all)

Najpierw warto zobaczyć, jak wymusić, żeby wszystkie połączenia typu ingress (przychodzące do podów) zostały zablokowane, aż do momentu wprowadzenia innych reguł.

> **Uwaga**: po utworzeniu **przynajmniej jednej** polityki w danym namespace, wszystkie pody w tym namespace (objęte pustym `podSelector`) będą miały ruch przychodzący zablokowany, chyba że zostaną dozwolone reguły.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-namespace
spec:
  podSelector: {}         # selektor pusty – obejmuje wszystkie pody w namespace “my-namespace”
  policyTypes:
    - Ingress             # dotyczy tylko ruchu przychodzącego
  # brak definicji ingress => wszystko jest zablokowane
```

* Po wywołaniu `kubectl apply -f default-deny-all.yaml` każdy ruch przychodzący (także kube-dns, liveness/readiness probes itp.) do podów w `my-namespace` zostanie odrzucony.
* Jeśli mamy np. usługę typu `ClusterIP` w tym samym namespace, jej próba dotarcia do poda zablokuje się – brak tego ustawienia może spowodować przerwanie komunikacji.

Aby przywrócić określony ruch, musimy dodać kolejną NetworkPolicy pozwalającą na konkretne połączenia (zobacz sekcję 4.2).

---

### 4.2. Zezwalanie na ruch wewnątrz namespace’u

Załóżmy, że w namespace `my-namespace` mamy 2 deploymenty:

* Service A z labelami `app: service-a`
* Service B z labelami `app: service-b`

Chcemy wymusić, by Service A mógł komunikować się z Service B po porcie 8080, ale nikt inny nie. Najpierw blokujemy wszystko tak jak wyżej, a następnie definiujemy regułę, która zezwoli tylko na taki ruch.

#### 4.2.1. Krok 1: default-deny (opisany w sekcji 4.1)

#### 4.2.2. Krok 2: zezwolenie na ruch

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-a-to-b
  namespace: my-namespace
spec:
  podSelector:
    matchLabels:
      app: service-b      # docelowe pody: pod o labelek “app=service-b”
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: service-a  # źródłowe pody: “app=service-a”
      ports:
        - protocol: TCP
          port: 8080
```

**Wyjaśnienie:**

* Tę politykę przypisujemy również do namespace `my-namespace`.
* `podSelector.matchLabels: app: service-b` – reguła dotyczy wszystkich podów z tym labelem.
* W `ingress.from` definiujemy, że tylko pody posiadające `app: service-a` mogą nawiązać połączenie.
* `ports` – tylko ruchem TCP na port 8080.

Teraz:

* Ruch z Service A do Service B: **dopuszczony** na TCP/8080.
* Inny ruch do Service B: **zablokowany**.
* Ruch między Service A a innymi podami (np. Service C) – nie jest dopuszczony, ponieważ nie zdefiniowaliśmy polityki pozwalającej.

> **Uwaga**: Jeśli chcemy zezwolić też na ruch wychodzący (Egress) z Service A, musimy stworzyć osobną politykę z `policyTypes: ["Egress"]`, ale w prostym scenariuszu często nie definiujemy egress, bo domyślnie jest on dozwolony, dopóki nie utworzymy polityki egress w tym namespace.

---

### 4.3. Izolacja namespace’ów

Często chcemy, aby pody w jednym namespace nie mogły komunikować się z podami w innym namespace. Jedna technika to:

1. W każdym namespace odfiltrować domyślny ruch (deny-all),
2. Dodać polityki zezwalające tylko na ruch wewnątrz własnego namespace (albo dozwolić dostęp z wybranych namespace’ów).

#### 4.3.1. Krok 1: default-deny we wszystkich namespace’ach

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Zastosuj ten manifest w każdym namespace, w którym chcesz wymusić izolację.

#### 4.3.2. Krok 2: zezwolenie na ruch wewnątrz namespace’u

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal-namespace
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: "dev"   # jeśli namespace dev ma taką labelkę
```

* `namespaceSelector.matchLabels: kubernetes.io/metadata.name: "dev"` – często systemowa labelka namespace’u (`kubernetes.io/metadata.name`) odpowiada nazwie namespace’u, ale można dodać też własne labelki.
* Dzięki temu tylko pody z namespace’u `dev` mogą komunikować się między sobą; pozostałe próby zostaną zablokowane.

Jeżeli chcemy, aby kilka namespace’ów miało wzajemny dostęp, należy w każdym z nich dodać odpowiednie reguły `from` wskazujące na pozostałe namespace’y.

---

### 4.4. Ograniczanie ruchu egress (wychodzącego)

Domyślnie, dopóki nie zdefiniujemy polityki `egress` w danym namespace, ruch wychodzący (np. do internetu) jest **dopuszczony**. Aby go ograniczyć:

#### 4.4.1. Przykład: zezwalamy tylko na połączenia HTTP do zewnętrznej usługi (np. API płatności)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-payments
  namespace: payments-namespace
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 52.23.45.0/24      # przykładowa podsieć zewnętrznego API
      ports:
        - protocol: TCP
          port: 443
```

**Wyjaśnienie:**

* `podSelector.matchLabels: app: payment-service` – dotyczy podsów z tym labelem.
* `policyTypes: ["Egress"]` – kontrolujemy tylko ruch wychodzący.
* `egress.to.ipBlock.cidr` – ograniczamy, do której sieci zewnętrznej mogą się łączyć pody.
* `ports` – tylko ruch TCP/443 (HTTPS).

W efekcie:

* Pody oznaczone `app=payment-service` mogą wysyłać ruch tylko do `52.23.45.0/24:443` (np. do zewnętrznego endpointu płatności).
* Wszelki inny ruch egress (np. do DNS, innych API) zostanie zablokowany. Jeśli potrzebujemy DNS, to warto dodać jeszcze regułę pozwalającą na UDP/53 do kube-dns.

#### 4.4.2. Pozwolenie na egress do kube-dns

Aby zachować funkcjonalność rozwiązywania nazw, dodajmy regułę:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns
  namespace: payments-namespace
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: "kube-system"
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

* Ten manifest zezwala na połączenia UDP/53 (DNS) do podów `kube-dns` w namespace `kube-system`.

> **Ważne**: Jeżeli chcemy mieć tylko **jedną** politykę egress, w której zdefiniujemy zarówno dostęp do zewnętrznej usługi płatności, jak i DNS, możemy połączyć obie podsekcje `egress` w jednej definicji, np.:
>
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: egress-payments-and-dns
>   namespace: payments-namespace
> spec:
>   podSelector:
>     matchLabels:
>       app: payment-service
>   policyTypes:
>     - Egress
>   egress:
>     - to:
>         - ipBlock:
>             cidr: 52.23.45.0/24
>       ports:
>         - protocol: TCP
>           port: 443
>     - to:
>         - namespaceSelector:
>             matchLabels:
>               kubernetes.io/metadata.name: "kube-system"
>           podSelector:
>             matchLabels:
>               k8s-app: kube-dns
>       ports:
>         - protocol: UDP
>           port: 53
> ```

---

### 4.5. Zaawansowane scenariusze

#### 4.5.1. Izolacja ruchu między labelkami w jednym namespace

Czasem chcemy wymusić, żeby tylko pody z określoną etykietą (label) wewnątrz jednego namespace’u mogły się komunikować, a reszta była odizolowana. Na przykład: pody z labelką `tier=frontend` niech komunikują się tylko z `tier=backend`, ale nie z innymi frontendami.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend-only
  namespace: my-namespace
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 80
```

* Blokujemy domyślnie egress z podów `tier=frontend`,
* Zezwalamy tylko na połączenia TCP/80 do podów `tier=backend`.
* Jeśli frontend ma inne potrzeby (np. do DNS), trzeba dodać reguły jak w sekcji 4.4.2.

#### 4.5.2. Użycie `ipBlock` z wyłączeniami

Może istnieć sytuacja, gdy chcemy zezwolić na cały ruch do sieci (np. 10.0.0.0/16), ale wyłączyć konkretną podsieć (np. 10.0.5.0/24). Używamy wówczas pola `except`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-except-special-subnet
  namespace: my-namespace
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/16
            except:
              - 10.0.5.0/24
```

W rezultacie:

* Ruch egress dla każdego poda w `my-namespace` do adresów z zakresu `10.0.0.0/16` jest dozwolony,
* ale ruch do `10.0.5.0/24` jest zablokowany.

#### 4.5.3. Polityka warstwy L7 (przykład z Calico)

Standardowe Network Policies Kubernetes operują na warstwach L3/L4 (adresy IP, porty, protokoły). Aby wprowadzić reguły L7 (np. filtrowanie HTTP po ścieżce, host header), trzeba korzystać z rozszerzeń oferowanych przez niektóre CNI, np. Calico Advanced Policy (typu GlobalNetworkPolicy albo używając CRD `NetworkPolicy` z dodatkowymi polami).

Przykład: blokujemy wszystkie żądania HTTP, które mają nagłówek `User-Agent` zawierający słowo “curl”, przy użyciu Calico (GlobalNetworkPolicy):

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: block-curl
spec:
  selector: app == 'frontend'
  order: 100
  ingress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        ports: [80, 443]
      http:
        - match:
            headers:
              User-Agent: 'curl*'
          action: Deny
    - action: Allow
  egress:
    - action: Allow
```

* `selector: app == 'frontend'` – stosujemy do podów z tym labelem.
* Pierwsza reguła ingress (`action: Allow`) dopuszcza ruch TCP na porty 80, 443, **ale** w sekcji `http.match.headers` definiujemy warunek, że jeśli `User-Agent` pasuje do wzorca `curl*`, to akcja będzie `Deny`.
* Druga reguła po prostu zezwala na pozostały ruch.

> **Uwaga**: powyższy przykład to już wykracza poza standardowe CRD Kubernetes NetworkPolicy i wymaga Calico (lub innej wtyczki wspierającej L7).

---

## 5. Testowanie i weryfikacja

Aby sprawdzić, czy Network Policy działa zgodnie z oczekiwaniami, najczęściej korzysta się z prostych narzędzi uruchomionych w podach, np. `curl`, `wget`, `nc` (netcat) czy `ping`. Poniżej przykładowy workflow:

1. **Przygotowanie dwóch tymczasowych podów testowych**

   ```bash
   # w namespace “my-namespace”
   kubectl run -it --rm test-a --image=busybox --restart=Never -- sh
   ```

   To uruchomi shell w podzie test-a (BusyBox). Następnie w innym terminalu:

   ```bash
   kubectl run -it --rm test-b --image=busybox --restart=Never -- sh
   ```

2. **Sprawdzenie, czy porty są otwarte lub zamknięte**
   W podzie test-a spróbuj wykonać połączenie do test-b:

   ```sh
   # załóżmy, że wiemy IP test-b (możemy je uzyskać przez `kubectl get pods -o wide`)
   nc -zv <IP_TEST_B> 80
   ```

    * Jeśli polityka zabrania ruchu, zobaczymy komunikat `connection refused` lub “no route to host”.
    * Jeśli polityka zezwala, zobaczymy “succeeded” lub “open\`.

3. **Test DNS**
   Jeśli mamy politykę egress ograniczającą DNS, przetestuj:

   ```sh
   nslookup google.com 10.96.0.10   # 10.96.0.10 to przykładowy IP kube-dns
   ```

   Błędy w tej części pomogą zidentyfikować brak reguł zezwalających na DNS.

4. **Test z użyciem HTTP**
   Jeżeli w sieci standardowo mamy usługę HTTP (np. serwis w klastrze z typem ClusterIP):

   ```sh
   wget -O- http://service-b.default.svc.cluster.local:8080
   ```

   gdzie `service-b.default.svc.cluster.local` – wirtualna nazwa usługi.

5. **Sprawdzanie Logów CNI / Calico**

    * W przypadku Calico możemy zajrzeć do logów komponentów:

      ```bash
      kubectl logs -n kube-system calico-node-<nazwa-poda>
      ```
    * Często dostarczają informacji o blokowanych pakietach, politykach dopasowanych do połączeń itd.

6. **Kubernetes Network Policy Status**
   Możliwe jest też użycie narzędzi takich jak `kubectl get networkpolicies -A` aby zobaczyć wszystkie polityki w klastrze. Warto sprawdzić, czy dany zasób został poprawnie utworzony i czy selektory dopasowują pody (np. `kubectl describe networkpolicy <nazwa>` pozwoli podejrzeć, które pody są objęte regułą).

---

## 6. Najczęstsze problemy i wskazówki praktyczne

1. **Brak wtyczki CNI wspierającej Network Policies**

    * Objawy: tworzenie NetworkPolicy przebiega bez błędów, ale ruch nie jest ograniczany.
    * Rozwiązanie: sprawdź dokumentację klastra, czy wybrana wtyczka CNI wspiera polityki; w razie potrzeby zainstaluj lub zmień na taką, która ma wsparcie (np. Calico, Cilium).

2. **Niepoprawny `podSelector` lub `namespaceSelector`**

    * Jeżeli selektor nie pasuje do żadnego poda, reguła nijak niczego nie zmienia.
    * Warto upewnić się, że labelki w manifestach zgadzają się z faktycznymi labelkami w podach.
    * Komenda do sprawdzenia:

      ```bash
      kubectl get pods --show-labels -n my-namespace
      ```

3. **Polityki wzajemnie się nakładają (kolejność)**

    * W Kubernetes kolejność polityk nie ma znaczenia – reguły są łączone. Jeśli jedna polityka zezwala na ruch, a inna zabrania, to **deny** (brak explicit allow) zwykle dominuje w przypadku, gdy nie ma wyraźnej reguły allow.
    * Dlatego lepiej czasem utworzyć jedną spójną politykę niż wiele rozproszonych, żeby uniknąć niezamierzonego denia.

4. **Ruch pomiędzy namespace’ami**

    * Jeżeli chcemy dopuszczać ruch cross-namespace, musimy korzystać z `namespaceSelector`.
    * Często zapomina się o dodaniu dodatkowej labels do namespace’ów:

      ```bash
      kubectl label namespace frontend-ns team=frontend
      kubectl label namespace backend-ns team=backend
      ```
    * A w polityce potem:

      ```yaml
      namespaceSelector:
        matchLabels:
          team: backend
      ```

5. **Poziom granularności portów**

    * Możemy nie tylko podać numer portu, ale także jego nazwę (jeśli w manifeście kontenera port jest nazwany).
    * Przykład:

      ```yaml
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: http
      ```
    * Zweryfikuj, czy port nazwa jest zgodny z tym, co deklarujesz w speczie kontenera.

6. **Egress do usług w klastrze**

    * Jeżeli tworzymy politykę egress i nie włączymy reguły dla ruchu do kube-dns lub do serwisu Headless (np. Elasticsearch), to pody mogą przestać działać poprawnie.
    * Upewnij się, że w każdej polityce egress, która ogranicza, dodasz zasady zezwalające na:

        * UDP/53 do kube-dns,
        * TCP odpowiednio do komponentów wewnętrznych (np. API Kubernetes, itp.).

7. **Spójność polityk w multi-tenancy**

    * W środowisku, gdzie wiele zespołów korzysta z tego samego klastra, warto wprowadzić standaryzację labeli w namespace’ach i podach, by łatwo definiować globalne polityki.
    * Zespół DevOps może zadbać o centralne polityki pozwalające tylko na minimalny niezbędny ruch, a zespoły deweloperskie dostosowują je w potrzebie wewnątrz własnych namespace’ów.

8. **Debugging za pomocą `kubectl` i narzędzi zewnętrznych**

    * `kubectl describe networkpolicy <nazwa>`
    * `kubectl exec -it <pod> -- /bin/sh` + `nc`, `curl`, `ping`
    * Narzędzia jak `kube-router` czy `cilium tool` dostarczają bardziej zaawansowane informacje, np. `cilium monitor` przy Cilium.
    * W przypadku Calico można użyć `calicoctl` do sprawdzenia statusu reguł:

      ```bash
      calicoctl get globalnetworkpolicy
      ```

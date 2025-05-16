# Dokumentacja Zespół 7 

### Misja 0 - kryptonim "Red Tape"
Najlepiej napisany program i najlepiej wdrożony system jest tykającą bombą bez odpowiedniej dokumentacji. Nikt nie lubi jej pisać, ale jest kluczowa dla łatwości późniejszego utrzymywania i efektywnej współpracy - a także do sprawdzania zadań! Dlatego udokumentuj wszystkie zadania, co najmniej oznaczając te które zostały wykonane, bo **tylko te zostaną sprawdzone**. Niektóre zadania wymagają pisemnej odpowiedzi, umieść je też w dokumentacji. Zalecany - nie, JEDYNIE SŁUSZNY - sposób na przedstawienie dokumentacji to utworzenie repozytorium GitHub (polecam utworzenie forka tego repo).

Opisz kroki wykonane w celu realizacji zadania, szczególnie lokalizacje zasobów, użyte opcje i komendy - nie musisz tego robić bardzo dokładnie, ale w razie wątpliwości będą one działać na twoją korzyść. Na przykład jeśli zadanie nie zostało do końca wykonane, ale znacząca część kroków jest opisana poprawnie, zaliczę za to częściowe punkty. Albo jeśli zadanie zostało wykonane, ale nie w sposób jakiego oczekiwałem, to opis będzie kluczem do uzyskania za nie punktów. To, co nie jest opisane, a nie jest oczywiste z interfejsu Ranchera, będzie rozstrzygane na twoją niekorzyść!
#### Rozwiązanie
- ***GOTOWE***

### Misja 1 - operacja "Otwarte Okno"
Potrzebujemy nowego serwera webowego do ogłaszania krytycznych informacji ze Sztabu Zarządzania Kryzysowego, ale minister cały dzień jadł ośmiorniczki i dlatego dopiero teraz dotarły rozkazy. Wszyscy inni poszli już do domu, więc cała nadzieja w waszym zespole.
Na klastrze "potyczki" utwórz projekt "szk-server" a w nim namespace "ogloszenia-krytyczne". W tym namespace uruchom serwer webowy. Nie mówią jaki, więc użyj dowolnego, ale ma być w najnowszej wersji. **3pkt**
- Utwórz usługę dzieki której można się odwołać do naszego serwera z całego klastra **3pkt**
- Instrukcje mówią o wysokiej dostępności tej usługi - nie masz dostępu do większej liczby maszyn, więc zrób co się da, żeby zwiększyć jej dostępność w obrębie istniejących zasobów **3pkt**
- Skonfiguruj serwer aby serwował załączony plik index.html **4pkt**
- Zapewnij dostępność usługi na internet. Nie masz czasu czekać do jutra aż administratorzy sieci udostępnią ci firmowy DNS, a potrzebujesz szybko przetestować dostępność, więc wymyśl jak zapewnić rozwiązywalny url wskazujący na IP hosta, na którym jest twój klaster "potyczki". **18pkt** (pełnym sukcesem operacji jest podanie adresu typu ogloszenia-krytyczne.xxxx.xxx rozwiązywalnego przy pomocy publicznego DNS z internetu, pod którym zgłosi się działająca strona internetowa);
#### Rozwiązanie
**Krok 1: Utworzenie Projektu/Namespace**

*   **Cel:** Logiczna izolacja zasobów dla serwera w dedykowanej przestrzeni nazw.
*   **Komenda:**
    ```bash
    kubectl create namespace ogloszenia-krytyczne
    ```
*   **Weryfikacja:** `kubectl get namespaces`

**Krok 2: Przygotowanie Treści HTML i utworzenie ConfigMap**

*   **Cel:** Udostępnienie pliku `index.html` w klastrze w formie ConfigMap, aby mógł być podmontowany do serwera webowego.
*   **Komenda:** (Zakładając, że plik `index.html` znajduje się w bieżącym katalogu)
    ```bash
    kubectl create configmap ogloszenia-html --namespace=ogloszenia-krytyczne --from-file=index.html
    ```
*   **Weryfikacja:** `kubectl get configmap ogloszenia-html --namespace=ogloszenia-krytyczne`

**Krok 3: Wdrożenie Serwera Webowego (Nginx)**

*   **Cel:** Uruchomienie serwera Nginx w najnowszej wersji, serwującego plik z ConfigMap, z konfiguracją zapewniającą podstawową wysoką dostępność (3 repliki, rozłożenie na węzłach, sondy zdrowia).
*   **Komenda:** (Zastosowanie manifestu `deployment.yaml` podanego wcześniej)
    ```bash
    kubectl apply -f deployment.yaml
    ```
*   **Weryfikacja:** `kubectl get deployment ogloszenia-nginx --namespace=ogloszenia-krytyczne` oraz `kubectl get pods --namespace=ogloszenia-krytyczne -l app=ogloszenia-nginx-app` (poczekaj aż pody będą Running i Ready).

**Krok 4: Utworzenie Usługi Dostępnej w Klastrze (ClusterIP)**

*   **Cel:** Udostępnienie serwera Nginx pod stabilnym adresem IP wewnątrz klastra (dostęp dla innych usług w klastrze, jeśli zajdzie taka potrzeba).
*   **Komenda:** (Zastosowanie manifestu `service-internal.yaml` podanego wcześniej)
    ```bash
    kubectl apply -f service-internal.yaml
    ```
*   **Weryfikacja:** `kubectl get service ogloszenia-local --namespace=ogloszenia-krytyczne`

**Krok 5: Udostępnienie Usługi w Internecie (LoadBalancer)**

*   **Cel:** Uzyskanie publicznego adresu IP dla usługi, umożliwiającego dostęp z zewnątrz.
*   **Komenda:** (Zastosowanie manifestu `service-external.yaml` podanego wcześniej)
    ```bash
    kubectl apply -f service-external.yaml
    ```
*   **Weryfikacja:** `kubectl get service ogloszenia-public --namespace=ogloszenia-krytyczne -o wide` (Najważniejsza jest kolumna `EXTERNAL-IP` - poczekaj, aż pojawi się adres IP, a nie będzie `<pending>`).

**Krok 6: Uzyskanie Publicznie Rozwiązywalnego URL**

*   **Cel:** Skonstruowanie adresu URL, który będzie rozwiązywalny z Internetu bez konieczności konfiguracji w firmowym DNS, wykorzystując publiczny IP z Krok 5 i serwisy typu `nip.io` lub `sslip.io`.
*   **Komendy:** (Wykonaj po upewnieniu się, że w Kroku 5 pojawił się EXTERNAL-IP)
    ```bash
    EXTERNAL_IP=$(kubectl get service ogloszenia-krytyczne-public --namespace=ogloszenia-krytyczne -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
    PUBLIC_URL="ogloszenia-krytyczne.${EXTERNAL_IP}.nip.io" # lub ogloszenia-krytyczne.${EXTERNAL_IP}.sslip.io
    echo "Publiczny URL do ogłoszeń: http://${PUBLIC_URL}/"
    ```
*   **Weryfikacja:** Otwórz wygenerowany URL w przeglądarce z Internetu. Powinieneś zobaczyć zawartość pliku `index.html`.

### Misja 2 - kryptonim "Long Horn"
Potrzebujemy Persistent Storage, szybko! Tylko musi być taki, żeby umożliwiał replikację! Wprawdzie i tak mamy tylko jeden serwer w klastrze, więc musimy ograniczyć liczbę replik do 1, ale sama obsługa replikacji jest ważna dla morale dowództwa - nieważne, że działa tylko na papierze... Nfs provisioner jest wykluczony, potrzebne jest coś lepszego. Gdyby tylko w Rancherze istniało jakieś repozytorium z łatwym w instalacji i obsłudze rozwiązaniem storage dla Kubernetes...

Zainstaluj rozwiązanie typu software-defined storage w najnowszej stabilnej wersji na klastrze "potyczki", ustawiając w konfiguracji instalacyjnej 1 replikę i domyślny StorageClass. **5pkt**
Sukces misji oznacza działającą aplikację storage oraz dostępną StorageClass.
#### Rozwiązanie
1.  **Ustaw kontekst `kubectl` na klaster docelowy** (np. "potyczki", jeśli taki jest jego nazwa w Twojej konfiguracji kubeconfig):
    ```bash
    kubectl config use-context potyczki
    ```

2.  **Dodaj oficjalne repozytorium Helm dla Longhorna:**
    ```bash
    helm repo add longhorn https://charts.longhorn.io
    ```

3.  **Zaktualizuj lokalną listę repozytoriów Helm:**
    ```bash
    helm repo update
    ```

4.  **Zainstaluj Longhorna przy użyciu Helm.** Installacja odbędzie się w dedykowanej przestrzeni nazw `longhorn-system`. Kluczowym elementem jest opcja `--set defaultSettings.replicaCount=1`, która nadpisuje domyślną liczbę replik (zazwyczaj 3) na wymaganą 1 replikę. Flagę `--create-namespace` dodajemy, aby Helm automatycznie utworzył przestrzeń nazw, jeśli jeszcze nie istnieje.
    ```bash
    helm install longhorn longhorn/longhorn \
      --namespace longhorn-system \
      --create-namespace \
      --set defaultSettings.replicaCount=1
    ```
    *Wyjaśnienie parametrów:*
    *   `helm install longhorn longhorn/longhorn`: Instaluje chart `longhorn/longhorn` pod nazwą `longhorn`.
    *   `--namespace longhorn-system`: Określa przestrzeń nazw do instalacji.
    *   `--create-namespace`: Tworzy przestrzeń nazw `longhorn-system`, jeśli jej nie ma.
    *   `--set defaultSettings.replicaCount=1`: Ustawia domyślną liczbę replik dla nowych wolumenów na 1.

5.  **Ustaw StorageClass `longhorn` jako domyślną (jeśli nie została ustawiona automatycznie).** Po instalacji Longhorn tworzy StorageClass o nazwie `longhorn`. Aby PV utworzone bez jawnego określenia StorageClass były obsługiwane przez Longhorn, należy je oznaczyć jako domyślne.
    *   Najpierw sprawdź, czy StorageClass `longhorn` jest już domyślna (szukaj adnotacji `[default]` w kolumnie `RECLAIMPOLICY` lub `PROVISIONER`).
        ```bash
        kubectl get storageclass
        ```
    *   Jeśli nie jest domyślna, użyj poniższej komendy. Pamiętaj, aby upewnić się, że żadna inna StorageClass nie jest obecnie oznaczona jako domyślna w klastrze (lub usuń domyślność innej przed dodaniem do Longhorna).
        ```bash
        kubectl patch storageclass longhorn \
          -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
        ```

### Misja 3 - operacja "Koci Pazur"
Kot prezesa się nudzi, należy mu zapewnić jakąś rozrywkę.
Dodaj nowe repozytorium do katalogu aplikacji Ranchera. URL repo: https://rancher.github.io/rodeo **5pkt**
Zainstaluj dowolną grę z nowo dodanego repo. **5 pkt**
#### Rozwiązanie
1. Wchodzimy na WebUI ranchera
2. Przechodzimy do zakładki `Apps > Repositories`
3. Klikamy `Create`
4. Wpisujemy nazwe oraz podajemy URL
5. Po zakończeniu przechodzimy do `Apps > Charts`
6. Filtrujemy tylko repozytorium o naszej nazwie
7. Wybieramy gre i ją pobieramy
8. Następnie otwieramy nasz URL gry w przeglądarce. Nasza gra jest [tutaj](http://tetris.default.193.187.67.116.sslip.io/)

### Misja 4 - kryptonim "Armor Plate"
Musimy wzmocnić nasze zabezpieczenia dedykowanymi rozwiązaniami security! Same firewalle to za mało w erze Kubernetes. Potrzebujemy najwyższej klasy ochrony - takiej jaka jest stosowana przez amerykańskie trzyliterowe służby.

Z katalogu aplikacji zainstaluj NeuVector w najnowszej stabilnej wersji. **5pkt**

Włącz funkcję Auto-scan **1 pkt**

Zbadaj czy istnieją podatności dla wersji Kubernetes uruchomionej na klastrze potyczki - podaj ich liczbę i jeśli nie jest równa zero, podnieś wersję Kubernetes klastra potyczki do 1.26. **5pkt**

Zbadaj czy istnieją podatności dla węzła (node) klastra potyczki - podaj ich liczbę i jeśli nie jest równa zero, dokonaj patchowania i aktualizacji systemu. **6pkt**

Ciekawe czy wrogie systemy mają podobne podatności, może dałoby się to wykorzystać?...
#### Rozwiązanie
##### Krok 1: Instalacja NeuVector (5pkt)

Zainstaluj NeuVector w najnowszej stabilnej wersji przy użyciu Helm.

1.  **Dodaj repozytorium Helm NeuVector:**
    ```bash
    helm repo add neuvector helm repo add neuvector https://neuvector.github.io/neuvector-helm/
    helm repo update
    ```

2.  **Zainstaluj NeuVector:**
    Zaleca się instalację w dedykowanej przestrzeni nazw `neuvector`.
    ```bash
    helm install neuvector neuvector/core --namespace neuvector --create-namespace
    ```

3.  **Zweryfikuj instalację:**
    Sprawdź, czy wszystkie pody NeuVector uruchomiły się poprawnie. Może to potrwać kilka minut.
    ```bash
    kubectl get pods -n neuvector
    ```
    Upewnij się, że wszystkie pody są w stanie `Running`.

##### Krok 2: Włączenie Funkcji Auto-scan (1pkt)

Włącz automatyczne skanowanie w konfiguracji NeuVector.

1.  **Zastosuj aktualizację konfiguracji przez Helm:**
    Użyj polecenia `helm upgrade`, aby zmodyfikować istniejącą instalację, włączając flagę `global.autoScan`.
    ```bash
    helm upgrade neuvector neuvector/core \
      --namespace neuvector \
      -- Reuse values, just set autScan=true \
      --set global.autoScan=true
    ```

2.  **Zweryfikuj zmianę:**
    Sprawdź ConfigMap NeuVector, aby upewnić się, że opcja `autoScan` została ustawiona na `true`.
    ```bash
    kubectl get cm neuvector-global -n neuvector -o yaml | grep -i autoScan
    ```
    Oczekiwany wynik powinien zawierać linię podobną do `autoScan: "true"`.

##### Krok 3: Skanowanie Podatności Wersji Kubernetes i Upgrade (5pkt)

Zbadaj wersję Kubernetes działającą na klastrze pod kątem znanych podatności i dokonaj upgrade'u do wersji 1.26.x, jeśli istnieją podatności.

1.  **Sprawdź aktualną wersję Kubernetes:**
    ```bash
    kubectl version --short | grep Server
    ```

2.  **Zainstaluj Trivy:**
    NeuVector ma wbudowane funkcje skanowania, ale na potrzeby tego zadania użyjemy popularnego narzędzia CLI - Trivy (od Aqua Security, podobnie jak NeuVector). Zainstaluj Trivy na maszynie, z której operujesz:
    ```bash
    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin
    ```

3.  **Zeskanuj klaster pod kątem podatności związanych z konfiguracją klastra/wersją K8s:**
    ```bash
    trivy k8s cluster --vuln-type config --format json > cluster-scan.json
    ```
    _Uwaga: Pełne skanowanie klastra przez Trivy może potrwać kilka minut._

4.  **Policz znalezione podatności CVE w konfiguracji klastra:**
    Użyj `jq` do parsowania wyników.
    ```bash
    VULN_COUNT=$(jq '[.[] | select(.Type=="Misconfiguration")] | length' cluster-scan.json)
    echo "Znaleziono $VULN_COUNT podatności konfiguracji/wersji K8s."
    ```
    _Uwaga: Czasami Trivy klasyfikuje problemy specyficzne dla wersji/konfiguracji K8s jako `Misconfiguration`, a nie bezpośrednio `CVE` związane z binarkami. W rzeczywistości obie kategorie mogą być ważne._

5.  **Wykonaj upgrade klastra, jeśli znaleziono podatności:**
    **JEŚLI liczba znalezionych podatności jest większa od zera (`$VULN_COUNT -ne 0`) LUB jeśli aktualna wersja K8s jest znacząco niższa niż 1.26.x**, przeprowadź upgrade klastra do wersji 1.26.x (np. 1.26.13 jako ostatni patch, jeśli dostępny w repozytoriach pakietów).

    **A. Upgrade Control Plane (wykonaj TYLKO na węźle Control Plane/Master):**
    *   Zaloguj się przez SSH do węzła Control Plane.
    *   Zaplanuj upgrade, aby zobaczyć sugerowane wersje:
        ```bash
        sudo kubeadm upgrade plan
        ```
    *   Zastosuj upgrade (wybierz najnowszy dostępny patch w serii 1.26.x, np. v1.26.13):
        ```bash
        sudo kubeadm upgrade apply v1.26.13 --allow-missing-images
        ```
        _Możesz dodać flagę `--allow-missing-images` jeśli masz problemy z pobieraniem obrazów._

6.  **Zweryfikuj zaktualizowaną wersję Kubernetes:**
    Ponownie sprawdź wersję z maszyny, z której operujesz `kubectl`.
    ```bash
    kubectl version --short | grep Server
    ```
    Powinna teraz wskazywać wersję 1.26.x.
**U nas musieliśmy zaktualizować kubernetsy do wersji: v1.26.15+rke2r1**

### Misja 5 - operacja Czyste Ręce
Wdrożenie AI byłoby niesłychanie użyteczne w naszych zadaniach, idealnie byłoby zacząć od narzędzia Ollama. Ale trzeba  się upewnić, że te obrazy nie zawierają podatności - najpierw musimy je przeskanować! 
Użyj NeuVector, żeby przeskanować repozytorium Ollama z rejestru https://registry.hub.docker.com ; jako rozwiązanie podaj nazwę image z największą ilością podatności, oraz liczbę tych podatności. **7pkt**

### Misja 6 - kryptonim Zero Zaufania
Nasz system wczesnego ostrzegania wykrył podejrzaną aktywność, którą przechwycilismy. Wdróż na klastrze zinfiltrowany zasób w odpowiednim namespace (podejrzany-agent.yaml). Natychmiast odetnij wszelką komunikację sieciową (przychodzącą i wychodzącą) z/do tego poda, żebyśmy mogli go szczegółowo przeanalizować (wymaga pomyślnego ukończenia Misji 4). **10pkt**

Przetestuj działanie zabezpieczeń dla połączeń przychodzących i wychodzących z podejrzanego poda. Załącz do odpowiedzi odpowiednie Security Violations z NeuVectora pokazujące zablokowaną próbę naruszenia blokady.**5pkt**

Utwórz pomocniczego poda nginx o nazwie detektor w namespace kwarantanna. Zmień reguły zabezpieczeń podejrzanego poda, tak, aby dopuszczały połączenie do poda detektor. Wygeneruj ruch sieciowy z podejrzanego-agenta do detektora przy pomocy curl. Użyj NeuVector aby dokonać przechwycenia pakietów z tej komunikacji. **10pkt**

Zablokuj narzędzie curl przy pomocy NeuVector. Potwierdź działanie blokady curl pomimo przepuszczenia ruchu sieciowego do poda detektor (załącz zrzut ekranu lub skopiowane w całości komunikaty shella wraz z poleceniem, które je wyzwoliło). **7pkt**
Wyeksportuj regułę jako CRD w trybie Protect i załącz do dokumentacji (**5 pkt**)

Dokonaj "analizy" przechwyconego pakietu (znajdź odpowiednie narzędzie) - do sukcesu misji wystarczy, że skopiujesz pola opisujące jeden z pakietów: źródłowe i docelowe IP, protokół, długość, info. **7pkt**

### Misja 7 - kryptonim Enigma Reactivation
Dzięki owocnej współpracy z wywiadami innych krajów NATO pomyślnie przechwyciliśmy zaszyfrowaną rosyjską transmisję. Niestety moduł deszyfrujący uległ awarii - po krótkim śledztwie okazało się, że tym razem to nie działanie wroga, ale zwyczajna niekompetecja - ktoś bardzo mądrze użył AI do wygenerowania konfiguracji i zastąpił wszystkie istniejące kopie. Przeanalizuj i napraw deszyfrator-pod-uszkodzony.yaml - bez niego przechwycona transmisja jest bezużyteczna!

Misja zakończona powodzeniem jesli pod deszyfrator przejdzie w stan Completed, a w jego logach pojawi się odszyfrowana wiadomość ("Zneutralizowac agenta KREML. Haslo: BURZA_MAJOWA") - załącz zrzut ekranu lub skopiowane w całości komunikaty shella wraz z poleceniem, które je wyzwoliło. **25pkt**

### Misja 8 - "For Your Eyes Only"
Nowo utworzony zespół szybkiego reagowania natychmiastowo potrzebuje tymczasowego dostępu read-only do logów aplikacji z naszego klastra. Zgodnie z zasadą zero-trust powninniśmy nadać im tylko niezbędne minimum uprawnień.

Utwórz nowego użytkownika rapid-response-agent i nadaj mu dostęp read-only do namespace ogloszenia-krytyczne. **3pkt**

Potrzebujemy dodatkowo kont serwisowych dla zautomatyzowanych narzędzi zespołu szybkiego reagowania. Utwórz specjalną rolę log-reader w namespace ogloszenia-krytyczne, która umożliwia wykonywanie tylko operacji *get pods* oraz *list pods* na podach, oraz - co kluczowe - daje dostęp do zasobu *logs* wewnątrz podów. Utwórz konto serwisowe automated-response-agent w namespace ogloszenia-krytyczne i przypisz mu rolę log-reader. **15pkt**

### Misja 9 - operacja Slinky Slingshot
Rosyjska machina nie ustaje w zalewaniu nas treściami propagandowymi używając wszelkich możliwych kanałów do siania dezinformacji, podgrzewania społecznej niezgody i szczucia na naszych sojuszników. Czas coś z tym zrobić! Masz za zadanie utworzenie Portalu Do Spraw Dez-DezInformacji, w skrócie PDSDDI (prawda, że chwytliwa nazwa?). Portal ma być oparty o wordpress. Utwórz namespace pdsddi i wdróż w nim wordpress.

Nasz portal musi posiadać persistent storage - użyj storage wdrożonego w Misji 2 - chyba, że zakończyła się ona niepowodzeniem, to użyj dowolnej dostępnej alternatywy. **7pkt**

Opublikuj na PDSDDI dwa dowolne fakty na temat Rosji - muszą być prawdziwe, np. "Putin to burak". **2pkt**

Wystaw Portal na świat pod adresem pdsddi.xxxx.xxx, analogicznie jak w Misji 1 **10 pkt**

Opisz podjęte środki w celu zabezpieczenia Portalu (w końcu to tylko wordpress i właśnie wystawiliście go na świat pełen wrogich agentów).**3pkt**

### Misja 10 - "Zaginiona Arka"
Kluczowy system archiwizacji został uszkodzony! Ale to nic dla naszego potężnego storage... Mam nadzieję, że Misja 2 się powiodła bo teraz potrzebujemy jej rezultatów!

Użyj archiwum-delta.yaml, aby utworzyć system archiwizacji. Po uruchomieniu poda, wejdź do jego konsoli i stwórz plik /dane/manifest-delta-v1.txt o treści "Protokół Delta aktywny.". **3pkt** Następnie, w interfejsie Longhorn, utwórz ręcznie backup wolumenu używanego przez tego poda. Teraz zasymuluj uszkodzenie danych poprzez usunięcie pliku manifest-delta-v1.txt z działającego poda. **5pkt**

Znajdź w interfejsie Longhorn ostatni dostępny backup dla wolumenu pvc-archiwum-delta. Następnie przywróć ten backup do *nowego* PersistentVolumeClaim o nazwie pvc-archiwum-delta-przywrocone (w tym samym namespace). Na koniec, przekonfiguruj archiwum-delta, aby używało tego przywróconego wolumenu. Oryginalny, "uszkodzony" PVC (pvc-archiwum-delta) powinien pozostać nietknięty ale odłączony od poda. **6pkt**

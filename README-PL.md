# Rozproszone uczenie maszynowe z Tensorflow i Kubernetes

Ten projekt zawiera Helm Chart, który oryginalnie był udostępniony na oficjalnej stronie platformy Helm. [link](https://hub.helm.sh/charts/stable/distributed-tensorflow)
W tym oddzielnym repozytorium zawarta jest dodatkowa instrukcja i pliki konfiguracyjne na potrzeby edukacyjne dla Politechniki Warszawskiej.

## Idea
Idea tego projektu składa się z 3 elementów.
Pierwszym z nich jest kontener z skryptem w języku Python implementujący rozproszoną architekturę biblioteki Tensorflow.
Architektura ta składa się z rozproszonych jednostek jednego z dwóch typów - *ps* oraz *worker*.
Jednostka *ps* jest skrótem od *Parameter Server* i jej zadaniem jest ustrzymywanie wartości sieci neuronowej, które są uaktualniane w czasie uczenia oraz dzielenie modelu sieci neuronowej w taki sposób, żeby wiele jednostek równolegle mogło go analizować i uczyć nowymi danymi.
Jednostka *worker* jest właściwą jednostką uczącą i przeprowadza proces uczenia na podstawie danych otrzymanych z jednostek *ps*.
Drugim elementem projektu jest wdrożenie rozproszonej architektury uczenia na platformę Kubernetes.
Platforma Kubernetes jest coraz popularniejsza w komercyjnym użyciu i na pewno każdy z nas nieświadomie korzysta z niej codziennie - korzystając z infrastruktury banku (np. PayPal, ING), czy nawet grając w gry online (np. League of Legends, Fortnite).
Jej zadaniem jest utrzymanie wielu maszyn połączonych ze sobą w *cluster* jako wysoko dostępnej platformy do wdrażania aplikacji zbudowanych w kontenerach.
W przypadku tego projektu wdrażane są dwa zestawy kontenerów - kontenerów pracujących jako jednostki *ps* oraz tych pracujących jako jednostki *worker*.
Trzecim elementem jest opakowanie manifestów (plików z konfiguracją wdrożenia) na platformę Kubernetes w taki sposób, żeby uruchomienie całego projektu było proste nawet dla niewprawnego operatora.
W tym celu wykorzystana jest platforma Helm, która generuje dynamicznie manifesty platformy Kubernetes korzystając z konfiguracji podanej w formie pliku *values.yml*.

## Instalacja krok po kroku

### Wymagania przed rozpoczęciem
Przed przystąpieniem do instalacji potrzebne jest kilka narzędzi linii poleceń:

- [ ] **Docker** - Jest to narzędzie do konteneryzacji. Jedno z najbardziej popularnych dla komputerów osobistych (do komercyjnych zastosowań popularniejsze jest containerd lub CRI-O, ale ich obsługa jest trochę bardziej skomplikowana). Instalując Docker na systemach MacOS oraz Windows należy się liczyś z tym, że w rzeczywistości instalujemy maszynę wirtualną z Linux'em oraz Docker'em. Do tego projektu zdecydowanie rekomendowane jest korzystanie z systemu Linux. [Instrukcja instalacji dla różnych systemów](https://docs.docker.com/get-docker/)
- [ ] **Kind** (Kubernetes in Docker) - Jest to przyjazne narzędzie, które na pojedynczej maszynie tworzy kilka kontenerów, a następnie na nich instaluje platformę Kubernetes. Więc w ten sposób posiadając jeden komputer możemy symulować wiele rozproszonych maszyn. [Instrukcja instalacji dla różnych systemów](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [ ] **kubectl** - Jest to narzędzie linii poleceń pozwalające na interakcję z *clusterem* platformy Kubernetes. [Instrukcja instalacji dla różnych systemów](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [ ] **Helm** - Do instalacji finalnej paczki z projektem potrzebne jest narzędzie linii poleceń Helm. [Instrukcja instalacji dla różnych systemów](https://helm.sh/docs/intro/install/)

### Przygotowanie środowiska

W pierwszym kroku należy przygotować środowisko. W tym przypadku będzie do platforma Kubernetes uruchomiona na symulowanych maszynach za pomocą kontenerów utworzonych przez narzędzie Kind.

W repozytorium projektu znajduje się plik *kind-config.yml*, któ©y zawiera konfigurację *clustra*.
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```
W powyższej konfiguracji zdefiniowane jest, że *cluster* ma posiadać jedną maszynę pracującą jako master (w platformie Kubernetes jest to maszyna obsługująca kontrolę nad całym *clustrem*. Możliwe jest utworzenie kilku takich maszyn pracujących równolegle, ale na potrzeby tego projektu wykorzystamy tylko jedną) oraz 3 pracujące jako zwykły worker.

Aby utworzyć *cluster* wystarczy polecenie `kind create cluster --config kind-config.yml`.

Jeśli wszystko zainstalowało się poprawnie, to polecenie powinno zwrócić następujące informacje:
```
$ kind create cluster --config kind-config.yml
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.18.2) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

### Konfiguracja parametrów projektu

Konfiguracja parametrów projektu jest realizowana za pomocą pliku *values.yml*, który następnie jest interpretowany przez platformę Helm.

Przykładowy plik wygląd następująco:
```
# Default values for distributed-tensorflow.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
worker:
  number: 2 # Liczba jednostek typu worker, które mają pojawić się w clustrze
  podManagementPolicy: Parallel # Jest to specyficzne pole dla zasobu StatefulSet, które definiuje róznoległe uruchomienie kontenerów na clustrze (domyślnie kontenery są uruchamiane jeden po drugim)
  image: # Ta sekcja opisuje parametry obrazu konteneru do uruchomienia
    repository: dysproz/distributed-tf # Repozytorium oraz nazwa obrazu do uruchomienia
    tag: 1.7.0 # Tag wersji kontenera
    pullPolicy: IfNotPresent # Polityka definiująca, żeby pobrać obraz z internetu tylko wtedy jeśli nie znajduje się już taki obraz lokalnie
  port: 9000 # Numer portu na którym ma działać aplikacja (jego zmiana wymaga również zmiany parametrów w skrypcie w kontenerze)
  limit: # Ta sekcja definiuje limity na zasoby wykorzystane przez pojedynczy kontener
    memory: 125Mi
    cpu: 200m
ps: # Konfiguracja tej sekcji wygląda tak samo, jak dla jednostki worker, ale odnosi się do jednostki ps. Dlatego opisy szczegółowe zostaną tutaj pominięte
  number: 2
  podManagementPolicy: Parallel
  image:
    repository: dysproz/distributed-tf
    tag: 1.7.0
    pullPolicy: IfNotPresent
  port: 8000
  limit:
    memory: 125Mi
    cpu: 200m
# optimize for training
hyperparams: # Ta sekcja definiuje parametry samego procesu uczenia
  batchsize: 20 # Rozmiar pojedynczego "batcha" w procesie uczenia
  learningrate: 0.001 # Poziom ograniczenia poziomu uczenia - często stosowany, żeby zapobiedz dużym wahaniom wartości w sieci neuronej
  trainsteps: 200 # Liczba kroków uczenia. Ustawienie tej wartości na 0 powoduje uczenie w nieskończoność
volumes: # Ta sekcja definiuje dostępne dyski dla kontenerów. Zapis wg standardów platformy Kubernetes.
  - name: logs-volume
    hostPath:
      path: /tmp/mnist # Ten dysk definiowany jest jako ścieżka fizyczna na maszynie uruchamiającej kontener.
volumeMounts: # Ta sekcja opisuje sposób zamontowania dysków do kontenerów
  - name: logs-volume
    mountPath: /tmp/mnist-log # Ta definicja podłącza wcześniej zdefiniowany dysk pod ścieżką /tmp/mnist-log
```

Szczegóły implementacyjne skryptu znajdującego się w kontenerze, którego kod znajduje się [tutaj](https://github.com/Dysproz/distributed-tensorflow).

### Instalacja projektu za pomocą platformy Helm

Posiadajac gotowy projekt można go zainstalować za pomocą komendy `helm install mnist distributed-tensorflow-chart/ --values values.yml`.

Poprawnie zainstalowany projekt powinien zwrócić następujące informacje:
```
$ helm install mnist distributed-tensorflow-chart/ --values values.yaml
NAME: mnist
LAST DEPLOYED: Sun Sep 13 11:56:55 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application Status by running these commands:

  echo "Get the statefulset of distributed tensorflow"
  kubectl get sts --namespace default -l "app=distributed-tensorflow,release=mnist"
```

### Obserwacja działającego projektu

Projekt już działa i mona go obserwować.

Pierwszym elementem jaki można sprawdzić jest stand zasobu StatefulSet przez komendę `kubectl get sts`.
```
$ kubectl get sts
NAME           READY   AGE
mnist-ps       0/2     2m28s
mnist-worker   0/2     2m28s
```

Widać tutaj, że zostały utworzone dwa zestawy Pod'ów - czyli najmniejszego zasoby platformy Kubernetes zawierającego jeden lub więcej kontenerów.
Widać również, że w każdym z nich gotowych jest 0/2 Pod'ów.

Może to być zastanawiająca informacja, ponieważ ani jeden kontener na razie nie działa.
Aby sprawdzić co się dzieje można wywołać komendę `kubectl get pod`.
```
$ kubectl get po
NAME             READY   STATUS              RESTARTS   AGE
mnist-ps-0       0/1     ContainerCreating   0          2m15s
mnist-ps-1       0/1     ContainerCreating   0          2m15s
mnist-worker-0   0/1     ContainerCreating   0          2m15s
mnist-worker-1   0/1     ContainerCreating   0          2m15s
```
Tutaj pokazane są wszystkie Pod'y uruchomione w *clustrze*.
Jak widać każdy z nich ma status *ContainerCreating*. Oznacza to, że na szczęście nie zdarzyła się awaria, a po prostu kontenery są tworzone (prawdopodobnie pobierany jest obraz).

Czekając kilka minut możemy sprawdzić ponownie statusy:
```
$ kubectl get po
NAME             READY   STATUS    RESTARTS   AGE
mnist-ps-0       1/1     Running   0          7m3s
mnist-ps-1       1/1     Running   0          7m3s
mnist-worker-0   1/1     Running   0          7m3s
mnist-worker-1   1/1     Running   0          7m3s
```
Jak widać wszystkie są w stanie *Running*.
Jeszcze dla potwierdzenia sprawdzenie statusu zasobów StatefulSet:
```
$ kubectl get sts
NAME           READY   AGE
mnist-ps       2/2     7m50s
mnist-worker   2/2     7m50s
```
Widać wyraźnie, że oba StatefulSet'y mają gotowy 2/2 Pod'y.
Oznacza to, że nasz projekt działa.

Jeśli chcemy sprawdzić aktualne postępy procesu uczenia możemy wywołać komedę bezpośrednio z jednym z kontenerów jednostki *ps*.
Komenda `kubectl exec -it mnist-ps-1 -- tail -f /tmp/mnist-log/deploy.log` będzie na bieżąco pokazywać logi z procesu uczenia.
```
$ kubectl exec -it mnist-ps-1 -- tail -f /tmp/mnist-log/deploy.log
2020-09-13 10:07:34,373 - __main__ - INFO - 1599991654.373432: Worker 0: training step 132 done (global step: 198)
INFO:__main__:1599991654.373432: Worker 0: training step 132 done (global step: 198)
2020-09-13 10:07:34,434 - __main__ - INFO - 1599991654.434183: Worker 0: training step 133 done (global step: 199)
INFO:__main__:1599991654.434183: Worker 0: training step 133 done (global step: 199)
2020-09-13 10:07:34,495 - __main__ - INFO - 1599991654.495170: Worker 0: training step 134 done (global step: 200)
INFO:__main__:1599991654.495170: Worker 0: training step 134 done (global step: 200)
2020-09-13 10:07:34,495 - __main__ - INFO - Training ends @ 1599991654.495587
INFO:__main__:Training ends @ 1599991654.495587
2020-09-13 10:07:34,495 - __main__ - INFO - Training elapsed time: 58.030548 s
INFO:__main__:Training elapsed time: 58.030548 s
```
Powyższe logi pokazują koniec procesu uczenia.
Widać w nich kilka ostatnich kroków uczenia i finalna wiadomość na temat skończonego procesu uczenia.
Warto zauważyć, że istnieją dla numery kroków uczenia - lokalny i globalny.
Lokalny krok uczenia odnosi się bezpośrednio do pracy kontenera, w którym obserwujemy logi, a globalny jest krokiem dla całego modelu uczenia zbiorczo.
Końcowa informacja wskazuje czas całkowity procesu uczenia w sekundach.

Warto zauważyć, że obecnie po zakończeniu uczenia następuje restart i ponowne uruchomienie programu - jest to spowodowane implementacją za pomocą StatefulSet, który składa się z Pod'ów, które oczekują ciągle działającego procesu, a zakończenie procesu jest równoznaczne z restartem. Lepszą implementacją mogłoby być wykorzystanie zasobu Job, który koniec procesu z kodem wyjścia 0 uznaje jako sukces i nie próbuje restartować kontenerów. Jednak aktualnie zasób StatefulSet jest wykorzystany ponieważ oferuje szczególne cechy pozwalające na realizację połączeń sieciowych pomiędzy kontenerami.

Aby zapobiec 'uciekaniu' logów zastosowano dysk zewnętrzny przyłączany do kontenera. Nawet po restarcie kontenera logi nadal są dostępne na maszynie (w tym przypadku kontenerze narzędzia Kind).

Aby wyświetlić logi należy podłączyć się do kontenera.
Aby to zrobić można sprawdzić na jakich węzłach umieszczone są kontenery
```
$ kubectl get po -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
mnist-ps-0       1/1     Running   0          21m   10.244.3.2   kind-worker3   <none>           <none>
mnist-ps-1       1/1     Running   0          21m   10.244.2.3   kind-worker    <none>           <none>
mnist-worker-0   1/1     Running   5          21m   10.244.2.2   kind-worker    <none>           <none>
mnist-worker-1   1/1     Running   5          21m   10.244.1.2   kind-worker2   <none>           <none>
```
Z powyższego przykładu widać, że kontener *mnist-ps-1* znajduje się na węźle *kind-worker*.

Znając nazwę kontenera wystarczy komenda `docker exec -it kind-worker sh`, aby znaleźć się w linii poleceń kontenera.

Następnie w pliku */tmp/mnist/deploy.log* znajdują się wszystkie logi.

### Odinstalowanie

Aby odinstallować projekt z *clustra* wystarczy polecenie `helm uninstall mnist`.

Następnie jeśli nie planujemy dalszych prac na *clustrze* wystarczy polecenie `kind delete cluster` aby usunąć kontenery i całą platformę Kubernetes.
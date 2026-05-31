# Laboratorium: Cloud Native Observability – Monitorowanie, Logi i Alerty w Kubernetes

## Wprowadzenie
Witajcie na zajęciach z nowoczesnego monitorowania systemów chmurowych (ang. *Observability*, czyli obserwowalność). W dzisiejszych czasach, gdy aplikacje składają się z dziesiątek małych elementów (mikrousług), które ciągle się tworzą i znikają, zwykłe sprawdzanie "czy serwer działa" to za mało. Musimy wiedzieć, **dlaczego** system zachowuje się tak, a nie inaczej.

Na tych zajęciach zbudujecie od zera pełne środowisko w chmurze Microsoft Azure, uruchomicie aplikację, celowo ją przeciążycie, a następnie zdiagnozujecie problem i ustawicie automatyczne powiadomienia (alerty) na Wasz komunikator.

Wykorzystamy do tego najpopularniejsze na rynku narzędzia:
* **Prometheus** – do zbierania danych liczbowych (metryk, np. zużycie procesora).
* **Loki** – do zbierania i przeszukiwania tekstu (logów z aplikacji).
* **Grafana** – do rysowania pięknych wykresów i wysyłania powiadomień o awariach.

---

## Część 0: Przygotowanie środowiska 

Aby rozpocząć pracę, musicie pobrać kod z mojego repozytorium i uruchomić infrastrukturę w chmurze. 

### Zadanie 0.1: Klonowanie repozytorium i logowanie
1. Klonowanie repozytorium, np. za pomocą GitHub Desktop - tak jak na ostatnich zajęciach. 

### Zadanie 0.2: Uruchomienie infrastruktury (CI/CD)
1. Przejdźcie na stronę repozytorium GitHub w przeglądarce.
2. Użyjcie kodu z ostatnich zajęć! Skonfigurujcie dostęp do chmury Azure. W ustawieniach repozytorium (**Settings -> Secrets and variables -> Actions**) dodajcie trzy sekrety (wartości poda prowadzący):
   * `AZURE_CLIENT_ID`
   * `TENANT_ID`
   * `SUBSCRIPTION_ID`
3. Przejdźcie do zakładki **Actions** na GitHubie, wybierzcie potok wdrażający infrastrukturę i wstępny monitoring (np. *Deploy AKS*) i uruchomcie go przyciskiem **Run workflow**. Proces ten potrwa kilka minut. Ten krok automatycznie zainstaluje Prometheusa oraz Grafanę na Waszym klastrze.

### Zadanie 0.3: Podłączenie terminala do Klastra
Gdy akcja na GitHubie zaświeci się na zielono, wróćcie do terminala Azure Cloud Shell i połączcie się z nowym klastrem:
```bash
az aks get-credentials --resource-group [TUTAJ_NAZWA_GRUPY_ZASOBOW] --name [TUTAJ_NAZWA_KLASTRA]
```
Sprawdźcie, czy macie połączenie, wpisując: `kubectl get nodes`.

---

## Część 1: Naprawa aplikacji testowej

Nasza aplikacja, która ma udawać duże obciążenie matematyczne, ma błędy w początkowej konfiguracji. Musimy podmienić jej kod na prawidłowy i naprawić porty sieciowe, żeby generator ruchu mógł w nią "trafić".

Proszę wykonać w terminalu poniższe komendy:

```bash
# 1. Zmieniamy obraz aplikacji na gotowy, działający program testowy od twórców Kubernetesa
kubectl set image deployment/cpu-burner-app python-app=registry.k8s.io/hpa-example

# 2. Zmieniamy konfigurację sieci, żeby ruch trafiał na prawidłowy port (80)
kubectl patch svc burner-service -p '{"spec": {"ports": [{"port": 80, "targetPort": 80}]}}'

# 3. Sprawdzamy, czy aplikacja działa (szukamy statusu "Running")
kubectl get pods
```

---

## Część 2: Stress Test i wykresy (Prometheus)

Teraz sprawdzimy, jak Kubernetes radzi sobie z nagłym zainteresowaniem naszą aplikacją (tzw. automatyczne skalowanie - HPA).

### Zadanie 2.1: Atak na aplikację
Włączymy teraz skrypt (generator ruchu), który zacznie zasypywać naszą aplikację tysiącami zapytań.

1. Wpiszcie w terminalu komendę, która będzie na żywo pokazywać obciążenie procesora (aby z niej wyjść, wciśnijcie później `Ctrl+C`):
   ```bash
   kubectl get hpa -w
   ```
2. Otwórzcie nową kartę w przeglądarce i zalogujcie się do Waszej **Grafany** (adres IP usługi znajdziecie wpisując `kubectl get svc -n monitoring`).
3. W Grafanie otwórzcie panel (dashboard) o nazwie: **Kubernetes / Compute Resources / Namespace (Pods)**.
4. Wróćcie do terminala, otwórzcie drugą kartę (lub wyjdźcie z podglądu HPA przez `Ctrl+C`) i odpalcie atak:
   ```bash
   kubectl scale deployment load-generator --replicas=1
   ```

### Zadanie 2.2: Wygłodzenie maszyny (Node Starvation)
Obserwujcie terminal i Grafanę. Obciążenie procesora (CPU) wystrzeli w kosmos (ponad 100%). Kubernetes zacznie ratować sytuację, tworząc nowe kopie aplikacji (zobaczycie, że liczba Podów wzrośnie np. z 1 do 5).

> **Ważne spostrzeżenie:** Możecie zauważyć, że Grafana nagle przestała rysować wykresy, zacięła się lub pokazuje błędy braku danych. To nie jest błąd w Waszej konfiguracji! To zjawisko nazywa się **Node Starvation** (wygłodzenie węzła). Wasz atak zużył 100% procesora całej maszyny wirtualnej, przez co programy monitorujące nie mają siły wysyłać danych do Grafany.

5. Wyłączmy atak, żeby system mógł "odetchnąć":
   ```bash
   kubectl scale deployment load-generator --replicas=0
   ```
Po kilku minutach Kubernetes zauważy, że jest bezpiecznie i usunie nadmiarowe kopie aplikacji.

---

## Część 3: Zbieranie logów tekstowych (Loki)

Kiedy aplikacje same się tworzą i kasują (jak przed chwilą), zwykłe czytanie ich logów w terminalu nie ma sensu – bo gdy aplikacja znika, jej logi znikają razem z nią. Rozwiążemy to, instalując system **Loki**, który zasysa teksty na bieżąco do bezpiecznej bazy danych.

### Zadanie 3.1: Instalacja bazy Loki
Zainstalujemy gotową paczkę oprogramowania przy użyciu narzędzia Helm. Wpiszcie w terminalu:

```bash
# 1. Dodajemy listę programów od firmy Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 2. Instalujemy bazę logów w folderze (namespace) o nazwie 'monitoring'
helm upgrade --install loki grafana/loki-stack --namespace monitoring
```

### Zadanie 3.2: Podłączenie Lokiego do Grafany
1. Przejdźcie do okna przeglądarki z Grafaną. W lewym menu wybierzcie **Connections -> Data sources**.
2. Kliknijcie niebieski przycisk **Add data source** i wybierzcie **Loki**.
3. W polu **URL** wpiszcie wewnętrzny adres bazy w klastrze:
   `http://loki:3100`
4. Zjedźcie na dół i kliknijcie **Save & test**. *(Jeśli przycisk zwróci błąd, zignorujcie to – w nowych wersjach czasem się tak zdarza, jeśli baza jest jeszcze pusta. Adres jest na pewno poprawny).*

### Zadanie 3.3: Wyszukiwanie logów i tworzenie z nich wykresów
Kliknijcie w Grafanie ikonę kompasu po lewej stronie (**Explore**). Upewnijcie się, że na samej górze wybrane jest źródło danych: **Loki**.
Przetestujcie te trzy komendy (tzw. język LogQL):

* **Wyświetlanie logów aplikacji:** Znajdźmy wszystko, co system zapisuje w domyślnym miejscu roboczym.
  ```logql
  {namespace="default"}
  ```
* **Szukanie błędów:** Wyszukajmy w systemie monitoringu tylko takie linijki, które zawierają słowo "error" (niezwykle przydatne przy awariach!).
  ```logql
  {namespace="monitoring"} |= "error"
  ```
* **Zmiana tekstu w matematykę:** Możemy policzyć, ile logów na sekundę generuje system, i narysować z tego wykres! *(Uwaga: pamiętajcie, aby wyłączyć przycisk "Live" w prawym górnym rogu i ustawić czas na np. "Last 15 minutes", żeby system mógł to obliczyć).*
  ```logql
  rate({namespace="monitoring"}[1m])
  ```

---

## Część 4: Automatyczne Alerty na komunikator (Discord/Teams)

Patrzenie w wykresy przez 8 godzin dziennie to zły pomysł. Dobry inżynier ustawia system tak, by sam wysłał mu powiadomienie na komunikator, gdy dzieje się coś złego.

### Zadanie 4.1: Przygotowanie połączenia (Contact Point)
1. W Waszym komunikatorze (np. na wyznaczonym kanale Discord) utwórzcie i skopiujcie link integracji typu **Webhook**.
2. W Grafanie wybierzcie z lewego menu **Alerting -> Contact points**.
3. Kliknijcie **Add contact point**, nazwijcie go np. `KanalDevOps` i wybierzcie rodzaj (np. Discord).
4. Wklejcie swój link Webhook i kliknijcie **Save**.

### Zadanie 4.2: Kierowanie ruchem (Notification Policy)
Musimy powiedzieć Grafanie, że ważne błędy mają lecieć właśnie na ten kanał.
1. Wybierzcie **Alerting -> Notification policies**.
2. Kliknijcie **New specific policy**.
3. W sekcji *Matching labels* wpiszcie regułę: `severity = critical`.
4. W sekcji *Contact point* wybierzcie Wasz nowo stworzony `KanalDevOps` i zapiszcie.

### Zadanie 4.3: Tworzenie alarmu (Alert Rule)
Na koniec ustalamy, kiedy podnieść alarm.
1. Wybierzcie **Alerting -> Alert rules -> New alert rule**.
2. Nazwijcie alarm: `WysokieZuzycieCPU`.
3. W polu zapytania (PromQL) wklejcie ten kod (oznacza on: sprawdź, czy wykorzystanie procesora w domyślnych aplikacjach przekracza 0.5):
   ```logql
   sum(rate(container_cpu_usage_seconds_total{namespace="default"}[1m])) > 0.5
   ```
4. Wybierzcie dowolny folder i **obowiązkowo** dodajcie etykietę (Label): `severity = critical` (dzięki temu Grafana wyśle to na Wasz komunikator, co ustawiliśmy w Zadaniu 4.2).
5. W opcjach czasu (Evaluation behavior) ustawcie sprawdzanie co `10s`, a pole **Pending period** ustawcie na `0m` (aby alarm włączył się od razu, bez 10-minutowego czekania).
6. Kliknijcie **Save rule**.

---

## Zadanie końcowe: Sprzątanie po sobie

Aby uniknąć niepotrzebnych opłat za zasoby w chmurze Azure, po zakończeniu laboratorium należy bezwzględnie usunąć całe utworzone środowisko.

Wróćcie do terminala Azure Cloud Shell i wpiszcie poniższą komendę, podmieniając nazwę grupy zasobów na tę, z której korzystaliście w zadaniu 0.3:
```bash
az group delete --name [TUTAJ_NAZWA_GRUPY_ZASOBOW] --yes --no-wait
```

**Gratulacje!** Samodzielnie wdrożyliście i system Obserwowalności (Prometheus, Loki, Grafana), używany codziennie przez inżynierów DevOps na całym świecie.
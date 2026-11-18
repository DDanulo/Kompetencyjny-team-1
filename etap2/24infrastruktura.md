## 6 Infrastruktura wdrożeniowa

System zostanie wdrożony w **chmurze publicznej** z użyciem **usług zarządzanych** (model PaaS). Przedsiębiorstwo nie kupuje ani nie administruje własnych serwerów, a utrzymanie systemu zleca firmie zewnętrznej, tak jak dotąd robi to z innymi procesami biznesowymi.

*Rozmieszczenie elementów systemu*

Węzły kontekstu serwera z diagramu perspektywy wdrożenia (rozdz. 3) zostaną zrealizowane następującymi usługami chmurowymi. Wszystkie zasoby znajdują się w jednym regionie na terenie UE, ze względu na przetwarzanie danych osobowych Klientów (RODO).

| Element systemu                         | Usługa chmurowa                                                                                                              | Uwagi                                                                                                                                                       |
|-----------------------------------------|------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Pliki statyczne SPA                     | Hosting plików statycznych z CDN                                                                                             | Brak serwera do utrzymania, skalowanie bez ograniczeń.                                                                                                     |
| Serwer WWW (reverse proxy)              | Zarządzany load balancer z certyfikatem TLS domeny                                                                           | Terminacja HTTPS, HSTS, mTLS dla terminali, sonda `/health`.                                                                                               |
| Serwer aplikacyjny (`parking-api`)      | Usługa uruchamiania kontenerów (np. AWS ECS Fargate, Azure Container Apps, Google Cloud Run)                                 | Plik wykonywalny Go w obrazie kontenera. Co najmniej 2 instancje w różnych strefach dostępności. Zadania okresowe działają w tych instancjach z blokadą w bazie (rozdz. 3). |
| Serwer bazy danych                      | Zarządzany PostgreSQL (np. Amazon RDS, Azure Database for PostgreSQL, Cloud SQL)                                             | Automatyczne kopie zapasowe z odtwarzaniem do punktu w czasie, instancja zapasowa w drugiej strefie z automatycznym przełączaniem.                          |
| Konfiguracja i sekrety                  | Menedżer sekretów dostawcy                                                                                                   | `DATABASE_URL`, klucz podpisu JWT, `MAX_CLIENT_RESERVATIONS` przekazywane instancjom jako zmienne środowiskowe.                                             |
| Dzienniki zdarzeń i monitoring          | Usługa logowania i alertów dostawcy                                                                                          | Zbiera wpisy `slog` ze wszystkich instancji i powiadamia o awarii.                                                                                        |

Instancje API i baza danych działają w prywatnej sieci wirtualnej. Z Internetu dostępny jest tylko load balancer i CDN. Terminale parkingowe łączą się z load balancerem przez sieć komórkową, więc po stronie parkingów nie jest potrzebna żadna infrastruktura poza zasilaniem, zgodnie z deklaracją dostawcy terminali.

Całą infrastrukturę opisuje się kodem (np. Terraform), a obraz kontenera buduje i wdraża potok CI/CD. Środowisko można dzięki temu odtworzyć, przekazać innej firmie utrzymującej system albo przenieść do innego dostawcy chmury. Nie zmienia to kodu aplikacji, bo PostgreSQL i kontenery są dostępne u wszystkich dużych dostawców.

*Uzasadnienie wyboru*

* **Brak kompetencji IT u Zamawiającego.** Przy usługach zarządzanych dostawca chmury odpowiada za sprzęt, system operacyjny, aktualizacje bazy danych, kopie zapasowe i przełączanie awaryjne. Firma zewnętrzna wdraża tylko nowe wersje aplikacji i reaguje na alerty. Przedsiębiorstwo nie musi zatrudniać administratorów.
* **Ciągła dostępność.** Szlabany na wszystkich parkingach zależą od systemu centralnego przez całą dobę. Instancje w kilku strefach dostępności i zapasowa baza danych sprawiają, że awaria jednego serwera nie zatrzymuje wjazdów i wyjazdów.
* **Rozproszone parkingi i dostęp z sieci publicznej.** Terminale na wielu parkingach i Klienci z przeglądarek muszą łączyć się z jednym punktem w Internecie. Chmura daje go od razu, razem z certyfikatem i ochroną przed atakami sieciowymi.
* **Koszty rosną razem z firmą.** Opłata zależy od zużycia, a nie od zakupu sprzętu. Przy kilku parkingach wystarczą dwie małe instancje API i mała baza. Kolejne parkingi wymagają tylko zwiększenia liczby instancji (rozdz. 3), bez inwestycji w sprzęt.

*Rozważane warianty*

| Wariant                                          | Ocena                                                                                                                                                                                               |
|--------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Własna serwerownia (*on-premise*)                | Odrzucony. Wymaga zakupu sprzętu, łącza z redundancją, zasilania awaryjnego i dyżurów administratorów 24/7, których Przedsiębiorstwo nie ma.                                                         |
| Maszyny wirtualne w chmurze (IaaS)               | Możliwy, tańszy przy stałym obciążeniu. Odpowiada wprost diagramowi (Nginx, Linux, PostgreSQL na osobnych maszynach), ale aktualizacje systemu, kopie bazy i przełączanie awaryjne spadają na firmę utrzymującą system. |
| **Usługi zarządzane w chmurze (PaaS)**           | **Wybrany.** Najmniej pracy przy utrzymaniu i wysoka dostępność bez własnych administratorów.                                                                                                     |

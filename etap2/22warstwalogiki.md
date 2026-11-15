## 4 Warstwa logiki

Warstwa logiki biznesowej odpowiada za tworzenie, opłacanie i anulowanie rezerwacji z uwzględnieniem rozmiaru parkingu, 
przyjmowanie zgłoszeń wjazdu i wyjazdu od Terminali parkingowych, obliczanie bieżącej zajętości parkingu na podstawie 
zdarzeń `ParkingEvents` oraz zarządzanie parkingami przez **Administratora**. Wykonuje też zadania aktora **System**, 
które są wyzwalane czasowo. Swoje operacje udostępnia pozostałym częściom systemu jako zdalne API REST z danymi w formacie JSON.

*Platforma*

Warstwa logiki jest implementowana w języku Go w wersji 1.27 i opiera się głównie na rozwiązaniach dostarczanych przez 
bibliotekę standardową tego języka. System nie używa zewnętrznego szkieletu aplikacyjnego ani serwera aplikacyjnego.

| Element             | Wybór                          | Rola w systemie                                                             |
|---------------------|--------------------------------|-----------------------------------------------------------------------------|
| Język programowania | Go 1.27                        | Implementacja warstwy logiki, kompilowanej do jednego pliku wykonywalnego.  |
| Serwer HTTP         | `net/http`                     | Wbudowany serwer HTTP obsługujący żądania API.                              |
| Routing             | `http.ServeMux`                | Kierowanie żądań do handlerów według metody i ścieżki.                      |
| Serializacja        | `encoding/json`                | Zamiana struktur Go na JSON i z powrotem.                                   |
| Zadania okresowe    | `time.Ticker` + goroutine      | Realizacja przypadków użycia aktora **System**, które są wyzwalane czasowo. |
| Logowanie           | `log/slog`                     | Strukturalne logi działania aplikacji.                                      |
| Testy               | `testing`, `net/http/httptest` | Testy logiki i handlerów HTTP bez uruchamiania serwera.                     |
| Dostęp do danych    | `sqlx`                         | Wykonywanie zapytań SQL i mapowanie wyników na struktury Go.                |
| Uwierzytelnianie    | middleware `net/http`          | Weryfikacja tożsamości Klienta i Administratora.                            |

*Udostępnienie logiki jako zdalnego API*

Każdy przypadek użycia z rozdziału 2 odpowiada jednemu punktowi końcowemu REST API, z którego korzysta aplikacja
kliencka i terminale parkingowe. Przykładowo przypadek użycia *Utwórz rezerwację* jest dostępny jako
`POST /api/reservations`:

```http
POST /api/reservations
Content-Type: application/json

{
  "vehicleId": "8f1c2e4a-...",
  "parkingId": "3b7d9a10-...",
  "reservationStartDate": "2026-12-01T08:00:00+01:00",
  "reservationEndDate": "2026-12-01T16:00:00+01:00"
}
```

```go
mux.HandleFunc("POST /api/reservations", h.createReservation)

func (h *Handler) createReservation(w http.ResponseWriter, r *http.Request) {
	var req reservation.NewReservation
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "niepoprawne dane", http.StatusBadRequest)
		return
	}
	res, err := h.reservations.Create(r.Context(), auth.ClientID(r.Context()), req)
	if errors.Is(err, reservation.ErrNoFreeSpace) {
		http.Error(w, err.Error(), http.StatusConflict)
		return
	}
	if err != nil {
		http.Error(w, "błąd serwera", http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(res)
}
```

Handler odczytuje JSON, wywołuje metodę logiki `Create`, która sprawdza warunki przejścia T1, i zamienia jej wynik na
odpowiedź HTTP: `201 Created` z rezerwacją w stanie *Nieopłacona* albo kod błędu, np. `409 Conflict` przy braku wolnych
miejsc. Metoda `Create` nie zależy od HTTP, więc tę samą logikę mogą wywoływać zadania cykliczne i testy.

*Wybór architektury: monolit*

Warstwa logiki zostanie zaimplementowana jako **monolit modularny** czyli jeden proces, w którym kod podzielono na pakiety
według obszarów dziedziny (`parking`, `vehicle`, `reservation`, `terminal`). Uzasadnienie:

* **Silnie powiązany model danych.** Sprawdzenie prawa wjazdu (P1, P2) wymaga danych z `Reservation`, `ParkingEvents`
  i `Parking`, a zakończenie rezerwacji (T7) zmienia stan postoju pojazdu (P4). Po podziale na mikroserwisy takie
  operacje wymagałyby transakcji rozproszonych, a w monolicie wystarcza jedna transakcja bazodanowa.
* **Skala systemu.** System obsługuje jedno przedsiębiorstwo. Żadna jego część nie wymaga niezależnego skalowania ani
  osobnego cyklu wdrożeń.
* **Zespół.** Sześcioosobowy zespół bez doświadczenia w Go łatwiej utrzyma jedną bazę kodu i jedno wdrożenie niż kilka
  usług komunikujących się przez sieć.

Monolit nie wyklucza skalowania horyzontalnego, co opisuje rozdział 3.

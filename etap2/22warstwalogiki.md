## 4 Warstwa logiki

Warstwa logiki obsługuje rezerwacje **Klientów**, wjazdy i wyjazdy zgłaszane przez Terminale parkingowe oraz zarządzanie 
parkingami przez **Administratora**. Wykonuje też zadania aktora **System** uruchamiane o określonej porze. 
Pozostałe części systemu korzystają z niej przez API REST, które przesyła dane w formacie JSON.

*Platforma*

Warstwa logiki jest napisana w Go 1.27 z użyciem szkieletu Gin. Serwer HTTP jest częścią samego programu, dlatego
całą aplikację uruchamia się jako jeden plik wykonywalny, bez serwera aplikacyjnego. Z bazą danych aplikacja
komunikuje się przez zwykłe zapytania SQL, bez ORM.

| Element            | Wybór                          | Do czego służy                                              |
|--------------------|--------------------------------|-------------------------------------------------------------|
| Język              | Go 1.27                        | Implementacja warstwy logiki.                               |
| Szkielet aplikacji | Gin                            | Obsługa żądań HTTP i zamiana JSON na struktury Go.          |
| Walidacja danych   | `go-playground/validator`      | Sprawdzanie poprawności danych w żądaniach.                 |
| Identyfikatory     | `google/uuid`                  | Identyfikatory obiektów generowane jako UUID.               |
| Dostęp do bazy     | `sqlx` + sterownik `pgx`       | Zapytania SQL do bazy PostgreSQL.                           |
| Migracje bazy      | `golang-migrate`               | Zmiany schematu bazy zapisane w skryptach SQL.              |
| Uwierzytelnianie   | `golang-jwt/jwt`               | Sprawdzanie tokenów i tożsamości kont.                      |
| Zadania okresowe   | `time.Ticker`                  | Uruchamianie zadań cyklicznych aktora **System**.           |
| Logowanie          | `log/slog`                     | Zapisywanie wpisów dziennika zdarzeń z działania aplikacji. |
| Testy              | `testing`, `net/http/httptest` | Testy logiki i API bez uruchamiania serwera.                |

*Udostępnienie logiki jako zdalnego API*

Przypadki użycia Klienta, Administratora i Terminala są dostępne jako punkty końcowe API. Przykładowo *Utwórz
rezerwację* to żądanie `POST /api/reservations`.

```go
package reservation

import (
	"errors"
	"time"

	"github.com/google/uuid"
)

var (
	ErrParkingNotFound = errors.New("reservation: parking not found")
	ErrVehicleNotOwned = errors.New("reservation: vehicle not owned by client")
	ErrOutsideLimits   = errors.New("reservation: outside parking time limits")
	ErrNoFreeSpace     = errors.New("reservation: no free space")
	ErrClientLimit     = errors.New("reservation: client reservation limit reached")
)

type CreateRequest struct {
	VehicleID uuid.UUID `json:"vehicleId" binding:"required"`
	ParkingID uuid.UUID `json:"parkingId" binding:"required"`
	Start     time.Time `json:"reservationStartDate" binding:"required"`
	End       time.Time `json:"reservationEndDate" binding:"required,gtfield=Start"`
}
```
Listing 2.2.1 - Dane żądania utworzenia rezerwacji w pakiecie `reservation`

```go
package api

import (
	"errors"
	"log/slog"
	"net/http"

	"github.com/gin-gonic/gin"

	"parking/auth"
	"parking/reservation"
)

type handler struct {
	reservations *reservation.Service
}

func NewRouter(reservations *reservation.Service) *gin.Engine {
	h := &handler{reservations: reservations}
	r := gin.New()
	r.POST("/api/reservations", auth.RequireClient(), h.createReservation) // krok 1
	return r
}

func (h *handler) createReservation(c *gin.Context) {
	// krok 2
	var req reservation.CreateRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": "niepoprawne dane"})
		return
	}

	// krok 3
	created, err := h.reservations.Create(c.Request.Context(), auth.ClientID(c), req)

	// krok 4
	switch {
	case err == nil:
		c.JSON(http.StatusCreated, created)
	case errors.Is(err, reservation.ErrParkingNotFound):
		c.JSON(http.StatusNotFound, gin.H{"error": "nie ma takiego parkingu"})
	case errors.Is(err, reservation.ErrVehicleNotOwned):
		c.JSON(http.StatusForbidden, gin.H{"error": "pojazd nie należy do klienta"})
	case errors.Is(err, reservation.ErrOutsideLimits):
		c.JSON(http.StatusBadRequest, gin.H{"error": "rezerwacja przekracza limity czasu parkingu"})
	case errors.Is(err, reservation.ErrNoFreeSpace):
		c.JSON(http.StatusConflict, gin.H{"error": "brak wolnych miejsc na parkingu"})
	case errors.Is(err, reservation.ErrClientLimit):
		c.JSON(http.StatusConflict, gin.H{"error": "osiągnięto limit rezerwacji klienta"})
	default:
		slog.ErrorContext(c.Request.Context(), "request failed", "err", err)
		c.JSON(http.StatusInternalServerError, gin.H{"error": "błąd serwera"})
	}
}
```
Listing 2.2.2 - Udostępnienie metody `Create` jako `POST /api/reservations` w pakiecie `api`

Struktura `CreateRequest` opisuje dane przesyłane w żądaniu. Handler korzysta z logiki rezerwacji
przez metodę `Create` typu `Service` z pakietu `reservation`. Metoda `Create` dostaje kontekst żądania, identyfikator Klienta 
i dane rezerwacji, a zwraca zapisaną rezerwację albo błąd. Typ `Service` zawiera właściwą logikę i korzysta z bazy danych. 
Przy starcie programu powstaje jeden obiekt `Service` i trafia do funkcji `NewRouter`. 
Ta tworzy `handler`, przypisuje adres `POST /api/reservations` do metody`createReservation` i poprzedza go sprawdzeniem 
tokenu przez `auth.RequireClient`.

Żądanie przechodzi przez cztery kroki.

1. `auth.RequireClient` sprawdza token i zapisuje w kontekście żądania identyfikator Klienta, który je wysłał.
   Brak tokenu albo błędny token kończy się odpowiedzią `401 Unauthorized`.
2. `ShouldBindJSON` zamienia JSON na strukturę `CreateRequest` i sprawdza reguły zapisane w tagach `binding`,
   np. czy koniec rezerwacji jest później niż początek. Błędne dane kończą się odpowiedzią `400 Bad Request`.
3. Handler wywołuje `Create` z kontekstem żądania, identyfikatorem Klienta odczytanym przez `auth.ClientID`
   i danymi rezerwacji. `Create` sprawdza pozostałe warunki i zapisuje rezerwację w bazie.
4. Handler zamienia wynik na odpowiedź. Udany zapis daje `201 Created` z rezerwacją w stanie *Nieopłacona*.
   Nieznany parking daje `404 Not Found`, cudzy pojazd `403 Forbidden`, złamanie warunku rezerwacji `400 Bad Request`,
   brak wolnych miejsc i osiągnięcie limitu rezerwacji Klienta `409 Conflict`, a każdy inny błąd trafia do dziennika zdarzeń i daje `500 Internal Server Error`.

*Wybór architektury*

Warstwa logiki będzie **monolitem**, czyli jedną aplikacją, w której kod podzielono na pakiety według obszarów
dziedziny (`parking`, `vehicle`, `reservation`, `terminal`). Przemawiają za tym trzy reguły biznesowe, które wymagają
aktualnego stanu kilku obszarów naraz.

* **Rezerwacje i wjazdy korzystają z jednej puli miejsc.** Gdy zostało jedno wolne miejsce, a w tej samej chwili
  Klient je rezerwuje i pod szlaban podjeżdża pojazd bez rezerwacji, tylko jedno z nich może je dostać. W monolicie
  obie operacje sprawdzają i zapisują stan w tej samej bazie, więc druga zaczeka na pierwszą. W mikroserwisach
  rezerwacje i wjazdy byłyby w osobnych usługach z osobnymi bazami, a konflikt wyszedłby dopiero po fakcie. Wtedy
  trzeba by cofnąć jedną z operacji, a pojazdu, który już wjechał, nie da się cofnąć.
* **Upływ czasu zmienia naraz rezerwację i postój.** Koniec rezerwacji przełącza postój pojazdu na płatny. W osobnych
  usługach do czasu dotarcia powiadomienia pojazd mógłby wyjechać bez opłaty.
* **Opłata decyduje o wyjeździe.** Wyliczanie opłat przejdzie z terminali do warstwy logiki, a szlaban podnosi się
  dopiero po zapłacie. Opłata musi więc opierać się na bieżącym stanie rezerwacji i postojów, a nie na kopii danych z
  innej usługi.

Wydzielić dałoby się tylko części, które czytają dane, np. premie lojalnościowe. Reszta i tak zostałaby w jednej
usłudze z jedną bazą, dlatego cała logika zostaje w jednym serwisie.

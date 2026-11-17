## 5 Warstwa danych

Dane systemu są zapisywane w relacyjnej bazie danych **PostgreSQL**. Warstwa logiki łączy się z nią przez bibliotekę
`sqlx` ze sterownikiem `pgx` i wysyła zwykłe zapytania SQL, bez ORM (z uwagi na wybór języka Go).

Klasy modelu odpowiadają tabelom: `client`, `vehicle`, `parking`, `reservation` i `parking_events`.
Bieżący stan zajętości parkingu i liczba wolnych miejsc nie mają własnych tabel, bo są wyliczane dodatkowymi zapytaniami.

*Uzasadnienie wyboru*

Z perspektywy technicznej:

* **Reguły biznesowe wymagają transakcji.** Limit miejsc na parkingu sprawdza się, licząc istniejące rezerwacje i
  pojazdy na parkingu, a potem zapisuje nową rezerwację albo wjazd. Oba kroki muszą wykonać się jako jedna
  transakcja, a równoległe żądania dla tego samego parkingu muszą czekać na siebie. PostgreSQL zapewnia to
  blokadą wiersza (`SELECT ... FOR UPDATE`). Przy kilku instancjach API baza jest jedynym wspólnym
  miejscem, w którym taką synchronizację da się przeprowadzić.
* **Dane są ze sobą powiązane.** Rezerwacje i zdarzenia wskazują pojazd i parking, a pojazd może wskazywać klienta.
  Klucze obce i ograniczenia `CHECK`,`UNIQUE` pilnują poprawności danych nawet przy błędzie w kodzie aplikacji.
* **Dane ulotne liczy się zapytaniami.** Zajętość parkingu i liczba nakładających się rezerwacji to proste
  zapytania z `count(*)` korzystające z indeksów.
* **Dobre wsparcie w Go.** Sterownik `pgx` obsługuje typy `uuid`, `timestamptz` i typy wyliczeniowe.

Z perspektywy zamawiającego:

* PostgreSQL jest dostępny jako usługa zarządzana u wielu dostawców infrastruktury w chmurze. Kopie zapasowe, aktualizacje, replikacja i
  przełączanie awaryjne leżą wtedy po stronie dostawcy, a nie przedsiębiorstwa.
* **Brak opłat licencyjnych i uzależnienia od dostawcy.** PostgreSQL jest otwartoźródłowy, więc bazę można przenieść
  do innego dostawcy infrastruktury w chmurze bez zmian w kodzie.
* **Wiarygodne rozliczenia.** Głównym problemem przedsiębiorstwa były niezarejestrowane usługi i nadużycia. Trwały i
  spójny zapis wszystkich wjazdów, wyjazdów i rezerwacji daje dane do rozliczeń i kontroli, także przed urzędem
  skarbowym.

Bazy nierelacyjne odrzucono, bo model danych nie ma struktury dokumentowej, a reguły limitów obejmują
kilka kolekcji naraz, więc i tak potrzebne byłyby transakcje i blokady, które w relacyjnej bazie są standardem.

*Odwołanie warstwy logiki do bazy danych*

Przy starcie programu powstaje jedna pula połączeń `*sqlx.DB`, która trafia do obiektów `Service` z poszczególnych
pakietów. Adres bazy i hasło pochodzą ze zmiennej środowiskowej, a nie z kodu.

Wymagania ograniczają łączną liczbę bieżących i przyszłych rezerwacji Klienta. Wartość tego limitu jest
parametrem konfiguracji aplikacji
(`MAX_CLIENT_RESERVATIONS`), wspólnym dla wszystkich Klientów, a nie kolumną w bazie.

```go
package main

import (
	"log"
	"os"
	"strconv"

	_ "github.com/jackc/pgx/v5/stdlib"
	"github.com/jmoiron/sqlx"

	"parking/api"
	"parking/reservation"
)

func main() {
	db, err := sqlx.Connect("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	db.SetMaxOpenConns(10) // limit na instancję, bo przy N instancjach API połączeń jest N razy więcej

	maxClientReservations, err := strconv.Atoi(os.Getenv("MAX_CLIENT_RESERVATIONS"))
	if err != nil {
		log.Fatal("niepoprawny MAX_CLIENT_RESERVATIONS: ", err)
	}

	reservations := reservation.NewService(db, maxClientReservations)
	log.Fatal(api.NewRouter(reservations).Run(":8080"))
}
```
Listing 2.3.1 Utworzenie puli połączeń z bazą w pakiecie `main`

Metoda `Create` z pakietu `reservation`, którą wywołuje handler `POST /api/reservations`. Realizuje
ona tworzenie rezerwacji w jednej transakcji.

```go
package reservation

import (
	"context"
	"database/sql"
	"errors"
	"time"

	"github.com/google/uuid"
	"github.com/jmoiron/sqlx"
)

type Reservation struct {
	ID        uuid.UUID `db:"reservation_id" json:"reservationId"`
	VehicleID uuid.UUID `db:"vehicle_id" json:"vehicleId"`
	ParkingID uuid.UUID `db:"parking_id" json:"parkingId"`
	Start     time.Time `db:"reservation_start_date" json:"reservationStartDate"`
	End       time.Time `db:"reservation_end_date" json:"reservationEndDate"`
	IsPaid    bool      `db:"is_paid" json:"isPaid"`
}

type parkingLimits struct {
	Capacity       int `db:"capacity"`
	MaxDurationMin int `db:"max_reservation_duration_min"`
	MaxAdvanceMin  int `db:"max_reservation_time_in_advance_min"`
}

var ErrClientLimit = errors.New("reservation: client reservation limit reached")

type Service struct {
	db                    *sqlx.DB
	maxClientReservations int
}

func NewService(db *sqlx.DB, maxClientReservations int) *Service {
	return &Service{db: db, maxClientReservations: maxClientReservations}
}

func (s *Service) Create(ctx context.Context, clientID uuid.UUID, req CreateRequest) (Reservation, error) {
	tx, err := s.db.BeginTxx(ctx, nil)
	if err != nil {
		return Reservation{}, err
	}
	defer tx.Rollback()

	var p parkingLimits
	err = tx.GetContext(ctx, &p, `
		SELECT capacity, max_reservation_duration_min, max_reservation_time_in_advance_min
		FROM parking
		WHERE parking_id = $1
		FOR UPDATE`, req.ParkingID)
	if errors.Is(err, sql.ErrNoRows) {
		return Reservation{}, ErrParkingNotFound
	}
	if err != nil {
		return Reservation{}, err
	}

	var owned bool
	err = tx.GetContext(ctx, &owned, `
		SELECT EXISTS (SELECT 1 FROM vehicle WHERE vehicle_id = $1 AND client_id = $2)`,
		req.VehicleID, clientID)
	if err != nil {
		return Reservation{}, err
	}
	if !owned {
		return Reservation{}, ErrVehicleNotOwned
	}

	now := time.Now()
	if req.Start.Before(now) ||
		req.Start.Sub(now) > time.Duration(p.MaxAdvanceMin)*time.Minute ||
		req.End.Sub(req.Start) > time.Duration(p.MaxDurationMin)*time.Minute {
		return Reservation{}, ErrOutsideLimits
	}

	_, err = tx.ExecContext(ctx, `SELECT 1 FROM client WHERE client_id = $1 FOR UPDATE`, clientID)
	if err != nil {
		return Reservation{}, err
	}
	var clientReservations int
	err = tx.GetContext(ctx, &clientReservations, `
		SELECT count(*)
		FROM reservation r
		JOIN vehicle v ON v.vehicle_id = r.vehicle_id
		WHERE v.client_id = $1
		  AND r.reservation_end_date > now()
		  AND r.cancelled_at IS NULL`, clientID)
	if err != nil {
		return Reservation{}, err
	}
	if clientReservations >= s.maxClientReservations {
		return Reservation{}, ErrClientLimit
	}

	var overlapping int
	err = tx.GetContext(ctx, &overlapping, `
		SELECT count(*)
		FROM reservation
		WHERE parking_id = $1
		  AND reservation_start_date < $3
		  AND reservation_end_date > $2
		  AND cancelled_at IS NULL`,
		req.ParkingID, req.Start, req.End)
	if err != nil {
		return Reservation{}, err
	}
	if overlapping >= p.Capacity {
		return Reservation{}, ErrNoFreeSpace
	}

	r := Reservation{
		ID:        uuid.New(),
		VehicleID: req.VehicleID,
		ParkingID: req.ParkingID,
		Start:     req.Start,
		End:       req.End,
		IsPaid:    false,
	}
	_, err = tx.NamedExecContext(ctx, `
		INSERT INTO reservation
			(reservation_id, vehicle_id, parking_id, reservation_start_date, reservation_end_date, is_paid)
		VALUES
			(:reservation_id, :vehicle_id, :parking_id, :reservation_start_date, :reservation_end_date, :is_paid)`, r)
	if err != nil {
		return Reservation{}, err
	}
	if err := tx.Commit(); err != nil {
		return Reservation{}, err
	}
	return r, nil
}
```
Listing 2.3.2 Utworzenie rezerwacji w bazie w pakiecie `reservation`

Struktura `Reservation` odpowiada wierszowi tabeli `reservation`. Tagi `db` łączą pola struktury z kolumnami tabeli,
dzięki czemu `sqlx` sam wypełnia struktury wynikami zapytań (`GetContext`) i podstawia ich pola do zapytań z
parametrami nazwanymi (`NamedExecContext`). Wartości z żądania trafiają do zapytań wyłącznie jako parametry (`$1`,
`:vehicle_id`), a nie przez łączenie tekstu, co chroni przed atakiem SQL injection.

Metoda przechodzi przez sześć kroków.

1. `BeginTxx` rozpoczyna transakcję. Jeżeli metoda zakończy się przed `Commit` jakimś błędem,
   to transakcja wycofuje wszystkie zmiany.
2. Zapytanie odczytuje parametry parkingu i blokuje jego wiersz (`FOR UPDATE`). Inna transakcja, która w tym samym
   czasie tworzy rezerwację na tym samym parkingu, czeka do końca bieżącej. Brak wiersza oznacza nieznany parking.
3. Zapytanie sprawdza, czy pojazd należy do Klienta. Następnie API sprawdza limity czasowe parkingu.
4. Pierwsze zapytanie blokuje wiersz Klienta, więc dwa równoległe żądania tego samego Klienta nie przekroczą jego
   limitu. Drugie liczy jego bieżące i przyszłe rezerwacje, z pominięciem anulowanych. Po osiągnięciu limitu
   rezerwacja jest odrzucana błędem `ErrClientLimit`, który handler zamienia na `409 Conflict`.
5. Zapytanie liczy nieanulowane rezerwacje na tym parkingu nakładające się na przedział `[start, koniec]`. Jeżeli
   osiągnięto pojemność parkingu, rezerwacja jest odrzucana.
6. Nowa rezerwacja jest zapisywana w stanie *nieopłacona* (`is_paid = false`), a `Commit` zatwierdza transakcję i
   zwalnia blokadę parkingu.

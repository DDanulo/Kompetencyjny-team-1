
## 1 Model danych

*Słownik pojęć*

Słownik powinien objaśniać pojęcia dziedzinowe występujące w projekcie, z podziałem na:
* Pojęcia występujące w modelu danych utrwalanych w ramach projektu (odwzorowanych w diagramie klas modelu dziedzinowego)
* Pojęcia odpowiadające danym ulotnym przetwarzanym przez system, o ile występują
* Pozostałe pojęcia, o ile występują

Słownik wymienia pojęcia w języku polskim. Jeżeli w realizacji projektu używane są identyfikatory anglojęzyczne należy podać anglojęzyczne odpowiedniki w nawiasie.

*Diagram klas modelu*

```mermaid
classDiagram
    class Vehicle{
      +UUID vehicleId
      +String plateNumber
      +Client client
    }
    class Parking{
      +UUID parkingId
      +Integer capacity
      +Duration maxReservationDuration
      +Duration maxReservationTimeInAdvance
      +Duration minCancellationTimeInAdvance
    }
    class Client{
      +UUID clientId
      +String fullName
    }
    class Reservation{
      +UUID reservationId
      +Vehicle vehicle
      +Parking parking
      +DateTime reservationStartDate
      +DateTime reservationEndDate
      +Boolean isPaid
    }
    class ParkingEvents{
      +UUID eventId
      +Vehicle vehicle
      +DateTime eventTime
      +EventType eventType
      +Parking parking
    }
    class EventType{
      <<enumeration>>
      ENTRY
      EXIT
    }
    Vehicle "0..*" --> "0..1" Client
    Reservation "0..*" --> "1" Vehicle
    Reservation "0..*" --> "1" Parking
    ParkingEvents "0..*" --> "1" Vehicle
    ParkingEvents ..> EventType
    ParkingEvents "0..*" --> "1" Parking
```

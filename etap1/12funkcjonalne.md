## 2 Wymagania funkcjonalne
*Słownik aktorów*

| Aktor  | Opis roli aktora                                                                                                                               |
|--------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Klient | Uwierzytelniony użytkownik korzystający z usług parkingowych dla przypisanych mu pojazdów i ich rezerwacji. Obiekt `Client` zawiera jego dane. |
| Gość   | Użytkownik bez uwierzytelnienia. Korzysta z funkcjonalności dostępnych bez logowania.                                                          |
| System | Aktor wykonujący działania bez inicjatywy użytkownika, w tym zadania wyzwalane czasowo.                                                        |

W modelu danych klient ma odpowiednik w klasie `Client`. Gość i System są rolami bez odrębnych klas.

*Diagramy przypadków użycia*

Dla każdej z wymienionych ról wymagany jest diagram UML przypadków użycia występujących w modelu dziedzinowym projektu, bez uwzględniania przypadków użycia związanych z technicznym aspektem uwierzytelniania, kontroli dostępu i obsługi kont systemu.

Jeżeli w modelu dziedzinowym występują przypadki użycia dostępne dla użytkownika nieuwierzytelnionego (anonimowego) należy wykazać taką rolę i zamieścić odpowiedni diagram.

Jeżeli w modelu dziedzinowym występują przypadki użycia realizowane automatycznie (nie z inicjatywy użytkownika, np. wyzwalane czasowo) należy wykazać rolę System i zamieścić odpowiedni diagram.

*Diagramy stanów / przejść*

Należy zaprezentować diagram(y) stanów (przejść) dla tych obiektów modelu dziedzinowego, które w ramach swojego cyklu życia zmieniają stan (w sensie zmiany
znaczenia dziedzinowego). Obowiązkiem zaprezentowania nie są objęte:
* zmiany stanów kont użytkowników systemu
* trywialne jednoprzejściowe i bezwarunkowe zmiany stanów np. obiekt aktywny i nieaktywny
* aktualizacje stanu agregatu wyliczeniowego, na przykład zaktualizowanie wartości koszyka zakupów po dodaniu produktu

UWAGA: nie traktuje się jako zmiany stanu obiektu zbudowania czy też porzucenia związku z innym obiektem, na przykład dodania produktu do koszyka zakupów nie
traktuje się jako zmiany stanu ani produktu, ani koszyka.

Dla każdego przejścia na diagramie należy opisać jaka akcja/przypadek użycia wraz z rolą powoduje dane przejście oraz ewentualne warunki danego przejścia. To, że
dany obiekt musi istnieć, aby mógł zmienić stan, należy przyjąć jako oczywistość.

Przykład opisu: Nauczyciel: zatwierdź zaliczenie / ocena nie mniejsza niż minimalna
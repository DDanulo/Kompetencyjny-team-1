## 2 Wymagania funkcjonalne
*Słownik aktorów*

- Gość
- Klient
- System
- Terminal parkingowy
- Administrator

Słownik aktorów (ról) powinien zawierać wszystkie role występujące w modelu dziedzinowym projektu.

*Diagramy przypadków użycia*

![gosc.png](img/gosc.png)

Wykres 1.2.1 - Diagram przypadków użycia dla aktora Gość

![klient.png](img/klient.png)

Wykres 1.2.2 - Diagram przypadków użycia dla aktora Klient

![system.png](img/system.png)

Wykres 1.2.3 - Diagram przypadków użycia dla aktora System

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
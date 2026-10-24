## 8 Problem przetwarzania współbieżnego
Należy przedstawić przykładowe scenariusze, w których na skutek przetwarzania współbieżnego żądań pochodzących od wielu klientów mogłoby dojść do utrwalenia
danych niespójnych z zasadami logiki aplikacji:
* naruszenie zasad danych np. przekroczenie limitu ilościowego związków
* naruszenie zasad logiki np. niedozwolona zmiana stanu obiektu czy też próba wykonania operacji na obiekcie, którego stan na to nie pozwala

Zalecana forma: przedstawienie na osi czasu przeplotu wykonania dwóch wątków wraz ze zmianami wprowadzanymi w zbiorze danych aplikacji przez każdy z wątków i
wyraźnym zaznaczeniem, czym końcowy stan danych różni się od oczekiwanego.

Dla każdego z przypadków scharakteryzować mechanizm (lub kombinację mechanizmów), który zostanie zastosowany w aplikacji w celu wyeliminowania
przedstawionych wcześniej problemów. Na prostym przykładzie pokazać sposób użycia takiego mechanizmu w platformie, która została wybrana do implementacji
warstwy logiki systemu.
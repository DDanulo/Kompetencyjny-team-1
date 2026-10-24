## 3 Architektura
*Diagram perspektywy wdrożenia*

Należy zaprezentować diagram perspektywy wdrożenia obejmujący wszystkie warstwy systemu. Na diagramie należy zaznaczyć granicę pomiędzy tymi częściami
systemu, które działają w kontekście użytkownika (są wykonywane w systemie operacyjnym kontrolowanym przez użytkownika) oraz tymi, które działają w kontekście
serwera (są wykonywane w systemie operacyjnym / systemach operacyjnych kontrolowanym / kontrolowanych przez administratorów wdrożenia systemu).

*Stan konwersacyjny*

Wskazać warstwę systemu, która odpowiada za przechowywanie stanu konwersacyjnego systemu z użytkownikiem (z technicznego punktu widzenia: między klientem a
kontekstem serwera).

*Udostępnienie systemu*

Wskazać protokół sieciowy, który ma być zastosowany do zapewnienia komunikacji między kontekstem użytkownika a kontekstem serwera.
Wskazać zagrożenia dla bezpieczeństwa danych przesyłanych pomiędzy kontekstem klienta a innymi elementami systemu oraz wskazać standardowe rozwiązanie
ochrony danych przed tymi zagrożeniami.

*Skalowalność*

Ocenić możliwość zastosowania skalowania horyzontalnego (multiplikowania instancji) do zaproponowanej architektury systemu.
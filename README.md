# Regiony w MeshCore

## Czym są regiony (technicznie)?

Regiony są ciągami bajtów (nie znaków). Jeśli urządzenie nadawcze zostanie odpowiednio poinstruowane, dokleja ten ciąg bajtów do wiadomości jako jej scope. Ma to swoją cenę - jeśli na kanale ustawisz region, okno wiadomości dla tego kanału zmniejsza się ze 130 do 120 znaków. Tracimy 10 znaków na to, żeby w limicie ramki zmieściły się dodatkowe bajty scope.

Z punktu widzenia użytkownika region to oczywiście jakiś ciąg znaków, np. PL, PL-MAZ czy dowolny inny. MeshCore traktuje te ciągi w ciekawy sposób - potrafi pociąć je na kawałki i zbudować strukturę hierarchiczną.

## Hierarchie regionów

Hierarchia regionów to bardzo fajne narzędzie i klucz do zrozumienia, jak to tak naprawdę działa. Powiedzmy, że ustawiamy na repeterze regiony:
- PL
- PL-MAZ
- PL-MAZ-WAW
- PL-MAZ-WAW-OCHOTA

MeshCore zrozumie to nie jako osobne regiony, tylko jako hierarchię PL -> MAZ -> WAW -> OCHOTA. Czyli MeshCore wie, że region PL-MAZ-WAW-OCHOTA jest częścią regionu PL-MAZ-WAW, ten zaś częścią regionu PL-MAZ a wszystko razem mieści się w PL. Nadrzędny do tego jest region globalny oznaczany jako "*". Co z tego wynika i jak działa filtrowanie regionów na podstawie tej hierarchii, wytłumaczę za chwilę. Najpierw muszę jednak opowiedzieć, skąd się te regiony biorą w wiadomościach.

## Scope

Za dodawanie regionów do wiadomości odpowiada urządzenie nadawcze, czyli companion. On sam scope wiadomości kompletnie ignoruje - przyjmuje każdy pakiet trafiający do anteny nie przejmując się oznaczeniem regionu.

Można za to ustawić region dla kanału. Każdy kanał może mieć dodany tylko jeden region. Jeśli na kanale z ustawionym regionem wyślemy wiadomość, companion doklei do niej scope, czyli oznaczenie regionu, do którego wiadomość jest adresowana. To wszystko co potrafi companion w temacie regionów i więcej nie musi.

## Filtrowanie

Scope przychodzące w wiadomości nie inseresuje companiona, ale bardzo interesuje repeter. Tak naprawdę informacja o scope jest adresowana wyłącznie do niego. Żeby repeter potrafił scope obsłużyć, trzeba najpierw zdefiniować na nim listę regionów. Powiedzmy, że będzie to repeter dla części Warszawy:
- PL-MAZ-WAW
- PL-MAZ-WAW-OCHOTA
- PL-MAZ-WAW-BIELANY

Jeśli taki repeter dostanie wiadomość do przesłania, przepuszcza ją przez filtr, który albo wiadomość akceptuje i repeter ją retransmituje, albo ją odrzuca i wiadomość nie jest przekazywana dalej. Filtrowanie wytłumaczę na przykładach. Powiedzmy, że przychodzi kilka wiadomości:
1. Bez scope
2. Scope *
3. Scope PL-MAZ-WAW
4. Scope PL-MAZ-WAW-WAWER
5. Scope PL-MAZ
6. Scope JAMA-SMOKA

Które z nich przejdą przez repeter?

**1. Bez Scope**

Protokół MeshCore traktuje regiony jako opcjonalne. Filtrowane są tylko te pakiety, którym ktoś ustawił scope. Jeśli scope nie zostało ustawione, repeter przepuszcza pakiet zawsze, bez gadania. **Ustawienie regionów na repeterze nie blokuje wiadomości bez regionu.**

**2. Scope \***

Skutki identyczne jak przy braku ustawionego scope - globalny region * przechodzi zawsze.

**3. Scope PL-MAZ-WAW**

Jest na liście, więc przechodzi.

**4. Scope PL-MAZ-WAW-WAWER**

Tutaj robi sie ciekawie. Tego regionu nie ma na liście, ale za to zaczyna działać hierarchiczność o której wspominałem wcześniej. MeshCore dopasowuje szukając najdłuższego prefiksu. Ciąg znaków PL-MAZ-WAW-WAWER zaczyna się od PL-MAZ-WAW, więc jest dopasowanie. To znaczy, że ten pakiet przejdzie, mimo że nie ma go na liście regionów repetera. Rozumowanie jest takie - skoro PL-MAZ-WAW jest na liście a w hierarchii obejmuje inne regiony z prefiksem PL-MAZ-WAW, to zaczynający się od tego prefiksu region PL-MAZ-WAW-WAWER jest sub-regionem regionu PL-MAZ-WAW i tak adresowana wiadomość powinna być przepuszczona. Jest to bardzo logiczne zachowanie, wynika z samej idei regionalizacji.

**5. Scope PL-MAZ**

W tym przypadku dzieje się kolejna ważna i ciekawa rzecz - regionu nie ma na liście i  nie przejdzie. I nie ma znaczenia, że jest on nadrzędny do regionu PL-MAZ-WAW. Tyle, że to nie PL-MAZ-WAW jest prefiksem dla PL-MAZ, tylko odwrotnie. Filtr sprawdza, czy regiony repetera są prefiksami dla scope, nie czy scope jest prefiksem dla regionów repetera. Pakiet nie przechodzi i tyle, Warszawa słyszy tylko Warszawę.

**6. Scope JAMA-SMOKA**

Regiony w companionie każdy może sobie ustawić takie jak sobie wymyśli. Później wysyłając wiadomości będzie doklejał je do nich. Od Warszawskiego repetera te wiadomości się odbiją.

Tyle w temacie filtrowania. Wynika z niego tak naprawdę cała idea regionalizacji - jeśli to dobrze się zrozumie, regiony przestaną być tajemnicą.

## Repeter "globalny"

Jeśli na repeterze nie ustawimy żadnych regionów, będzie przesyłał dowolne wiadomości, nie ma znaczenia czy będą miały scope czy nie.

Jest jeszcze możliwość dodania na repeterze regionu **\***. To powoduje, że repeter zacznie działać jakby nie miał ustawionego żadnego regionu - będzie przepuszczał wszystko. Dla takiego repetera mija się z celem ustawianie jakichkolwiek innych regionów - i tak niczego nie zmienią w zachowaniu repetera, są po prostu zbędne.

## Co z tego wynika?

Dlaczego MeshCore wprowadziło hierarchię?
Bo:
- pozwala filtrować ruch na poziomie kraju/województwa/miasta/osiedla
- repeater może być „lokalny”, ale nadal przepuszczać ruch nadrzędny
- można budować regiony zgodne z topologią sieci
- można ograniczyć „zalewanie” pakietami z daleka

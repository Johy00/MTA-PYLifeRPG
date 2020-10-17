# Architektura

![architecture overview](img/architecture_overview.png)

Kierunek strzałek oznacza kierunek nawiązywania połączeń, nie kierunek wymiany danych.

Na czerwono zaznaczone są komponenty absolutnie wymagane.

Usługi zewnętrzne to np. API ServerProject, API haveibeenpwned.com do bezpiecznej i poufnej weryfikacji czy hasła wprowadzane przez graczy nie są publicznie znane.

## Wykorzystanie bazy PostgreSQL

Baza MySQL mimo wielu swoich limitów bardzo dobrze spełnia swoje zadanie w roli źródła prawdy dla serwera gier.
MTA oferuje natywne i działające niezawodnie mechanizmy do interakcji z nią.
Niestety z powodu specyfiki działania serwera MTA, zapytania bazodanowe są wykonywane w wątku logicznym, w związku z czym ewentualne długo trwające zapytanie wprost zatrzymuje ten wątek.

Aby temu zapobiec wdrożona została dodatkowa baza PostgreSQL z interfejsem [postgrest](http://postgrest.org/). Dzięki temu można wykorzystać interfejs HTTP REST i realizować zapytania asynchronicznie, tj. możemy wrzucić do bazy danych zapytanie które będzie trwało nawet kilka sekund, serwer może kontynuować wątek logiczny i za jakiś czas obsłużyć callback kiedy dane bedą dostępne.

Co więcej, możliwe jest w ten sposób również sięgnięcie po dane zapisane w bazie MySQL:
- za pomocą [SymmetricDS](https://www.symmetricds.org/) możemy replikować dane na żywo z jednej bazy do drugiej, a także transformować je w dodatkowe sposoby lub odkładać do backupów
- korzystając modułów `mysql foreign data wrapper` możemy bezpośrednio z PostgreSQL odpytywać się o struktury w bazie MySQL, w tym je modyfikować.

Wykorzystanie jednego z powyższych, obu lub nawet żadnego zależne jest od konkretnych potrzeb projektowych.

Na uwadze należy mieć, że interfejs REST API Postgrest jest mniej elastyczny niż bezpośredni dostęp do bazy danych. Wykonywanie skomplikowanych zapytań, składających się np. z łączeń albo subselectów jest trudne lub niemożliwe. Należy to obchodzić za pomocą widoków (które w definicji zawierają podane łączenia) lub za pomocą procedur.

## Elementy pominięte

Oprócz ww. elementów istnieje jeszcze kilka innych, które są poza scope tego repozytorium, ale warto o nich nadmienić:
- serwer WWW z oprogramowaniem forum Invision Power Board
- trzecia baza danych do forum - znacznie wyższe wymagania niż bazy serwera gier
- dwa serwery pocztowe - jeden jako smarthost pozwalający na wysyłkę maili na świat, drugi jako pełnoprawne MTA
- serwery/kontenery do obsługi wymienionych wcześniej komponentów (symmetricds, postgrest, bazy danych) a także mniejszych kontenerów pomocnicznych
- serwer backup + storage backup
- [Elastic Stack](https://www.elastic.co/elastic-stack) do agregacji, przetwarzania i przeglądania logów
- CloudFlare (jakże obowiązkowy przy obecnej scenie MTA).

Przy rozważaniu wdrożenia tych usług dla siebie warto rozważyć wykorzystanie rozwiązań zarządzanych oferowanych np. przez Amazon Web Services lub inne firmy. Ilość czynności związanych z zarządzaniem, aktualizowaniem, monitorowaniem serwerów pocztowych, serwerów bazodanowych, serwerów replikacyjnch, usługami backupu może być większa niż ilość czynności związanych z rozwojem kodu.

Przy wyborze hostingu dla serwera gier od lat polecamy usługi [ServerProject](https://serverproject.pl/). Uruchomienie ww. usług wymagało zaangażowania kilku dodatkowych serwerów fizycznych i teoretycznie było możliwe uruchomienie na nich również serwera MTA. Jednak jakość usług oferowanych przez SP, profesjonalne podejście i niskie czasy reakcji na ewentualne incydenty czy zgłoszenia supportowe przekonały nas że warto powierzyć hosting MTA właśnie firmie ServerProject.

# Kod

## Rozwój kodu

Kod był i jest rozwijany w sposób bazarowy, tj. tak jak opisano w eseju ESR [Katedra i Bazar](http://www.mkgajwer.jgora.net/katedra.html). Zachęcamy każdego do zapoznania się z tym sposobem tworzenia oprogramowania.
Podejście bazarowe pozwala tworzyć kod dla siebie, do realizacji własnych celów. Podejście te jest też zwyczajnie tańsze, szczególnie jeśli nie jesteśmy nastawieni na zysk. Tworząc kod serwera w takim modelu możemy zaczynać od małego modelu i stopniowo go rozbudowywać.

## Automaty skończone

W wielu zasobach można spotkać konstrukcję automatu skończonego (FSM - Finite State Machines).
```
local vg={} -- fsm
vg.STANY={"PATROLUJ","ZLOKALIZUJ","SLEDZ","SERWIS"}
vg.STANYDESC={
  ["PATROLUJ"]="Patroluj mapę i czekaj na wyznaczenie celu.",
  ["ZLOKALIZUJ"]="Zlokalizuj oznaczony cel.",
  ["SLEDZ"]="Rozpocznij obserwację i/lub śledzenie celu.",
  ["SERWIS"]="Uzupełnij poziom paliwa i/lub napraw helikopter",
}
vg.brain=function() ...
```

Automaty skończone są bardzo eleganckim sposobem zapisu stanu mechanizmu, szczególnie przydatnym w grach komputerowych. Automat ma ograniczoną, stałą ilość stanów i jasne warunki przejścia pomiędzy nimi. W każdej chwili można jasno określić w jakim jest stanie i do jakich stanów (i pod jakimi warunkami) może przejść. Pozwala to m.in. uniknąć sytuacji niezdefiniowanych, błędów z tym związanych a także gąszczu ifologii która jest zwyczajnie trudna do czytania i debugowania.

Więcej o automatach stanowych: 
- https://pl.wikipedia.org/wiki/Automat_sko%C5%84czony
- https://gamedevelopment.tutsplus.com/tutorials/finite-state-machines-theory-and-implementation--gamedev-11867



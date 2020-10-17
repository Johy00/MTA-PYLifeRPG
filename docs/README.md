# Widok architektury

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

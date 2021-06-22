# Projekt: Wysokie loty.

Twoim zadaniem jest zaimplementowanie zdefiniowanego poniżej API.

## Opis systemu

Napisz prosty system umożliwiający śledzenie lotów i korzystający z funkcji geograficznych oferowanych przez bibliotekę PostGIS.
Dokładny opis funkcji API do zaimplementowania znajduje się w końcowej części dokumentu.

## Technologie

System Linux. Język programowania - python. Baza danych – PostgreSQL. Testy przeprowadzałem na komputerze z Ubuntu 20.04 LTS, PostgreSQL 12.7, PostGIS 3.0. (pakiety `postgresql-12` oraz `postgis`, dokładne wersje PostgreSQL i PostGIS nie mają znaczenia, muszą jednak do siebie pasować, w szczególności być zainstalowane w odpowiednich katalogach).

##  Przechowywanie danych geograficznych

Użyj rozszerzenia PostGIS, wszelkie obliczenia dotyczące długości i szerokości geograficznej wykonuj z użyciem typu `geography` używając [WGS84 — SRID 4326](https://en.wikipedia.org/wiki/World_Geodetic_System#WGS84). Pamiętaj w opisie punktów podaje się długość przed szerokością geograficzną: `POINT(51.107883 17.038538)` (Jemen) vs. `POINT(17.038538 51.107883)` (Wrocław). Użyj odpowiedniego indeksu. 

[Dokumentacja PostGIS 3.0 - typ geography](https://postgis.net/docs/manual-3.0/using_postgis_dbmanagement.html#PostGIS_Geography)

Przydatne fragmenty kodu, w tym przykłady obliczania odległości z użyciem funkcji `ST_Distance`:

```
-- odległość pomiędzy podanymi dwoma punktami na Połwyspie Arabskim
select ST_Distance('SRID=4326;POINT(51.107883 17.038538)'::geography,'SRID=4326;POINT(50.671062 17.926126)'::geography,
    true)

   st_distance
-----------------
 108637.97748665
(1 row)

-- minimalna odległość (w km) samolotu lecącego z Seattle do Londynu (LINESTRING(-122.33 47.606, 0.0 51.5)),
-- a Reykjavikiem (POINT(-21.96 64.15)) - zakładamy lot po najkrótszej możliwej trasie
SELECT ST_Distance('LINESTRING(-122.33 47.606, 0.0 51.5)'::geography, 'POINT(-21.96 64.15)'::geography)/1000 as distance;
      distance      
--------------------
 122.23523815667001
(1 row)

-- minimalna odległość pomiędzy trasą WRO -> GDN i KTW -> KRK
SELECT ST_Distance('LINESTRING(16.89 51.1, 18.47 54.38)'::geography, 'LINESTRING(19.1 50.5, 19.8 50.08)'::geography)/1000 as distance;


create table tab (name text, geog geography);

create index on tab using Gist (geog);

insert into tab  values ('Wroclaw', 'SRID=4326;POINT(17.038538 51.107883)');
insert into tab  values ('Opole', 'SRID=4326;POINT(17.926126 50.671062)');


select name, ST_AsText(geog) from tab limit 5;             
  name   |         st_astext          
---------+----------------------------
 Wrocław | POINT(17.038538 51.107883)
 Opole   | POINT(17.926126 50.671062)
 
SELECT name FROM tab WHERE ST_DWithin(geog, ST_GeographyFromText('SRID=4326;POINT(17.038538 51.107883)'), 80000);
  name   
---------
 Wroclaw
 Opole

SELECT 'SRID=4326;POINT(' || longitude::text || ' ' || latitude::text || ')' 
   FROM city 
   WHERE name='Wrocław';
          ?column?           
-----------------------------
 SRID=4326;POINT(17.03 51.1)

SELECT ST_AsText(('SRID=4326;POINT(' || longitude::text || ' ' || latitude::text || ')')::geography) 
  FROM city 
  WHERE name='Warszawa';
     st_astext      
--------------------
 POINT(21.02 52.23)
```

## Implementacja

Ze względu na to, że interesuje nas przede wszystkim tematyka baz danych kolejne wywołania funkcji API należy wczytywać ze standardowego wejścia, a wyniki zapisywać na standardowe wyjście.
Wszystkie dane powinny być przechowywane w bazie danych, efekt działania każdej funkcji modyfikującej bazę, dla której wypisano potwierdzenie wykonania (wartość OK) powinien być utrwalony. 
**Dane dostępowe do bazy danych**: baza danych: `student`, login: `app`, password: `qwerty`.

Program będzie uruchamiany wielokrotnie z następującymi parametrami:

- pierwsze uruchomienie - program wywołany z parametrem `--init`

Wejście puste. 

- kolejne uruchomienia

Wejście zawiera wywołania kolejnych funkcji API.

## Dodatkowe informacje i założenia 
- Można założyć, że przed uruchomieniem z parametrem `--init` baza zawiera wyłącznie dwie tabele: `city` i `airport` (z bazy `mondial`).
- Zawartość tabel `city` oraz `airport` nigdy nie będzie zmieniana.
- Baza danych `student` oraz użytkownik `app` z hasłem `qwerty` będą istnieli w momencie pierwszego uruchomienia bazy, dostępne będzie również rozszerzenie PostGIS (zainstalowany pakiet `postgis` oraz wydane polecenie `create extension postgis`).
- Przy pierwszym uruchomieniu program powinien utworzyć wszystkie niezbędne elementy bazy danych (tabele, więzy, funkcje, wyzwalacze) zgodnie z przygotowanym przez studenta modelem fizycznym.
- Baza nie będzie modyfikowana pomiędzy kolejnymi uruchomieniami.
- Program nie będzie miał praw do tworzenia i zapisywania jakichkolwiek plików. 
- Program będzie mógł czytać pliki z bieżącego katalogu (np. dołączony do rozwiązania studenta plik .sql zawierający polecenia tworzące niezbędne elementy bazy).

## Format wejścia

Każda linia pliku wejściowego zawiera obiekt JSON (http://www.json.org/json-pl.html). Każdy z obiektów opisuje wywołanie jednej funkcji API wraz z argumentami.

Przykład: 

Obiekt (dla uproszczenia sformatowany w wielu liniach):
```
{
   "function":"flight",
   "params":{
      "id":"12345",
      "airports":[
         {
            "airport":"WAW",
            "takeoff_time":"2021-06-01 20:26:44.229109+02"
         },
         {
            "airport":"WRO",
            "takeoff_time":"2021-06-01 21:46:44.229109+02",
            "landing_time":"2021-06-01 21:26:44.229109+02"
         },
         {
            "airport":"GDN",
            "landing_time":"2021-06-01 22:46:44.229109+02"
         }
      ]
   }
}
```
oznacza wywołanie funkcji o nazwie `flight` z podanymi parametrami `id` typu `string` oraz `airports`, który jest tablicą zawierającą obiekty z kluczami `airport`, `takeoff_time`, `landing_time`.  

## Format wyjścia

Dla każdego wywołania wypisz w osobnej linii obiekt JSON zawierający obiekt z polami: status (zwracane zawsze), data (tylko dla funkcji zwracających krotki), debug (opcjonalnie). 

Wartość pola status to "OK" albo "ERROR".

Tabela `data` zawiera krotki wynikowe. Każda krotka to obiekt zawierający wartości wszystkich podanych w specyfikacji atrybutów.

Dopuszczalne jest dodatkowe pole o kluczu `debug` i wartości typu `string` z ew. informacją przydatną w debugowaniu (jest ona całkowicie dobrowolna i będzie ignorowana w czasie testowania, powinna mieć niewielki rozmiar).


## Przykładowe wejście i wyjście

###### Pierwsze uruchomienie (z parametrem `--init`)
wejście puste (pusty plik).

###### Oczekiwane wyjście
```
{"status": "OK"}
```

###### Kolejne uruchomienie:
```
{"function":"flight", "params":{"id":"12345", "airports":[{"airport":"WAW","takeoff_time":"2021-06-01 20:26:44.229109+02"},{"airport":"WRO","takeoff_time":"2021-06-01 21:46:44.229109+02", "landing_time":"2021-06-01 21:26:44.229109+02"}, {"airport":"GDN", "landing_time":"2021-06-01 22:46:44.229109+02"}]}}
{"function":"list_flights", "params":{"id":"12345"}}
{"function":"flight", "params":{"id":"12346", "airports":[{"airport":"KTW","takeoff_time":"2021-06-02 12:00:00.229109+02"},{"airport":"POZ", "landing_time":"2021-06-01 13:00:00.229109+02"}]}}
{"function":"list_flights", "params":{"id":"12346"}}
```

###### Oczekiwane wyjście (dla czytelności zawiera znaki nowej linii)
```
{"status": "OK"}
{"status": "OK", "data": []}
{"status": "OK"}
{
   "status":"OK",
   "data":[
      {
         "rid":"12345",
         "from":"WRO",
         "to":"GDN",
         "takeoff_time":"2021-06-01 21:46:44.229109+02"
      },
      {
         "rid":"12345",
         "from":"WAW",
         "to":"WRO",
         "takeoff_time":"2021-06-01 20:26:44.229109+02"
      }
   ]
}
```

## Format opisu API

```
<function> <arg1> <arg2> … <argn> // nazwa funkcji oraz nazwy jej argumentów
```
opis działania funkcji

```
// lista atrybutów wynikowych tabeli data lub informacja o braku tego pola
```

## Wywołania API

**Weryfikację poprawności zapytań przeprowadza inna warstwa systemu i nie musimy się tą weryfikacją przejmować. Można założyć, że wszystkie wywołania będą zawsze w prawidłowym formacie, a wszystkie wartości będą odpowiedniego typu.**


Wartość `<password>` jest typu `string`, jej długość nie przekracza 128 znaków.

###### Status "ERROR"

Aplikacja będzie testowana wyłącznie na danych spełniających niniejszą specyfikację, jednak w razie wykrycia ew. niezgodności można zwrócić status "ERROR" (nie piszemy wariantów fail-safe itp.).

## Funkcje API

###### flight

```
flight <id> <airports>
```

Dodaje informację o nowym locie z następującymi parametrami: 
- `<id>` - unikalne id lotu, typu `string`,
- `<airports>` - tablica obiektów zawierająca opisy kolejnych _segmentów_ lotu. Każdy obiekt posiada 2 lub 3 pary klucz-wartość:
  - kod IATA `<iatacode>` danego lotniska (parametr obecny zawsze), 
  - planowany moment startu `<takeoff_time>`, typ BD `timestamp with time zone`, typ JSON `string` (obecny zawsze za wyjątkiem lotniska docelowego), 
  - planowany moment lądowania `<landing_time>`, typ BD `timestamp with time zone`, typ JSON `string` (obecny zawsze za wyjątkiem lotniska początkowego).

Dla początkowego lotniska podana jest wyłącznie informacja o czasie startu, a dla docelowego - wyłącznie o czasie lądowania, dla międzylądowań podane są zawsze oba te parametry. Tablica zawiera co najmniej dwa obiekty - opisy początkowego i docelowego lotniska. Kolejność obiektów w tablicy ma znaczenie. 
Wszystkie lotniska w ramach każdego lotu są parami różne.

// nie zwraca krotek

###### list_flights

```
list_flights <id>
```
Zwraca dane wszystkich segmentów lotów takich, że najmniejsza odległość pomiędzy trasą tego segmentu, a trasą dowolnego segmentu lotu `<id>` wynosi 0.
Dla każdego segmentu zwróć identyfikator jego lotu `<rid>`, kod IATA lotniska startu `<from>` oraz lotniska lądowania `<to>`, a także `<takeoff_time>` lotniska startu danego segmentu. Lista powinna być posortowana wg `<takeoff_time>` malejąco, w drugiej kolejności wg `<rid>` rosnąco.

Nie jest wykluczone, że w wynikach znajdą się różne segmenty tego samego lotu. Nie należy zwracać żadnego z segmentów lotu o `<id>` podanym na wejściu.

Atrybuty zwracanych krotek:
```
// <rid> <from> <to> <takeoff_time>
```
###### list_cities

```
list_cities <id> <dist>
```
Zwraca listę miast położonych bliżej niż `dist` km od trasy lotu `id` posortowaną wg nazwy miasta rosnąco.

Atrybuty zwracanych krotek:
```
// <name> <prov> <country>
```
###### list_airport

```
list_airport <iatacode> <n>
```

Zwraca identyfikatory `n` ostatnich (wg `<takeoff_time>`) lotów startujących z lotniska `<iatacode>` posortowaną wg `<takeoff_time>` zaczynając od najnowszego (tj. malejąco), w drugiej kolejności wg `<id>` rosnąco. Lotnisko `<iatacode>` może być zarówno lotniskiem początkowym jak i lotniskiem międzylądowania dla zwracanych lotów. Sortując uwzględnij `<takeoff_time>` danego segmentu lotu.


Atrybuty zwracanych krotek:
```
// <id>
```

###### list_city 
 
```
list_city <name> <prov> <country> <n> <dist>
```
Znajduje `<n>` ostatnich (wg `<takeoff_time>`) lotów przelatujących bliżej niz `dist` km od centrum miasta wyznaczonego przez `<name>, <prov>, <country>`, posortowaną wg `<takeoff_time>` zaczynając od najnowszego (tj. malejąco), w drugiej kolejności wg `<rid>` rosnąco. 
Dla każdego lotu zwraca jego identyfikator `<rid>` oraz minimalną odległość od centrum miasta `<mdist>`.
Każde `<rid>` wypisz co najwyżej raz. Sortując uwzględnij odpowiedni `<takeoff_time>` dla tego segmentu lotu `<rid>`, w którym osiągana jest `<mdist>`. Zwracane odległości powinny być zaokrąglone do pełnego kilometra (funkcja `round`).

Atrybuty zwracanych krotek:
```
// <rid> <mdist>
```

## Punktacja

Maksymalna liczba punktów: **100 pkt.**.
Rozwiązania bez modelu konceptualnego lub poprawnej implementacji funkcji `flight` oraz `list_flights` otrzymują 0 punktów.

Punktacja:
- Przygotowanie modelu konceptualnego: **20 pkt.** (obowiązkowo)
- Implementacja funkcji `flight` oraz `list_flights`  po **20 pkt.** (obowiązkowo).
- Implementacja pozostałych funkcji  **po 10 pkt**.
- Odpowiednie indeksowanie wyszukiwań: **10 pkt.** 

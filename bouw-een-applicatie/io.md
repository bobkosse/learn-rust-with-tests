# IO en sorteren

**[Je vindt alle code voor dit hoofdstuk hier](https://github.com/quii/learn-go-with-tests/tree/main/io)**

[In het vorige hoofdstuk](json.md) hebben we onze applicatie verder geïnterpreteerd door een nieuw eindpunt `/league` toe te voegen. Onderweg leerden we hoe we met JSON om moesten gaan, typen moesten insluiten en routeren.

Onze product owner is enigszins verontrust door het feit dat de software de scores kwijtraakt wanneer de server opnieuw wordt opgestart. Dit komt doordat onze implementatie van onze store in-memory is. Ze is er ook niet blij mee dat we het eindpunt `/league` niet zo hebben geïnterpreteerd dat het de spelers zou moeten retourneren, gesorteerd op het aantal overwinningen!

## De code tot nu toe

```go
// server.go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strings"
)

// PlayerStore stores score information about players
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}

// Player stores a name with a number of wins
type Player struct {
	Name string
	Wins int
}

// PlayerServer is a HTTP interface for player information
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

// NewPlayerServer creates a PlayerServer with routing configured
func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
}

func (p *PlayerServer) playersHandler(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}

func (p *PlayerServer) showScore(w http.ResponseWriter, player string) {
	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter, player string) {
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

```go
// in_memory_player_store.go
package main

func NewInMemoryPlayerStore() *InMemoryPlayerStore {
	return &InMemoryPlayerStore{map[string]int{}}
}

type InMemoryPlayerStore struct {
	store map[string]int
}

func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
}

func (i *InMemoryPlayerStore) RecordWin(name string) {
	i.store[name]++
}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return i.store[name]
}
```

```go
// main.go
package main

import (
	"log"
	"net/http"
)

func main() {
	server := NewPlayerServer(NewInMemoryPlayerStore())
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Je vindt de bijbehorende tests in de link bovenaan het hoofdstuk.

## Sla de data op

Er zijn tientallen databases die we hiervoor zouden kunnen gebruiken, maar we kiezen voor een heel eenvoudige aanpak. We slaan de data voor deze applicatie op in een JSON-bestand.

Hierdoor blijven de data zeer makkelijk overdraagbaar en is het relatief eenvoudig te implementeren.

Het zal niet bijzonder goed schalen, maar aangezien dit een prototype is, voldoet het voorlopig prima. Mochten onze omstandigheden veranderen en het niet langer geschikt zijn, dan is het eenvoudig om het te vervangen door iets anders dankzij de abstractie `PlayerStore` die we hebben gebruikt.

We behouden de `InMemoryPlayerStore` voorlopig, zodat de integratietests blijven slagen terwijl we onze nieuwe store ontwikkelen. Zodra we er zeker van zijn dat onze nieuwe implementatie voldoende is om de integratietest te laten slagen, vervangen we deze en verwijderen we `InMemoryPlayerStore`.

## Schrijf eerst de test

Je zou nu bekend moeten zijn met de interfaces rond de standaardbibliotheek voor het lezen van data (`io.Reader`), het schrijven van data (`io.Writer`) en hoe we de standaardbibliotheek kunnen gebruiken om deze functies te testen zonder echte bestanden te hoeven gebruiken.

Om dit werk af te ronden, moeten we `PlayerStore` implementeren. We schrijven tests voor onze winkel die de methoden aanroepen die we moeten implementeren. We beginnen met `GetLeague`.

```go
//file_system_store_test.go
func TestFileSystemStore(t *testing.T) {

	t.Run("league from a reader", func(t *testing.T) {
		database := strings.NewReader(`[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)

		store := FileSystemPlayerStore{database}

		got := store.GetLeague()

		want := []Player{
			{"Cleo", 10},
			{"Chris", 33},
		}

		assertLeague(t, got, want)
	})
}
```

We gebruiken `strings.NewReader`, wat ons een `Reader` retourneert, wat onze `FileSystemPlayerStore` gebruikt om gegevens te lezen. In `main` openen we een bestand, wat ook een `Reader` is.

## Probeer de test uit te voeren

```
# github.com/quii/learn-go-with-tests/io/v1
./file_system_store_test.go:15:12: undefined: FileSystemPlayerStore
```

## Schrijf de minimale hoeveelheid code voor de test om uit te voeren en controleer de mislukte testuitvoer.

Laten we `FileSystemPlayerStore` definiëren in een nieuw bestand.

```go
//file_system_store.go
type FileSystemPlayerStore struct{}
```

Probeer opnieuw

```
# github.com/quii/learn-go-with-tests/io/v1
./file_system_store_test.go:15:28: too many values in struct initializer
./file_system_store_test.go:17:15: store.GetLeague undefined (type FileSystemPlayerStore has no field or method GetLeague)
```

Het klaagt omdat we een `Reader` doorgeven, maar er geen verwachten, en omdat `GetLeague` nog niet is gedefinieerd.

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.Reader
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
	return nil
}
```

En we proberen het nog een keer ...

```
=== RUN   TestFileSystemStore//league_from_a_reader
    --- FAIL: TestFileSystemStore//league_from_a_reader (0.00s)
        file_system_store_test.go:24: got [] want [{Cleo 10} {Chris 33}]
```

## Schrijf voldoende code om het te laten slagen

We hebben al eerder JSON van een reader gelezen

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() []Player {
	var league []Player
	json.NewDecoder(f.database).Decode(&league)
	return league
}
```

De test zou nu moeten slagen

## Refactor

We _hebben_ dit al eerder gedaan! Onze testcode voor de server moest de JSON uit de respons decoderen.

Laten we proberen dit op te bouwen tot een functie.

Maak een nieuw bestand met de naam `league.go` en plaats dit erin.

```go
//league.go
func NewLeague(rdr io.Reader) ([]Player, error) {
	var league []Player
	err := json.NewDecoder(rdr).Decode(&league)
	if err != nil {
		err = fmt.Errorf("problem parsing league, %v", err)
	}

	return league, err
}
```

Roep dit aan in onze implementatie en in onze testhelper `getLeagueFromResponse` in `server_test.go`

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() []Player {
	league, _ := NewLeague(f.database)
	return league
}
```

We hebben nog geen strategie om met parserfouten om te gaan, maar laten we doorgaan.

### Problemen zoeken

Er zit een fout in onze implementatie. Laten we eerst eens kijken hoe `io.Reader` is gedefinieerd.

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

Met ons bestand kun je je voorstellen dat het byte voor byte tot het einde wordt gelezen. Wat gebeurt er als je `Read` voor een tweede keer probeert uit te voeren?

Voeg het volgende toe aan het einde van onze huidige test.

```go
//file_system_store_test.go

// read again
got = store.GetLeague()
assertLeague(t, got, want)
```

We willen dat dit slaagt, maar als je de test uitvoert, lukt dat niet.

Het probleem is dat onze `Reader` het einde heeft bereikt, dus er valt niets meer te lezen. We hebben een manier nodig om hem te vertellen terug te gaan naar het begin.

[ReadSeeker](https://golang.org/pkg/io/#ReadSeeker) is een andere interface in de standaardbibliotheek die hierbij kan helpen.

```go
type ReadSeeker interface {
	Reader
	Seeker
}
```

Herinner je je de embedding nog? Dit is een interface die bestaat uit `Reader` en [`Seeker`](https://golang.org/pkg/io/#Seeker)

```go
type Seeker interface {
	Seek(offset int64, whence int) (int64, error)
}
```

Klinkt goed. Kunnen we `FileSystemPlayerStore` aanpassen zodat deze interface wordt gebruikt?

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadSeeker
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
	f.database.Seek(0, io.SeekStart)
	league, _ := NewLeague(f.database)
	return league
}
```

Probeer de test uit te voeren, hij slaagt nu! Gelukkig implementeert `strings.NewReader`, dat we in onze test gebruikten, ook `ReadSeeker`, dus hoefden we geen andere wijzigingen aan te brengen.

Vervolgens implementeren we `GetPlayerScore`.

## Schrijf eerst de test

```go
//file_system_store_test.go
t.Run("get player score", func(t *testing.T) {
	database := strings.NewReader(`[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)

	store := FileSystemPlayerStore{database}

	got := store.GetPlayerScore("Chris")

	want := 33

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
})
```

## Probeer de test uit te voeren

```
./file_system_store_test.go:38:15: store.GetPlayerScore undefined (type FileSystemPlayerStore has no field or method GetPlayerScore)
```

## Schrijf de minimale hoeveelheid code die nodig is om de test uit te voeren en controleer de mislukte testuitvoer.

We moeten de methode toevoegen aan ons nieuwe type om de test te compileren.

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {
	return 0
}
```

Nu compileert het en mislukt de test

```
=== RUN   TestFileSystemStore/get_player_score
    --- FAIL: TestFileSystemStore//get_player_score (0.00s)
        file_system_store_test.go:43: got 0 want 33
```

## Schrijf voldoende code om de test te laten slagen

We kunnen de competitie doorlopen om de speler te vinden en hun score te retourneren

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	var wins int

	for _, player := range f.GetLeague() {
		if player.Name == name {
			wins = player.Wins
			break
		}
	}

	return wins
}
```

## Refactor

Je hebt tientallen testhulp-refactorings gezien, dus ik laat het aan jou over om het werkend te krijgen

```go
//file_system_store_test.go
t.Run("get player score", func(t *testing.T) {
	database := strings.NewReader(`[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)

	store := FileSystemPlayerStore{database}

	got := store.GetPlayerScore("Chris")
	want := 33
	assertScoreEquals(t, got, want)
})
```

Ten slotte moeten we beginnen met het registreren van scores met `RecordWin`.

## Schrijf eerst de test

Onze aanpak is vrij kortzichtig voor schrijfbewerkingen. We kunnen niet (zomaar) één "rij" in een JSON-bestand bijwerken. We moeten de _gehele_ nieuwe representatie van onze database bij elke schrijfbewerking opslaan.

Hoe schrijven we? Normaal gesproken gebruiken we een `Writer`, maar we hebben al onze `ReadSeeker`. We zouden potentieel twee afhankelijkheden kunnen hebben, maar de standaardbibliotheek heeft al een interface voor `ReadWriteSeeker` waarmee we alles kunnen doen wat we met een bestand moeten doen.

Laten we ons type bijwerken

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadWriteSeeker
}
```

Bekijk of het compileert

```
./file_system_store_test.go:15:34: cannot use database (type *strings.Reader) as type io.ReadWriteSeeker in field value:
    *strings.Reader does not implement io.ReadWriteSeeker (missing Write method)
./file_system_store_test.go:36:34: cannot use database (type *strings.Reader) as type io.ReadWriteSeeker in field value:
    *strings.Reader does not implement io.ReadWriteSeeker (missing Write method)
```

Het is niet zo verrassend dat `strings.Reader` `ReadWriteSeeker` niet implementeert, dus wat doen we?

We hebben twee opties:

- Maak een tijdelijk bestand voor elke test. `*os.File` implementeert `ReadWriteSeeker`. Het voordeel hiervan is dat het meer een integratietest wordt; we lezen en schrijven echt van en naar het bestandssysteem, wat ons een zeer hoge mate van betrouwbaarheid geeft. De nadelen zijn dat we de voorkeur geven aan unittests omdat ze sneller en over het algemeen eenvoudiger zijn. We zullen ook meer werk moeten verzetten rond het maken van tijdelijke bestanden en ervoor zorgen dat ze na de test worden verwijderd.
- We zouden een bibliotheek van derden kunnen gebruiken. [Mattetti](https://github.com/mattetti) heeft een bibliotheek geschreven [filebuffer](https://github.com/mattetti/filebuffer) die de interface implementeert die we nodig hebben en het bestandssysteem niet aantast.

Ik denk niet dat er een specifiek fout antwoord is, maar door te kiezen voor een externe bibliotheek zou ik afhankelijkheidsbeheer moeten uitleggen! Dus gebruiken we in plaats daarvan bestanden.

Voordat we onze test toevoegen, moeten we onze andere tests compileren door `strings.Reader` te vervangen door `os.File`.

Laten we een aantal hulpfuncties maken die een tijdelijk bestand met wat gegevens aanmaken en onze scoretests abstraheren.

```go
//file_system_store_test.go
func createTempFile(t testing.TB, initialData string) (io.ReadWriteSeeker, func()) {
	t.Helper()

	tmpfile, err := os.CreateTemp("", "db")

	if err != nil {
		t.Fatalf("could not create temp file %v", err)
	}

	tmpfile.Write([]byte(initialData))

	removeFile := func() {
		tmpfile.Close()
		os.Remove(tmpfile.Name())
	}

	return tmpfile, removeFile
}

func assertScoreEquals(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

[CreateTemp](https://pkg.go.dev/os#CreateTemp) maakt een tijdelijk bestand aan dat we kunnen gebruiken. De `"db"`-waarde die we hebben doorgegeven, is een voorvoegsel voor een willekeurige bestandsnaam die het bestand aanmaakt. Dit om te voorkomen dat het bestand per ongeluk met andere bestanden conflicteert.

Je zult merken dat we niet alleen onze `ReadWriteSeeker` (het bestand) retourneren, maar ook een functie. We moeten ervoor zorgen dat het bestand wordt verwijderd zodra de test is voltooid. We willen geen details van de bestanden in de test laten lekken, omdat dit foutgevoelig en oninteressant is voor de lezer. Door een `removeFile`-functie te retourneren, kunnen we de details in onze helper verwerken en hoeft de aanroeper alleen `defer cleanDatabase()` uit te voeren.

```go
//file_system_store_test.go
func TestFileSystemStore(t *testing.T) {

	t.Run("league from a reader", func(t *testing.T) {
		database, cleanDatabase := createTempFile(t, `[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)
		defer cleanDatabase()

		store := FileSystemPlayerStore{database}

		got := store.GetLeague()

		want := []Player{
			{"Cleo", 10},
			{"Chris", 33},
		}

		assertLeague(t, got, want)

		// read again
		got = store.GetLeague()
		assertLeague(t, got, want)
	})

	t.Run("get player score", func(t *testing.T) {
		database, cleanDatabase := createTempFile(t, `[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)
		defer cleanDatabase()

		store := FileSystemPlayerStore{database}

		got := store.GetPlayerScore("Chris")
		want := 33
		assertScoreEquals(t, got, want)
	})
}
```

Voer de tests uit en ze zouden moeten slagen! Er zijn behoorlijk wat wijzigingen aangebracht, maar nu lijkt het erop dat we onze interfacedefinitie compleet hebben en dat het heel eenvoudig zou moeten zijn om vanaf nu nieuwe tests toe te voegen.

Laten we de eerste keer een overwinning voor een bestaande speler vastleggen.

```go
//file_system_store_test.go
t.Run("store wins for existing players", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer cleanDatabase()

	store := FileSystemPlayerStore{database}

	store.RecordWin("Chris")

	got := store.GetPlayerScore("Chris")
	want := 34
	assertScoreEquals(t, got, want)
})
```

## Probeer de test uit te voeren

`./file_system_store_test.go:67:8: store.RecordWin undefined (type FileSystemPlayerStore heeft geen veld of methode RecordWin)`

## Schrijf de minimale hoeveelheid code voor de test en controleer de mislukte testuitvoer

Voeg de nieuwe methode toe

```go
//file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {

}
```

```
=== RUN   TestFileSystemStore/store_wins_for_existing_players
    --- FAIL: TestFileSystemStore/store_wins_for_existing_players (0.00s)
        file_system_store_test.go:71: got 33 want 34
```

Onze implementatie is leeg, dus de oude score wordt geretourneerd.

## Schrijf voldoende code om het te laten slagen

```go
//file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()

	for i, player := range league {
		if player.Name == name {
			league[i].Wins++
		}
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

Je vraagt je misschien af waarom ik `league[i].Wins++` gebruik in plaats van `player.Wins++`.

Wanneer je een slice met een `range` uitvoert, krijg je de huidige index van de lus (in ons geval `i`) en een kopie van het element op die index terug. Het wijzigen van de `Wins`-waarde van een kopie heeft geen effect op de `league`-slice waarop we itereren. Daarom moeten we de referentie naar de werkelijke waarde verkrijgen door `league[i]` uit te voeren en vervolgens die waarde te wijzigen.

Als je de tests uitvoert, zouden ze nu moeten slagen.

## Refactor

In `GetPlayerScore` en `RecordWin` itereren we over `[]Player` om een speler op naam te vinden.

We zouden deze algemene code kunnen refactoren in de interne werking van `FileSystemStore`, maar het lijkt me dat dit misschien nuttige code is die we naar een nieuw type kunnen tillen. Tot nu toe werkten we altijd met een "League" met `[]Player`, maar we kunnen een nieuw type creëren met de naam `League`. Dit zal voor andere ontwikkelaars gemakkelijker te begrijpen zijn en dan kunnen we nuttige methoden aan dat type koppelen die we zelf kunnen gebruiken.

Voeg binnen `league.go` het volgende toe

```go
//league.go
type League []Player

func (l League) Find(name string) *Player {
	for i, p := range l {
		if p.Name == name {
			return &l[i]
		}
	}
	return nil
}
```

Als iemand nu een `League` heeft, kan hij of zij gemakkelijk een bepaalde speler vinden.

Wijzig onze `PlayerStore`-interface zodat deze `League` retourneert in plaats van `[]Player`. Probeer de tests opnieuw uit te voeren; je krijgt een compilatieprobleem omdat we de interface hebben aangepast, maar dit is heel eenvoudig op te lossen; verander gewoon het retourtype van `[]Player` naar `League`.

Hierdoor kunnen we onze methoden in `file_system_store` vereenvoudigen.

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	player := f.GetLeague().Find(name)

	if player != nil {
		return player.Wins
	}

	return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()
	player := league.Find(name)

	if player != nil {
		player.Wins++
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

Dit ziet er veel beter uit en we zien hoe we andere nuttige functionaliteit rond `League` kunnen vinden die we kunnen refactoren.

We moeten nu het scenario van het registreren van overwinningen van nieuwe spelers afhandelen.

## Schrijf eerst de test

```go
//file_system_store_test.go
t.Run("store wins for new players", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer cleanDatabase()

	store := FileSystemPlayerStore{database}

	store.RecordWin("Pepper")

	got := store.GetPlayerScore("Pepper")
	want := 1
	assertScoreEquals(t, got, want)
})
```

## Probeer de test uit te voeren

```
=== RUN   TestFileSystemStore/store_wins_for_new_players#01
    --- FAIL: TestFileSystemStore/store_wins_for_new_players#01 (0.00s)
        file_system_store_test.go:86: got 0 want 1
```

## Schrijf voldoende code om het te laten slagen

We hoeven alleen het scenario af te handelen waarbij `Find` `nil` retourneert omdat de speler niet gevonden kon worden.

```go
//file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()
	player := league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		league = append(league, Player{name, 1})
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

Het 'happy path' ziet er goed uit, dus we kunnen nu onze nieuwe 'Store' in de integratietest proberen te gebruiken. Dit geeft ons meer vertrouwen dat de software werkt en vervolgens kunnen we de overbodige `InMemoryPlayerStore` verwijderen.

Vervang de oude `Store` in `TestRecordingWinsAndRetrievingThem`.

```go
//server_integration_test.go
database, cleanDatabase := createTempFile(t, "")
defer cleanDatabase()
store := &FileSystemPlayerStore{database}
```

Als je de test uitvoert, zou deze moeten slagen en kunnen we `InMemoryPlayerStore` verwijderen. `main.go` zal nu compilatieproblemen hebben, wat ons motiveert om onze nieuwe opslag in de "echte" code te gebruiken.

```go
// main.go
package main

import (
	"log"
	"net/http"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store := &FileSystemPlayerStore{db}
	server := NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

- We maken een bestand aan voor onze database.
- Met het 2e argument van `os.OpenFile` kun je de rechten voor het openen van het bestand definiëren. In ons geval betekent `O_RDWR` dat we willen lezen en schrijven _en_ `os.O_CREATE` betekent dat we het bestand moeten aanmaken als het nog niet bestaat.
- Met het 3e argument worden de rechten voor het bestand ingesteld. In ons geval kunnen alle gebruikers het bestand lezen en schrijven. [(Zie superuser.com voor een meer gedetailleerde uitleg)](https://superuser.com/questions/295591/what-is-the-meaning-of-chmod-666).

Het uitvoeren van het programma bewaart nu de gegevens in een bestand tussen herstarts door, hoera!

## Meer refactoring en prestatieproblemen

Elke keer dat iemand `GetLeague()` of `GetPlayerScore()` aanroept, lezen we het volledige bestand en parseren we het naar JSON. Dat zou niet nodig moeten zijn, omdat `FileSystemStore` volledig verantwoordelijk is voor de status van de competitie; het zou het bestand alleen moeten lezen wanneer het programma opstart en het alleen moeten bijwerken wanneer de gegevens veranderen.

We kunnen een constructor maken die een deel van deze initialisatie voor ons uitvoert en de competitie als een waarde in onze `FileSystemStore` opslaat, die vervolgens bij het lezen wordt gebruikt.

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadWriteSeeker
	league   League
}

func NewFileSystemPlayerStore(database io.ReadWriteSeeker) *FileSystemPlayerStore {
	database.Seek(0, io.SeekStart)
	league, _ := NewLeague(database)
	return &FileSystemPlayerStore{
		database: database,
		league:   league,
	}
}
```

Op deze manier hoeven we maar één keer van schijf te lezen. We kunnen nu al onze eerdere aanroepen om de competitie van schijf te halen vervangen en gewoon `f.league` gebruiken.

```go
//file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() League {
	return f.league
}

func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	player := f.league.Find(name)

	if player != nil {
		return player.Wins
	}

	return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
	player := f.league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		f.league = append(f.league, Player{name, 1})
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(f.league)
}
```

Als je de tests probeert uit te voeren, zal het nu klagen over het initialiseren van `FileSystemPlayerStore`, dus los ze gewoon op door onze nieuwe constructor aan te roepen.

### Nog een probleem

Er zit nog meer naïviteit in de manier waarop we met bestanden omgaan, wat op termijn een vervelende bug _zou_ kunnen veroorzaken.

Wanneer we `RecordWin` gebruiken, `zoeken` we terug naar het begin van het bestand en schrijven we vervolgens de nieuwe gegevens weg. Maar wat als de nieuwe gegevens kleiner zijn dan de vorige?

In ons huidige geval is dit onmogelijk. We bewerken of verwijderen nooit scores, dus de gegevens kunnen alleen maar groter worden. Het zou echter onverantwoord zijn om de code zo te laten; het is niet ondenkbaar dat er een verwijderscenario zou kunnen ontstaan.

Hoe gaan we dit testen? Wat we moeten doen, is eerst onze code refactoren, zodat we de _soort gegevens die we schrijven, scheiden van de schrijfbewerking_. We kunnen dat dan apart testen om te controleren of het werkt zoals we hopen.

We maken een nieuw type aan om onze functionaliteit "als we schrijven, beginnen we vanaf het begin" te definiëren. Ik noem het `Tape`. Maak een nieuw bestand met de volgende code:

```go
// tape.go
package main

import "io"

type tape struct {
	file io.ReadWriteSeeker
}

func (t *tape) Write(p []byte) (n int, err error) {
	t.file.Seek(0, io.SeekStart)
	return t.file.Write(p)
}
```

Merk op dat we nu alleen `Write` implementeren, omdat het het `Seek`-gedeelte inkapselt. Dit betekent dat onze `FileSystemStore` gewoon naar een `Writer` kan verwijzen.

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database io.Writer
	league   League
}
```

Werk de constructor bij om `Tape` te gebruiken

```go
//file_system_store.go
func NewFileSystemPlayerStore(database io.ReadWriteSeeker) *FileSystemPlayerStore {
	database.Seek(0, io.SeekStart)
	league, _ := NewLeague(database)

	return &FileSystemPlayerStore{
		database: &tape{database},
		league:   league,
	}
}
```

Eindelijk kunnen we de geweldige beloning krijgen die we wilden door de `Seek`-aanroep uit `RecordWin` te verwijderen. Ja, het voelt misschien niet veel, maar het betekent in ieder geval dat we erop kunnen vertrouwen dat onze `Write` zich gedraagt zoals we willen, ook als we andere schrijfbewerkingen uitvoeren. Bovendien kunnen we nu de potentieel problematische code apart testen en oplossen.

Laten we de test schrijven waarbij we de volledige inhoud van een bestand willen bijwerken met iets dat kleiner is dan de oorspronkelijke inhoud.

## Schrijf eerst de test

Onze test maakt een bestand met wat inhoud aan, probeert ernaar te schrijven met behulp van de `tape` en leest alles opnieuw om te zien wat er in het bestand staat. In `tape_test.go`:

```go
//tape_test.go
func TestTape_Write(t *testing.T) {
	file, clean := createTempFile(t, "12345")
	defer clean()

	tape := &tape{file}

	tape.Write([]byte("abc"))

	file.Seek(0, io.SeekStart)
	newFileContents, _ := io.ReadAll(file)

	got := string(newFileContents)
	want := "abc"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

## Probeer de test uit te voeren

```
=== RUN   TestTape_Write
--- FAIL: TestTape_Write (0.00s)
    tape_test.go:23: got 'abc45' want 'abc'
```

Zoals we al dachten! Het schrijft de gewenste gegevens weg, maar laat de rest van de originele gegevens staan.

## Schrijf voldoende code om het te laten werken

`os.File` heeft een afkapfunctie waarmee we het bestand effectief kunnen legen. We zouden dit gewoon moeten kunnen aanroepen om te krijgen wat we willen.

Verander `tape` in het volgende:

```go
//tape.go
type tape struct {
	file *os.File
}

func (t *tape) Write(p []byte) (n int, err error) {
	t.file.Truncate(0)
	t.file.Seek(0, io.SeekStart)
	return t.file.Write(p)
}
```

De compiler zal op een aantal plaatsen falen waar we een `io.ReadWriteSeeker` verwachten, maar we sturen `*os.File` mee. Je zou deze problemen nu zelf moeten kunnen oplossen, maar als je vastloopt, controleer dan gewoon de broncode.

Zodra je het refactoren onder de knie hebt, zou onze `TestTape_Write`-test moeten slagen!

### Nog een kleine refactoring

In `RecordWin` hebben we de regel `json.NewEncoder(f.database).Encode(f.league)`.

We hoeven niet elke keer dat we schrijven een nieuwe encoder te maken, we kunnen er een initialiseren in onze constructor en die gebruiken.

Sla een verwijzing naar een `Encoder` op in ons type en initialiseer deze in de constructor:

```go
//file_system_store.go
type FileSystemPlayerStore struct {
	database *json.Encoder
	league   League
}

func NewFileSystemPlayerStore(file *os.File) *FileSystemPlayerStore {
	file.Seek(0, io.SeekStart)
	league, _ := NewLeague(file)

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}
}
```

Gebruik dit in `RecordWin`.

```go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	player := f.league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		f.league = append(f.league, Player{name, 1})
	}

	f.database.Encode(f.league)
}
```

## Hebben we daar niet gewoon wat regels overtreden? Privé-items testen? Geen interfaces?

### Over het testen van privé-typen

Het klopt dat je _over het algemeen_ beter geen privé-items kunt testen, omdat dat er soms toe kan leiden dat je tests te nauw gekoppeld zijn aan de implementatie, wat toekomstige refactoring kan belemmeren.

We mogen echter niet vergeten dat tests ons _vertrouwen_ moeten geven.

We hadden er geen vertrouwen in dat onze implementatie zou werken als we bewerkings- of verwijderfunctionaliteit zouden toevoegen. We wilden de code niet zo laten, vooral niet als er door meer dan één persoon aan gewerkt werd die zich mogelijk niet bewust was van de tekortkomingen van onze oorspronkelijke aanpak.

Tenslotte is het maar één test! Als we besluiten de werking te veranderen, is het geen ramp om de test zomaar te verwijderen, maar we hebben in ieder geval voldaan aan de eisen voor toekomstige beheerders.

### Interfaces

We begonnen de code met `io.Reader`, omdat dat de makkelijkste manier was om onze nieuwe `PlayerStore` te testen. Tijdens het ontwikkelen van de code gingen we verder met `io.ReadWriter` en vervolgens met `io.ReadWriteSeeker`. We ontdekten toen dat er niets in de standaardbibliotheek stond dat dit daadwerkelijk implementeerde, behalve `*os.File`. We hadden er ook voor kunnen kiezen om onze eigen te schrijven of een open source-variant te gebruiken, maar het voelde pragmatisch om gewoon tijdelijke bestanden voor de tests te maken.

Tot slot hadden we `Truncate` nodig, dat ook op `*os.File` staat. Het zou een optie zijn geweest om onze eigen interface te maken die aan deze vereisten voldeed.

```go
type ReadWriteSeekTruncate interface {
	io.ReadWriteSeeker
	Truncate(size int64) error
}
```

Maar wat levert dit ons nu echt op? Houd er rekening mee dat we _niet aan het mocken_ zijn en dat het onrealistisch is dat een **bestandssysteem** een ander type dan `*os.File` accepteert, dus we hebben het polymorfisme dat interfaces ons bieden niet nodig.

Wees niet bang om typen te wijzigen en te experimenteren zoals we hier hebben gedaan. Het mooie van een statisch getypeerde taal is dat de compiler je bij elke wijziging helpt.

## Foutafhandeling

Voordat we beginnen met sorteren, moeten we ervoor zorgen dat we tevreden zijn met onze huidige code en eventuele technische problemen oplossen. Het is een belangrijk principe om zo snel mogelijk werkende software te krijgen (voorkom de rode status), maar dat betekent niet dat we foutgevallen moeten negeren!

Als we teruggaan naar `FileSystemStore.go`, hebben we `league, _ := NewLeague(f.database)` in onze constructor.

`NewLeague` kan een foutmelding retourneren als de competitie niet kan worden geparsed vanuit de `io.Reader` die we hebben aangeleverd.

Het was pragmatisch om dat eerst te negeren, aangezien we al falende tests hadden. Als we hadden geprobeerd het tegelijkertijd aan te pakken, zouden we twee dingen tegelijk hebben moeten doen.

Laten we ervoor zorgen dat onze constructor een foutmelding kan retourneren.

```go
//file_system_store.go
func NewFileSystemPlayerStore(file *os.File) (*FileSystemPlayerStore, error) {
	file.Seek(0, io.SeekStart)
	league, err := NewLeague(file)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
	}

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}, nil
}
```

Onthoud dat het erg belangrijk is om nuttige foutmeldingen te geven (net als bij je tests). Mensen op internet zeggen gekscherend dat de meeste Go-code er zo uitziet:

```go
if err != nil {
	return err
}
```

**Dat is 100% niet de realiteit.** Door contextuele informatie (d.w.z. wat je deed waardoor de fout ontstond) aan je foutmeldingen toe te voegen, wordt het bedienen van je software veel eenvoudiger.

Als je probeert te compileren, krijg je fouten.

```
./main.go:18:35: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:35:36: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:57:36: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:70:36: multiple-value NewFileSystemPlayerStore() in single-value context
./file_system_store_test.go:85:36: multiple-value NewFileSystemPlayerStore() in single-value context
./server_integration_test.go:12:35: multiple-value NewFileSystemPlayerStore() in single-value context
```

Meestal willen we het programma afsluiten en de foutmelding weergeven.

```go
//main.go
store, err := NewFileSystemPlayerStore(db)

if err != nil {
	log.Fatalf("problem creating file system player store, %v ", err)
}
```

In de tests moeten we bevestigen dat er geen fouten zijn. We kunnen een hulpmiddel maken om dit te verhelpen.

```go
//file_system_store_test.go
func assertNoError(t testing.TB, err error) {
	t.Helper()
	if err != nil {
		t.Fatalf("didn't expect an error but got one, %v", err)
	}
}
```

Los de andere compilatieproblemen op met behulp van deze helper. Uiteindelijk zou je een falende test moeten hebben:

```
=== RUN   TestRecordingWinsAndRetrievingThem
--- FAIL: TestRecordingWinsAndRetrievingThem (0.00s)
    server_integration_test.go:14: didn't expect an error but got one, problem loading player store from file /var/folders/nj/r_ccbj5d7flds0sf63yy4vb80000gn/T/db841037437, problem parsing league, EOF
```

We kunnen de league niet parseren omdat het bestand leeg is. We kregen voorheen geen fouten omdat we ze altijd negeerden.

Laten we onze grote integratietest repareren door er een geldige JSON-code in te plaatsen:

```go
//server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[]`)
	//etc...
}
```

Nu alle tests succesvol zijn, moeten we het scenario afhandelen waarin het bestand leeg is.

## Schrijf eerst de test

```go
//file_system_store_test.go
t.Run("works with an empty file", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, "")
	defer cleanDatabase()

	_, err := NewFileSystemPlayerStore(database)

	assertNoError(t, err)
})
```

## Probeer de test uit te voeren

```
=== RUN   TestFileSystemStore/works_with_an_empty_file
    --- FAIL: TestFileSystemStore/works_with_an_empty_file (0.00s)
        file_system_store_test.go:108: didn't expect an error but got one, problem loading player store from file /var/folders/nj/r_ccbj5d7flds0sf63yy4vb80000gn/T/db019548018, problem parsing league, EOF
```

## Schrijf voldoende code om de test te laten slagen

Wijzig onze constructor naar het volgende

```go
//file_system_store.go
func NewFileSystemPlayerStore(file *os.File) (*FileSystemPlayerStore, error) {

	file.Seek(0, io.SeekStart)

	info, err := file.Stat()

	if err != nil {
		return nil, fmt.Errorf("problem getting file info from file %s, %v", file.Name(), err)
	}

	if info.Size() == 0 {
		file.Write([]byte("[]"))
		file.Seek(0, io.SeekStart)
	}

	league, err := NewLeague(file)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
	}

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}, nil
}
```

`file.Stat` retourneert statistieken over ons bestand, waarmee we de bestandsgrootte kunnen controleren. Als het leeg is, Schrijven we met `Write` een lege JSON-array en zoeken we met `Seek` terug naar het begin, klaar voor de rest van de code.

## Refactor

Onze constructor is nu een beetje rommelig, dus laten we de initialisatiecode extraheren naar een functie:

```go
//file_system_store.go
func initialisePlayerDBFile(file *os.File) error {
	file.Seek(0, io.SeekStart)

	info, err := file.Stat()

	if err != nil {
		return fmt.Errorf("problem getting file info from file %s, %v", file.Name(), err)
	}

	if info.Size() == 0 {
		file.Write([]byte("[]"))
		file.Seek(0, io.SeekStart)
	}

	return nil
}
```

```go
//file_system_store.go
func NewFileSystemPlayerStore(file *os.File) (*FileSystemPlayerStore, error) {

	err := initialisePlayerDBFile(file)

	if err != nil {
		return nil, fmt.Errorf("problem initialising player db file, %v", err)
	}

	league, err := NewLeague(file)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %s, %v", file.Name(), err)
	}

	return &FileSystemPlayerStore{
		database: json.NewEncoder(&tape{file}),
		league:   league,
	}, nil
}
```

## Sorteren

Onze Product Owner wil dat `/league` de spelers sorteert op hun scores, van hoog naar laag.

De belangrijkste beslissing die hier genomen moet worden, is waar in de software dit moet gebeuren. Als we een "echte" database zouden gebruiken, zouden we dingen zoals `ORDER BY` gebruiken, zodat het sorteren supersnel gaat. Daarom lijkt het erop dat implementaties van `PlayerStore` hiervoor verantwoordelijk zouden moeten zijn.

## Schrijf eerst de test

We kunnen de bewering in onze eerste test in `TestFileSystemStore` bijwerken:

```go
//file_system_store_test.go
t.Run("league sorted", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer cleanDatabase()

	store, err := NewFileSystemPlayerStore(database)

	assertNoError(t, err)

	got := store.GetLeague()

	want := League{
		{"Chris", 33},
		{"Cleo", 10},
	}

	assertLeague(t, got, want)

	// read again
	got = store.GetLeague()
	assertLeague(t, got, want)
})
```

De volgorde waarin de JSON binnenkomt, is verkeerd en onze `want` controleert of deze in de juiste volgorde naar de aanroeper wordt teruggestuurd.

## Probeer de test uit te voeren

```
=== RUN   TestFileSystemStore/league_from_a_reader,_sorted
    --- FAIL: TestFileSystemStore/league_from_a_reader,_sorted (0.00s)
        file_system_store_test.go:46: got [{Cleo 10} {Chris 33}] want [{Chris 33} {Cleo 10}]
        file_system_store_test.go:51: got [{Cleo 10} {Chris 33}] want [{Chris 33} {Cleo 10}]
```

## Schrijf genoeg code om de test te laten slagen

```go
func (f *FileSystemPlayerStore) GetLeague() League {
	sort.Slice(f.league, func(i, j int) bool {
		return f.league[i].Wins > f.league[j].Wins
	})
	return f.league
}
```

[`sort.Slice`](https://golang.org/pkg/sort/#Slice)

> Slice sorteert de meegeleverde slice, rekening houdend met de meegeleverde functie.

Makkelijk!

## Samenvattend

### Wat we hebben behandeld

- De `Seeker`-interface en de relatie met `Reader` en `Writer`.
- Werken met bestanden.
- Een gebruiksvriendelijke helper maken voor het testen met bestanden die alle rommel verbergt.
- `sort.Slice` voor het sorteren van slices.
- De compiler gebruiken om veilig structurele wijzigingen in de applicatie aan te brengen.

### Regels overtreden

- De meeste regels in software engineering zijn eigenlijk geen regels, maar gewoon best practices die in 80% van de gevallen werken.
- We ontdekten een scenario waarin een van onze eerdere "regels" om interne functies niet te testen niet nuttig voor ons was, dus hebben we die regel overtreden.
- Het is belangrijk om bij het overtreden van regels te begrijpen welke afweging je maakt. In ons geval vonden we het prima, omdat het slechts één test was en het scenario anders erg moeilijk zou zijn geweest om uit te voeren.
- Om de regels te kunnen overtreden, **moet je ze eerst begrijpen**. Een analogie is met gitaar leren spelen. Het maakt niet uit hoe creatief je denkt te zijn, je moet de basisprincipes begrijpen en oefenen.

### Waar onze software staat

- We hebben een HTTP API waarmee je spelers kunt aanmaken en hun score kunt verhogen.
- We kunnen een competitie van ieders scores als JSON retourneren.
- De gegevens worden opgeslagen als een JSON-bestand.

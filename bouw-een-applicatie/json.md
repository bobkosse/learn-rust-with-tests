# JSON, routing & embedding

**[Alle code voor dit hoofdstuk vind je hier](https://github.com/quii/learn-go-with-tests/tree/main/json)**

[In het vorige hoofdstuk](http-server.md) hebben we een webserver gemaakt om op te slaan hoeveel wedstrijden spelers hebben gewonnen.

Onze product owner heeft een nieuwe vereiste: een nieuw eindpunt genaamd `/league` dat een lijst met alle opgeslagen spelers retourneert. Ze wil dat dit als JSON wordt geretourneerd.

## Dit is de code die we tot nu toe hebben

```go
// server.go
package main

import (
	"fmt"
	"net/http"
	"strings"
)

type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
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
	server := &PlayerServer{NewInMemoryPlayerStore()}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Je vindt de bijbehorende tests in de link bovenaan het hoofdstuk.

We beginnen met het maken van het eindpunt voor de ranglijst.

## Schrijf eerst de test

We breiden de bestaande suite uit, omdat we een aantal handige testfuncties en een fake-`PlayerStore` hebben om te gebruiken.

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := &PlayerServer{&store}

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

Voordat we ons druk maken over de daadwerkelijke scores en JSON, proberen we de wijzigingen klein te houden en itereren we naar ons doel toe. De eenvoudigste manier om te beginnen is door te controleren of we op `/league` kunnen aanroepen en een `OK` terugkrijgen.

## Probeer de test uit te voeren

```
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:101: status code is wrong: got 404, want 200
FAIL
FAIL	playerstore	0.221s
FAIL
```

Onze `PlayerServer` retourneert een `404 Not Found`, alsof we de winst van een onbekende speler proberen te achterhalen. Als we kijken naar hoe `server.go` `ServeHTTP` implementeert, zien we dat het er altijd van uitgaat dat het wordt aangeroepen met een URL die naar een specifieke speler verwijst:

```go
player := strings.TrimPrefix(r.URL.Path, "/players/")
```

In het vorige hoofdstuk noemden we dit een vrij naïeve manier om de router te gebruiken. Onze test informeert ons terecht dat we een concept nodig hebben voor hoe we met verschillende aanvraagpaden moeten omgaan.

## Schrijf voldoende code om het te laten slagen

Go heeft een ingebouwd routeringsmechanisme genaamd [`ServeMux`](https://golang.org/pkg/net/http/#ServeMux) (request multiplexer) waarmee je `http.Handler`s aan specifieke aanvraagpaden kunt koppelen.

Laten we een paar zonden begaan en de tests zo snel mogelijk laten slagen, wetende dat we ze veilig kunnen refactoren zodra we weten dat de tests slagen.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()

	router.Handle("/league", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	router.Handle("/players/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		player := strings.TrimPrefix(r.URL.Path, "/players/")

		switch r.Method {
		case http.MethodPost:
			p.processWin(w, player)
		case http.MethodGet:
			p.showScore(w, player)
		}
	}))

	router.ServeHTTP(w, r)
}
```

- Wanneer de aanvraag start, maken we een router aan en geven we hem de opdracht om voor het pad `x` de handler `y` te gebruiken.
- Dus voor ons nieuwe eindpunt gebruiken we `http.HandlerFunc` en een _anonieme functie_ voor `w.WriteHeader(http.StatusOK)` wanneer `/league` wordt aangevraagd om onze nieuwe test te laten slagen.
- Voor de route `/players/` knippen en plakken we onze code in een andere `http.HandlerFunc`.
- Ten slotte handelen we de binnenkomende aanvraag af door `ServeHTTP` van onze nieuwe router aan te roepen (merk je op dat `ServeMux` _ook_ een `http.Handler` is?).

De tests zouden nu moeten slagen.

## Refactor

`ServeHTTP` lijkt nogal groot, we kunnen de zaken wat opsplitsen door onze handlers te refactoren in aparte methoden.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	router.ServeHTTP(w, r)
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
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
```

Het is nogal vreemd (en inefficiënt) om een router in te stellen terwijl er een verzoek binnenkomt en deze vervolgens aan te roepen. Idealiter zouden we een soort `NewPlayerServer`-functie willen hebben die onze afhankelijkheden gebruikt en de router eenmalig instelt. Elk verzoek kan dan gewoon die ene instantie van de router gebruiken.

```go
//server.go
type PlayerServer struct {
	store  PlayerStore
	router *http.ServeMux
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := &PlayerServer{
		store,
		http.NewServeMux(),
	}

	p.router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	p.router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	return p
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	p.router.ServeHTTP(w, r)
}
```

- `PlayerServer` moet nu een router opslaan.
- We hebben de routing-aanmaak verplaatst van `ServeHTTP` naar onze `NewPlayerServer`, zodat dit slechts één keer hoeft te gebeuren, niet per aanvraag.
- Je moet alle test- en productiecode, waar we voorheen `PlayerServer{&store}` gebruikten, bijwerken met `NewPlayerServer(&store)`.

### Nog een laatste refactoring

Probeer de code als volgt aan te passen.

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}
```

Vervang vervolgens `server := &PlayerServer{&store}` door `server := NewPlayerServer(&store)` in `server_test.go`, `server_integration_test.go` en `main.go`.

Verwijder tot slot `func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request)`, want deze is niet langer nodig!

## Embedding

We hebben de tweede eigenschap van `PlayerServer` gewijzigd door de genoemde eigenschap `router http.ServeMux` te verwijderen en te vervangen door `http.Handler`; dit wordt _embedding_ genoemd.

> Go biedt niet de typische, type-gedreven notie van subklassen, maar het heeft wel de mogelijkheid om delen van een implementatie te "lenen" door typen in een struct of interface te embedden.

[Effectieve Go - Embedding](https://golang.org/doc/effective_go.html#embedding)

Dit betekent dat onze `PlayerServer` nu alle methoden heeft die `http.Handler` heeft, namelijk `ServeHTTP`.

Om de `http.Handler` te "invullen", wijzen we deze toe aan de `router` die we in `NewPlayerServer` hebben aangemaakt. Dit is mogelijk omdat `http.ServeMux` de methode `ServeHTTP` heeft.

Hiermee kunnen we onze eigen `ServeHTTP`-methode verwijderen, omdat we er al een beschikbaar stellen via het embedded type.

Embedden is een zeer interessante taalfunctie. Je kunt het gebruiken met interfaces om nieuwe interfaces samen te stellen.

```go
type Animal interface {
	Eater
	Sleeper
}
```

En je kunt het ook gebruiken met concrete typen, niet alleen interfaces. Zoals je zou verwachten, heb je bij het embedden van een concreet type toegang tot alle openbare methoden en velden.

### Nadelen?

Je moet voorzichtig zijn met het embedden van typen, omdat je daarmee alle openbare methoden en velden van het type dat je embed, blootlegt. In ons geval is dat geen probleem, omdat we alleen de _interface_ hebben embed die we wilden blootleggen (`http.Handler`).

Als we lui waren geweest en in plaats daarvan `http.ServeMux` hadden embed (het concrete type), zou het nog steeds werken, _maar_ zouden gebruikers van `PlayerServer` nieuwe routes naar onze server kunnen toevoegen, omdat `Handle(pad, handler)` openbaar zou zijn.

**Denk bij het embedden van typen goed na over de impact die dit heeft op je openbare API.**

Het is een _veel_ gemaakte fout om embedding verkeerd te gebruiken, waardoor je je API's vervuilt en de interne werking van je type blootlegt.

Nu we onze applicatie hebben geherstructureerd, kunnen we eenvoudig nieuwe routes toevoegen en het begin van het `/league`-eindpunt vastleggen. We moeten er nu voor zorgen dat het nuttige informatie retourneert.

We moeten een JSON-bestand moeten retourneren dat er ongeveer zo uitziet.

```json
[
   {
      "Name":"Bill",
      "Wins":10
   },
   {
      "Name":"Alice",
      "Wins":15
   }
]
```

## Schrijf eerst de test

We beginnen met het proberen om het antwoord om te zetten in iets zinvols.

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := NewPlayerServer(&store)

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

### Waarom test je de JSON-string niet?

Je zou kunnen stellen dat een eenvoudigere eerste stap zou zijn om gewoon te beweren dat de responsbody een specifieke JSON-string bevat.

In mijn ervaring hebben tests die beweringen doen tegen JSON-strings de volgende problemen.

- *Het is broos*. Als je het datamodel wijzigt, mislukken je tests.
- *Moeilijk te debuggen*. Het kan lastig zijn om te begrijpen wat het werkelijke probleem is bij het vergelijken van twee JSON-strings.
- *Slechte bedoeling*. Hoewel de uitvoer JSON zou moeten zijn, is het vooral belangrijk wat de data precies is, en niet hoe deze is gecodeerd.
- *De standaardbibliotheek opnieuw testen*. Het is niet nodig om te testen hoe de standaardbibliotheek JSON uitvoert, deze is al getest. Test niet de code van anderen.

In plaats daarvan zouden we de JSON moeten parsen in datastructuren die relevant zijn om mee te testen.

### Datamodellering

Gezien het JSON-datamodel lijkt het erop dat we een `Player`-array met enkele velden nodig hebben. Daarom hebben we een nieuw type gemaakt om dit vast te leggen.

```go
//server.go
type Player struct {
	Name string
	Wins int
}
```

### JSON decoding

```go
//server_test.go
var got []Player
err := json.NewDecoder(response.Body).Decode(&got)
```

Om JSON in ons datamodel te parsen, maken we een `Decoder` aan vanuit het `encoding/json`-pakket en roepen we vervolgens de `Decode`-methode aan. Om een `Decoder` te maken, is een `io.Reader` nodig om te lezen, wat in ons geval de `Body` van onze respons is.

`Decode` neemt het adres van het object dat we proberen te decoderen, daarom declareren we een lege slice van `Player` in de regel ervoor.

Het parsen van JSON kan mislukken, dus `Decode` kan een `error` retourneren. Het heeft geen zin om de test voort te zetten als dat mislukt, dus controleren we op de fout en stoppen we de test met `t.Fatalf` als die optreedt. Merk op dat we de responsbody samen met de fout afdrukken, omdat het belangrijk is voor iemand die de test uitvoert om te zien welke string niet kan worden geparsed.

## Probeer de test uit te voeren

```
=== RUN   TestLeague/it_returns_200_on_/league
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:107: Unable to parse response from server '' into slice of Player, 'unexpected end of JSON input'
```

Ons eindpunt retourneert momenteel geen body, dus deze kan niet in JSON worden geparsed.

## Schrijf voldoende code om het te laten slagen

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	leagueTable := []Player{
		{"Chris", 20},
	}

	json.NewEncoder(w).Encode(leagueTable)

	w.WriteHeader(http.StatusOK)
}
```

De test slaagd nu.

### Coderen en decoderen

Let op de mooie symmetrie in de standaardbibliotheek.

- Om een `Encoder` te maken, heb je een `io.Writer` nodig, wat `http.ResponseWriter` implementeert.
- Om een `Decoder` te maken, heb je een `io.Reader` nodig, wat het `Body`-veld van onze antwoord-spy implementeert.

In dit boek hebben we `io.Writer` gebruikt en dit is wederom een demonstratie van de prevalentie ervan in de standaardbibliotheek en hoe veel bibliotheken er gemakkelijk mee werken.

## Refactor

Het zou mooi zijn om een scheiding van taken te introduceren tussen onze handler en het ophalen van de `leagueTable`, aangezien we weten dat we dat binnenkort niet meer hard hoeven te coderen.

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.getLeagueTable())
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) getLeagueTable() []Player {
	return []Player{
		{"Chris", 20},
	}
}
```

Vervolgens willen we onze test uitbreiden, zodat we precies kunnen bepalen welke gegevens we terug willen.

## Schrijf eerst de test

We kunnen de test bijwerken om te bevestigen dat de ranglijst een aantal spelers bevat die we in onze opslag willen opnemen.

Werk `StubPlayerStore` bij zodat er een competitie kan worden opgeslagen, wat slechts een deel is van `Player`. We slaan onze verwachte gegevens daarin op.

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}
```

Werk vervolgens onze huidige test bij door een aantal spelers in de competitie-eigenschap van onze stub te plaatsen en te bevestigen dat ze van onze server worden teruggestuurd.

```go
//server_test.go
func TestLeague(t *testing.T) {

	t.Run("it returns the league table as JSON", func(t *testing.T) {
		wantedLeague := []Player{
			{"Cleo", 32},
			{"Chris", 20},
			{"Tiest", 14},
		}

		store := StubPlayerStore{nil, nil, wantedLeague}
		server := NewPlayerServer(&store)

		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)

		if !reflect.DeepEqual(got, wantedLeague) {
			t.Errorf("got %v want %v", got, wantedLeague)
		}
	})
}
```

## Probeer de test uit te voeren

```
./server_test.go:33:3: too few values in struct initializer
./server_test.go:70:3: too few values in struct initializer
```

## Schrijf de minimale hoeveelheid code voor de test om uit te voeren en controleer de uitvoer van de mislukte test.

Je moet de andere tests bijwerken, aangezien we een nieuw veld hebben in `StubPlayerStore`; stel dit in op nul voor de andere tests.

Probeer de tests opnieuw uit te voeren en je zou het volgende moeten krijgen.

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: got [{Chris 20}] want [{Cleo 32} {Chris 20} {Tiest 14}]
```

## Schrijf voldoende code om het te laten slagen

We weten dat de gegevens in onze `StubPlayerStore` staan en we hebben die afgeleid naar een interface `PlayerStore`. We moeten dit bijwerken, zodat iedereen die ons in een `PlayerStore` doorgeeft, ons de gegevens voor de competities kan verstrekken.

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}
```

Nu kunnen we onze handlercode bijwerken om dit aan te roepen in plaats van een hardgecodeerde lijst te retourneren. Verwijder onze methode `getLeagueTable()` en werk vervolgens `leagueHandler` bij om `GetLeague()` aan te roepen.

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.store.GetLeague())
	w.WriteHeader(http.StatusOK)
}
```

## Probeer de testen uit te voeren.

```
# github.com/quii/learn-go-with-tests/json-and-io/v4
./main.go:9:50: cannot use NewInMemoryPlayerStore() (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_integration_test.go:11:27: cannot use store (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:36:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:74:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:106:29: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
```

De compiler klaagt omdat `InMemoryPlayerStore` en `StubPlayerStore` niet de nieuwe methode hebben die we aan onze interface hebben toegevoegd.

Voor `StubPlayerStore` is het vrij eenvoudig: retourneer gewoon het veld `league` dat we eerder hebben toegevoegd.

```go
//server_test.go
func (s *StubPlayerStore) GetLeague() []Player {
	return s.league
}
```

Hierbij een herinnering over hoe `InMemoryStore` is geïmplementeerd.


```go
//in_memory_player_store.go
type InMemoryPlayerStore struct {
	store map[string]int
}
```

Hoewel het vrij eenvoudig zou zijn om `GetLeague` "correct" te implementeren door over de map te itereren, onthoud dat we gewoon proberen _zo min mogelijk code te schrijven om de tests te laten slagen_.

Dus laten we de compiler voorlopig tevreden stellen en leven met het ongemakkelijke gevoel van een onvolledige implementatie in onze `InMemoryStore`.

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	return nil
}
```

Wat dit ons eigenlijk vertelt, is dat we dit _later_ willen testen, maar laten we dat even laten rusten.

Probeer de tests uit te voeren, de compiler zou moeten slagen en de tests zouden moeten slagen!

## Refactor

De testcode brengt onze bedoeling niet goed over en bevat veel boilerplate-elementen die we kunnen wegrefactoren.

```go
//server_test.go
t.Run("it returns the league table as JSON", func(t *testing.T) {
	wantedLeague := []Player{
		{"Cleo", 32},
		{"Chris", 20},
		{"Tiest", 14},
	}

	store := StubPlayerStore{nil, nil, wantedLeague}
	server := NewPlayerServer(&store)

	request := newLeagueRequest()
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := getLeagueFromResponse(t, response.Body)
	assertStatus(t, response.Code, http.StatusOK)
	assertLeague(t, got, wantedLeague)
})
```

Hier zijn de nieuwe helpers

```go
//server_test.go
func getLeagueFromResponse(t testing.TB, body io.Reader) (league []Player) {
	t.Helper()
	err := json.NewDecoder(body).Decode(&league)

	if err != nil {
		t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", body, err)
	}

	return
}

func assertLeague(t testing.TB, got, want []Player) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}

func newLeagueRequest() *http.Request {
	req, _ := http.NewRequest(http.MethodGet, "/league", nil)
	return req
}
```

Een laatste ding dat we moeten doen om onze server te laten werken, is ervoor zorgen dat we een `content-type`-header in het antwoord retourneren, zodat machines kunnen herkennen dat we `JSON` retourneren.

## Schrijf eerst de test

Voeg deze bewering toe aan de bestaande test


```go
//server_test.go
if response.Result().Header.Get("content-type") != "application/json" {
	t.Errorf("response did not have content-type of application/json, got %v", response.Result().Header)
}
```

## Probeer de test uit te voeren

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: response did not have content-type of application/json, got map[Content-Type:[text/plain; charset=utf-8]]
```

## Schrijf voldoende code om het te laten slagen

Werk `leagueHandler` bij

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", "application/json")
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

De test zou moeten slagen.

## Refactor

Maak een constante voor "application/json" en gebruik deze in `leagueHandler`

```go
//server.go
const jsonContentType = "application/json"

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

Voeg vervolgens een helper toe voor `assertContentType`.

```go
//server_test.go
func assertContentType(t testing.TB, response *httptest.ResponseRecorder, want string) {
	t.Helper()
	if response.Result().Header.Get("content-type") != want {
		t.Errorf("response did not have content-type of %s, got %v", want, response.Result().Header)
	}
}
```

Gebruik het in de test.

```go
//server_test.go
assertContentType(t, response, jsonContentType)
```

Nu we `PlayerServer` hebben opgelost, kunnen we ons richten op `InMemoryPlayerStore`. Als we dit nu aan de producteigenaar zouden proberen te demonstreren, werkt `/league` niet.

De snelste manier om wat vertrouwen te krijgen, is door onze integratietest uit te breiden. We kunnen het nieuwe eindpunt testen en controleren of we de juiste respons van `/league` terugkrijgen.

## Schrijf eerst de test

We kunnen `t.Run` gebruiken om deze test wat op te splitsen en we kunnen de helpers van onze servertests hergebruiken, wat opnieuw het belang van het refactoren van tests aantoont.

```go
//server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := NewInMemoryPlayerStore()
	server := NewPlayerServer(store)
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	t.Run("get score", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newGetScoreRequest(player))
		assertStatus(t, response.Code, http.StatusOK)

		assertResponseBody(t, response.Body.String(), "3")
	})

	t.Run("get league", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newLeagueRequest())
		assertStatus(t, response.Code, http.StatusOK)

		got := getLeagueFromResponse(t, response.Body)
		want := []Player{
			{"Pepper", 3},
		}
		assertLeague(t, got, want)
	})
}
```

## Probeer de test uit te voeren

```
=== RUN   TestRecordingWinsAndRetrievingThem/get_league
    --- FAIL: TestRecordingWinsAndRetrievingThem/get_league (0.00s)
        server_integration_test.go:35: got [] want [{Pepper 3}]
```

## Schrijf voldoende code om het te laten slagen

`InMemoryPlayerStore` retourneert `nil` wanneer je `GetLeague()` aanroept, dus dat moeten we oplossen.

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
}
```

Het enige wat we hoeven te doen is over de map te itereren en elke sleutel/waarde om te zetten naar een `Player`.

De test zou nu moeten slagen.

## Samenvattend

We zijn doorgegaan met het veilig itereren op ons programma met behulp van TDD, waardoor het nieuwe eindpunten op een onderhoudbare manier met een router ondersteunt en nu JSON voor onze gebruikers kan retourneren. In het volgende hoofdstuk zullen we het persistent maken van de data en het sorteren van onze league behandelen.

Wat we hebben behandeld:

- **Routing**. De standaardbibliotheek biedt een gebruiksvriendelijk type voor routing. Het omarmt de `http.Handler`-interface volledig, doordat je routes toewijst aan `Handlers` en de router zelf ook een `Handler` is. Het mist echter enkele functies die je misschien zou verwachten, zoals padvariabelen (bijv. `/users/{id}`). Je kunt deze informatie eenvoudig zelf parsen, maar je kunt overwegen om naar andere routingbibliotheken te kijken als het een last wordt. De meeste populaire bibliotheken houden vast aan de filosofie van de standaardbibliotheek om ook `http.Handler` te implementeren.
- **Type-embedding**. We hebben deze techniek al even kort aangestipt, maar je kunt er [meer over leren op Effective Go](https://golang.org/doc/effective_go.html#embedding). Eén ding dat je hieruit moet onthouden, is dat het enorm nuttig kan zijn, maar _denk altijd na over je publieke API en stel alleen beschikbaar wat relevant is_.
- **JSON-deserialisatie en serialisatie**. De standaardbibliotheek maakt het serialiseren en deserialiseren van je data heel eenvoudig. Het is ook open voor configuratie en je kunt de werking van deze datatransformaties indien nodig aanpassen.

# HTTP Server

**[Je kunt alle code voor dit hoofdstuk hier vinden](https://github.com/quii/learn-go-with-tests/tree/main/http-server)**

Je bent gevraagd een webserver te maken waar gebruikers kunnen bijhouden hoeveel games spelers hebben gewonnen.

- `GET /players/{name}` moet een getal retourneren dat het totale aantal overwinningen aangeeft.
- `POST /players/{name}` moet een overwinning voor die naam registreren, oplopend bij elke volgende `POST`.

We volgen de TDD-aanpak: zo snel mogelijk werkende software krijgen en gaan vervolgens kleine iteratieve verbeteringen aanbrengen totdat we de oplossing hebben. Door deze aanpak te volgen:

- Houden we het probleemgebied te allen tijde klein.
- Vermijden we valkuilen.
- Mochten we ooit vastlopen/verdwalen, dan gaat er door versiebeheer geen werk verloren.

## Rood, groen, refactor

In dit boek hebben we de nadruk gelegd op het TDD-proces: schrijf een test en zie hem falen (rood), schrijf de _minimale_ hoeveelheid code om hem te laten werken (groen) en refactor vervolgens.

Deze discipline van het schrijven van de minimale hoeveelheid code is belangrijk vanwege de veiligheid die TDD biedt. Je moet ernaar streven om zo snel mogelijk uit de "rode" situatie te komen.

Kent Beck beschrijft het als volgt:

> Zorg dat de test snel werkt en bega daarbij alle nodige fouten.

Je kunt deze fouten begaan, omdat je daarna refactored, ondersteund door de veiligheid van de tests.

### Wat als je dit niet doet?

Hoe meer wijzigingen je aanbrengt terwijl je in het rood staat, hoe groter de kans dat je meer problemen toevoegt die niet door tests worden gedekt.

Het idee is om iteratief nuttige code te schrijven met kleine stapjes, aangestuurd door tests, zodat je niet urenlang na uren werk in een valkuil stapt.

### Kip en ei

Hoe kunnen we dit incrementeel bouwen? We kunnen geen speler ophalen met `GET` zonder iets opgeslagen te hebben en het lijkt moeilijk om te weten of `POST` heeft gewerkt zonder dat het `GET`-eindpunt al bestaat.

Dit is waar _mocking_ schittert.

- `GET` heeft een `PlayerStore`-_ding_ nodig om scores voor een speler op te halen. Dit zou een interface moeten zijn, zodat we tijdens het testen een eenvoudige stub kunnen maken om onze code te testen zonder dat we daadwerkelijke opslagcode hoeven te implementeren.
- Voor `POST` kunnen we de aanroepen van `PlayerStore` _bespioneren_ om er zeker van te zijn dat spelers correct worden opgeslagen. Onze implementatie van opslaan zal niet gekoppeld zijn aan ophalen.
- Om snel werkende software te hebben, kunnen we een zeer eenvoudige in-memory implementatie maken en later een implementatie creëren die wordt ondersteund door het opslagmechanisme dat we verkiezen.

## Schrijf eerst de test

We kunnen een test schrijven en deze laten slagen door een hardgecodeerde waarde te retourneren om ons op weg te helpen. Kent Beck noemt dit "faken". Zodra we een werkende test hebben, kunnen we meer tests schrijven om die constante te verwijderen.

Door deze zeer kleine stap te zetten, kunnen we de belangrijke start maken om een algehele projectstructuur correct te laten werken, zonder ons al te veel zorgen te hoeven maken over onze applicatielogica.

Om een webserver in Go te maken, roep je doorgaans [ListenAndServe](https://golang.org/pkg/net/http/#ListenAndServe) aan.

```go
func ListenAndServe(addr string, handler Handler) error
```

Hiermee start je een webserver die op een poort luistert, voor elke aanvraag een goroutine aanmaakt en deze uitvoert op een [`Handler`](https://golang.org/pkg/net/http/#Handler).

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

Een type implementeert de Handler-interface door de `ServeHTTP`-methode te implementeren, die twee argumenten verwacht: het eerste is waar we _ons antwoord schrijven_ en het tweede is het HTTP-verzoek dat naar de server is verzonden.

Laten we een bestand met de naam `server_test.go` aanmaken en een test schrijven voor een functie `PlayerServer` die deze twee argumenten accepteert. Het verzonden verzoek is bedoeld om de score van een speler op te vragen, waarvan we verwachten dat deze `"20"` is.

```go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		got := response.Body.String()
		want := "20"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

Om onze server te testen, hebben we een `Request` nodig om te versturen en willen we met _spy_ in de gaten houden wat onze handler naar de `ResponseWriter` schrijft.

- We gebruiken `http.NewRequest` om een request aan te maken. Het eerste argument is de methode van de request en het tweede is het pad van de request. Het `nil`-argument verwijst naar de body van de request, die we in dit geval niet hoeven in te stellen.
- `net/http/httptest` heeft een spy die al voor ons is aangemaakt: `ResponseRecorder`, dus die kunnen we gebruiken. Deze heeft veel handige methoden om te controleren wat er als antwoord is geschreven.

## Probeer de test uit te voeren

`./server_test.go:13:2: undefined: PlayerServer`

## Schrijf de minimale hoeveelheid code voor de test om uit te voeren en controleer de mislukte testuitvoer.

De compiler staat klaar om te helpen, luister daar naar.

Maak een bestand met de naam `server.go` en definieer `PlayerServer`

```go
func PlayerServer() {}
```

Probeer opnieuw

```
./server_test.go:13:14: too many arguments in call to PlayerServer
    have (*httptest.ResponseRecorder, *http.Request)
    want ()
```

Voeg de argumenten toe aan de functie

```go
import "net/http"

func PlayerServer(w http.ResponseWriter, r *http.Request) {

}
```

De code compileert nu en de test mislukt

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- FAIL: TestGETPlayers/returns_Pepper's_score (0.00s)
        server_test.go:20: got '', want '20'
```

## Schrijf voldoende code om het te laten slagen

In het DI-hoofdstuk hebben we HTTP-servers met een `Greet`-functie besproken. We hebben geleerd dat de `ResponseWriter` van net/http ook de `Writer` van io implementeert, zodat we `fmt.Fprint` kunnen gebruiken om strings als HTTP-reacties te versturen.

```go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "20")
}
```

De test zou nu moeten slagen.

## Voltooi de voorbereiding

We willen dit in een applicatie integreren. Dit is belangrijk omdat

- We _werkelijke werkende software_ hebben, we willen geen tests schrijven zonder doel, het is goed om de code in actie te zien.
- Naarmate we onze code refactoren, is het waarschijnlijk dat we de structuur van het programma zullen wijzigen. We willen ervoor zorgen dat dit ook in onze applicatie tot uiting komt als onderdeel van de incrementele aanpak.

Maak een nieuw `main.go`-bestand voor onze applicatie en plaats daar deze code in

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(PlayerServer)
	log.Fatal(http.ListenAndServe(":5000", handler))
}
```

Tot nu toe staat al onze applicatiecode in één bestand. Dit is echter niet de beste manier voor grotere projecten waarbij je dingen in verschillende bestanden wilt splitsen.

Om dit uit te voeren, voer je `go build` uit. Dit programma neemt alle `.go`-bestanden in de directory en bouwt een programma voor je. Je kunt het vervolgens uitvoeren met `./myprogram`.

### `http.HandlerFunc`

Eerder hebben we besproken dat de `Handler`-interface is wat we moeten implementeren om een server te maken. _Normaal gesproken_ doen we dat door een `struct` te creëren en deze de interface te laten implementeren door zijn eigen ServeHTTP-methode te implementeren. De use-case voor structs is echter om data te bewaren, maar _momenteel_ hebben we geen status, dus het voelt niet goed om er een te creëren.

[HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc) laat ons dit vermijden.

> Het type HandlerFunc is een adapter die het gebruik van gewone functies als HTTP-handlers mogelijk maakt. Als f een functie is met de juiste signatuur, is HandlerFunc(f) een Handler die f aanroept.

```go
type HandlerFunc func(ResponseWriter, *Request)
```

Uit de documentatie blijkt dat het type `HandlerFunc` de `ServeHTTP`-methode al heeft geïmplementeerd.
Door onze `PlayerServer`-functie ermee te typecasten, hebben we nu de vereiste `Handler` geïmplementeerd.

### `http.ListenAndServe(":5000"...)`

`ListenAndServe` accepteert een poort om te luisteren op een `Handler`. Als er een probleem is, retourneert de webserver een foutmelding. Een voorbeeld hiervan is de poort waarnaar al wordt geluisterd. Daarom verpakken we de aanroep in `log.Fatal` om de fout aan de gebruiker te loggen.

Wat we nu gaan doen, is een _andere_ test schrijven om onszelf te dwingen een positieve wijziging aan te brengen om te proberen af te stappen van de hardgecodeerde waarde.

## Schrijf eerst de test

We voegen nog een subtest toe aan onze suite die de score van een andere speler probeert te bepalen, wat onze hardgecodeerde aanpak zal verstoren.

```go
t.Run("returns Floyd's score", func(t *testing.T) {
	request, _ := http.NewRequest(http.MethodGet, "/players/Floyd", nil)
	response := httptest.NewRecorder()

	PlayerServer(response, request)

	got := response.Body.String()
	want := "10"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

Je hebt misschien gedacht:

> We hebben toch wel een soort opslagconcept nodig om te bepalen welke speler welke score krijgt? Het is vreemd dat de waarden in onze tests zo willekeurig lijken.

Bedenk dat we gewoon proberen zo klein mogelijke stappen te zetten, dus we proberen voorlopig alleen de constante te doorbreken.

## Probeer de test uit te voeren

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- PASS: TestGETPlayers/returns_Pepper's_score (0.00s)
=== RUN   TestGETPlayers/returns_Floyd's_score
    --- FAIL: TestGETPlayers/returns_Floyd's_score (0.00s)
        server_test.go:34: got '20', want '10'
```

## Schrijf genoeg code om de test te laten slagen

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	if player == "Pepper" {
		fmt.Fprint(w, "20")
		return
	}

	if player == "Floyd" {
		fmt.Fprint(w, "10")
		return
	}
}
```

Deze test dwong ons om daadwerkelijk naar de URL van het verzoek te kijken en een beslissing te nemen. Dus hoewel we ons in gedachten misschien zorgen maakten over de store en interfaces van de speler, lijkt de volgende logische stap eigenlijk te gaan over _routing_.

Als we met de store-code waren begonnen, zouden we in vergelijking hiermee veel meer wijzigingen moeten doorvoeren. **Dit is een kleinere stap richting ons uiteindelijke doel en is gebaseerd op tests**.

We weerstaan nu de verleiding om routingbibliotheken te gebruiken, alleen de kleinste stap om onze test te laten slagen.

`r.URL.Path` retourneert het pad van het verzoek, waarmee we vervolgens [`strings.TrimPrefix`](https://golang.org/pkg/strings/#TrimPrefix) kunnen gebruiken om `/players/` weg te halen en de gevraagde speler te krijgen. Het is niet erg robuust, maar het voldoet voorlopig.

## Refactor

We kunnen de `PlayerServer` vereenvoudigen door het ophalen van de score op te splitsen in een functie

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	fmt.Fprint(w, GetPlayerScore(player))
}

func GetPlayerScore(name string) string {
	if name == "Pepper" {
		return "20"
	}

	if name == "Floyd" {
		return "10"
	}

	return ""
}
```

En we kunnen een deel van de code in de tests _DRY_ maken door een aantal helpers te maken

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

Toch zouden we niet blij moeten zijn. Het voelt niet goed dat onze server de scores kent.

Onze refactoring heeft het vrij duidelijk gemaakt wat we moeten doen.

We hebben de scoreberekening uit de hoofdtekst van onze handler verplaatst naar de functie `GetPlayerScore`. Dit voelt als de juiste plek om de problemen te scheiden met behulp van interfaces.

Laten we de functie die we hebben gerefactored, verplaatsen naar een interface.

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
}
```

Om onze `PlayerServer` een `PlayerStore` te laten gebruiken, heeft deze een verwijzing naar een `PlayerStore` nodig. Dit voelt als het juiste moment om onze architectuur aan te passen, zodat onze `PlayerServer` nu een `struct` is.

```go
type PlayerServer struct {
	store PlayerStore
}
```

Ten slotte implementeren we de `Handler`-interface door een methode toe te voegen aan onze nieuwe struct en onze bestaande handlercode te plaatsen.

```go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

De enige andere verandering is dat we nu onze `store.GetPlayerScore` aanroepen om de score op te halen, in plaats van de lokale functie die we hebben gedefinieerd (die we nu kunnen verwijderen).

Hier is de volledige code van onze server.

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

### Los de problemen op

Dit waren nogal wat wijzigingen en we weten dat onze tests en applicatie niet meer zullen compileren, maar ontspan en laat de compiler het werk doen.

`./main.go:9:58: type PlayerServer is not an expression`

We moeten onze tests aanpassen om in plaats daarvan een nieuwe instantie van onze `PlayerServer` te maken en vervolgens de methode `ServeHTTP` aan te roepen.

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	server := &PlayerServer{}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

Merk op dat we ons nog niet druk maken over het maken van stores (nog niet), we willen alleen dat de compiler zo snel mogelijk reageert.

Je zou er een gewoonte van moeten maken om prioriteit te geven aan code die compileert en vervolgens code die de tests doorstaat.

Door meer functionaliteit toe te voegen (zoals stub stores) terwijl de code niet compileert, stellen we onszelf bloot aan mogelijk _meer_ compilatieproblemen.

Nu compileert `main.go` om dezelfde reden niet.

```go
func main() {
	server := &PlayerServer{}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Eindelijk wordt alles gecompileerd, maar de tests mislukken


```
=== RUN   TestGETPlayers/returns_the_Pepper's_score
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
    panic: runtime error: invalid memory address or nil pointer dereference
```

Dit komt omdat we in onze tests geen `PlayerStore` hebben ingevoerd. We moeten er een stub van maken.

```go
//server_test.go
type StubPlayerStore struct {
	scores map[string]int
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}
```

Een `map` is een snelle en gemakkelijke manier om een stub-sleutel/waarde-opslag voor onze tests te maken. Laten we nu een van deze opslagplaatsen voor onze tests maken en deze naar onze `PlayerServer` sturen.

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

Onze tests zijn nu geslaagd en zien er beter uit. De bedoeling van onze code is nu duidelijker dankzij de introductie van de store. We vertellen de lezer dat, omdat we deze gegevens in een `PlayerStore` hebben staan, je de volgende reacties zou moeten krijgen wanneer je ze gebruikt met een `PlayerServer`.

### Voer de applicatie uit

Nu onze tests succesvol zijn, moeten we als laatste nog controleren of onze applicatie werkt om deze refactoring te voltooien. Het programma zou moeten opstarten, maar je krijgt een vreselijke reactie als je probeert de server te bereiken via `http://localhost:5000/players/Pepper`.

De reden hiervoor is dat we geen `PlayerStore` hebben ingevoerd.

We moeten er een implementeren, maar dat is momenteel lastig omdat we geen relevante gegevens opslaan, dus die moeten we voorlopig hardcoderen.

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return 123
}

func main() {
	server := &PlayerServer{&InMemoryPlayerStore{}}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Als je `go build` opnieuw uitvoert en dezelfde URL gebruikt, zou je `"123"` moeten krijgen. Niet geweldig, maar totdat we gegevens hebben opgeslagen, is dit het beste wat we kunnen doen.
Het voelde ook niet geweldig dat onze hoofdapplicatie wel opstartte, maar niet echt werkte. We moesten handmatig testen om het probleem te achterhalen.

We hebben een paar opties voor de volgende stap:

- Het scenario afhandelen waarin de speler niet bestaat
- Het scenario `POST /players/{name}` afhandelen

Hoewel het `POST`-scenario ons dichter bij het "happy path" brengt, denk ik dat het makkelijker zal zijn om eerst het scenario met de ontbrekende speler aan te pakken, omdat we ons al in die context bevinden. We komen later op de rest.

## Schrijf eerst de test

Voeg een ontbrekend spelerscenario toe aan onze bestaande suite

```go
//server_test.go
t.Run("returns 404 on missing players", func(t *testing.T) {
	request := newGetScoreRequest("Apollo")
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := response.Code
	want := http.StatusNotFound

	if got != want {
		t.Errorf("got status %d want %d", got, want)
	}
})
```

## Probeer de test uit te voeren

```
=== RUN   TestGETPlayers/returns_404_on_missing_players
    --- FAIL: TestGETPlayers/returns_404_on_missing_players (0.00s)
        server_test.go:56: got status 200 want 404
```

## Schrijf genoeg code om het te laten slagen

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	w.WriteHeader(http.StatusNotFound)

	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

Soms rol ik echt met mijn ogen als voorstanders van TDD zeggen: "Zorg ervoor dat je alleen de minimale hoeveelheid code schrijft om het te laten slagen", want dat kan nogal overdreven perfectionistisch overkomen.

Maar dit scenario illustreert het voorbeeld goed. Ik heb het absolute minimum gedaan (wetende dat het niet correct is), namelijk een `StatusNotFound` schrijven voor **alle reacties**, maar al onze tests slagen!

**Door het absolute minimum te doen om de tests te laten slagen, kunnen hiaten in je tests zichtbaar worden**. In ons geval beweren we niet dat we een `StatusOK` zouden moeten krijgen wanneer spelers _wel_ in de winkel staan.

Werk de andere twee tests bij om de status te bevestigen en de code te corrigeren.

Hier zijn de nieuwe tests

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "10")
	})

	t.Run("returns 404 on missing players", func(t *testing.T) {
		request := newGetScoreRequest("Apollo")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusNotFound)
	})
}

func assertStatus(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("did not get correct status, got %d, want %d", got, want)
	}
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

We controleren nu de status in al onze tests, dus ik heb een helper aangemaakt: `assertStatus` om dat te vergemakkelijken.

Onze eerste twee tests mislukken nu vanwege de 404 in plaats van 200, dus we kunnen `PlayerServer` zo instellen dat het alleen 'niet gevonden' retourneert als de score 0 is.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

### Scores opslaan

Nu we scores uit een opslag kunnen halen, is het logisch om nieuwe scores op te slaan.

## Schrijf eerst de test

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it returns accepted on POST", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodPost, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)
	})
}
```

Laten we om te beginnen controleren of we de juiste statuscode krijgen als we de specifieke route met POST bereiken. Dit stelt ons in staat om een ander type verzoek te accepteren en anders af te handelen dan `GET /players/{name}`. Zodra dit werkt, kunnen we beginnen met het claimen van de interactie van onze handler met de opslag.

## Probeer de test uit te voeren

```
=== RUN   TestStoreWins/it_returns_accepted_on_POST
    --- FAIL: TestStoreWins/it_returns_accepted_on_POST (0.00s)
        server_test.go:70: did not get correct status, got 404, want 202
```

## Schrijf voldoende code om het te laten slagen

Onthoud dat we opzettelijk zonden begaan, dus een `if`-statement gebaseerd op de methode van het verzoek is voldoende.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	if r.Method == http.MethodPost {
		w.WriteHeader(http.StatusAccepted)
		return
	}

	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

## Refactor

De handler ziet er nu wat rommelig uit. Laten we de code opsplitsen om het makkelijker te volgen te maken en de verschillende functionaliteiten isoleren in nieuwe functies.

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	switch r.Method {
	case http.MethodPost:
		p.processWin(w)
	case http.MethodGet:
		p.showScore(w, r)
	}

}

func (p *PlayerServer) showScore(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter) {
	w.WriteHeader(http.StatusAccepted)
}
```

Dit maakt het routingaspect van `ServeHTTP` iets duidelijker en betekent dat onze volgende iteraties voor opslag gewoon binnen `processWin` kunnen plaatsvinden.

Vervolgens willen we controleren of onze `PlayerStore` de opdracht krijgt om de winst te registreren wanneer we onze `POST /players/{name}` uitvoeren.

## Schrijf eerst de test

We kunnen dit bereiken door onze `StubPlayerStore` uit te breiden met een nieuwe `RecordWin`-methode en vervolgens de aanroepen ervan te bespioneren.

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}
```

Breid nu onze test uit om het aantal aanroepen voor een start te controleren

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it records wins when POST", func(t *testing.T) {
		request := newPostWinRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Errorf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}
	})
}

func newPostWinRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodPost, fmt.Sprintf("/players/%s", name), nil)
	return req
}
```

## Probeer de test uit te voeren

```
./server_test.go:26:20: too few values in struct initializer
./server_test.go:65:20: too few values in struct initializer
```

## Schrijf de minimale hoeveelheid code om de test uit te voeren en controleer de mislukte testuitvoer.

We moeten onze code bijwerken waar we een `StubPlayerStore` aanmaken, omdat we een nieuw veld hebben toegevoegd.

```go
//server_test.go
store := StubPlayerStore{
	map[string]int{},
	nil,
}
```

```
--- FAIL: TestStoreWins (0.00s)
    --- FAIL: TestStoreWins/it_records_wins_when_POST (0.00s)
        server_test.go:80: got 0 calls to RecordWin want 1
```

## Schrijf voldoende code om het te laten slagen

Omdat we alleen het aantal aanroepen aangeven in plaats van de specifieke waarden, wordt onze initiële iteratie iets kleiner.

We moeten het idee van `PlayerServer` over wat een `PlayerStore` is bijwerken door de interface aan te passen, zodat we `RecordWin` kunnen aanroepen.

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}
```

Door dit te doen compileert `main` niet meer

```
./main.go:17:46: cannot use InMemoryPlayerStore literal (type *InMemoryPlayerStore) as type PlayerStore in field value:
    *InMemoryPlayerStore does not implement PlayerStore (missing RecordWin method)
```

De compiler vertelt ons wat er mis is. Laten we `InMemoryPlayerStore` updaten zodat deze methode beschikbaar is.

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) RecordWin(name string) {}
```

Probeer de tests uit te voeren en we zouden weer code moeten kunnen compileren, maar de test mislukt nog steeds.

Nu `PlayerStore` een `RecordWin` heeft, kunnen we het binnen onze `PlayerServer` aanroepen.

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter) {
	p.store.RecordWin("Bob")
	w.WriteHeader(http.StatusAccepted)
}
```

Voer de tests uit en het zou moeten lukken! `"Bob"` is duidelijk niet precies wat we naar `RecordWin` willen sturen, dus laten we de test verder verfijnen.

## Schrijf eerst de test

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
		nil,
	}
	server := &PlayerServer{&store}

	t.Run("it records wins on POST", func(t *testing.T) {
		player := "Pepper"

		request := newPostWinRequest(player)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}

		if store.winCalls[0] != player {
			t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], player)
		}
	})
}
```

Nu we weten dat er één element in onze `winCalls`-slice zit, kunnen we veilig naar het eerste element verwijzen en controleren of het gelijk is aan `player`.

## Probeer de test uit te voeren

```
=== RUN   TestStoreWins/it_records_wins_on_POST
    --- FAIL: TestStoreWins/it_records_wins_on_POST (0.00s)
        server_test.go:86: did not store correct winner got 'Bob' want 'Pepper'
```

## Schrijf genoeg code om het te laten slagen

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

We hebben `processWin` aangepast om `http.Request` te gebruiken, zodat we de URL kunnen bekijken om de naam van de speler te extraheren. Zodra we die hebben, kunnen we onze `store` aanroepen met de juiste waarde om de test te laten slagen.

## Refactor

We kunnen deze code wat opschonen met DRY principes, aangezien we de naam van de speler op twee plaatsen op dezelfde manier extraheren.

```go
//server.go
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

Hoewel onze tests succesvol zijn, hebben we nog geen werkende software. Als je `main` probeert te draaien en de software gebruikt zoals bedoeld, werkt het niet omdat we er nog niet aan toegekomen zijn om `PlayerStore` correct te implementeren. Dit is echter prima; door ons te concentreren op onze handler hebben we de interface geïdentificeerd die we nodig hebben, in plaats van te proberen deze van tevoren te ontwerpen.

We _zouden_ kunnen beginnen met het schrijven van tests rond onze `InMemoryPlayerStore`, maar die is er slechts tijdelijk totdat we een robuustere manier implementeren om spelersscores te bewaren (d.w.z. een database).

Wat we nu gaan doen, is een _integratietest_ schrijven tussen onze `PlayerServer` en `InMemoryPlayerStore` om de functionaliteit af te ronden. Dit stelt ons in staat om ons doel te bereiken: er zeker van zijn dat onze applicatie werkt, zonder `InMemoryPlayerStore` direct te hoeven testen. Sterker nog, zodra we `PlayerStore` met een database implementeren, kunnen we die implementatie testen met dezelfde integratietest.

### Integratietests

Integratietests kunnen nuttig zijn om te testen of grotere delen van je systeem werken, maar je moet rekening houden met het volgende:

- Ze zijn moeilijker te schrijven.
- Als ze mislukken, kan het moeilijk zijn om te achterhalen waarom (meestal is het een bug in een component van de integratietest) en dus moeilijker te verhelpen.
- Ze zijn soms trager in uitvoering (omdat ze vaak worden gebruikt met "echte" componenten, zoals een database).

Daarom is het raadzaam om _De Testpiramide_ te onderzoeken.

## Schrijf eerst de test.

Om het kort te houden, laat ik je de definitieve refactored integratietest zien.

```go
// server_integration_test.go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := InMemoryPlayerStore{}
	server := PlayerServer{&store}
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	response := httptest.NewRecorder()
	server.ServeHTTP(response, newGetScoreRequest(player))
	assertStatus(t, response.Code, http.StatusOK)

	assertResponseBody(t, response.Body.String(), "3")
}
```

- We zijn bezig met het creëren van de twee componenten waarmee we proberen te integreren: `InMemoryPlayerStore` en `PlayerServer`.
- Vervolgens versturen we 3 verzoeken om 3 overwinningen voor `player` te registreren. We maken ons in deze test niet al te veel zorgen over de statuscodes, omdat die niet relevant zijn voor de goede integratie.
- Het volgende antwoord vinden we wel belangrijk (daarom slaan we een variabele `response` op), omdat we de score van `player` willen achterhalen.

## Probeer de test uit te voeren

```
--- FAIL: TestRecordingWinsAndRetrievingThem (0.00s)
    server_integration_test.go:24: response body is wrong, got '123' want '3'
```

## Schrijf genoeg code om het te laten slagen

Ik neem hier wat vrijheden en schrijf meer code dan je misschien prettig vindt zonder een test te schrijven.

_Dit is toegestaan!_ We hebben nog steeds een test die controleert of alles correct zou moeten werken, maar deze test is niet gericht op de specifieke unit waarmee we werken (`InMemoryPlayerStore`).

Mocht ik in dit scenario vastlopen, dan zou ik mijn wijzigingen terugdraaien naar de mislukte test en vervolgens specifiekere unittests schrijven rond `InMemoryPlayerStore` om me te helpen een oplossing te vinden.

```go
//in_memory_player_store.go
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

- We moeten de gegevens opslaan, dus heb ik een `map[string]int` toegevoegd aan de `InMemoryPlayerStore`-structuur.
- Voor het gemak heb ik `NewInMemoryPlayerStore` gemaakt om de opslag te initialiseren en de integratietest bijgewerkt om deze te gebruiken:
    ```go
    //server_integration_test.go
    store := NewInMemoryPlayerStore()
    server := PlayerServer{store}
    ```
- De rest van de code draait gewoon om de `map`.

De integratietest is geslaagd, nu hoeven we alleen nog `main` te wijzigen om `NewInMemoryPlayerStore()` te gebruiken.


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

Bouw het, voer het uit en gebruik vervolgens `curl` om het te testen.

- Voer dit een paar keer uit, verander eventueel de spelersnamen `curl -X POST http://localhost:5000/players/Pepper`
- Controleer de scores met `curl http://localhost:5000/players/Pepper`

Geweldig! Je hebt een REST-achtige service gemaakt. Om dit verder te ontwikkelen, kun je een dataopslag kiezen die de scores langer bewaart dan de looptijd van het programma.

- Kies een opslag (Bolt? Mongo? Postgres? Bestandssysteem?)
- Laat `PostgresPlayerStore` `PlayerStore` implementeren (of welke opslag je ook gekozen hebt)
- TDD de functionaliteit zodat je zeker weet dat het werkt
- Sluit het aan op de integratietest en controleer of het nog steeds werkt
- Sluit het ten slotte aan op `main`

## Refactor

We zijn er bijna! Laten we ons inspannen om gelijktijdigheidsfouten zoals deze te voorkomen.

```
fatal error: concurrent map read and map write
```

Door mutexen toe te voegen, zorgen we voor gelijktijdigheidsbeveiliging, met name voor de teller in onze `RecordWin`-functie. Lees meer over mutexen in het hoofdstuk over synchronisatie.

## Samenvattend

### `http.Handler`

- Implementeer deze interface om webservers te maken
- Gebruik `http.HandlerFunc` om gewone functies om te zetten in `http.Handler`s
- Gebruik `httptest.NewRecorder` om in te voeren als een `ResponseWriter`, zodat je de reacties die je handler verstuurt kunt bespioneren
- Gebruik `http.NewRequest` om de verzoeken te construeren die je verwacht in je systeem

### Interfaces, Mocking en DI

- Hiermee kun je het systeem iteratief in kleinere delen opbouwen
- Hiermee kun je een handler ontwikkelen die opslagruimte nodig heeft zonder daadwerkelijke opslagruimte
- TDD om de benodigde interfaces aan te sturen

### Zonden committen, vervolgens refactoren (en vervolgens committen naar bronbeheer)

- Je moet falende compilatie of falende tests behandelen als een rode situatie waar je zo snel mogelijk uit moet komen.
- Schrijf alleen de code die nodig is om dit te bereiken. _Refactor_ vervolgens en maak de code netjes.
- Door te veel wijzigingen door te voeren terwijl de code niet compileert of de tests mislukken, loop je het risico de problemen te verergeren.
- Door deze aanpak te volgen, dwing je jezelf om kleine tests te schrijven, wat kleine wijzigingen betekent, wat helpt om het werken aan complexe systemen beheersbaar te houden.

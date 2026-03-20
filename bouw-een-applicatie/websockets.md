# WebSockets

**[Alle code voor dit hoofdstuk vindt je hier](https://github.com/quii/learn-go-with-tests/tree/main/websockets)**

In dit hoofdstuk leren we hoe we WebSockets kunnen gebruiken om onze applicatie te verbeteren.

## Projectsamenvatting

We hebben twee applicaties in onze pokercodebase.

- *Command line app*. Vraagt de gebruiker om het aantal spelers in een spel in te voeren. Vanaf dat moment worden de spelers geïnformeerd over de waarde van de "blind bet", die in de loop van de tijd toeneemt. Op elk moment kan een gebruiker `"{Playername} wins"` invoeren om het spel te voltooien en de winnaar in een store te registreren.
- *Web app*. Stelt gebruikers in staat om winnaars van spellen te registreren en toont een ranglijst. Deelt dezelfde store als de command line app.

## Volgende stappen

De product owner is enthousiast over de command line app, maar zou het beter vinden als we die functionaliteit naar de browser zouden brengen. Ze stelt zich een webpagina voor met een tekstvak waarmee de gebruiker het aantal spelers kan invoeren. Wanneer de gebruiker het formulier verzendt, toont de pagina de blinde waarde en werkt deze automatisch bij wanneer dat nodig is. Net als bij de commandline-applicatie kan de gebruiker de winnaar aankondigen en deze wordt opgeslagen in de database.

Op het eerste gezicht klinkt het vrij eenvoudig, maar zoals altijd moeten we de nadruk leggen op een iteratieve aanpak bij het schrijven van software.

Eerst moeten we HTML aanbieden. Tot nu toe hebben al onze HTTP-eindpunten platte tekst of JSON geretourneerd. We zouden dezelfde technieken kunnen gebruiken die we kennen (aangezien het uiteindelijk allemaal strings zijn), maar we kunnen ook het pakket [html/template](https://golang.org/pkg/html/template/) gebruiken voor een schonere oplossing.

We moeten ook asynchroon berichten naar de gebruiker kunnen sturen met de tekst `The blind is now *y*` zonder de browser te hoeven vernieuwen. We kunnen [WebSockets](https://en.wikipedia.org/wiki/WebSocket) gebruiken om dit te vergemakkelijken.

> WebSocket is een computercommunicatieprotocol dat full-duplex communicatiekanalen biedt via één TCP-verbinding.

Aangezien we verschillende technieken gebruiken, is het des te belangrijker dat we dit, zoals altijd, aanpakken met verschillende _iteraties_.

Daarom maken we eerst een webpagina met een formulier waarmee de gebruiker een winnaar kan registreren. In plaats van een standaardformulier te gebruiken, gebruiken we WebSockets om die gegevens naar onze server te sturen, zodat deze kunnen worden geregistreerd.

Daarna werken we aan de blinde meldingen, waarna we een stukje infrastructuurcode hebben opgezet.

### Hoe zit het met tests voor de JavaScript?

Er zal wat JavaScript geschreven worden om dit te doen, maar ik ga niet in op het schrijven van tests.

Het is natuurlijk mogelijk, maar om het kort te houden zal ik er geen uitleg over geven.

Sorry mensen. Vraag O'Reilly om me te betalen voor het maken van een "Leer JavaScript met tests".

## Schrijf eerst de test

Het eerste wat we moeten doen, is wat HTML aan gebruikers aanbieden wanneer ze op `/game` klikken.

Hier is een herinnering aan de relevante code op onze webserver

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

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

Het _eenvoudigste_ wat we nu kunnen doen is controleren of we bij `GET /game` een `200` krijgen.


```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request, _ := http.NewRequest(http.MethodGet, "/game", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

## Probeer de test uit te voeren
```
--- FAIL: TestGame (0.00s)
=== RUN   TestGame/GET_/game_returns_200
    --- FAIL: TestGame/GET_/game_returns_200 (0.00s)
    	server_test.go:109: did not get correct status, got 404, want 200
```

## Schrijf voldoende code om de test te laten slagen

Onze server is voorzien van een router, dus het is relatief eenvoudig op te lossen.

Voeg toe aan onze router

```go
router.Handle("/game", http.HandlerFunc(p.game))
```

Schrijf vervolgens de `game` methode

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}
```

## Refactor

De servercode is al prima, omdat we heel gemakkelijk meer code in de bestaande, goed gefactoriseerde code kunnen inpassen.

We kunnen de test wat opschonen door een testhulpfunctie toe te voegen: `newGameRequest`, die de aanvraag naar `/game` stuurt. Probeer dit zelf maar eens te schrijven.

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request := newGameRequest()
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response, http.StatusOK)
	})
}
```

Je zult ook merken dat ik `assertStatus` heb aangepast om `response` te accepteren in plaats van `response.Code`, omdat ik vind dat dat beter leest.

Nu moeten we het eindpunt wat HTML laten retourneren, hier is de HTML code.

```html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Let's play poker</title>
</head>
<body>
<section id="game">
    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>
</section>
</body>
<script type="application/javascript">

    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    if (window['WebSocket']) {
        const conn = new WebSocket('ws://' + document.location.host + '/ws')

        submitWinnerButton.onclick = event => {
            conn.send(winnerInput.value)
        }
    }
</script>
</html>
```

We hebben een heel eenvoudige webpagina

- Een tekstinvoer waarmee de gebruiker de winnaar kan invoeren
- Een knop waarop de gebruiker kan klikken om de winnaar bekend te maken.
- Wat JavaScript om een WebSocket-verbinding met onze server te openen en het indrukken van de verzendknop te verwerken

`WebSocket` is ingebouwd in de meeste moderne browsers, dus we hoeven ons geen zorgen te maken over het toevoegen van bibliotheken. De webpagina werkt niet in oudere browsers, maar voor dit scenario is dat geen probleem.

### Hoe testen we of we de juiste markup retourneren?

Er zijn verschillende manieren. Zoals in het boek al benadrukt, is het belangrijk dat de tests die je schrijft voldoende waarde hebben om de kosten te rechtvaardigen.

1. Schrijf een browsergebaseerde test met bijvoorbeeld Selenium. Deze tests zijn de meest "realistische" van alle benaderingen, omdat ze een echte webbrowser starten en een gebruiker simuleren die ermee communiceert. Deze tests kunnen je veel vertrouwen geven dat je systeem werkt, maar zijn moeilijker te schrijven dan unittests en veel langzamer om uit te voeren. Voor ons product is dit overdreven.
2. Voer een exacte stringmatch uit. Dit _kan_ wel goed zijn, maar dit soort tests zijn uiteindelijk erg kwetsbaar. Zodra iemand de markup wijzigt, mislukt een test, terwijl er in de praktijk niets _echts kapot_ is.
3. Controleer of we de juiste template aanroepen. We gebruiken een templatebibliotheek uit de standaardbibliotheek om de HTML te serveren (hierover later meer). We kunnen _thing_ injecteren om de HTML te genereren en de aanroep ervan te controleren om te controleren of we het goed doen. Dit zou een impact hebben op het ontwerp van onze code, maar test niet veel; behalve dat we het aanroepen met het juiste templatebestand. Aangezien we maar één template in ons project zullen hebben, lijkt de kans op falen hier klein.

Dus in het boek "Leer Go met Tests" gaan we voor het eerst geen test schrijven.

Zet de markup in een bestand met de naam `game.html`.

Vervolgens wijzigen we het eindpunt dat we zojuist hebben geschreven naar het volgende:

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	tmpl, err := template.ParseFiles("game.html")

	if err != nil {
		http.Error(w, fmt.Sprintf("problem loading template %s", err.Error()), http.StatusInternalServerError)
		return
	}

	tmpl.Execute(w, nil)
}
```

[`html/template`](https://golang.org/pkg/html/template/) is een Go-pakket voor het maken van HTML. In ons geval roepen we `template.ParseFiles` aan, met het pad naar ons HTML-bestand. Ervan uitgaande dat er geen fout is, kun je de template uitvoeren, die deze naar een `io.Writer` schrijft. In ons geval willen we dat deze naar het internet schrijft, dus geven we hem onze `http.ResponseWriter`.

Omdat we geen test hebben geschreven, is het verstandig om onze webserver handmatig te testen om er zeker van te zijn dat alles werkt zoals we hopen. Ga naar `cmd/webserver` en voer het bestand `main.go` uit. Ga naar `http://localhost:5000/game`.

Je _zou_ een foutmelding moeten krijgen dat je de template niet kon vinden. Je kunt het pad relatief maken ten opzichte van je map, of je kunt een kopie van 'game.html' in de map 'cmd/webserver' plaatsen. Ik heb ervoor gekozen om een symbolische link ('ln -s ../../game.html game.html') te maken naar het bestand in de root van het project, zodat eventuele wijzigingen worden doorgevoerd wanneer de server draait.

Als je deze wijziging aanbrengt en de server opnieuw start, zou je onze gebruikersinterface moeten zien.

Nu moeten we testen of we een string die via een WebSocket-verbinding met onze server wordt verzonden, kunnen uitroepen tot winnaar van een spel.

## Schrijf eerst de test

Voor het eerst gaan we een externe bibliotheek gebruiken om met WebSockets te kunnen werken.

Voer `go get github.com/gorilla/websocket` uit.

Hiermee wordt de code voor de uitstekende [Gorilla WebSocket](https://github.com/gorilla/websocket)-bibliotheek opgehaald. Nu kunnen we onze tests bijwerken voor onze nieuwe vereiste.

```go
t.Run("when we get a message over a websocket it is a winner of a game", func(t *testing.T) {
	store := &StubPlayerStore{}
	winner := "Ruth"
	server := httptest.NewServer(NewPlayerServer(store))
	defer server.Close()

	wsURL := "ws" + strings.TrimPrefix(server.URL, "http") + "/ws"

	ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", wsURL, err)
	}
	defer ws.Close()

	if err := ws.WriteMessage(websocket.TextMessage, []byte(winner)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}

	AssertPlayerWin(t, store, winner)
})
```

Zorg ervoor dat je een import hebt voor de `websocket`-bibliotheek. Mijn IDE deed dit automatisch voor me, dat zou die van jou dus ook moeten kunnen.

Om te testen wat er vanuit de browser gebeurt, moeten we onze eigen WebSocket-verbinding openen en ernaar schrijven.

Onze eerdere tests op onze server riepen alleen methoden aan op onze server, maar nu hebben we een permanente verbinding met onze server nodig. Hiervoor gebruiken we `httptest.NewServer`, die een `http.Handler` gebruikt en deze opstart en luistert naar verbindingen.

Met `websocket.DefaultDialer.Dial` proberen we in te bellen op onze server en vervolgens proberen we een bericht te versturen met onze `winnaar`.

Tot slot bevestigen we de speleropslag om te controleren of de winnaar is geregistreerd.

## Probeer de test uit te voeren
```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:124: could not open a ws connection on ws://127.0.0.1:55838/ws websocket: bad handshake
```

We hebben onze server nog niet aangepast om WebSocket-verbindingen op `/ws` te accepteren, dus we schudden elkaar nog niet de hand.

## Schrijf voldoende code om het te laten werken

Voeg nog een vermelding toe aan onze router

```go
router.Handle("/ws", http.HandlerFunc(p.webSocket))
```

Voeg vervolgens onze nieuwe `webSocket`-handler toe

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	upgrader.Upgrade(w, r, nil)
}
```

Om een WebSocket-verbinding te accepteren, `upgraden` we het verzoek. Als je de test nu opnieuw uitvoert, krijg je de volgende fout.

```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:132: got 0 calls to RecordWin want 1
```

Nu de verbinding is geopend, willen we luisteren naar een bericht en dit vervolgens vastleggen als winnaar.

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	conn, _ := upgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

(Ja, we negeren momenteel veel fouten!)

`conn.ReadMessage()` blokkeert het wachten op een bericht op de verbinding. Zodra we er een ontvangen, gebruiken we het om `RecordWin` te gebruiken. Dit zou de WebSocket-verbinding definitief sluiten.

Als je de test probeert uit te voeren, mislukt deze nog steeds.

Het probleem zit in de timing. Er zit een vertraging tussen het moment waarop onze WebSocket-verbinding het bericht leest en de win vastlegt, en onze test is voltooid voordat dit gebeurt. Je kunt dit testen door een korte `time.Sleep` voor de laatste bewering te plaatsen.

Laten we dat voorlopig zo houden, maar we erkennen dat het gebruiken van willekeurige slaapstanden in tests **een zeer slechte gewoonte** is.

```go
time.Sleep(10 * time.Millisecond)
AssertPlayerWin(t, store, winner)
```

## Refactor

We hebben veel fouten gemaakt om deze test te laten werken, zowel in de servercode als in de testcode, maar onthoud dat dit voor ons de makkelijkste manier is om te werken.

We hebben vervelende, vreselijke, _werkende_ software die wordt ondersteund door een test, dus nu kunnen we het goed maken en weten we dat we niets per ongeluk kapotmaken.

Laten we beginnen met de servercode.

We kunnen de `upgrader` naar een privéwaarde in ons pakket verplaatsen, omdat we deze niet bij elke WebSocket-verbindingsaanvraag opnieuw hoeven te declareren.

```go
var wsUpgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

Onze aanroep van `template.ParseFiles("game.html")` wordt uitgevoerd bij elke `GET /game`, wat betekent dat we bij elke aanvraag naar het bestandssysteem gaan, ook al hoeven we de template niet opnieuw te parsen. Laten we onze code zo aanpassen dat we de template één keer parsen in `NewPlayerServer`. We moeten ervoor zorgen dat deze functie nu een foutmelding kan retourneren als er problemen zijn met het ophalen van de template van schijf of het parsen ervan.

Hier zijn de relevante wijzigingen aan `PlayerServer`

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
}

const htmlTemplatePath = "game.html"

func NewPlayerServer(store PlayerStore) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.template = tmpl
	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))
	router.Handle("/game", http.HandlerFunc(p.game))
	router.Handle("/ws", http.HandlerFunc(p.webSocket))

	p.Handler = router

	return p, nil
}

func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	p.template.Execute(w, nil)
}
```

Door de signatuur van `NewPlayerServer` te wijzigen, hebben we nu compilatieproblemen. Probeer ze zelf op te lossen of raadpleeg de broncode als je er moeite mee hebt.

Voor de testcode heb ik een helper gemaakt genaamd `mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer`, zodat ik de foutmeldingen kon verbergen voor de tests.

```go
func mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer {
	server, err := NewPlayerServer(store)
	if err != nil {
		t.Fatal("problem creating player server", err)
	}
	return server
}
```

Op dezelfde manier heb ik nog een helper aangemaakt: `mustDialWS`, zodat ik vervelende foutmeldingen kon verbergen bij het aanmaken van de WebSocket-verbinding.

```go
func mustDialWS(t *testing.T, url string) *websocket.Conn {
	ws, _, err := websocket.DefaultDialer.Dial(url, nil)

	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", url, err)
	}

	return ws
}
```

Ten slotte kunnen we in onze testcode een helper creëren om het verzenden van berichten op te ruimen

```go
func writeWSMessage(t testing.TB, conn *websocket.Conn, message string) {
	t.Helper()
	if err := conn.WriteMessage(websocket.TextMessage, []byte(message)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}
}
```

Nu de tests succesvol zijn, probeer de server te starten en een aantal winnaars te benoemen in `/game`. Je zou ze moeten zien in `/league`. Onthoud dat we elke keer dat we een winnaar hebben de verbinding _sluiten_. Je moet de pagina vernieuwen om de verbinding weer te openen.

We hebben een eenvoudig webformulier gemaakt waarmee gebruikers de winnaar van een spel kunnen registreren. Laten we dit itereren zodat de gebruiker een spel kan starten door een aantal spelers op te geven. De server zal dan berichten naar de client sturen met informatie over de blindwaarde naarmate de tijd verstrijkt.

Werk eerst `game.html` bij om onze client-side code bij te werken aan de nieuwe vereisten.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Lets play poker</title>
</head>
<body>
<section id="game">
    <div id="game-start">
        <label for="player-count">Number of players</label>
        <input type="number" id="player-count"/>
        <button id="start-game">Start</button>
    </div>

    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>

    <div id="blind-value"/>
</section>

<section id="game-end">
    <h1>Another great game of poker everyone!</h1>
    <p><a href="/league">Go check the league table</a></p>
</section>

</body>
<script type="application/javascript">
    const startGame = document.getElementById('game-start')

    const declareWinner = document.getElementById('declare-winner')
    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    const blindContainer = document.getElementById('blind-value')

    const gameContainer = document.getElementById('game')
    const gameEndContainer = document.getElementById('game-end')

    declareWinner.hidden = true
    gameEndContainer.hidden = true

    document.getElementById('start-game').addEventListener('click', event => {
        startGame.hidden = true
        declareWinner.hidden = false

        const numberOfPlayers = document.getElementById('player-count').value

        if (window['WebSocket']) {
            const conn = new WebSocket('ws://' + document.location.host + '/ws')

            submitWinnerButton.onclick = event => {
                conn.send(winnerInput.value)
                gameEndContainer.hidden = false
                gameContainer.hidden = true
            }

            conn.onclose = evt => {
                blindContainer.innerText = 'Connection closed'
            }

            conn.onmessage = evt => {
                blindContainer.innerText = evt.data
            }

            conn.onopen = function () {
                conn.send(numberOfPlayers)
            }
        }
    })
</script>
</html>
```

De belangrijkste veranderingen zijn de toevoeging van een sectie om het aantal spelers in te voeren en een sectie om de blindwaarde weer te geven. We hebben een beetje logica om de gebruikersinterface te tonen/verbergen, afhankelijk van de fase van het spel.

Elk bericht dat we ontvangen via `conn.onmessage`, nemen we aan als blinde meldingen en daarom stellen we de `blindContainer.innerText` dienovereenkomstig in.

Hoe versturen we de blinde meldingen? In het vorige hoofdstuk introduceerden we het idee van `Game`, zodat onze CLI-code een `Game` kon aanroepen en al het andere zou worden geregeld, inclusief het plannen van blinde meldingen. Dit bleek een goede scheiding van aandachtspunten te zijn.

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

Wanneer de gebruiker in de CLI werd gevraagd om het aantal spelers, werd het spel `Start`, wat de blinde waarschuwingen activeerde. Wanneer de gebruiker de winnaar bekendmaakte, werd het spel `Finish`. Dit zijn dezelfde vereisten als nu, alleen een andere manier om de invoer te verkrijgen; we zouden dit concept dus moeten hergebruiken als dat mogelijk is.

Onze "echte" implementatie van `Game` is `TexasHoldem`.

```go
type TexasHoldem struct {
	alerter BlindAlerter
	store   PlayerStore
}
```

Door een `BlindAlerter` in te sturen kan `TexasHoldem` blinde waarschuwingen plannen die naar _waar dan ook_ worden verzonden

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

En ter herinnering, hier is onze implementatie van de `BlindAlerter` die we gebruiken in de CLI.

```go
func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

Dit werkt in de CLI omdat we de waarschuwingen altijd naar `os.Stdout` willen sturen, maar dit werkt niet voor onze webserver. Voor elke aanvraag ontvangen we een nieuwe `http.ResponseWriter`, die we vervolgens upgraden naar `*websocket.Conn`. We kunnen dus niet weten waar onze waarschuwingen naartoe moeten tijdens het samenstellen van onze afhankelijkheden.

Daarom moeten we `BlindAlerter.ScheduleAlertAt` aanpassen, zodat deze een bestemming voor de waarschuwingen krijgt, zodat we deze opnieuw kunnen gebruiken op onze webserver.

Open `blind_alerter.go` en voeg de parameter toe aan `io.Writer`

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int, to io.Writer)
}

type BlindAlerterFunc func(duration time.Duration, amount int, to io.Writer)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int, to io.Writer) {
	a(duration, amount, to)
}
```

Het idee van een `StdoutAlerter` past niet in ons nieuwe model, dus hernoemen we het gewoon naar `Alerter`

```go
func Alerter(duration time.Duration, amount int, to io.Writer) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(to, "Blind is now %d\n", amount)
	})
}
```

Als je probeert te compileren, mislukt het in `TexasHoldem` omdat het `ScheduleAlertAt` aanroept zonder bestemming. Om de boel _voorlopig_ weer te laten compileren, hardcodeer je het naar `os.Stdout`.

Probeer de tests uit te voeren en ze mislukken omdat `SpyBlindAlerter` `BlindAlerter` niet meer implementeert. Los dit op door de handtekening van `ScheduleAlertAt` bij te werken. Voer de tests uit en we zouden nog steeds groen moeten zijn.

Het heeft geen zin dat `TexasHoldem` weet waar blinde waarschuwingen naartoe moeten worden gestuurd. Laten we nu `Game` bijwerken, zodat je bij het starten van een spel aangeeft _waar_ de waarschuwingen naartoe moeten.

```go
type Game interface {
	Start(numberOfPlayers int, alertsDestination io.Writer)
	Finish(winner string)
}
```

Laat de compiler je vertellen wat je moet aanpassen. De verandering is niet zo ingewikkeld:

- Werk `TexasHoldem` bij zodat `Game` correct wordt geïmplementeerd.
- Geef in `CLI`, wanneer we het spel starten, onze `out`-eigenschap (`cli.game.Start(numberOfPlayers, cli.out)`) mee.
- In de test van `TexasHoldem` gebruik ik `game.Start(5, io.Discard)` om het compilatieprobleem op te lossen en de waarschuwingsuitvoer zo in te stellen dat deze wordt verwijderd.

Als je alles goed hebt, zou alles groen moeten zijn! Nu kunnen we `Game` binnen `Server` proberen te gebruiken.

## Schrijf eerst de test

De vereisten van `CLI` en `Server` zijn hetzelfde! Alleen het aflevermechanisme is anders.

Laten we ter inspiratie eens kijken naar onze `CLI`-test.

```go
t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
	game := &GameSpy{}

	out := &bytes.Buffer{}
	in := userSends("3", "Chris wins")

	poker.NewCLI(in, out, game).PlayPoker()

	assertMessagesSentToUser(t, out, poker.PlayerPrompt)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, "Chris")
})
```

Het lijkt erop dat we een vergelijkbaar resultaat kunnen testen met `GameSpy`.

Vervang de oude websocket-test door het volgende:

```go
t.Run("start a game with 3 players and declare Ruth the winner", func(t *testing.T) {
	game := &poker.GameSpy{}
	winner := "Ruth"
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
})
```

- Zoals besproken maken we een spion `Game` aan en geven deze door aan `mustMakePlayerServer` (zorg ervoor dat de helper dit ondersteunt).
- Vervolgens sturen we de websocketberichten voor een game.
- Ten slotte bevestigen we dat de game is gestart en voltooid met wat we verwachten.

## Probeer de test uit te voeren

Je zult een aantal compilatiefouten rond `mustMakePlayerServer` tegenkomen in andere tests. Introduceer een niet-geëxporteerde variabele `dummyGame` en gebruik deze in alle tests die niet gecompileerd worden.


```go
var (
	dummyGame = &GameSpy{}
)
```

De laatste fout is wanneer we proberen `Game` door te geven aan `NewPlayerServer`, maar deze ondersteunt dit nog niet

```
./server_test.go:21:38: too many arguments in call to "github.com/quii/learn-go-with-tests/WebSockets/v2".NewPlayerServer
	have ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore, "github.com/quii/learn-go-with-tests/WebSockets/v2".Game)
	want ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore)
```

## Schrijf de minimale hoeveelheid code om de test uit te voeren en controleer de mislukte testuitvoer.

Voeg het voorlopig gewoon als argument toe om de test uit te voeren.

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error)
```

Eindelijk!

```
=== RUN   TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.01s)
    --- FAIL: TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner (0.01s)
    	server_test.go:146: wanted Start called with 3 but got 0
    	server_test.go:147: expected finish called with 'Ruth' but got ''
FAIL
```

## Schrijf voldoende code om het te laten slagen

We moeten `Game` als veld toevoegen aan `PlayerServer`, zodat het gebruikt kan worden wanneer het verzoeken ontvangt.

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
	game     Game
}
```

(We hebben al een methode genaamd `game`, dus hernoem die naar `playGame`)

Laten we deze vervolgens toewijzen in onze constructor

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.game = game

	// etc
}
```

Hoe kunnen we onze `Game` gebruiken binnen `webSocket`.

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)

	_, numberOfPlayersMsg, _ := conn.ReadMessage()
	numberOfPlayers, _ := strconv.Atoi(string(numberOfPlayersMsg))
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	_, winner, _ := conn.ReadMessage()
	p.game.Finish(string(winner))
}
```

Hoera! De tests zijn geslaagd.

We gaan de blinde berichten _nog_ nergens naartoe sturen, want daar moeten we nog even over nadenken. Wanneer we `game.Start` aanroepen, sturen we `io.Discard` mee, die alle berichten die erop geschreven worden gewoon weggooit.

Start nu de webserver op. Je moet `main.go` bijwerken om een `Game` door te geven aan de `PlayerServer`

```go
func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.Alerter), store)

	server, err := poker.NewPlayerServer(store, game)

	if err != nil {
		log.Fatalf("problem creating player server %v", err)
	}

	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Afgezien van het feit dat we nog geen blinde meldingen ontvangen, werkt de app wel! We zijn erin geslaagd om `Game` te hergebruiken met `PlayerServer` en alles is tot in de puntjes geregeld. Zodra we erachter komen hoe we onze blinde meldingen naar de websockets kunnen sturen in plaats van ze te verwijderen, _zou_ alles moeten werken.

Maar laten we eerst wat code opschonen.

## Refactor

De manier waarop we WebSockets gebruiken is vrij eenvoudig en de foutafhandeling is vrij naïef, dus ik wilde dat in een type vastleggen om die rommel uit de servercode te halen. We willen er later misschien nog op terugkomen, maar voor nu ruimt dit de boel wat op.

```go
type playerServerWS struct {
	*websocket.Conn
}

func newPlayerServerWS(w http.ResponseWriter, r *http.Request) *playerServerWS {
	conn, err := wsUpgrader.Upgrade(w, r, nil)

	if err != nil {
		log.Printf("problem upgrading connection to WebSockets %v\n", err)
	}

	return &playerServerWS{conn}
}

func (w *playerServerWS) WaitForMsg() string {
	_, msg, err := w.ReadMessage()
	if err != nil {
		log.Printf("error reading from websocket %v\n", err)
	}
	return string(msg)
}
```

Nu is de servercode een beetje vereenvoudigd

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

Zodra we erachter komen hoe we de blinde berichten niet moeten weggooien, zijn we klaar.

### Laten we _geen_ test schrijven!

Soms, als we niet zeker weten hoe we iets moeten doen, is het het beste om gewoon wat te experimenteren en dingen uit te proberen! Zorg ervoor dat je werk eerst gecommit is, want zodra we een manier hebben gevonden, moeten we het door een test halen.

De problematische regel code die we hebben is

```go
p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!
```

We moeten een `io.Writer` doorgeven aan de game om de blind alerts naartoe te schrijven.

Zou het niet handig zijn als we onze `playerServerWS` van voorheen konden doorgeven? Het is onze wrapper rond onze WebSocket, dus het _voelt_ alsof we die naar onze `Game` zouden moeten kunnen sturen om berichten naar te sturen.

Probeer het eens:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)
	//etc...
}
```

De compiler klaagt

```
./server.go:71:14: cannot use ws (type *playerServerWS) as type io.Writer in argument to p.game.Start:
	*playerServerWS does not implement io.Writer (missing Write method)
```

Het lijkt voor de hand liggend om ervoor te zorgen dat `playerServerWS` `io.Writer` _implementeert_. Om dit te doen gebruiken we de onderliggende `*websocket.Conn` om `WriteMessage` te gebruiken om het bericht via de websocket te versturen.

```go
func (w *playerServerWS) Write(p []byte) (n int, err error) {
	err = w.WriteMessage(websocket.TextMessage, p)

	if err != nil {
		return 0, err
	}

	return len(p), nil
}
```

Dit lijkt te makkelijk! Probeer de applicatie te starten en kijk of het werkt.

Bewerk eerst `TexasHoldem` zodat de blinde incrementtijd korter is, zodat je het in actie kunt zien.

```go
blindIncrement := time.Duration(5+numberOfPlayers) * time.Second // (rather than a minute)
```

Je zou het moeten zien werken! De blinde hoeveelheid neemt als bij toverslag toe in de browser.

Laten we nu de code terugdraaien en bedenken hoe we die kunnen testen. Om het te _implementeren_ hebben we alleen `playerServerWS` doorgegeven aan `StartGame` in plaats van `io.Discard`, dus je zou kunnen denken dat we de aanroep moeten bespioneren om te controleren of deze werkt.

Spioneren is geweldig en helpt ons implementatiedetails te controleren, maar we moeten altijd proberen om het _echte_ gedrag te testen als dat mogelijk is, want wanneer je besluit te refactoren, zijn het vaak spionagetests die mislukken, omdat ze meestal implementatiedetails controleren die je probeert te wijzigen.

Onze test opent momenteel een websocketverbinding met onze actieve server en verstuurt berichten om deze te laten werken. Op dezelfde manier zouden we de berichten moeten kunnen testen die onze server via de websocketverbinding terugstuurt.

## Schrijf eerst de test

We gaan onze bestaande test bewerken.

Momenteel stuurt onze `GameSpy` geen gegevens naar `out` wanneer je `Start` aanroept. We moeten dit aanpassen zodat we het kunnen configureren om een standaardbericht te versturen en vervolgens kunnen controleren of dat bericht naar de websocket wordt verzonden. Dit zou ons de zekerheid moeten geven dat we alles correct hebben geconfigureerd en tegelijkertijd het gewenste gedrag vertonen.

```go
type GameSpy struct {
	StartCalled     bool
	StartCalledWith int
	BlindAlert      []byte

	FinishedCalled   bool
	FinishCalledWith string
}
```

Voeg het veld `BlindAlert` toe.

Werk `GameSpy` `Start` bij om het standaardbericht naar `out` te sturen.

```go
func (g *GameSpy) Start(numberOfPlayers int, out io.Writer) {
	g.StartCalled = true
	g.StartCalledWith = numberOfPlayers
	out.Write(g.BlindAlert)
}
```
Dit betekent nu dat wanneer we `PlayerServer` gebruiken om het spel te `Starten`, er berichten via de websocket moeten worden verzonden als alles goed werkt.

Ten slotte kunnen we de test bijwerken.

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)

	_, gotBlindAlert, _ := ws.ReadMessage()

	if string(gotBlindAlert) != wantedBlindAlert {
		t.Errorf("got blind alert %q, want %q", string(gotBlindAlert), wantedBlindAlert)
	}
})
```

- We hebben een `wantedBlindAlert` toegevoegd en onze `GameSpy` geconfigureerd om deze naar `out` te sturen als `Start` wordt aangeroepen.
- We hopen dat het via de websocketverbinding wordt verzonden, dus hebben we een aanroep naar `ws.ReadMessage()` toegevoegd om te wachten tot er een bericht is verzonden en vervolgens te controleren of het het verwachte bericht is.

## Probeer de test uit te voeren

Je zou moeten zien dat de test permanent vastloopt. Dit komt doordat `ws.ReadMessage()` blokkeert totdat er een bericht wordt ontvangen, wat nooit gebeurt.

## Schrijf de minimale hoeveelheid code voor de test om uit te voeren en controleer de mislukte testuitvoer.

We zouden nooit tests moeten hebben die vastlopen, dus laten we een manier introduceren om code af te handelen waarvan we een time-out willen.

```go
func within(t testing.TB, d time.Duration, assert func()) {
	t.Helper()

	done := make(chan struct{}, 1)

	go func() {
		assert()
		done <- struct{}{}
	}()

	select {
	case <-time.After(d):
		t.Error("timed out")
	case <-done:
	}
}
```

Wat `within` doet, is de functie `assert` als argument gebruiken en deze vervolgens in een go-routine uitvoeren. Als/wanneer de functie klaar is, geeft deze via het `done`-kanaal aan dat dit is voltooid.

Terwijl dat gebeurt, gebruiken we een `select`-statement waarmee we kunnen wachten tot een kanaal een bericht verzendt. Vanaf hier is het een race tussen de `assert`-functie en `time.After`, die een signaal verzendt wanneer de duur is verstreken.

Tot slot heb ik een hulpfunctie voor onze assertie gemaakt, om het wat overzichtelijker te maken.

```go
func assertWebsocketGotMsg(t *testing.T, ws *websocket.Conn, want string) {
	_, msg, _ := ws.ReadMessage()
	if string(msg) != want {
		t.Errorf(`got "%s", want "%s"`, string(msg), want)
	}
}
```

Dit is hoe de test er nu uitziet:

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(tenMS)

	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
	within(t, tenMS, func() { assertWebsocketGotMsg(t, ws, wantedBlindAlert) })
})
```

Als je de test nu uitvoert...

```
=== RUN   TestGame
=== RUN   TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.02s)
    --- FAIL: TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner (0.02s)
    	server_test.go:143: timed out
    	server_test.go:150: got "", want "Blind is 100"
```

## Schrijf voldoende code om het te laten slagen

Ten slotte kunnen we nu onze servercode aanpassen, zodat deze onze WebSocket-verbinding naar het spel stuurt wanneer het start.

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

## Refactor

De servercode was een heel kleine wijziging, dus er valt hier niet veel te veranderen, maar de testcode heeft nog steeds een `time.Sleep`-aanroep omdat we moeten wachten tot onze server zijn werk asynchroon uitvoert.

We kunnen onze helpers `assertGameStartedWith` en `assertFinishCalledWith` refactoren, zodat ze hun beweringen korte tijd opnieuw kunnen proberen voordat ze mislukken.

Hieronder lees je hoe je dit kunt doen voor `assertFinishCalledWith`. Je kunt dezelfde aanpak gebruiken voor de andere helper.

```go
func assertFinishCalledWith(t testing.TB, game *GameSpy, winner string) {
	t.Helper()

	passed := retryUntil(500*time.Millisecond, func() bool {
		return game.FinishCalledWith == winner
	})

	if !passed {
		t.Errorf("expected finish called with %q but got %q", winner, game.FinishCalledWith)
	}
}
```

Hier zie je hoe `retryUntil` is gedfinieerd

```go
func retryUntil(d time.Duration, f func() bool) bool {
	deadline := time.Now().Add(d)
	for time.Now().Before(deadline) {
		if f() {
			return true
		}
	}
	return false
}
```

## Samenvattend

Onze applicatie is nu compleet. Een pokerspel kan worden gestart via een webbrowser en gebruikers worden via WebSockets geïnformeerd over de waarde van de blinde inzet. Na afloop van het spel kunnen ze de winnaar vastleggen, die wordt bijgehouden met behulp van code die we een paar hoofdstukken geleden hebben geschreven. De spelers kunnen achterhalen wie de beste (of gelukkigste) pokerspeler is met behulp van het `/league`-eindpunt op de website.

Tijdens de reis hebben we fouten gemaakt, maar met de TDD-flow zijn we nooit ver verwijderd geweest van werkende software. We hadden de vrijheid om te blijven itereren en experimenteren.

Het laatste hoofdstuk blikt terug op de aanpak, het ontwerp waar we toe zijn gekomen en lost een aantal losse eindjes op.

We hebben in dit hoofdstuk een aantal dingen behandeld:

### WebSockets

- Een handige manier om berichten te versturen tussen clients en servers, waarbij de client de server niet hoeft te blijven pollen. Zowel de client- als de servercode die we hebben, is heel eenvoudig.
- Triviaal om te testen, maar je moet rekening houden met het asynchrone karakter van de tests.

### Code verwerken in tests die vertraagd kan zijn of nooit kan worden voltooid.

- Hulpfuncties creëren om beweringen opnieuw te proberen en time-outs toe te voegen.
- We kunnen go-routines gebruiken om ervoor te zorgen dat de beweringen niets blokkeren en vervolgens kanalen gebruiken om aan te geven of ze al dan niet voltooid zijn.
- Het `time`-pakket heeft een aantal handige functies die ook signalen via kanalen over gebeurtenissen in de tijd verzenden, zodat we time-outs kunnen instellen.

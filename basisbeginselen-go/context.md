# Context

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/context)

Software start vaak langlopende, resource-intensieve processen (vaak in goroutines). Als de actie die dit veroorzaakt, om een ​​of andere reden wordt geannuleerd of mislukt, moet je deze processen op een consistente manier via je applicatie stoppen.

Als je dit niet doet, kan het zijn dat je snelle Go-applicatie, waar je zo trots op bent, te maken krijgt met prestatieproblemen die lastig te debuggen zijn.

In dit hoofdstuk gebruiken we het pakket `context` om langlopende processen te beheren.

We beginnen met een klassiek voorbeeld van een webserver die, wanneer deze wordt aangeroepen, een mogelijk langdurig proces start om gegevens op te halen en in het antwoord te retourneren.

We voeren een scenario uit waarin een gebruiker het verzoek annuleert voordat de gegevens kunnen worden opgehaald. We zorgen ervoor dat het proces de opdracht krijgt om op te geven.

Ik heb wat code op het Happy Path gezet om ons op weg te helpen. Hier is onze servercode.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, store.Fetch())
	}
}
```

De functie `Server` gebruikt een `Store` en retourneert een `http.HandlerFunc`. Store is gedefinieerd als:

```go
type Store interface {
	Fetch() string
}
```

De geretourneerde functie roept de `Fetch`-methode van de `store` aan om de gegevens op te halen en schrijft deze naar het antwoord.

We hebben een overeenkomstige observator voor `Store` die we in een test gebruiken.

```go
type SpyStore struct {
	response string
}

func (s *SpyStore) Fetch() string {
	return s.response
}

func TestServer(t *testing.T) {
	data := "hello, world"
	svr := Server(&SpyStore{data})

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
}
```

Nu we een happy path hebben, willen we een realistischer scenario creëren waarin de `Store` een `Fetch` niet kan voltooien voordat de gebruiker de aanvraag annuleert.

## Schrijf eerst je test

Onze handler heeft een manier nodig om de `Store` te vertellen dat deze de werkzaamheden moet annuleren en dus de interface moet bijwerken.

```go
type Store interface {
	Fetch() string
	Cancel()
}
```

We moeten onze observator aanpassen, zodat het enige tijd duurt om gegevens te retourneren en een manier om te weten dat de opdracht is gegeven om te annuleren. We moeten `Cancel` toevoegen als methode om de `Store`-interface te implementeren.

```go
type SpyStore struct {
	response  string
	cancelled bool
}

func (s *SpyStore) Fetch() string {
	time.Sleep(100 * time.Millisecond)
	return s.response
}

func (s *SpyStore) Cancel() {
	s.cancelled = true
}
```

Laten we een nieuwe test toevoegen waarbij we het verzoek annuleren vóór 100 milliseconden en dan de store controleren om te zien of het verzoek wordt geannuleerd.

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if !store.cancelled {
		t.Error("store was not told to cancel")
	}
})
```

Tekst uit [Go Blog: Context](https://blog.golang.org/context)

> Het context pakket biedt functies om nieuwe context waarden af ​​te leiden uit bestaande waarden. Deze waarden vormen een boomstructuur: wanneer een context wordt geannuleerd, worden alle contexten die ervan zijn afgeleid ook geannuleerd.

Het is belangrijk dat je je contexten afleidt, zodat annuleringen worden verspreid over de hele aanroepstack voor een bepaalde aanvraag.

Wat we doen is een nieuwe `cancellingCtx` afleiden uit onze aanvraag die ons een `cancel`functie retourneert. Vervolgens plannen we dat die functie over 5 milliseconden wordt aangeroepen met behulp van `time.AfterFunc`. Ten slotte gebruiken we deze nieuwe context in onze aanvraag door `request.WithContext` aan te roepen.

## Probeer de test uit te voeren

De test faalt, zoals verwacht

```
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.00s)
    	context_test.go:62: store was not told to cancel
```

## Schrijf genoeg code om de test te laten slagen

Vergeet niet om gedisciplineerd te zijn met TDD. Schrijf de minimale hoeveelheid code om onze test te laten slagen.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		store.Cancel()
		fmt.Fprint(w, store.Fetch())
	}
}
```

Daarmee is deze test geslaagd, maar het voelt niet goed, toch? We zouden `Cancel()` toch niet moeten annuleren voordat we _elke aanvraag_ hebben opgehaald?

Door de discipline is er een fout in onze tests aan het licht gekomen, en dat is positief!

We moeten onze 'happy path'-test bijwerken om te garanderen dat deze niet wordt geannuleerd.

```go
t.Run("returns data from store", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}

	if store.cancelled {
		t.Error("it should not have cancelled the store")
	}
})
```

Voer beide tests uit. De 'happy path'-test zou nu moeten mislukken en we zijn nu gedwongen om een ​​verstandigere implementatie te doen.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		data := make(chan string, 1)

		go func() {
			data <- store.Fetch()
		}()

		select {
		case d := <-data:
			fmt.Fprint(w, d)
		case <-ctx.Done():
			store.Cancel()
		}
	}
}
```

Wat hebben we hier gedaan?

`context` heeft een methode `Done()` die een kanaal retourneert dat een signaal ontvangt wanneer de context "klaar" of "geannuleerd" is. We willen naar dat signaal luisteren en `Store.Cancel` aanroepen zodra we het signaal ontvangen, maar we willen het negeren als onze `Store` erin slaagt om eerder  `Fetch` uit te voeren.

Om dit te regelen, draaien we `Fetch` in een goroutine en schrijven we het resultaat naar een nieuw kanaal. Vervolgens gebruiken we `select` om effectief naar de twee asynchrone processen te racen en schrijven we een respons of roepen we `Cancel` aan.

## Refactor

We kunnen onze testcode een beetje refactoren door asser methoden op onze observator te maken

```go
type SpyStore struct {
	response  string
	cancelled bool
	t         *testing.T
}

func (s *SpyStore) assertWasCancelled() {
	s.t.Helper()
	if !s.cancelled {
		s.t.Error("store was not told to cancel")
	}
}

func (s *SpyStore) assertWasNotCancelled() {
	s.t.Helper()
	if s.cancelled {
		s.t.Error("store was told to cancel")
	}
}
```

Vergeet niet om `*testing.T` mee te nemen bij het aanmaken van de spion.

```go
func TestServer(t *testing.T) {
	data := "hello, world"

	t.Run("returns data from store", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)
		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		if response.Body.String() != data {
			t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
		}

		store.assertWasNotCancelled()
	})

	t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)

		cancellingCtx, cancel := context.WithCancel(request.Context())
		time.AfterFunc(5*time.Millisecond, cancel)
		request = request.WithContext(cancellingCtx)

		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		store.assertWasCancelled()
	})
}
```

Deze aanpak is prima, maar is hij logisch?

Heeft het zin dat onze webserver zich bezighoudt met het handmatig annuleren van `Store`? Wat als `Store` ook afhankelijk is van andere trage processen? We moeten ervoor zorgen dat `Store.Cancel` de annulering correct doorgeeft aan al zijn afhankelijke processen.

Een van de belangrijkste punten van de `context` is dat het een consistente manier is om annulering aan te bieden.

[Uit de go doc](https://golang.org/pkg/context/)

> Inkomende verzoeken aan een server moeten een context aanmaken en uitgaande aanroepen naar servers moeten een context accepteren. De reeks functieaanroepen tussen deze verzoeken moet de context propageren en deze eventueel vervangen door een afgeleide context die is gemaakt met WithCancel, WithDeadline, WithTimeout of WithValue. Wanneer een context wordt geannuleerd, worden alle contexten die ervan zijn afgeleid ook geannuleerd.

Opnieuw uit de [Go Blog: Context](https://blog.golang.org/context):

> Bij Google vereisen we dat Go-programmeurs een contextparameter als eerste argument doorgeven aan elke functie in het aanroeppad tussen inkomende en uitgaande verzoeken. Dit zorgt ervoor dat Go-code die door verschillende teams is ontwikkeld, goed kan samenwerken. Het biedt eenvoudige controle over time-outs en annuleringen en zorgt ervoor dat kritieke waarden, zoals beveiligingsreferenties, Go-programma's correct overbrengen.

(Stop even en denk na over de gevolgen van het feit dat elke functie een context moet hebben, en de ergonomie daarvan.)

Voel je je een beetje ongemakkelijk? Goed. Laten we die aanpak volgen en in plaats daarvan de `context` doorgeven aan onze `Store` en die de verantwoordelijkheid laten dragen. Op die manier kan de Store de `context` ook doorgeven aan zijn afhankelijken, en ook zij kunnen verantwoordelijk zijn voor het stoppen.

## Schrijf eerst je test

We zullen onze bestaande tests moeten aanpassen, aangezien hun verantwoordelijkheden veranderen. Het enige waar onze handler nu verantwoordelijk voor is, is ervoor zorgen dat er context naar de downstream `Store` wordt gestuurd en dat de fout die uit de `Store` komt wanneer deze wordt geannuleerd, wordt afgehandeld.

Laten we onze `Store` interface bijwerken om de nieuwe verantwoordelijkheden te tonen.

```go
type Store interface {
	Fetch(ctx context.Context) (string, error)
}
```

Verwijder voorlopig de code in onze handler

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
	}
}
```

Pas onze `SpyStore` aan&#x20;

```go
type SpyStore struct {
	response string
	t        *testing.T
}

func (s *SpyStore) Fetch(ctx context.Context) (string, error) {
	data := make(chan string, 1)

	go func() {
		var result string
		for _, c := range s.response {
			select {
			case <-ctx.Done():
				log.Println("spy store got cancelled")
				return
			default:
				time.Sleep(10 * time.Millisecond)
				result += string(c)
			}
		}
		data <- result
	}()

	select {
	case <-ctx.Done():
		return "", ctx.Err()
	case res := <-data:
		return res, nil
	}
}
```

We moeten onze observator laten functioneren als een echte methode die met de `context` werkt.

We simuleren een langzaam proces waarbij we het resultaat langzaam opbouwen door de string, teken voor teken, toe te voegen in een goroutine. Wanneer de goroutine klaar is met zijn werk, schrijft hij de string naar het `data`kanaal. De goroutine luistert naar de `ctx.Done` en stopt met werken als er een signaal in dat kanaal wordt verzonden.

Ten slotte gebruikt de code nog een `select` om te wachten tot de goroutine klaar is met zijn werk of tot de annulering plaatsvindt.

Het is vergelijkbaar met onze eerdere aanpak: we gebruiken Go's concurrency oplossing om twee asynchrone processen tegen elkaar te laten racen om te bepalen wat we retourneren.

Je zult een vergelijkbare aanpak hanteren wanneer je je eigen functies en methoden schrijft die een `context` accepteren. Zorg er dus voor dat je begrijpt wat er gebeurt.

Eindelijk kunnen we onze tests bijwerken. Commentaar geven op onze annuleringstest, zodat we eerst de happy path-test kunnen repareren.

```go
t.Run("returns data from store", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data, t: t}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
})
```

## Probeer de test uit te voeren

```
=== RUN   TestServer/returns_data_from_store
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/returns_data_from_store (0.00s)
    	context_test.go:22: got "", want "hello, world"
```

## Schrijf genoeg code om de test te laten slagen

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, _ := store.Fetch(r.Context())
		fmt.Fprint(w, data)
	}
}
```

Ons gelukkige pad zou... gelukkig moeten zijn. Nu kunnen we de andere test oplossen.

## Schrijf eerst je test

We moeten testen of we geen enkel antwoord op de foutmelding schrijven. Helaas heeft `httptest.ResponseRecorder` geen manier om dit te achterhalen, dus we zullen onze eigen observator moeten bouwen om dit te testen.

```go
type SpyResponseWriter struct {
	written bool
}

func (s *SpyResponseWriter) Header() http.Header {
	s.written = true
	return nil
}

func (s *SpyResponseWriter) Write([]byte) (int, error) {
	s.written = true
	return 0, errors.New("not implemented")
}

func (s *SpyResponseWriter) WriteHeader(statusCode int) {
	s.written = true
}
```

Onze `SpyResponseWriter` implementeert `http.ResponseWriter`, zodat we het in de tests kunnen gebruiken.

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data, t: t}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := &SpyResponseWriter{}

	svr.ServeHTTP(response, request)

	if response.written {
		t.Error("a response should not have been written")
	}
})
```

## Probeer de test uit te voeren

```
=== RUN   TestServer
=== RUN   TestServer/tells_store_to_cancel_work_if_request_is_cancelled
--- FAIL: TestServer (0.01s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.01s)
    	context_test.go:47: a response should not have been written
```

## Schrijf genoeg code om de test te laten slagen

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, err := store.Fetch(r.Context())

		if err != nil {
			return // todo: log error however you like
		}

		fmt.Fprint(w, data)
	}
}
```

Hierna kunnen we zien dat de servercode is vereenvoudigd, omdat deze niet langer expliciet verantwoordelijk is voor annuleringen. Deze gaat simpelweg door de `context` heen en vertrouwt erop dat de functies lager in de keten rekening houden met eventuele annuleringen.

## Samenvattend

### Wat we hebben behandeld

* Hoe je een HTTP-handler test waarvan de aanvraag door de client is geannuleerd
* Hoe je context kunt gebruiken om annuleringen te beheren.
* Hoe je een functie schrijft die `context` accepteert en deze gebruikt om zichzelf te annuleren met behulp van goroutines, `select` en channels.
* Volg de richtlijnen van Google om annuleringen te beheren door de context van het verzoekbereik via uw call-stack te verspreiden.
* Hoe je zelf een observator voor `http.ResponseWriter` kunt maken als je dat nodig hebt.

### Hoe zit het met context.Value ?

[Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/) hebben een gelijke mening.

> Als je ctx.Value gebruikt in mijn (niet-bestaande) bedrijf, word je ontslagen

Sommige ingenieurs zijn voorstander van het doorgeven van waarden via `context`, omdat dit _handig lijkt_.

Gemak is vaak de oorzaak van slechte code.

Het probleem met `context.Values` ​​is dat het gewoon een niet getypeerde map is, dus je hebt geen typesafety en je moet ermee omgaan zonder dat je waarde erin zit. Je moet een koppeling van mapsleutels van de ene module naar de andere maken, en als iemand iets verandert, gaan dingen stuk.

Kortom, **als een functie waarden nodig heeft, zet ze dan als getypte parameters in plaats van te proberen ze op te halen uit `context.Value`**. Dit zorgt ervoor dat de functie statisch wordt gecontroleerd en gedocumenteerd, zodat iedereen kan zien wat het doet.

#### Maar...

Aan de andere kant kan het nuttig zijn om informatie op te nemen die loodrecht staat op een verzoek in een context, zoals een trace-ID. Deze informatie is mogelijk niet nodig voor elke functie in je call-stack en zou je functionele handtekeningen erg rommelig maken.

[Jack Lindamood zegt **Context.Value moet informeren, niet controleren**](https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39)

> De inhoud van context.Value is bedoeld voor beheerders, niet voor gebruikers. Het mag nooit verplichte invoer zijn voor gedocumenteerde of verwachte resultaten.

### Aanvullend materiaal

* Ik heb echt genoten van het lezen van [Context should go away for Go 2 van Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/). Hij betoogt dat het overal moeten doorgeven van `context` slechte code (code smell) is, dat het wijst op een tekortkoming in de taal met betrekking tot annulering. Hij zegt dat het beter zou zijn als dit op een of andere manier op taalniveau zou worden opgelost, in plaats van op bibliotheekniveau. Totdat dat gebeurt, heb je `context` nodig om langlopende processen te beheren.
* Het [Go blog beschrijft verder de motivatie om met `context` te werken en geeft een aantal voorbeelden](https://blog.golang.org/context)

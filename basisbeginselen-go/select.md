# Select

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/select)

Je bent gevraagd een functie genaamd `WebsiteRacer` te maken die twee URL's gebruikt en deze "racet" door ze te benaderen met een HTTP GET en de URL te retourneren die als eerste is geretourneerd. Als geen van beide binnen 10 seconden retourneert, zou er een `error` worden getourneerd.

Hiervoor gaan we gebruik maken van:

* `net/http` om HTTP verzoeken te versturen.
* `net/http/httptest` om te helpen bij het testen.
* goroutines.
* `select` om processen te synchroniseren.

## Schrijf eerst je test

Laten we om te beginnen met iets naïefs.

```go
func TestRacer(t *testing.T) {
	slowURL := "http://www.facebook.com"
	fastURL := "http://www.quii.dev"

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

We weten dat dit niet perfect is en dat er problemen zijn, maar het is een begin. Het is belangrijk om niet te gefixeerd te zijn op het idee om alles in één keer perfect te doen.

## Probeer de test uit te voeren

`./racer_test.go:14:9: undefined: Racer`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func Racer(a, b string) (winner string) {
	return
}
```

`racer_test.go:25: got '', want 'http://www.quii.dev'`

## Schrijf genoeg code om de test te laten slagen

```go
func Racer(a, b string) (winner string) {
	startA := time.Now()
	http.Get(a)
	aDuration := time.Since(startA)

	startB := time.Now()
	http.Get(b)
	bDuration := time.Since(startB)

	if aDuration < bDuration {
		return a
	}

	return b
}
```

Voor iedere URL:

1. Gebruiken we `time.Now()` om de tijd vast te leggen voordat we de `URL` proberen op te halen.
2. Dan gebruiken we [`http.Get`](https://golang.org/pkg/net/http/#Client.Get) om een HTTP `GET` verzoek uit te voeren op de `URL`. Deze functie geeft een [`http.Response`](https://golang.org/pkg/net/http/#Response) en een `error` terug, maar op dit moment zijn we nog niet geïnteresseerd in deze waarden.
3. `time.Since` neemt de start tijd en geeft een `time.Duration` van het verschil.

Zodra we dit hebben gedaan, vergelijken we eenvoudigweg de tijdsduren om te zien welke het snelst is.

### Problemen

Dit kan ertoe leiden dat de test wel of niet slaagt, dit is vooraf niet zeggen. Het probleem is dat we gebruik maken van externe websites om onze eigen logica te testen.

Het testen van code die HTTP gebruikt, is zo gebruikelijk dat Go hulpmiddelen in de standaardbibliotheek heeft waarmee je dit kunt doen.

In de hoofdstukken over [mocking](mocking.md) en [dependency injection](dependency-injection.md) hebben we besproken hoe we idealiter niet afhankelijk willen zijn van externe services om onze code te testen, omdat ze

* Langzaam
* Instabiel
* Geen randgevallen kunnen testen

In de standaardbibliotheek is een pakket met de naam [`net/http/httptest`](https://golang.org/pkg/net/http/httptest/) beschikbaar waarmee gebruikers eenvoudig een mock-HTTP-server kunnen maken.

Laten we onze tests veranderen en gebruikmaken van mocks, zodat we betrouwbare servers hebben om op te testen en die we kunnen controleren.

```go
func TestRacer(t *testing.T) {

	slowServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(20 * time.Millisecond)
		w.WriteHeader(http.StatusOK)
	}))

	fastServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	slowServer.Close()
	fastServer.Close()
}
```

De syntaxis ziet er misschien wat druk uit, maar neem er gerust de tijd voor.

`httptest.NewServer` gebruikt een `http.HandlerFunc` die we via een _anonieme functie_ versturen.

`http.HandlerFunc` is een type dat er als volgt uitziet: `type HandlerFunc func(ResponseWriter, *Request)`.

Het zegt eigenlijk alleen dat er een functie nodig is die een `ResponseWriter` en een `Request` accepteert, wat niet zo verrassend is voor een HTTP-server.

Het blijkt dat er eigenlijk geen extra magie aan te pas komt; **dit is ook hoe je een&#x20;**_**echte**_**&#x20;HTTP-server in Go zou schrijven**. Het enige verschil is dat we hem in een `httptest.NewServer` verpakken, wat hem makkelijker te gebruiken maakt bij het testen, omdat hij een open poort vindt om op te luisteren en je hem vervolgens kunt sluiten als je klaar bent met je test.

Binnen onze twee servers zorgen we ervoor dat de langzame server een korte `time.Sleep` heeft wanneer we een verzoek krijgen om hem langzamer te maken dan de andere. Beide servers sturen vervolgens een `OK`-antwoord met `w.WriteHeader(http.StatusOK)` terug naar de aanroeper.

Als je de test opnieuw uitvoert, zal hij nu zeker slagen en zou hij sneller moeten gaan. Experimenteer met deze slaapstanden om de test opzettelijk te verstoren.

## Refactor

Er is sprake van enige duplicatie in zowel onze productiecode als onze testcode.

```go
func Racer(a, b string) (winner string) {
	aDuration := measureResponseTime(a)
	bDuration := measureResponseTime(b)

	if aDuration < bDuration {
		return a
	}

	return b
}

func measureResponseTime(url string) time.Duration {
	start := time.Now()
	http.Get(url)
	return time.Since(start)
}
```

Door het opdrogen is onze `Racer`-code een stuk makkelijker te lezen.

```go
func TestRacer(t *testing.T) {

	slowServer := makeDelayedServer(20 * time.Millisecond)
	fastServer := makeDelayedServer(0 * time.Millisecond)

	defer slowServer.Close()
	defer fastServer.Close()

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

func makeDelayedServer(delay time.Duration) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(delay)
		w.WriteHeader(http.StatusOK)
	}))
}
```

We hebben het aanmaken van onze nepservers geherstructureerd in een functie genaamd `makeDelayedServer`. Hiermee halen we een deel van de saaie code uit de test en verminderen we de herhaling.

### `defer`

Door een functieaanroep vooraf te laten gaan door `defer`, wordt de functie `aangeroepen aan het einde van de bevattende functie`.

Soms moet je bronnen opschonen, bijvoorbeeld door een bestand te sluiten of, in ons geval, door een server te sluiten zodat deze niet langer naar een poort luistert.

Je wilt dat dit aan het einde van de functie wordt uitgevoerd, maar je wilt de instructie om dat te doen in de buurt houden van de plek waar je de server hebt gemaakt. Dit is handig voor toekomstige lezers van de code.

Onze refactoring is een verbetering en vormt een redelijke oplossing gezien de Go-functies die tot nu toe zijn besproken. We kunnen de oplossing echter nog eenvoudiger maken.

### Processen synchroniseren

* Waarom testen we de snelheid van websites één voor één, terwijl Go uitstekend presteert op het gebied van gelijktijdigheid? We zouden beide tegelijk moeten kunnen testen.
* _De exacte reactietijd_ van de verzoeken interesseert ons niet zozeer. We willen alleen weten welk verzoek het eerst wordt beantwoord.

Om dit te doen, introduceren we een nieuwe constructie genaamd `select` waarmee we processen heel eenvoudig en duidelijk kunnen synchroniseren.

```go
func Racer(a, b string) (winner string) {
	select {
	case <-ping(a):
		return a
	case <-ping(b):
		return b
	}
}

func ping(url string) chan struct{} {
	ch := make(chan struct{})
	go func() {
		http.Get(url)
		close(ch)
	}()
	return ch
}
```

#### `ping`

We hebben een functie `ping` gedefinieerd die een `chan struct{}` aanmaakt en retourneert.

In ons geval maakt het niet uit welk type bericht er naar het kanaal wordt verzonden. _We willen alleen aangeven dat we klaar_ zijn en het kanaal sluiten werkt perfect!

Waarom `struct{}` en niet een ander type zoals een `bool`? Nou, een `chan struct{}` is het kleinste beschikbare datatype vanuit geheugenperspectief, dus we krijgen geen toewijzing ten opzichte van een `bool`. Aangezien we de chan sluiten en niets verzenden, waarom zouden we dan iets toewijzen?

Binnen dezelfde functie starten we een goroutine die een signaal naar dat kanaal stuurt zodra we `http.Get(url)` hebben voltooid.

**Maak altijd gebruik van  `make` bij het aanmaken van een kanaal**

Merk op hoe we `make` moeten gebruiken bij het aanmaken van een kanaal, in plaats van bijvoorbeeld `var ch chan struct{}`. Wanneer je `var` gebruikt, wordt de variabele geïnitialiseerd met de "zero"-waarde van het type. Dus voor `string` is dat `""`, voor `int` is dat 0, enz.

Voor kanalen is de nulwaarde `nil` en als je probeert daar wat heen te sturen met `<-` wordt dat geblokkeerd omdat je niets naar `nil`kanalen kunt sturen

[Je kunt dit in actie zien in The Go Playground](https://play.golang.org/p/IIbeAox5jKA)

#### `select`

Je herinnert je misschien uit het hoofdstuk over [concurrency](concurrency.md) dat je kunt wachten tot er waarden naar een kanaal worden verzonden met `myVar := <-ch`. Dit is een blokkerende aanroep, omdat je wacht op een waarde.

Met `select` kun je op _meerdere kanalen_ wachten. De eerste die een waarde verzendt, "wint" en de code onder de `case` wordt uitgevoerd.

We gebruiken `ping` in onze `select` om twee kanalen in te stellen, één voor elk van onze `URL`'s. De code van het kanaal dat als eerste naar zijn kanaal schrijft, wordt uitgevoerd in de `select`, wat resulteert in het retourneren van de `URL` (en dus de winnaar).

Na deze wijzigingen is het doel van onze code veel duidelijker en is de implementatie zelfs eenvoudiger geworden.

### Timeouts

Onze laatste vereiste was om een ​​foutmelding te retourneren als `Racer` langer dan 10 seconden nodig heeft.

## Schrijf eerst de test

```go
func TestRacer(t *testing.T) {
	t.Run("compares speeds of servers, returning the url of the fastest one", func(t *testing.T) {
		slowServer := makeDelayedServer(20 * time.Millisecond)
		fastServer := makeDelayedServer(0 * time.Millisecond)

		defer slowServer.Close()
		defer fastServer.Close()

		slowURL := slowServer.URL
		fastURL := fastServer.URL

		want := fastURL
		got, _ := Racer(slowURL, fastURL)

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("returns an error if a server doesn't respond within 10s", func(t *testing.T) {
		serverA := makeDelayedServer(11 * time.Second)
		serverB := makeDelayedServer(12 * time.Second)

		defer serverA.Close()
		defer serverB.Close()

		_, err := Racer(serverA.URL, serverB.URL)

		if err == nil {
			t.Error("expected an error but didn't get one")
		}
	})
}
```

We hebben ervoor gezorgd dat onze testservers langer dan 10 seconden nodig hebben om te herstellen van dit scenario en we verwachten dat `Racer` nu twee waarden retourneert: de winnende URL (die we in deze test negeren met \_) en een `error`.

Merk op dat we de foutmeldingen in onze oorspronkelijke test ook hebben afgehandeld. We gebruiken nu `_` om er zeker van te zijn dat de tests worden uitgevoerd.

## Probeer de test uit te voeren

`./racer_test.go:37:10: assignment mismatch: 2 variables but Racer returns 1 value`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	}
}
```

Verander de aanroep van `Racer` om de winnaar en een `error` te retourneren. Retourneer `nil` voor onze gelukkige gevallen.

De compiler zal klagen dat je eerste test maar naar één waarde zoekt. Wijzig die regel dus in `got, err := Racer(slowURL, fastURL)`, wetende dat we moeten controleren of we in ons gelukkige scenario geen fout krijgen.

Als je de test nu uitvoert, zal deze na 11 seconden mislukken.

```
--- FAIL: TestRacer (12.00s)
    --- FAIL: TestRacer/returns_an_error_if_a_server_doesn't_respond_within_10s (12.00s)
        racer_test.go:40: expected an error but didn't get one
```

## Schrijf genoeg code om de test te laten slagen

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(10 * time.Second):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

`time.After` is een erg handige functie bij het gebruik van `select`. Hoewel dit in ons geval niet gebeurde, kun je mogelijk code schrijven die permanent blokkeert als de kanalen waarnaar je luistert nooit een waarde retourneren. `time.After` retourneert een `chan` (zoals `ping`) en stuurt een signaal ernaartoe na de door jou gedefinieerde tijd.

Voor ons is dit perfect; als `a` of `b` terug kunnen, hebben zij gewonnen, maar als wij 10 seconden halen, dan is onze `time.After` een signaal en geven wij een `error`.

### Langzame tests

Het probleem is dat deze test 10 seconden duurt. Voor zo'n simpele logica voelt dat niet echt prettig.

Wat we wel kunnen doen, is de time-out configureerbaar maken. In onze test kunnen we dus een zeer korte time-out instellen, en wanneer de code in productie wordt gebruikt, kan deze worden ingesteld op 10 seconden.

```go
func Racer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

Onze tests kunnen nu niet worden gecompileerd omdat we geen time-out opgeven.

Voordat we deze standaardwaarde aan beide tests toevoegen, gaan we eerst naar de resultaten _luisteren_.

* Maken we ons zorgen over de time-out in de "happy flow"?
* De eisen waren expliciet over de timeout.

Gegeven deze kennis, kunnen we een kleine refactoring uitvoeren om zowel onze tests als de gebruikers van onze code beter van dienst te zijn.

```go
var tenSecondTimeout = 10 * time.Second

func Racer(a, b string) (winner string, error error) {
	return ConfigurableRacer(a, b, tenSecondTimeout)
}

func ConfigurableRacer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

Onze gebruikers en onze eerste test kunnen `Racer` gebruiken (dat onder de motorkap ConfigurableRacer gebruikt) en onze _sad path-test_ kan `ConfigurableRacer` gebruiken.

```go
func TestRacer(t *testing.T) {

	t.Run("compares speeds of servers, returning the url of the fastest one", func(t *testing.T) {
		slowServer := makeDelayedServer(20 * time.Millisecond)
		fastServer := makeDelayedServer(0 * time.Millisecond)

		defer slowServer.Close()
		defer fastServer.Close()

		slowURL := slowServer.URL
		fastURL := fastServer.URL

		want := fastURL
		got, err := Racer(slowURL, fastURL)

		if err != nil {
			t.Fatalf("did not expect an error but got one %v", err)
		}

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("returns an error if a server doesn't respond within the specified time", func(t *testing.T) {
		server := makeDelayedServer(25 * time.Millisecond)

		defer server.Close()

		_, err := ConfigurableRacer(server.URL, server.URL, 20*time.Millisecond)

		if err == nil {
			t.Error("expected an error but didn't get one")
		}
	})
}
```

Ik heb nog een laatste controle aan de eerste test toegevoegd om te verifiëren dat we geen `error` terugkrijgen van de functieaanroep.

## Samenvattend

### `select`

* Helpt je bij het wachten op meerdere kanalen.
* Soms wil je `time.After` gebruiken in `cases` om te voorkomen dat je systeem blokkeert

### `httptest`

* Een handige manier om testservers te maken, zodat je betrouwbare en controleerbare tests hebt.
* Maakt gebruik van dezelfde interfaces als de "echte" `net/http`-servers, wat consistent is en minder kost om te leren.

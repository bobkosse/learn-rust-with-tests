# Fout types

**[Je kunt alle code voor dit hoofdstuk hier vinden](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/error-types)**

**Het aanmaken van je eigen typen voor fouten kan een elegante manier zijn om je code op te schonen, waardoor je code gemakkelijker te gebruiken en te testen is.**

Pedro op de Gopher Slack vraagt:

> Als ik een fout aanmaak zoals `fmt.Errorf("%s must be foo, got %s", bar, baz)`, is er dan een manier om gelijkheid te testen zonder de stringwaarde te vergelijken?

Laten we een functie bedenken om dit idee te verkennen.

```go
// DumbGetter will get the string body of url if it gets a 200
func DumbGetter(url string) (string, error) {
	res, err := http.Get(url)

	if err != nil {
		return "", fmt.Errorf("problem fetching from %s, %v", url, err)
	}

	if res.StatusCode != http.StatusOK {
		return "", fmt.Errorf("did not get 200 from %s, got %d", url, res.StatusCode)
	}

	defer res.Body.Close()
	body, _ := io.ReadAll(res.Body) // ignoring err for brevity

	return string(body), nil
}
```

Het is niet ongebruikelijk om een functie te schrijven die om verschillende redenen kan mislukken en we willen er zeker van zijn dat we elk scenario correct afhandelen.

Zoals Pedro zegt, _zouden_ we een test voor de statusfout als volgt kunnen schrijven.

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	want := fmt.Sprintf("did not get 200 from %s, got %d", svr.URL, http.StatusTeapot)
	got := err.Error()

	if got != want {
		t.Errorf(`got "%v", want "%v"`, got, want)
	}
})
```

Deze test creëert een server die altijd `StatusTeapot` retourneert. Vervolgens gebruiken we de URL ervan als argument voor `DumbGetter`, zodat we kunnen zien dat deze niet-`200`-reacties correct verwerkt.

## Problemen met deze manier van testen

Dit boek benadrukt het belang van _luister naar je tests_ en deze test voelt niet _goed_ aan:

- We construeren dezelfde string als productiecode om deze te testen.
- Het is vervelend om te lezen en te schrijven.
- Is de exacte string met de foutmelding waar we _eigenlijk mee bezig zijn_?

Wat vertelt dit ons? De ergonomie van onze test zou zich weerspiegelen in een ander stukje code dat onze code probeert te gebruiken.

Hoe reageert een gebruiker van onze code op het specifieke soort fouten dat we retourneren? Het beste wat ze kunnen doen is kijken naar de string met de fout, die extreem foutgevoelig en moeilijk te schrijven is.

## Wat we zouden moeten doen

Met TDD hebben we het voordeel dat we in de volgende mindset komen:

> Hoe zou _ik_ deze code willen gebruiken?

Wat we voor `DumbGetter` zouden kunnen doen, is gebruikers een manier bieden om het typesysteem te gebruiken om te begrijpen wat voor soort fout er is opgetreden.

Wat als `DumbGetter` ons zoiets zou kunnen teruggeven?

```go
type BadStatusError struct {
	URL    string
	Status int
}
```

In plaats van een magische string hebben we daadwerkelijke _data_ om mee te werken.

Laten we onze bestaande test aanpassen om aan deze behoefte te voldoen.

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	got, isStatusErr := err.(BadStatusError)

	if !isStatusErr {
		t.Fatalf("was not a BadStatusError, got %T", err)
	}

	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

We moeten ervoor zorgen dat `BadStatusError` de foutinterface implementeert.

```go
func (b BadStatusError) Error() string {
	return fmt.Sprintf("did not get 200 from %s, got %d", b.URL, b.Status)
}
```

### Wat doet de test?

In plaats van de exacte string van de fout te controleren, voeren we een [type assertion](https://tour.golang.org/methods/15) uit op de fout om te zien of het een `BadStatusError` is. Dit weerspiegelt onze wens voor een _soort_ foutopheldering. Ervan uitgaande dat de assertion slaagt, kunnen we vervolgens controleren of de eigenschappen van de fout correct zijn.

Wanneer we de test uitvoeren, geeft deze aan dat we niet het juiste type fout hebben geretourneerd.

```
--- FAIL: TestDumbGetter (0.00s)
    --- FAIL: TestDumbGetter/when_you_dont_get_a_200_you_get_a_status_error (0.00s)
    	error-types_test.go:56: was not a BadStatusError, got *errors.errorString
```

Laten we `DumbGetter` repareren door onze foutverwerkingscode bij te werken om ons type te gebruiken

```go
if res.StatusCode != http.StatusOK {
	return "", BadStatusError{URL: url, Status: res.StatusCode}
}
```

Deze wijziging heeft een aantal _echt positieve effecten_ gehad.

- Onze `DumbGetter`-functie is eenvoudiger geworden. Hij houdt zich niet langer bezig met de complexiteit van een foutstring, maar creëert gewoon een `BadStatusError`.
- Onze tests weerspiegelen (en documenteren) nu wat een gebruiker van onze code _zou_ kunnen doen als hij besluit om geavanceerdere foutafhandeling te gebruiken dan alleen loggen. Voer gewoon een type-assertie uit en je krijgt eenvoudig toegang tot de eigenschappen van de fout.
- Het is nog steeds "gewoon" een `fout`, dus als ze dat willen, kunnen ze deze doorgeven aan de call stack of loggen zoals elke andere `fout`.

## Samenvattend

Als je test op meerdere foutcondities, trap dan niet in de valkuil van het vergelijken van de foutmeldingen.

Dit leidt tot onbetrouwbare en moeilijk te lezen/schrijven tests en het weerspiegelt de moeilijkheden die de gebruikers van je code zullen ondervinden als ze ook dingen anders moeten gaan doen, afhankelijk van het soort fouten dat is opgetreden.

Zorg er altijd voor dat je tests weerspiegelen hoe je je code wilt gebruiken. Overweeg daarom om fouttypen te creëren die je fouten inkapselen. Dit maakt het verwerken van verschillende soorten fouten eenvoudiger voor gebruikers van je code en maakt het schrijven van je code voor foutverwerking eenvoudiger en leesbaarder.

## Addendum

Vanaf Go 1.13 zijn er nieuwe manieren om met fouten om te gaan in de standaardbibliotheek. Deze worden behandeld in de [Go Blog](https://blog.golang.org/go1.13-errors)

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	var got BadStatusError
	isBadStatusError := errors.As(err, &got)
	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if !isBadStatusError {
		t.Fatalf("was not a BadStatusError, got %T", err)
	}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

In dit geval gebruiken we [`errors.As`](https://pkg.go.dev/errors#example-As) om te proberen onze fout in ons aangepaste type te extraheren. Het retourneert een `bool` om succes aan te geven en extraheert deze voor ons naar `got`.

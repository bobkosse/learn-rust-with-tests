# Maps

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/maps)

In [arrays en slices](arrays-and-slices.md) heb je gezien hoe je waarden in volgorde kunt opslaan. Nu gaan we kijken naar een manier om items op sleutel op te slaan en ze snel op te zoeken.

Met Maps kun je items opslaan op een manier die vergelijkbaar is met een woordenboek. Je kunt de `key` zien als het woord en de `value` als de definitie. En wat is er nou beter dan Maps te leren kennen door je eigen woordenboek te bouwen?

Ten eerste, ervan uitgaande dat er al een aantal woorden met hun definities in het woordenboek staan, zou het woordenboek, als we naar een woord zoeken, de definitie ervan moeten teruggeven.

## Schrijf eerst je test

In `dictionary_test.go`

```go
package main

import "testing"

func TestSearch(t *testing.T) {
	dictionary := map[string]string{"test": "this is just a test"}

	got := Search(dictionary, "test")
	want := "this is just a test"

	if got != want {
		t.Errorf("got %q want %q given, %q", got, want, "test")
	}
}
```

Het declareren van een map lijkt enigszins op een array. Behalve dat het begint met het sleutelwoord `map` en twee typen vereist. Het eerste is het sleuteltype, dat tussen de `[]` staat. Het tweede is het waardetype, dat direct na de `[]` komt.

Het sleuteltype is speciaal. Het kan alleen een vergelijkbaar type zijn, want zonder de mogelijkheid om te bepalen of twee sleutels gelijk zijn, kunnen we niet garanderen dat we de juiste waarde krijgen. Vergelijkbare typen worden uitgebreid uitgelegd in de [taalspecificatie](https://golang.org/ref/spec#Comparison_operators).

Het waardetype kan daarentegen elk gewenst type zijn. Het kan zelfs een andere map zijn.

Al het overige in de test zou je bekend voor moeten komen.

## Probeer de test uit te voeren

Door het uitvoeren van `go test` zal de compiler falen met `./dictionary_test.go:8:9: undefined: Search`.

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

In `dictionary.go`

```go
package main

func Search(dictionary map[string]string, word string) string {
	return ""
}
```

Je test zou nu moeten mislukken met een _duidelijke foutmelding_

`dictionary_test.go:12: got '' want 'this is just a test' given, 'test'`.

## Schrijf genoeg code om de test te laten slagen

```go
func Search(dictionary map[string]string, word string) string {
	return dictionary[word]
}
```

Het ophalen van een waarde uit een Map is hetzelfde als het ophalen van een waarde uit een Array `map[key]`.

## Refactor

```go
func TestSearch(t *testing.T) {
	dictionary := map[string]string{"test": "this is just a test"}

	got := Search(dictionary, "test")
	want := "this is just a test"

	assertStrings(t, got, want)
}

func assertStrings(t testing.TB, got, want string) {
	t.Helper()

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Ik heb besloten een `assertStrings`-helper te maken om de implementatie algemener te maken.

### Maak gebruik van een custom type

We kunnen het gebruik van ons woordenboek verbeteren door een nieuw type rondom map te creëren en van Search een methode te maken.

In `dictionary_test.go`:

```go
func TestSearch(t *testing.T) {
	dictionary := Dictionary{"test": "this is just a test"}

	got := dictionary.Search("test")
	want := "this is just a test"

	assertStrings(t, got, want)
}
```

Hier maken we gebruik van het `Dictionary`-type, dat nog niet gedefinieerd is. Vervolgens roepen we `Search` aan op de Dictionary-instantie.

We hoeven hiervoor niet de `assertStrings`-helper aan te passen.

In `dictionary.go`:

```go
type Dictionary map[string]string

func (d Dictionary) Search(word string) string {
	return d[word]
}
```

Hier hebben we een woordenboektype gemaakt dat als een dunne wrapper rond de map fungeert. Met het aangepaste type gedefinieerd, kunnen we de `Search` methode maken.

## Schrijf eerst je test

De basis zoekfunctie was heel eenvoudig te implementeren, maar wat gebeurt er als we een woord invoeren dat niet in ons woordenboek voorkomt?

We krijgen eigenlijk niets terug. Dit is goed, omdat het programma gewoon door kan blijven draaien, maar er is een betere aanpak. De functie kan melden dat het woord niet in het woordenboek staat. Zo hoeft de gebruiker zich niet af te vragen of het woord niet bestaat of dat er gewoon geen definitie is (dit lijkt misschien niet erg nuttig voor een woordenboek. Het is echter een scenario dat in andere gevallen cruciaal zou kunnen zijn).

```go
func TestSearch(t *testing.T) {
	dictionary := Dictionary{"test": "this is just a test"}

	t.Run("known word", func(t *testing.T) {
		got, _ := dictionary.Search("test")
		want := "this is just a test"

		assertStrings(t, got, want)
	})

	t.Run("unknown word", func(t *testing.T) {
		_, err := dictionary.Search("unknown")
		want := "could not find the word you were looking for"

		if err == nil {
			t.Fatal("expected to get an error.")
		}

		assertStrings(t, err.Error(), want)
	})
}
```

De manier om dit scenario in Go af te handelen is om een ​​tweede argument te retourneren, namelijk van het type `Error`.

Merk op dat we in de sectie over [pointers en errors](pointers-and-errors.md) hebben gezien dat we, om de foutmelding te bevestigen, eerst moeten controleren of de fout niet `nil` is. Vervolgens gebruiken we de methode `.Error()` om de tekenreeks te verkrijgen die we vervolgens aan de bevestiging kunnen doorgeven.

## Probeer de test uit te voeren

Dit zal niet compileren

```
./dictionary_test.go:18:10: assignment mismatch: 2 variables but 1 values
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func (d Dictionary) Search(word string) (string, error) {
	return d[word], nil
}
```

Je test zou nu moeten mislukken en er verschijnt een veel duidelijkere foutmelding.

`dictionary_test.go:22: expected to get an error.`

## Schrijf genoeg code om de test te laten slagen

```go
func (d Dictionary) Search(word string) (string, error) {
	definition, ok := d[word]
	if !ok {
		return "", errors.New("could not find the word you were looking for")
	}

	return definition, nil
}
```

Om dit mogelijk te maken, gebruiken we een interessante eigenschap van de map lookup. Deze kan twee waarden retourneren. De tweede waarde is een boolean die aangeeft of de sleutel succesvol is gevonden.

Dankzij deze eigenschap kunnen we onderscheid maken tussen een woord dat niet bestaat en een woord dat geen definitie heeft.

## Refactor

```go
var ErrNotFound = errors.New("could not find the word you were looking for")

func (d Dictionary) Search(word string) (string, error) {
	definition, ok := d[word]
	if !ok {
		return "", ErrNotFound
	}

	return definition, nil
}
```

We kunnen de magische fout in onze `Search`-functie verwijderen door deze in een variabele te extraheren. Dit stelt ons ook in staat om een ​​betere test te doen.

```go
t.Run("unknown word", func(t *testing.T) {
	_, got := dictionary.Search("unknown")
	if got == nil {
		t.Fatal("expected to get an error.")
	}
	assertError(t, got, ErrNotFound)
})
```

```go
func assertError(t testing.TB, got, want error) {
	t.Helper()

	if got != want {
		t.Errorf("got error %q want %q", got, want)
	}
}
```

Door een nieuwe helper te maken, konden we onze test vereenvoudigen en zijn we onze variabele `ErrNotFound` gaan gebruiken. Hierdoor mislukt de test niet meer als we in de toekomst de fouttekst wijzigen.

## Schrijf eerst je teste

We hebben een geweldige manier om het woordenboek te doorzoeken. We kunnen echter geen nieuwe woorden aan ons woordenboek toevoegen.

```go
func TestAdd(t *testing.T) {
	dictionary := Dictionary{}
	dictionary.Add("test", "this is just a test")

	want := "this is just a test"
	got, err := dictionary.Search("test")
	if err != nil {
		t.Fatal("should find added word:", err)
	}

	assertStrings(t, got, want)
}
```

In deze test maken we gebruik van onze `Search` functie om de validatie van het woordenboek iets eenvoudiger te maken.

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

In `dictionary.go`

```go
func (d Dictionary) Add(word, definition string) {
}
```

Je test zou nu moeten falen:

```
dictionary_test.go:31: should find added word: could not find the word you were looking for
```

## Schrijf genoeg code om test te laten slagen

```go
func (d Dictionary) Add(word, definition string) {
	d[word] = definition
}
```

Toevoegen aan een map is vergelijkbaar met een array. Je hoeft alleen een sleutel op te geven en deze gelijk te stellen aan een waarde.

### Pointers, kopieën, etc

Een interessante eigenschap van `maps` is dat je ze kunt aanpassen zonder dat je er een adres aan hoeft door te geven (bijvoorbeeld `&myMap`)

Daardoor voelen ze misschien aan als een 'referentietype', [maar zoals Dave Cheney het beschrijft](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it), zijn ze dat niet.

> Een mapwaarde is een aanwijzer naar een runtime.hmap-structuur.

Wanneer je dus een map doorgeeft aan een functie/methode, kopieer je deze feitelijk, maar alleen het pointer-gedeelte, niet de onderliggende gegevensstructuur die de data bevat.

Een valkuil bij maps is dat ze een `nil`-waarde kunnen hebben. Een `nil`-map gedraagt ​​zich tijdens het lezen als een lege map, maar pogingen om naar een `nil`-map te schrijven veroorzaken runtime-panic. Je kunt [hier](https://blog.golang.org/go-maps-in-action) meer over maps lezen.

Daarom mag u nooit een nil map-variabele initialiseren:

```go
var m map[string]string
```

In plaats daarvan kun je een lege map initialiseren of het trefwoord `make` gebruiken om een ​​map voor je te maken:

```go
var dictionary = map[string]string{}

// OR

var dictionary = make(map[string]string)
```

Beide benaderingen creëren een lege `hashmap` en verwijzen `dictionary` ernaar. Dit zorgt ervoor dat je nooit een runtime-panic krijgt.

## Refactor

Er valt niet veel te refactoren in onze implementatie, maar de test zou wel wat vereenvoudigd kunnen.

```go
func TestAdd(t *testing.T) {
	dictionary := Dictionary{}
	word := "test"
	definition := "this is just a test"

	dictionary.Add(word, definition)

	assertDefinition(t, dictionary, word, definition)
}

func assertDefinition(t testing.TB, dictionary Dictionary, word, definition string) {
	t.Helper()

	got, err := dictionary.Search(word)
	if err != nil {
		t.Fatal("should find added word:", err)
	}
	assertStrings(t, got, definition)
}
```

We hebben variabelen voor woord en definitie gemaakt en de definitie-vergelijking naar een eigen hulpfunctie verplaatst.

Onze `Add` ziet er goed uit. Alleen hebben we niet nagedacht over wat er gebeurt als de waarde die we proberen toe te voegen al bestaat!

Map geeft geen foutmelding als de waarde al bestaat. In plaats daarvan overschrijven ze de waarde met de nieuw opgegeven waarde. Dit kan in de praktijk handig zijn, maar maakt onze functienaam minder nauwkeurig. `Add` mag bestaande waarden niet wijzigen. Het mag alleen nieuwe woorden aan ons woordenboek toevoegen.

## Schrijf eerst je test

```go
func TestAdd(t *testing.T) {
	t.Run("new word", func(t *testing.T) {
		dictionary := Dictionary{}
		word := "test"
		definition := "this is just a test"

		err := dictionary.Add(word, definition)

		assertError(t, err, nil)
		assertDefinition(t, dictionary, word, definition)
	})

	t.Run("existing word", func(t *testing.T) {
		word := "test"
		definition := "this is just a test"
		dictionary := Dictionary{word: definition}
		err := dictionary.Add(word, "new test")

		assertError(t, err, ErrWordExists)
		assertDefinition(t, dictionary, word, definition)
	})
}
```

Voor deze test hebben we `Add` aangepast om een ​​fout te retourneren, die we valideren met een nieuwe foutvariabele, `ErrWordExists`. We hebben ook de vorige test aangepast om te controleren op een nulfout.

## Probeer de test uit te voeren

De compiler zal falen omdat we geen waarde voor `Add` retourneren.

```
./dictionary_test.go:30:13: dictionary.Add(word, definition) used as value
./dictionary_test.go:41:13: dictionary.Add(word, "new test") used as value
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

In `dictionary.go`

```go
var (
	ErrNotFound   = errors.New("could not find the word you were looking for")
	ErrWordExists = errors.New("cannot add word because it already exists")
)

func (d Dictionary) Add(word, definition string) error {
	d[word] = definition
	return nil
}
```

Nu krijgen we nog twee fouten. We zijn de waarde nog steeds aan het aanpassen en retourneren een `nil`-fout.

```
dictionary_test.go:43: got error '%!q(<nil>)' want 'cannot add word because it already exists'
dictionary_test.go:44: got 'new test' want 'this is just a test'
```

## Schrijf genoeg code om de test te laten slagen

```go
func (d Dictionary) Add(word, definition string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		d[word] = definition
	case nil:
		return ErrWordExists
	default:
		return err
	}

	return nil
}
```

Hier gebruiken we een `switch`-statement om de fout te matchen. Een dergelijke `switch` biedt een extra vangnet voor het geval `Search` een andere fout dan `ErrNotFound` retourneert.

## Refactor

Er valt niet heel veel te refactoren, maar naarmate het aantal fout-afhandelingen toeneemt, kunnen we een paar aanpassingen doen.

```go
const (
	ErrNotFound   = DictionaryErr("could not find the word you were looking for")
	ErrWordExists = DictionaryErr("cannot add word because it already exists")
)

type DictionaryErr string

func (e DictionaryErr) Error() string {
	return string(e)
}
```

We hebben de fout-meldingen constant gemaakt; hiervoor moesten we ons eigen `DictionaryErr`-type maken dat de `error`interface implementeert. Je kunt meer over de details lezen in [dit uitstekende artikel van Dave Cheney](https://dave.cheney.net/2016/04/07/constant-errors). Simpel gezegd: het maakt de fouten herbruikbaarder en onveranderlijker.

Laten we nu een functie maken om de definitie van een woord bij te werken.

## Schrijf eerste je test

```go
func TestUpdate(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{word: definition}
	newDefinition := "new definition"

	dictionary.Update(word, newDefinition)

	assertDefinition(t, dictionary, word, newDefinition)
}
```

`Update` is nauw verwant aan `Add` en zal onze volgende implementatie zijn.

## Probeer de test uit te voeren

```
./dictionary_test.go:53:2: dictionary.Update undefined (type Dictionary has no field or method Update)
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

We weten al hoe we met een dergelijke fout moeten omgaan. We moeten onze functie definiëren.

```go
func (d Dictionary) Update(word, definition string) {}
```

Nu we dat weten, zien we dat we de definitie van het woord moeten veranderen.

```
dictionary_test.go:55: got 'this is just a test' want 'new definition'
```

## Schrijf genoeg code om de test te laten slagen

We hebben al gezien hoe we dit kunnen doen toen we het probleem met `Add` oplosten. Laten we dus iets implementeren dat erg lijkt op `Add`.

```go
func (d Dictionary) Update(word, definition string) {
	d[word] = definition
}
```

We hoeven hier geen refactoring op uit te voeren, aangezien het een eenvoudige wijziging was. We hebben nu echter hetzelfde probleem als met `Add`. Als we een nieuw woord doorgeven, voegt `Update` het toe aan het woordenboek.

## Schrijf eerst je test

```go
t.Run("existing word", func(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{word: definition}
	newDefinition := "new definition"

	err := dictionary.Update(word, newDefinition)

	assertError(t, err, nil)
	assertDefinition(t, dictionary, word, newDefinition)
})

t.Run("new word", func(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{}

	err := dictionary.Update(word, definition)

	assertError(t, err, ErrWordDoesNotExist)
})
```

We hebben nog een fouttype toegevoegd voor wanneer het woord niet bestaat. Ook hebben we `Update` aangepast om een ​​`error`waarde te retourneren.

## Probeer de test uit te voeren

```
./dictionary_test.go:53:16: dictionary.Update(word, newDefinition) used as value
./dictionary_test.go:64:16: dictionary.Update(word, definition) used as value
./dictionary_test.go:66:23: undefined: ErrWordDoesNotExist
```

Deze keer krijgen we 3 fouten, maar we weten hoe we daarmee om moeten gaan.

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
const (
	ErrNotFound         = DictionaryErr("could not find the word you were looking for")
	ErrWordExists       = DictionaryErr("cannot add word because it already exists")
	ErrWordDoesNotExist = DictionaryErr("cannot perform operation on word because it does not exist")
)

func (d Dictionary) Update(word, definition string) error {
	d[word] = definition
	return nil
}
```

We hebben ons eigen fouttype toegevoegd en retourneren een `nil`-fout.

Door deze wijzigingen krijgen we nu een heel duidelijke foutmelding:

```
dictionary_test.go:66: got error '%!q(<nil>)' want 'cannot update word because it does not exist'
```

## Schrijf genoeg code om de test te laten slagen

```go
func (d Dictionary) Update(word, definition string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		return ErrWordDoesNotExist
	case nil:
		d[word] = definition
	default:
		return err
	}

	return nil
}
```

Deze functie lijkt bijna hetzelfde als `Add`, behalve dat we we nu een update doen in de `dictionary` wanneer we geen fout krijgen.

### Opmerking over het declareren van een nieuwe fout voor Update

We zouden `ErrNotFound` kunnen hergebruiken zonder een nieuwe fout toe te voegen. Het is echter vaak beter om een ​​specifieke foutmelding te hebben voor wanneer een update mislukt.

Door specifieke fouten te hebben, krijg je meer informatie over wat er misgaat. Hier is een voorbeeld in een webapp:

> Je kunt de gebruiker omleiden wanneer `ErrNotFound` wordt gegeven, maar een foutmelding tonen wanneer `ErrWordDoesNotExist` wordt gegeven.

Laten we nu een functie maken om een ​​woord uit het woordenboek te verwijderen.

## Schrijf je eerste test

```go
func TestDelete(t *testing.T) {
	word := "test"
	dictionary := Dictionary{word: "test definition"}

	dictionary.Delete(word)

	_, err := dictionary.Search(word)
	assertError(t, err, ErrNotFound)
}
```

Onze test maakt een `Dictionary` met een woord en controleert vervolgens of het woord is verwijderd.

## Probeer de test uit te voeren

Door `go test` uit te voeren, krijgen we:

```
./dictionary_test.go:74:6: dictionary.Delete undefined (type Dictionary has no field or method Delete)
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func (d Dictionary) Delete(word string) {

}
```

Nadat we dit hebben toegevoegd, geeft de test aan dat we het woord niet verwijderd is.

```
dictionary_test.go:78: got error '%!q(<nil>)' want 'could not find the word you were looking for'
```

## Schrijf genoeg code om de test te laten slagen

```go
func (d Dictionary) Delete(word string) {
	delete(d, word)
}
```

Go heeft een ingebouwde functie `delete` die werkt op maps. Deze functie accepteert twee argumenten en retourneert niets. Het eerste argument is de map en het tweede de te verwijderen sleutel.

## Refactor

Er valt niet veel te refactoren, maar we kunnen dezelfde logica uit `Update` implementeren om gevallen af ​​te handelen waarin het woord niet bestaat.

```go
func TestDelete(t *testing.T) {
	t.Run("existing word", func(t *testing.T) {
		word := "test"
		dictionary := Dictionary{word: "test definition"}

		err := dictionary.Delete(word)

		assertError(t, err, nil)

		_, err = dictionary.Search(word)

		assertError(t, err, ErrNotFound)
	})

	t.Run("non-existing word", func(t *testing.T) {
		word := "test"
		dictionary := Dictionary{}

		err := dictionary.Delete(word)

		assertError(t, err, ErrWordDoesNotExist)
	})
}
```

## Voer de test uit

De compiler zal falen omdat we geen waarde voor `Delete` retourneren.

```
./dictionary_test.go:77:10: dictionary.Delete(word) (no value) used as value
./dictionary_test.go:90:10: dictionary.Delete(word) (no value) used as value
```

## Schrijf genoeg code om test te laten slagen

```go
func (d Dictionary) Delete(word string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		return ErrWordDoesNotExist
	case nil:
		delete(d, word)
	default:
		return err
	}

	return nil
}
```

We gebruiken opnieuw een switch-instructie om de fout te detecteren die optreedt wanneer we een woord proberen te verwijderen dat niet bestaat.

## Samenvattend

In deze sectie hebben we veel behandeld. We hebben een volledige CRUD (Create, Read, Update en Delete) API voor ons woordenboek gemaakt. Tijdens dit proces hebben we geleerd hoe we:

* Maps aan kunnen maken
* Kunnen zoeken naar items binnen die maps
* Nieuwe items aan maps toe kunnen voegen
* Items in een map kunnen updaten
* Items van een map kunnen verwijderen
* Geleerd over errors
  * Hoe we errors kunnen maken als constanten
  * Hoe we error wrappers kunnen schrijven

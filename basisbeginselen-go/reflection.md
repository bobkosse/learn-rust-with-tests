# Reflection

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/reflection)

[Van Twitter](https://twitter.com/peterbourgon/status/1011403901419937792?s=09)

> golang challenge: write a function `walk(x interface{}, fn func(string))` which takes a struct `x` and calls `fn` for all strings fields found inside. difficulty level: recursively.

Om dit te doen, moeten we _reflection_ gebruiken.

> Reflection in de informatica is het vermogen van een programma om zijn eigen structuur te onderzoeken, met name door middel van typen; het is een vorm van meta-programmeren. Het is ook een grote bron van verwarring.

Bron [The Go Blog: Reflection](https://blog.golang.org/laws-of-reflection)

## Wat is `interface{}`?

We zijn erg blij met de typeveiligheid die Go ons biedt in termen van functies die werken met bekende typen, zoals `string`, `int` en onze eigen typen zoals `BankAccount`.

Dit betekent dat we een deel van de documentatie gratis krijgen en dat de compiler zal klagen als je probeert het verkeerde type aan een functie door te geven.

Er kunnen zich echter situaties voordoen waarin je een functie wilt schrijven waarvan je het type tijdens de compilatie nog niet weet.

Met Go kunnen we dit omzeilen met het type `interface{}`, dat je kun beschouwen als elk willekeurig type (in Go is `any` zelfs een [alias](https://cs.opensource.google/go/go/+/master:src/builtin/builtin.go;drc=master;l=95) voor `interface{}`).

Dus `walk(x interface{}, fn func(string))` accepteert elke waarde voor `x`.

### Waarom zou je `interface{}` dan niet voor alles gebruiken en echt flexibele functies hebben?

* Als gebruiker van een functie die `interface{}` gebruikt, verlies je de typesafety. Wat als je `Herd.species` van het type `string` aan een functie wilde doorgeven, maar in plaats daarvan `Herd.count`, een `int`, gebruikte? De compiler kan je niet op de hoogte stellen van je fout. Je hebt ook geen idee wat je aan een functie mag doorgeven. Weten dat een functie bijvoorbeeld een `UserService` gebruikt, is erg handig.
* Als schrijver van zo'n functie moet je _alles_ wat je krijgt kunnen inspecteren en proberen te achterhalen wat het type is en wat je ermee kunt doen. Dit doe je met behulp van _reflection_. Dit kan nogal onhandig en moeilijk leesbaar zijn en is over het algemeen minder performant (omdat je controles tijdens runtime moet uitvoeren).

Kortom, gebruik reflection alleen als het echt nodig is.

Als je polymorfe functies wilt, kun je overwegen om deze te ontwerpen rond een interface (niet `interface{}`, wat verwarrend is) zodat gebruikers je functie met meerdere typen kunnen gebruiken als ze de methoden implementeren die je nodig hebt om de functie te laten werken.

Onze functie moet met veel verschillende dingen kunnen werken. Zoals altijd hanteren we een iteratieve aanpak: we schrijven tests voor elk nieuw ding dat we willen ondersteunen en refactoren gaandeweg tot we klaar zijn.

## Schrijf eerst je test

We willen onze functie aanroepen met een struct die een string-veld bevat (`x`). Vervolgens kunnen we de meegegeven functie (`fn`) bekijken om te zien of deze wordt aangeroepen.

```go
func TestWalk(t *testing.T) {

	expected := "Chris"
	var got []string

	x := struct {
		Name string
	}{expected}

	walk(x, func(input string) {
		got = append(got, input)
	})

	if len(got) != 1 {
		t.Errorf("wrong number of function calls, got %d want %d", len(got), 1)
	}
}
```

* We willen een deel van de strings (`got`) opslaan, waarin staat welke strings via `walk` aan `fn` zijn doorgegeven. In voorgaande hoofdstukken hebben we hiervoor vaak speciale typen gemaakt om functie-/methodeaanroepen te bespioneren, maar in dit geval kunnen we gewoon een anonieme functie voor `fn` doorgeven die `got` afsluit.
* We gebruiken een anonieme `struct` met een veld `Name` van het type string om het eenvoudigste "happy" pad te bereiken.
* Roep ten slotte `walk` aan met `x` en spy en controleer voor nu alleen de lengte van `got`. We zullen specifieker worden met onze beweringen zodra we iets heel basaals werkend hebben.

## Probeer de test uit te voeren

```
./reflection_test.go:21:2: undefined: walk
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

We moeten `walk` definiëren&#x20;

```go
func walk(x interface{}, fn func(input string)) {

}
```

Probeer en voer de test opnieuw uit

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:19: wrong number of function calls, got 0 want 1
FAIL
```

## Schrijf genoeg code om de test te laten slagen

We kunnen de observator met een willekeurige tekenreeks aanroepen om dit te doen.

```go
func walk(x interface{}, fn func(input string)) {
	fn("I still can't believe South Korea beat Germany 2-0 to put them last in their group")
}
```

De test zou nu moeten slagen. Het volgende wat we moeten doen, is een specifiekere bewering doen over waarmee onze `fn` wordt aangeroepen.

## Schrijf eerst je test

Voeg het volgende toe aan de bestaande test om te controleren of de aan `fn` doorgegeven string correct is

```go
if got[0] != expected {
	t.Errorf("got %q, want %q", got[0], expected)
}
```

## Probeer de test uit te voeren

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:23: got 'I still can't believe South Korea beat Germany 2-0 to put them last in their group', want 'Chris'
FAIL
```

## Schrijf genoeg code om de test te laten slagen

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)
	field := val.Field(0)
	fn(field.String())
}
```

Deze code is _erg onveilig_ en _erg naïef_, maar vergeet niet: ons doel wanneer we in het "rood" zitten (wanneer de tests mislukken), is om zo min mogelijk code te schrijven. Vervolgens schrijven we meer tests om onze zorgen weg te nemen.

We moeten reflection gebruiken om naar `x` te kijken en proberen de eigenschappen ervan te bestuderen.

Het [reflect-pakket](https://pkg.go.dev/reflect) heeft een functie `ValueOf` die ons een `waarde` van een gegeven variabele retourneert. Dit biedt ons manieren om een ​​waarde te inspecteren, inclusief de velden die we op de volgende regel gebruiken.

Vervolgens maken we in onze code een aantal zeer optimistische aannames over de doorgegeven waarde:

* We kijken naar het eerste en enige veld. Het kan echter zijn dat er helemaal geen velden zijn, wat een paniek-fout zou veroorzaken.
* Vervolgens roepen we `String()` aan, die de onderliggende waarde als een string retourneert. Dit zou echter onjuist zijn als het veld iets anders dan een string teruggeeft.

## Refactor

Onze code is geschikt voor het eenvoudige geval, maar we weten dat onze code veel tekortkomingen heeft.

We gaan een aantal tests schrijven waarin we verschillende waarden doorgeven en de array van strings controleren waarmee `fn` is aangeroepen.

We moeten onze test omzetten in een tabelgebaseerde test, zodat we makkelijker nieuwe scenario's kunnen blijven testen.

```go
func TestWalk(t *testing.T) {

	cases := []struct {
		Name          string
		Input         interface{}
		ExpectedCalls []string
	}{
		{
			"struct with one string field",
			struct {
				Name string
			}{"Chris"},
			[]string{"Chris"},
		},
	}

	for _, test := range cases {
		t.Run(test.Name, func(t *testing.T) {
			var got []string
			walk(test.Input, func(input string) {
				got = append(got, input)
			})

			if !reflect.DeepEqual(got, test.ExpectedCalls) {
				t.Errorf("got %v, want %v", got, test.ExpectedCalls)
			}
		})
	}
}
```

Nu kunnen we eenvoudig een scenario toevoegen om te zien wat er gebeurt als we meer dan één string veld hebben.

## Schrijf eerst je test

Voeg het volgende scenario toe aan de `cases`.

```
{
    "struct with two string fields",
    struct {
        Name string
        City string
    }{"Chris", "London"},
    []string{"Chris", "London"},
}
```

## Probeer de test uit te voeren

```
=== RUN   TestWalk/struct_with_two_string_fields
    --- FAIL: TestWalk/struct_with_two_string_fields (0.00s)
        reflection_test.go:40: got [Chris], want [Chris London]
```

## Schrijf genoeg code om de test te laten slagen

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)
		fn(field.String())
	}
}
```

`val` heeft een methode `NumField` die het aantal velden in de waarde retourneert. Hiermee kunnen we over de velden itereren en `fn` aanroepen, wat onze test doet slagen.

## Refactor

Het lijkt er niet op dat er duidelijke refactoringen zijn die de code zouden verbeteren, dus laten we doorgaan.

De volgende tekortkoming van `walk` is dat het ervan uitgaat dat elk veld een `string` is. Laten we een test voor dit scenario schrijven.

## Schrijf eerst de test

Voeg de volgende test case toe

```
{
    "struct with non string field",
    struct {
        Name string
        Age  int
    }{"Chris", 33},
    []string{"Chris"},
},
```

## Probeer de test uit te voeren

```
=== RUN   TestWalk/struct_with_non_string_field
    --- FAIL: TestWalk/struct_with_non_string_field (0.00s)
        reflection_test.go:46: got [Chris <int Value>], want [Chris]
```

## Schrijf genoeg code om de test te laten slagen

We moeten controleren of het type van het veld een `string` is.

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		if field.Kind() == reflect.String {
			fn(field.String())
		}
	}
}
```

Dat kunnen we doen door het soort met [`Kind`](https://pkg.go.dev/reflect#Kind) te controleren.

## Refactor

Ook hier lijkt de code vooralsnog redelijk goed te werken.

Het volgende scenario is: wat als het geen "platte" `struct` is? Met andere woorden, wat gebeurt er als we een `struct` hebben met een aantal geneste velden?

## Schrijf eerst je test

We hebben de anonieme struct-syntaxis gebruikt om ad-hoc typen te declareren voor onze tests, zodat we dat op deze manier konden blijven doen.

```
{
    "nested fields",
    struct {
        Name string
        Profile struct {
            Age  int
            City string
        }
    }{"Chris", struct {
        Age  int
        City string
    }{33, "London"}},
    []string{"Chris", "London"},
},
```

Maar we zien dat de syntaxis wat rommelig wordt wanneer je geneste anonieme structuren krijgt. [Er is een voorstel om de syntaxis mooier te maken](https://github.com/golang/go/issues/12854).

Laten we dit refactoren door een bekend type voor dit scenario te maken en ernaar te verwijzen in de test. Er is een kleine omweg, aangezien een deel van de code voor onze test buiten de test valt, maar lezers zouden de structuur van de `struct` moeten kunnen afleiden door naar de initialisatie te kijken.

Voeg de volgende typedeclaraties ergens in je testbestand toe (persoonlijk zou ik dat boven de tests doen)

```go
type Person struct {
	Name    string
	Profile Profile
}

type Profile struct {
	Age  int
	City string
}
```

Nu kunnen we dit toevoegen aan onze cases, wat veel duidelijker leest dan voorheen

```
{
    "nested fields",
    Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## Probeer de test uit te voeren

```
=== RUN   TestWalk/Nested_fields
    --- FAIL: TestWalk/nested_fields (0.00s)
        reflection_test.go:54: got [Chris], want [Chris London]
```

Het probleem is dat we alleen itereren op de velden op het eerste niveau van de hiërarchie van het type.

## Schrijf genoeg code om de test te laten slagen

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		if field.Kind() == reflect.String {
			fn(field.String())
		}

		if field.Kind() == reflect.Struct {
			walk(field.Interface(), fn)
		}
	}
}
```

De oplossing is vrij eenvoudig: we bekijken opnieuw de soort met de `Kind` functie, en als deze een `struct` is, roepen we gewoon opnieuw een `walk` aan op die geneste `struct`.

## Refactor

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

Wanneer je dezelfde waarde meermaals vergelijkt, verbetert het refactoren naar een `switch` doorgaans de leesbaarheid en zorgt het ervoor dat je code gemakkelijker uit te breiden is.

Maar wat nu als de waarde van de doorgegeven struct een pointer is?

## Schrijf eerste je test

Voeg deze case toe

```
{
    "pointers to things",
    &Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## Probeer de test uit te voeren

```
=== RUN   TestWalk/pointers_to_things
panic: reflect: call of reflect.Value.NumField on ptr Value [recovered]
    panic: reflect: call of reflect.Value.NumField on ptr Value
```

## Schrijf genoeg code om de test te laten slagen

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Pointer {
		val = val.Elem()
	}

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

Je kunt `NumField` niet gebruiken voor een pointer `Value`. We moeten eerst de onderliggende waarde extraheren met behulp van `Elem()`.

## Refactor

Laten we de verantwoordelijkheid van het extraheren van de `reflect.Value` uit een gegeven `interface{}` in een functie verpakken.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}

func getValue(x interface{}) reflect.Value {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Pointer {
		val = val.Elem()
	}

	return val
}
```

Dit voegt weliswaar meer code toe, maar ik vind dat het abstractieniveau hierdoor beter is.

* Haal de `reflect.Value` van `x` op, zodat ik het kan inspecteren, het maakt me niet uit hoe.
* Ittereer over de velden en voer de nodige handelingen uit, afhankelijk van het veldtype.

Vervoilgens moeten we de slices afhandelen

## Schrijf eerst je test

```
{
    "slices",
    []Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## Probeer de test uit te voeren

```
=== RUN   TestWalk/slices
panic: reflect: call of reflect.Value.NumField on slice Value [recovered]
    panic: reflect: call of reflect.Value.NumField on slice Value
```

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Dit is vergelijkbaar met het pointer-scenario hiervoor: we proberen `NumField` aan te roepen op onze `reflect.Value`, maar deze heeft er geen omdat het geen struct is.

## Schrijf genoeg code om de test te laten slagen

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	if val.Kind() == reflect.Slice {
		for i := 0; i < val.Len(); i++ {
			walk(val.Index(i).Interface(), fn)
		}
		return
	}

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

## Refactor

Dit werkt, maar het is niet fraai. Geen zorgen, we hebben werkende code die wordt ondersteund door tests, dus we kunnen naar hartenlust sleutelen.

Als je een beetje abstract denkt, willen we de `walk` op beide aanroepen

* Elk veld in een struct
* Elk _ding_ in een slice

Onze code doet dit momenteel wel, maar geeft de werking niet goed weer. We controleren aan het begin of het een slice is (met een `return` om de rest van de code te stoppen) en als dat niet zo is, gaan we ervan uit dat het een struct is.

Laten we de code aanpassen, zodat we _eerst_ het type controleren en daarna ons werk doen.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	switch val.Kind() {
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walk(val.Field(i).Interface(), fn)
		}
	case reflect.Slice:
		for i := 0; i < val.Len(); i++ {
			walk(val.Index(i).Interface(), fn)
		}
	case reflect.String:
		fn(val.String())
	}
}
```

Dat ziet er veel beter uit! Als het een struct of slice is, itereren we over de waarden ervan en roepen we `walk` aan. Anders, als het een `reflect.String` is, kunnen we `fn` aanroepen.

Toch voelt het voor mij alsof het beter zou kunnen. De handeling van het itereren over velden/waarden en vervolgens het aanroepen van `walk` wordt herhaald, maar conceptueel zijn ze hetzelfde.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice:
		numberOfValues = val.Len()
		getField = val.Index
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

Als de `value` een `reflect.String` is, roepen we gewoon `fn` aan zoals normaal.

Anders zal onze `switch` twee dingen uitvoeren, afhankelijk van het type

* Hoeveel velden er zijn
* Hoe de waarde (`Field` of `Index`) te extraheren

Zodra we deze zaken hebben bepaald, kunnen we door `numberOfValues` ​​itereren en `walk` aanroepen met het resultaat van de `getField`-functie.

Nu we dit hebben gedaan, zou het verwerken van arrays eenvoudig moeten zijn.

## Scrhijf eerst je test

Voeg de volgende cases toe

```
{
    "arrays",
    [2]Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## Probeer de test uit te voeren

```
=== RUN   TestWalk/arrays
    --- FAIL: TestWalk/arrays (0.00s)
        reflection_test.go:78: got [], want [London Reykjavík]
```

## Schrijf genoeg code om de test te laten slagen

Arrays kunnen op dezelfde manier worden behandeld als slices, dus voeg het gewoon toe aan de case met een komma

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice, reflect.Array:
		numberOfValues = val.Len()
		getField = val.Index
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

Het volgende type dat we willen gebruiken is `map`.

## Schrijf eerst je test

```
{
    "maps",
    map[string]string{
        "Cow": "Moo",
        "Sheep": "Baa",
    },
    []string{"Moo", "Baa"},
},
```

## Probeer de test uit te voeren

```
=== RUN   TestWalk/maps
    --- FAIL: TestWalk/maps (0.00s)
        reflection_test.go:86: got [], want [Moo Baa]
```

## Schrijf genoeg code om de test te laten slagen

Als je er wat abstracter over nadenkt, zie je dat `map` erg op `struct` lijkt. Het enige verschil is dat de sleutels tijdens het compileren nog onbekend zijn.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice, reflect.Array:
		numberOfValues = val.Len()
		getField = val.Index
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walk(val.MapIndex(key).Interface(), fn)
		}
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

Het is echter zo dat je per index geen waarden uit een map kunt halen. Dat gebeurt alleen per sleutel, dus dat verstoort onze abstractie, verdorie.

## Refactor

Hoe voelt de code nu? Het voelde eerst misschien als een mooie abstractie, maar nu voelt de code een beetje vreemd aan.

_Dat is oké!_ Refactoring is een proces en soms maken we fouten. Een belangrijk voordeel van TDD is dat het ons de vrijheid geeft om deze dingen uit te proberen.

Door kleine stapjes te zetten, ondersteund door tests, is dit absoluut geen onomkeerbare situatie. Laten we het gewoon terugbrengen naar hoe het was vóór de refactoring.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	}
}
```

We hebben `walkValue` geïntroduceerd, die de aanroepen binnen onze `switch` opschoont, zodat ze alleen de `reflect.Values` ​​uit val hoeven te halen.

### Een laatste probleem

Houd er rekening mee dat maps in Go geen volgorde garanderen. Je tests zullen dus soms mislukken omdat we beweren dat de aanroepen van `fn` in een bepaalde volgorde worden uitgevoerd.

Om dit te verhelpen, moeten we onze bewering met de maps verplaatsen naar een nieuwe test, waarbij de volgorde ons niet uitmaakt.

```go
t.Run("with maps", func(t *testing.T) {
	aMap := map[string]string{
		"Cow":   "Moo",
		"Sheep": "Baa",
	}

	var got []string
	walk(aMap, func(input string) {
		got = append(got, input)
	})

	assertContains(t, got, "Moo")
	assertContains(t, got, "Baa")
})
```

Dit is hoe `assertContains` wordt gedefinieerd

```go
func assertContains(t testing.TB, haystack []string, needle string) {
	t.Helper()
	contains := false
	for _, x := range haystack {
		if x == needle {
			contains = true
		}
	}
	if !contains {
		t.Errorf("expected %v to contain %q but it didn't", haystack, needle)
	}
}
```

Omdat we maps in een nieuwe test hebben geëxtraheerd, hebben we de foutmelding niet gezien. Breek de `with maps`-test hier opzettelijk af, zodat je de foutmelding kunt controleren en deze vervolgens opnieuw kunt oplossen, zodat alle tests slagen.

Het volgende type waar we mee om willen kunnen gaan is `chan`.

## Schrijf eerst je test

```go
t.Run("with channels", func(t *testing.T) {
	aChannel := make(chan Profile)

	go func() {
		aChannel <- Profile{33, "Berlin"}
		aChannel <- Profile{34, "Katowice"}
		close(aChannel)
	}()

	var got []string
	want := []string{"Berlin", "Katowice"}

	walk(aChannel, func(input string) {
		got = append(got, input)
	})

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

## Probeer de test uit te voeren

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_channels (0.00s)
        reflection_test.go:115: got [], want [Berlin Katowice]
```

## Schrijf genoeg code om de test te laten slagen

We kunnen door alle waarden itereren die via het kanaal zijn verzonden totdat het is gesloten met Recv()

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for {
			if v, ok := val.Recv(); ok {
				walkValue(v)
			} else {
				break
			}
		}
	}
}
```

Het volgende type dat we willen afvangen is `func`.

## Schrijf eerst je test

```go
t.Run("with function", func(t *testing.T) {
	aFunction := func() (Profile, Profile) {
		return Profile{33, "Berlin"}, Profile{34, "Katowice"}
	}

	var got []string
	want := []string{"Berlin", "Katowice"}

	walk(aFunction, func(input string) {
		got = append(got, input)
	})

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

## Probeer de test uit te voeren

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_function (0.00s)
        reflection_test.go:132: got [], want [Berlin Katowice]
```

## Schrijf genoeg code om de test te laten slagen

Functies zonder nulargument lijken in dit scenario niet erg zinvol. Maar we moeten wel rekening houden met willekeurige retourwaarden.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for v, ok := val.Recv(); ok; v, ok = val.Recv() {
			walkValue(v)
		}
	case reflect.Func:
		valFnResult := val.Call(nil)
		for _, res := range valFnResult {
			walkValue(res)
		}
	}
}
```

## Samenvattend

* Enkele concepten uit het `reflect`-pakket geïntroduceerd.
* Recursie wordt gebruikt om willekeurige datastructuren te doorkruisen.
* We hebben, achteraf gezien, een slechte refactoring uitgevoerd, maar hebben ons er niet al te druk over gemaakt. Door iteratief met tests te werken, is het niet zo'n groot probleem.
* Dit besloeg slechts een klein aspect van reflection. [De Go-blog heeft een uitstekende post die meer details bevat.](https://blog.golang.org/laws-of-reflection)
* Nu je weet wat reflection inhoudt, weet je dat je dit beter niet meer kunt gebruiken.

# Generics

**[Je kunt hier alle code van dit hoofdstuk vinden](https://github.com/quii/learn-go-with-tests/tree/main/generics)**

Dit hoofdstuk geeft je een introductie tot generieke functies (generics), neemt eventuele bedenkingen erover weg en geeft je een idee hoe je in de toekomst een deel van je code kunt vereenvoudigen. Na het lezen hiervan weet je hoe je het volgende schrijft:

- Een functie die generieke argumenten accepteert
- Een generieke datastructuur


## Onze eigen test helpers (`AssertEqual`, `AssertNotEqual`)

Om generics te verkennen, schrijven we een aantal testhelpers.

### Bevestigen over gehele getallen

Laten we beginnen met iets eenvoudigs en itereren naar ons doel

```go
import "testing"

func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})
}

func AssertEqual(t *testing.T, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d, want %d", got, want)
	}
}

func AssertNotEqual(t *testing.T, got, want int) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %d", got)
	}
}
```


### Beweringen over strings

Het is geweldig om te kunnen asserteren op de gelijkheid van gehele getallen, maar wat als we een assert willen uitvoeren op een string?

```go
t.Run("asserting on strings", func(t *testing.T) {
	AssertEqual(t, "hello", "hello")
	AssertNotEqual(t, "hello", "Grace")
})
```

Je zult een foutmelding krijgen

```
# github.com/quii/learn-go-with-tests/generics [github.com/quii/learn-go-with-tests/generics.test]
./generics_test.go:12:18: cannot use "hello" (untyped string constant) as int value in argument to AssertEqual
./generics_test.go:13:21: cannot use "hello" (untyped string constant) as int value in argument to AssertNotEqual
./generics_test.go:13:30: cannot use "Grace" (untyped string constant) as int value in argument to AssertNotEqual
```

Als je de tijd neemt om de fout te lezen, zul je zien dat de compiler klaagt dat we proberen een `string` door te geven aan een functie die een `integer` verwacht.

#### Korte samenvatting van typeveiligheid (type-safety)

Als je de voorgaande hoofdstukken van dit boek hebt gelezen of ervaring hebt met statisch getypeerde talen, zal dit je niet verbazen. De Go-compiler verwacht dat je je functies, structs, enz. schrijft door te beschrijven met welke typen je wilt werken.

Je kunt geen `string` doorgeven aan een functie die een `integer` verwacht.

Hoewel dit misschien als overdreven aanvoelt, kan het enorm nuttig zijn. Door deze beperkingen te beschrijven,

- Maak je de functie-implementatie eenvoudiger. Door aan de compiler te beschrijven met welke typen je werkt, **beperk je het aantal mogelijke geldige implementaties**. Je kunt geen `Persoon` en een `Bankrekening` "toevoegen". Je kunt een `integer` niet met een hoofdletter schrijven. In software zijn beperkingen vaak enorm nuttig.
- Worden ze voorkomen dat er per ongeluk gegevens aan een functie worden doorgegeven die je niet wilde.

Go biedt je een manier om abstracter te zijn met je typen met [interfaces](./structs-methods-and-interfaces.md), zodat je functies kunt ontwerpen die geen concrete typen gebruiken, maar typen die het gewenste gedrag bieden. Dit geeft je enige flexibiliteit, terwijl de typesafety behouden blijft.

### Een functie die een string of een integer gebruikt? (of eigenlijk andere dingen)

Een andere optie die Go heeft om je functies flexibeler te maken, is door het type van je argument te declareren als `interface{}`, wat "alles" betekent.

Probeer de aanroep parameters aan te passen om in plaats van het 'vaste' type, dit type te gebruiken.


```go
func AssertEqual(got, want interface{})

func AssertNotEqual(got, want interface{})

```

De tests zouden nu moeten compileren en slagen. Als je ze probeert te laten mislukken, zul je zien dat de uitvoer wat zwak is, omdat we de integer `%d`-opmaak gebruiken om onze berichten weer te geven. Verander ze daarom naar de algemene `%+v`-opmaak voor een betere uitvoer van elke waarde.

### Het probleem met `interface{}`

Onze `AssertX`-functies zijn vrij naïef, maar verschillen conceptueel niet veel van de manier waarop andere [populaire bibliotheken deze functionaliteit bieden](https://github.com/matryer/is/blob/master/is.go#L150)

```go
func (is *I) Equal(a, b interface{})
```

Wat is dan het probleem?

Door `interface{}` te gebruiken, kan de compiler ons niet helpen bij het schrijven van onze code, omdat we hem niets nuttigs vertellen over de typen dingen die aan de functie worden doorgegeven. Probeer twee verschillende typen te vergelijken.

```go
AssertEqual(1, "1")
```

In dit geval komen we ermee weg; de test compileert en mislukt zoals we hadden gehoopt, hoewel de foutmelding `got 1, want 1` onduidelijk is. Maar willen we strings met integers kunnen vergelijken? Hoe zit het met het vergelijken van een `Person` met een `Airport`?

Het schrijven van functies die 'interface{}' gebruiken, kan extreem uitdagend en foutgevoelig zijn, omdat we onze beperkingen _kwijt_ zijn en we tijdens het compileren geen informatie hebben over het soort gegevens waarmee we te maken hebben.

Dit betekent dat **de compiler ons niet kan helpen** en dat we in plaats daarvan eerder **runtime-fouten** krijgen die onze gebruikers kunnen beïnvloeden, storingen kunnen veroorzaken of erger.

Vaak moeten ontwikkelaars reflectie gebruiken om deze *ehm* generieke functies te implementeren, wat ingewikkeld kan zijn om te lezen en te schrijven en de prestaties van je programma kan schaden.

## Onze eigen testhelpers met generieke functies

Idealiter willen we geen specifieke `AssertX`-functies hoeven te maken voor elk type waarmee we te maken hebben. We willen _één_ `AssertEqual`-functie die met _elk_ type werkt, maar waarmee je geen [appels met peren](https://en.wikipedia.org/wiki/Apples_and_oranges) kunt vergelijken.

Generieke functies bieden ons een manier om abstracties (zoals interfaces) te maken door ons in staat te stellen **onze beperkingen te beschrijven**. Ze stellen ons in staat om functies te schrijven die een vergelijkbare flexibiliteit hebben als `interface{}`, maar die typesafety behouden en een betere ontwikkelaarservaring bieden voor aanroepers.

```go
func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})

	t.Run("asserting on strings", func(t *testing.T) {
		AssertEqual(t, "hello", "hello")
		AssertNotEqual(t, "hello", "Grace")
	})

	// AssertEqual(t, 1, "1") // uncomment to see the error
}

func AssertEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
}

func AssertNotEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %v", got)
	}
}
```

Om generieke functies in Go te schrijven, moet je "typeparameters" opgeven, wat gewoon een mooie manier is om te zeggen: "beschrijf je generieke type en geef het een label".

In ons geval is het type van onze typeparameter `comparable` en hebben we het label `T` gegeven. Met dit label kunnen we vervolgens de typen van de argumenten van onze functie beschrijven (`got, want T`).

We gebruiken `comparable` omdat we aan de compiler willen laten weten dat we de operatoren `==` en `!=` willen gebruiken voor dingen van het type `T` in onze functie, we willen vergelijken! Als je het type probeert te wijzigen naar `any`,


```go
func AssertNotEqual[T any](got, want T)
```

Krijg je de volgende foutmelding:

```
prog.go2:15:5: cannot compare got != want (operator != not defined for T)
```

Dat is heel logisch, want je kunt die operatoren niet op elk (of `any`) type gebruiken.

### Is een generieke functie met [`T any`](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md#the-constraint) hetzelfde als `interface{}`?

Bekijk deze twee functies:


```go
func GenericFoo[T any](x, y T)
```

```go
func InterfaceyFoo(x, y interface{})
```

Wat is hier het nut van generieke functies? Beschrijft `any` niet... alles?

In termen van beperkingen betekent `any` wel degelijk "alles", net als `interface{}`. Sterker nog, `any` werd toegevoegd in 1.18 en is _gewoon een alias voor `interface{}`_.

Het verschil met de generieke versie is dat je _nog steeds een specifiek type beschrijft_ en dat betekent dat we deze functie nog steeds hebben beperkt tot slechts _één_ type.

Dit betekent dat je `InterfaceyFoo` kunt aanroepen met elke combinatie van typen (bijv. `InterfaceyFoo(apple, orange)`). `GenericFoo` biedt echter nog steeds enkele beperkingen, omdat we hebben aangegeven dat het slechts met _één_ type werkt, `T`.

Geldig:

- `GenericFoo(apple1, apple2)`
- `GenericFoo(orange1, orange2)`
- `GenericFoo(1, 2)`
- `GenericFoo("one", "two")`

Niet geldig (faalt bij compilatie):

- `GenericFoo(apple1, orange1)`
- `GenericFoo("1", 1)`

Als je functie het generieke type retourneert, kan de aanroeper het type ook gebruiken zoals het was, in plaats van dat er een typebevestiging moet worden gemaakt. Als een functie `interface{}` retourneert, kan de compiler namelijk geen garanties geven over het type.

## Volgende: Generieke gegevenstypen

We gaan een [stack](https://en.wikipedia.org/wiki/Stack_(abstract_data_type))-gegevenstype aanmaken. Stacks zouden vanuit het oogpunt van vereisten vrij eenvoudig te begrijpen moeten zijn. Het zijn een verzameling items waarbij je items naar de "top" kunt pushen en om items weer terug te krijgen, kun je items van bovenaf "poppen" (LIFO - last in, first out).

Om het kort te houden, heb ik het TDD-proces weggelaten dat me tot de volgende code voor een stack van `int`s en een stack van `string`s opleverde.


```go
type StackOfInts struct {
	values []int
}

func (s *StackOfInts) Push(value int) {
	s.values = append(s.values, value)
}

func (s *StackOfInts) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfInts) Pop() (int, bool) {
	if s.IsEmpty() {
		return 0, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}

type StackOfStrings struct {
	values []string
}

func (s *StackOfStrings) Push(value string) {
	s.values = append(s.values, value)
}

func (s *StackOfStrings) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfStrings) Pop() (string, bool) {
	if s.IsEmpty() {
		return "", false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

Ik heb een aantal andere assertiefuncties gemaakt om je vooruit te helpen

```go
func AssertTrue(t *testing.T, got bool) {
	t.Helper()
	if !got {
		t.Errorf("got %v, want true", got)
	}
}

func AssertFalse(t *testing.T, got bool) {
	t.Helper()
	if got {
		t.Errorf("got %v, want false", got)
	}
}
```

En hier zijn de tests

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(StackOfInts)

		// check stack is empty
		AssertTrue(t, myStackOfInts.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// add another thing, pop it back again
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())
	})

	t.Run("string stack", func(t *testing.T) {
		myStackOfStrings := new(StackOfStrings)

		// check stack is empty
		AssertTrue(t, myStackOfStrings.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfStrings.Push("123")
		AssertFalse(t, myStackOfStrings.IsEmpty())

		// add another thing, pop it back again
		myStackOfStrings.Push("456")
		value, _ := myStackOfStrings.Pop()
		AssertEqual(t, value, "456")
		value, _ = myStackOfStrings.Pop()
		AssertEqual(t, value, "123")
		AssertTrue(t, myStackOfStrings.IsEmpty())
	})
}
```

### Problemen

- De code voor zowel `StackOfStrings` als `StackOfInts` is vrijwel identiek. Hoewel duplicatie niet altijd het einde van de wereld is, is het wel meer code om te lezen, schrijven en onderhouden.
- Omdat we de logica over twee typen dupliceren, moesten we ook de tests dupliceren.

We willen het _idee_ van een stack in één type vastleggen en er één set tests voor hebben. We zouden nu onze refactoring-hoed moeten opzetten, wat betekent dat we de tests niet moeten wijzigen, omdat we hetzelfde gedrag willen behouden.

Zonder generieke typen is dit wat we _zouden_ kunnen doen


```go
type StackOfInts = Stack
type StackOfStrings = Stack

type Stack struct {
	values []interface{}
}

func (s *Stack) Push(value interface{}) {
	s.values = append(s.values, value)
}

func (s *Stack) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack) Pop() (interface{}, bool) {
	if s.IsEmpty() {
		var zero interface{}
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

- We aliassen onze eerdere implementaties van `StackOfInts` en `StackOfStrings` naar een nieuw, uniform type `Stack`.
- We hebben de typesafety van de `Stack` verwijderd door `values` een [slice](https://github.com/quii/learn-go-with-tests/blob/main/arrays-and-slices.md) van `interface{}` te maken.

Om deze code te proberen, moet je de typebeperkingen van onze assert-functies verwijderen:

```go
func AssertEqual(t *testing.T, got, want interface{})
```

Als je dit doet, slagen onze tests nog steeds. Wie heeft er nog generics nodig?

### Het probleem met het weggooien van typesafety

Het eerste probleem is hetzelfde als dat we zagen met onze `AssertEquals`: we zijn de typesafety kwijt. Ik kan nu appels op een stapel peren `Pushen`.

Zelfs als we de discipline hebben om dit niet te doen, is de code nog steeds onaangenaam om mee te werken, omdat methoden die **`interface{}` retourneren, vreselijk zijn om mee te werken**.

Voeg de volgende test toe:

```go
t.Run("interface stack DX is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()
	AssertEqual(t, firstNum+secondNum, 3)
})
```

Er verschijnt een compilerfout, die de zwakte van het verlies van typeveiligheid aantoont:

```
invalid operation: operator + not defined on firstNum (variable of type interface{})
```

Wanneer `Pop` `interface{}` retourneert, betekent dit dat de compiler geen informatie heeft over de data en daarom onze mogelijkheden ernstig beperkt. De compiler kan niet weten dat het een geheel getal moet zijn, dus staat hij ons de `+` operator niet toe.

Om dit te omzeilen, moet de aanroeper voor elke waarde een [type assertion](https://golang.org/ref/spec#Type_assertions) uitvoeren.

```go
t.Run("interface stack dx is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()

	// get our ints from out interface{}
	reallyFirstNum, ok := firstNum.(int)
	AssertTrue(t, ok) // need to check we definitely got an int out of the interface{}

	reallySecondNum, ok := secondNum.(int)
	AssertTrue(t, ok) // and again!

	AssertEqual(t, reallyFirstNum+reallySecondNum, 3)
})
```

De onaangenaamheid die van deze test uitgaat, zou zich herhalen voor elke potentiële gebruiker van onze `Stack`-implementatie, bah.

### Generieke datastructuren schieten te hulp

Net zoals je generieke argumenten voor functies kunt definiëren, kun je ook generieke datastructuren definiëren.

Hier is onze nieuwe `Stack`-implementatie, met een generiek gegevenstype.

```go
type Stack[T any] struct {
	values []T
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

Hier zie je de tests, die laten zien hoe ze werken zoals wij dat graag zouden zien, met volledige typesafety.

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(Stack[int])

		// check stack is empty
		AssertTrue(t, myStackOfInts.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// add another thing, pop it back again
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// can get the numbers we put in as numbers, not untyped interface{}
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```

Je zult merken dat de syntaxis voor het definiëren van generieke datastructuren consistent is met de syntaxis voor het definiëren van generieke argumenten voor functies.

```go
type Stack[T any] struct {
	values []T
}
```

Het is _bijna_ hetzelfde als voorheen, alleen zeggen we dat het **type van de stack beperkt met welk type waarden je kunt werken**.

Zodra je een `Stack[Orange]` of een `Stack[Apple]` hebt aangemaakt, laten de methoden die op onze stack zijn gedefinieerd je alleen het specifieke type stack waarmee je werkt doorgeven en retourneren:


```go
func (s *Stack[T]) Pop() (T, bool)
```

Je kunt zich voorstellen dat de typen implementaties op de een of andere manier voor jeu worden gegenereerd, afhankelijk van het type stack dat je creëert:

```go
func (s *Stack[Orange]) Pop() (Orange, bool)
```

```go
func (s *Stack[Apple]) Pop() (Apple, bool)
```

Nu we deze refactoring hebben uitgevoerd, kunnen we de string stack test veilig verwijderen, omdat we niet steeds dezelfde logica hoeven te bewijzen.

Merk op dat we tot nu toe in de voorbeelden van het aanroepen van generieke functies de generieke typen niet hoefden te specificeren. Om bijvoorbeeld `AssertEqual[T]` aan te roepen, hoeven we niet te specificeren wat het type `T` is, omdat dit kan worden afgeleid uit de argumenten. In gevallen waarin de generieke typen niet kunnen worden afgeleid, moet je de typen specificeren bij het aanroepen van de functie. De syntaxis is hetzelfde als bij het definiëren van de functie, d.w.z. je specificeert de typen tussen vierkante haken vóór de argumenten.

Voor een concreet voorbeeld kun je overwegen een constructor te maken voor `Stack[T]`.
```go
func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}
```
Om deze constructor te gebruiken om bijvoorbeeld een stapel ints en een stapel strings te maken, roep je hem als volgt aan:
```go
myStackOfInts := NewStack[int]()
myStackOfStrings := NewStack[string]()
```

Hier zie je de `Stack`-implementatie en de tests na het toevoegen van de constructor.

```go
type Stack[T any] struct {
	values []T
}

func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := NewStack[int]()

		// check stack is empty
		AssertTrue(t, myStackOfInts.IsEmpty())

		// add a thing, then check it's not empty
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// add another thing, pop it back again
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// can get the numbers we put in as numbers, not untyped interface{}
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```

Met behulp van een generiek gegevenstype hebben we:

- Verminderde duplicatie van belangrijke logica.
- `Pop` retourneert `T`, zodat we bij het aanmaken van een `Stack[int]` in de praktijk `int` van `Pop` terugkrijgen; we kunnen nu `+` gebruiken zonder dat we type-assertie-gymnastiek hoeven te doen.
- Misbruik tijdens compileren voorkomen. Je kunt geen sinaasappels `Pushen` naar een appel stack.

## Samenvattend

Dit hoofdstuk zou je een voorproefje moeten hebben gegeven van de syntaxis van generieke typen en enkele ideeën moeten hebben gegeven waarom generieke typen nuttig kunnen zijn. We hebben onze eigen `Assert`-functies geschreven die we veilig kunnen hergebruiken om te experimenteren met andere ideeën rond generieke typen, en we hebben een eenvoudige datastructuur geïmplementeerd om elk gewenst type data op een typeveilige manier op te slaan.

### Generics zijn in de meeste gevallen eenvoudiger dan `interface{}`

Als je geen ervaring hebt met statisch getypeerde talen, is het nut van generics misschien niet meteen duidelijk, maar ik hoop dat de voorbeelden in dit hoofdstuk hebben geïllustreerd waar de Go-taal niet zo expressief is als we zouden willen. Met name het gebruik van `interface{}` maakt je code:

- Minder veilig (een mix van appels en peren), vereist meer foutafhandeling
- Minder expressief, `interface{}` vertelt je niets over de data
- Waarschijnlijker dat het vertrouwt op [reflectie](https://github.com/bobkosse/leer-go-met-tests/basisbeginselen-go/reflection.md), type-asserties, enz., wat je code lastiger te gebruiken en foutgevoeliger maakt, omdat het controles van compile-time naar runtime verplaatst

Het gebruik van statisch getypeerde talen is een manier om beperkingen te beschrijven. Als je het goed doet, creëer je code die niet alleen veilig en eenvoudig te gebruiken is, maar ook eenvoudiger te schrijven, omdat de mogelijke oplossingsruimte kleiner is.

Generics biedt ons een nieuwe manier om beperkingen in onze code uit te drukken. Zoals aangetoond, stelt dit ons in staat om code te samen te voegen en te vereenvoudigen die tot Go 1.18 niet mogelijk was.

### Zullen generics Go in Java veranderen?

- Nee.

Er is veel [FUD (fear, uncertainty and doubt)](https://en.wikipedia.org/wiki/Fear,_uncertainty,_and_doubt) in de Go-community over het feit dat generics leiden tot nachtmerrieachtige abstracties en verbijsterende codebases. Dit wordt meestal gecompenseerd door "ze moeten zorgvuldig worden gebruikt".

Hoewel dit waar is, is het geen bijzonder nuttig advies, omdat dit voor elke taalfunctie geldt.

Niet veel mensen klagen over ons vermogen om interfaces te definiëren, wat net als generics een manier is om beperkingen in onze code te beschrijven. Wanneer je een interface beschrijft, maak je een ontwerpkeuze die _slecht zou kunnen zijn_. generics zijn niet uniek in hun vermogen om code verwarrend en vervelend te maken.

### Je gebruikt al generics

Als je bedenkt dat als je arrays, slices of maps hebt gebruikt, je _al een consument van generics_ bent geweest.

```
var myApples []Apple
// You can't do this!
append(myApples, Orange{})
```

### Abstractie is geen vies woord

Het is makkelijk om [AbstractSingletonProxyFactoryBean](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/framework/AbstractSingletonProxyFactoryBean.html) te gebruiken, maar laten we niet doen alsof een codebase zonder enige abstractie niet net zo slecht is. Het is jouw taak om gerelateerde concepten te _verzamelen_ wanneer dat nodig is, zodat je systeem gemakkelijker te begrijpen en te wijzigen is; in plaats van een verzameling van uiteenlopende functies en typen met een gebrek aan duidelijkheid.

### [Laat het werken, maak het goed, maak het snel](https://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast#:~:text=%22Make%20it%20work%2C%20make%20it,to%20DesignForPerformance%20ahead%20of%20time.)

Mensen lopen tegen problemen aan met generics wanneer ze te snel abstraheren zonder voldoende informatie om goede ontwerpbeslissingen te nemen.

De TDD-cyclus van rood, groen, refactoring betekent dat je meer richting hebt met betrekking tot welke code je _werkelijk_ nodig hebt om je gedrag te leveren, **in plaats van dat je van tevoren abstracties bedenkt**; maar je moet nog steeds voorzichtig zijn.

Er zijn hier geen vaste regels, maar maak dingen pas generiek als je ziet dat je een bruikbare generalisatie hebt. Toen we de verschillende `Stack`-implementaties creëerden, begonnen we met concreet gedrag zoals `StackOfStrings` en `StackOfInts`, ondersteund door tests. Vanuit onze echte code konden we echte patronen zien, en op basis van onze tests konden we refactoring onderzoeken voor een meer algemene oplossing.

Developers adviseren vaak om pas te generaliseren als je dezelfde code drie keer ziet, wat een goede vuistregel lijkt.

Een veelvoorkomend pad dat ik in andere programmeertalen heb gevolgd, is:

- Eén TDD-cyclus om bepaald gedrag aan te sturen
- Nog een TDD-cyclus om andere gerelateerde scenario's te oefenen

> Hmm, deze dingen lijken op elkaar, maar een beetje duplicatie is beter dan koppeling aan een slechte abstractie

- Slaap er een nachtje over
- Een nieuwe TDD-cyclus

> Oké, ik wil graag proberen of ik dit kan generaliseren. Gelukkig ben ik zo slim en knap omdat ik TDD gebruik, dus ik kan refactoren wanneer ik wil. Het proces heeft me geholpen te begrijpen welk gedrag ik echt nodig heb voordat ik te veel ontwerp.

- Deze abstractie voelt prettig! De tests slagen nog steeds en de code is eenvoudiger.
- Ik kan nu een aantal tests verwijderen. Ik heb de _essentie_ van het gedrag vastgelegd en onnodige details verwijderd.

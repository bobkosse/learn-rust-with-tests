# Structs, methods & interfaces

[**Je kunt hier alle code voor dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/arrays)

Stel dat we wat geometrische code nodig hebben om de omtrek van een rechthoek te berekenen op basis van een hoogte en breedte. We kunnen een functie `Perimeter(breedte float64, hoogte float64)` schrijven, waarbij `float64` voor kommagetallen zoals `123,45` bedoeld is.

De TDD cyclus zal je inmiddels bekend zijn.

## Schrijf eerst je test

```go
func TestPerimeter(t *testing.T) {
	got := Perimeter(10.0, 10.0)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

Zie je de nieuwe formatstring? De `f` staat voor onze `float64` en de `.2` betekent dat er 2 decimalen worden afgedrukt.

Let ook op dat decimale getallen in programmeertalen met een _decimale punt_ geschreven worden. Waar in het Nederlands vaak komma's gebruikt worden, is dat in programmeertalen dus de Engelse notatie.

## Probeer de test uit te voeren

`./shapes_test.go:6:9: undefined: Perimeter`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func Perimeter(width float64, height float64) float64 {
	return 0
}
```

Dit resulteert in: `shapes_test.go:10: got 0.00 want 40.00`.

## Schrijf genoeg code om de test te laten slagen

```go
func Perimeter(width float64, height float64) float64 {
	return 2 * (width + height)
}
```

Tot zover is het nog vrij eenvoudig. Laten we nu een functie genaamd `Area(breedte, hoogte, float64)` maken die de oppervlakte van een rechthoek retourneert.

Probeer dit eerst eens zelf te doen door de TDD cyclus te volgen.

Je zou uit moeten komen op tests die hierop lijken:

```go
func TestPerimeter(t *testing.T) {
	got := Perimeter(10.0, 10.0)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}

func TestArea(t *testing.T) {
	got := Area(12.0, 6.0)
	want := 72.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

En code zoals de onderstaande

```go
func Perimeter(width float64, height float64) float64 {
	return 2 * (width + height)
}

func Area(width float64, height float64) float64 {
	return width * height
}
```

## Refactor

Onze code doet wat het moet doen, maar bevat geen expliciete informatie over rechthoeken. Een onoplettende ontwikkelaar zou kunnen proberen de breedte en hoogte van een driehoek aan deze functies te verstrekken zonder te beseffen dat ze het verkeerde antwoord zullen retourneren.

We zouden de functies specifiekere namen kunnen geven, zoals `RectangleArea`. Een nettere oplossing is om ons eigen type te definiëren, genaamd `Rectangle`, dat dit concept voor ons omvat.

We kunnen een eenvoudig type maken met behulp van een **struct**. [Een struct](https://golang.org/ref/spec#Struct_types) is gewoon een benoemde verzameling velden waarin je gegevens kunt opslaan.

Declareer een struct in je `shapes.go` bestand zoals hieronder

```go
type Rectangle struct {
	Width  float64
	Height float64
}
```

Laten we de tests nu refactoren zodat `Rectangle` wordt gebruikt in plaats van gewone `float64`'s.

```go
func TestPerimeter(t *testing.T) {
	rectangle := Rectangle{10.0, 10.0}
	got := Perimeter(rectangle)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}

func TestArea(t *testing.T) {
	rectangle := Rectangle{12.0, 6.0}
	got := Area(rectangle)
	want := 72.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

Vergeet niet om je tests uit te voeren voordat je probeert het probleem te verhelpen. De tests zouden een nuttige foutmelding moeten laten zien, zoals:

```
./shapes_test.go:7:18: not enough arguments in call to Perimeter
    have (Rectangle)
    want (float64, float64)
```

Je kunt de velden van een struct benaderen met de syntaxis `myStruct.field`.

Verander de twee functies zodat de test werkt.

```go
func Perimeter(rectangle Rectangle) float64 {
	return 2 * (rectangle.Width + rectangle.Height)
}

func Area(rectangle Rectangle) float64 {
	return rectangle.Width * rectangle.Height
}
```

Ik hoop dat je het ermee eens bent dat het doorgeven van een `Rectangle` aan een functie onze bedoeling duidelijker overbrengt. Er zijn echter nog meer voordelen aan het gebruik van structs die we later zullen bespreken.

Onze volgende vereiste is het schrijven van een `Area` functie voor cirkels.

## Schrijf eerst je test

```go
func TestArea(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		got := Area(rectangle)
		want := 72.0

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		got := Area(circle)
		want := 314.1592653589793

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

}
```

Zoals je ziet, is de `f` vervangen door `g`, en dat is niet voor niets. Het gebruik van `g` zorgt ervoor dat er een nauwkeuriger decimaal getal in de foutmelding wordt weergegeven ([fmt-opties](https://golang.org/pkg/fmt/)). Bijvoorbeeld, bij een straal van 1,5 in een cirkeloppervlakteberekening, zou `f` `7,068583` opleveren, terwijl `g` een waarde van `7,0685834705770345` zou opleveren.

## Probeer de test uit te voeren

`./shapes_test.go:28:13: undefined: Circle`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

We moeten het `Circle` type definiëren.

```go
type Circle struct {
	Radius float64
}
```

Probeer de test nu opnieuw uit te voeren

`./shapes_test.go:29:14: cannot use circle (type Circle) as type Rectangle in argument to Area`

Sommige programmeertalen bieden de mogelijkheid om zoiets te doen:

```go
func Area(circle Circle) float64       {}
func Area(rectangle Rectangle) float64 {}
```

Maar dat werkt niet binnen Go

`./shapes.go:20:32: Area redeclared in this block`

We hebben hier twee keuzes:

* Je kunt functies met dezelfde naam in verschillende _pakketten_ laten declareren. We zouden onze `Area(Circle)` dus in een nieuw pakket kunnen aanmaken, maar dat voelt hier overdreven.
* In plaats daarvan kunnen we [methods](https://golang.org/ref/spec#Method_declarations) definiëren voor onze nieuw gedefinieerde typen.

### Wat zijn methods?

Tot nu toe hebben we alleen _functies_ geschreven, maar we hebben ook enkele _methods_ gebruikt. Wanneer we `t.Errorf` aanroepen, roepen we de _method_ `Errorf` aan op de instantie van onze `t` (`testing.T`).

Een _method_ is een functie met een ontvanger. Een _method_-declaratie koppelt een identificatie, de _method_-naam, aan een _method_ en koppelt de methode aan het basistype van de ontvanger.

_Methods_ lijken erg op functies, maar ze worden aangeroepen door ze aan te roepen op een instantie van een bepaald type. Waar je functies gewoon kunt aanroepen waar je maar wilt, zoals `Area(rectangle)`, kun je _methods_ alleen aanroepen op "dingen".

Een voorbeeld kan helpen, dus laten we eerst onze tests aanpassen en _methods_ aanroepen. Daarna gaan we de code aanpassen.

```go
func TestArea(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		got := rectangle.Area()
		want := 72.0

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		got := circle.Area()
		want := 314.1592653589793

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

}
```

Als we proberen deze tests uit te voeren krijgen we

```
./shapes_test.go:19:19: rectangle.Area undefined (type Rectangle has no field or method Area)
./shapes_test.go:29:16: circle.Area undefined (type Circle has no field or method Area)
```

> type Circle has no field or method Area

Ik wil nogmaals benadrukken hoe geweldig de compiler hier is. Het is zo belangrijk om de tijd te nemen om de foutmeldingen die je krijgt rustig door te lezen, het zal je op de lange termijn helpen.

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Laten we enkele methoden aan onze typen toevoegen

```go
type Rectangle struct {
	Width  float64
	Height float64
}

func (r Rectangle) Area() float64 {
	return 0
}

type Circle struct {
	Radius float64
}

func (c Circle) Area() float64 {
	return 0
}
```

De syntaxis voor het declareren van methoden is vrijwel gelijk aan die van functies, omdat ze zo op elkaar lijken. Het enige verschil is de syntaxis van de methode-ontvanger. `func (receiverName ReceiverType) MethodName(args)`.

Wanneer je methode wordt aangeroepen voor een variabele van dat type, krijg je de referentie naar de gegevens via de variabele `receiverName`. In veel andere programmeertalen gebeurt dit impliciet en krijg je toegang tot de ontvanger via het `this`-sleutelwoord.

In Go is het gebruikelijk dat de ontvangende variabele de eerste letter van het type is.

```
r Rectangle
```

Als je de tests opnieuw wilt uitvoeren, worden ze nu gecompileerd en krijg je een mislukte uitvoer.

## Schrijf genoeg code om de test te laten slagen

Laten we nu onze rechthoektests laten slagen door onze nieuwe methode te repareren

```go
func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}
```

Als je de tests opnieuw uitvoert, zouden de rechthoek-tests moeten slagen, maar de cirkel-tests zouden nog steeds moeten mislukken.

Om de `Aria` functie van de cirkel te laten slagen, lenen we de `Pi`-constante uit het `math` pakket (vergeet niet deze te importeren).

```go
func (c Circle) Area() float64 {
	return math.Pi * c.Radius * c.Radius
}
```

## Refactor

Er is enige overlapping in onze testen.

Het enige wat we willen doen is een verzameling _shapes_ nemen, de `Area()`-methode erop aanroepen en vervolgens het resultaat controleren.

We willen een soort `checkArea`-functie schrijven waaraan we zowel `Rectangle`s als `Circle`s kunnen doorgeven, maar die niet kan worden gecompileerd als we iets anders dan een vorm proberen door te geven.

Met Go kunnen we deze intentie vastleggen met **interfaces**.

[Interfaces](https://golang.org/ref/spec#Interface_types) vormen een zeer krachtig concept in statisch getypeerde talen zoals Go, omdat je hiermee functies kunt maken die met verschillende typen kunnen worden gebruikt. Ook kun je hiermee sterk ontkoppelde code creëren, terwijl de typeveiligheid behouden blijft.

Laten we dit introduceren door onze tests te refactoren.

```go
func TestArea(t *testing.T) {

	checkArea := func(t testing.TB, shape Shape, want float64) {
		t.Helper()
		got := shape.Area()
		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	}

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		checkArea(t, rectangle, 72.0)
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		checkArea(t, circle, 314.1592653589793)
	})

}
```

We maken een hulpfunctie zoals we die in andere oefeningen hebben gedaan, maar deze keer vragen we om een ​​vorm (`Shape`) mee te geven. Als we deze functie proberen aan te roepen met iets dat geen vorm is, zal deze niet compileren.

Hoe wordt iets een vorm? We vertellen Go gewoon wat een `Shape` is met behulp van een interfacedeclaratie.

```go
type Shape interface {
	Area() float64
}
```

We maken een nieuw `type`, net zoals we dat met `Rectangle` en `Circle` deden, maar dit keer is het een `interface` in plaats van een `struct`.

Zodra je dit aan de code toevoegt, zullen de tests slagen.

### Wacht, wat?

Dit verschilt aanzienlijk van interfaces in de meeste andere programmeertalen. Normaal gesproken moet je code schrijven om bijvoorbeeld te zeggen: `Mijn type Foo implementeert interface Bar`.

Maar in ons geval

* `Rectangle` heeft een methode genaamd `Area` die een `float64` retourneert, zodat deze voldoet aan de `Shape`-interface
* `Circle` heeft een method genaamd `Area` die een `float64` retourneert en voldoet dus aan de `Shape`-interface.
* `String` heeft geen dergelijke methode en voldoet dus niet aan de interface.
* Enz.

In de Go-**interface is resolutie impliciet**. Als het type dat je opgeeft overeenkomt met wat de interface vraagt, wordt het gecompileerd.

### Decoupling

Merk op hoe onze helper zich niet hoeft te bekommeren om de vraag of de vorm een ​​`Rectangle`, `Circle` of `Triangle` is. Door een interface te declareren, wordt de helper losgekoppeld (decoupled) van de concrete typen en beschikt hij alleen over de methode die hij nodig heeft om zijn werk te doen.

Deze aanpak, waarbij interfaces **alleen datgene aangeven wat je nodig hebt**, is erg belangrijk bij softwareontwerp. In latere secties wordt hier dieper op ingegaan.

## Further refactoring

Nu je enige kennis hebt van structs, kunnen we "table driven tests" introduceren.

[Table driven tests](https://go.dev/wiki/TableDrivenTests) zijn handig als je een lijst met testcases wilt samenstellen die op dezelfde manier kunnen worden getest.

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		shape Shape
		want  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Circle{10}, 314.1592653589793},
	}

	for _, tt := range areaTests {
		got := tt.shape.Area()
		if got != tt.want {
			t.Errorf("got %g want %g", got, tt.want)
		}
	}

}
```

De enige nieuwe syntaxis hier is het aanmaken van een "anonieme struct", `areaTests`. We declareren een deel van de structs met behulp van `[]struct` met twee velden: de `shape` en de `want`. Vervolgens vullen we het deel met cases.

Vervolgens itereren we eroverheen, net zoals we met elke andere slice doen, waarbij we de struct-velden gebruiken om onze tests uit te voeren.

Je ziet hoe eenvoudig het voor een ontwikkelaar is om een ​​nieuwe shape te introduceren, `Area` te implementeren en deze vervolgens aan de testcases toe te voegen. Bovendien is het, als er een bug in `Area` wordt gevonden, heel eenvoudig om een ​​nieuwe testcase toe te voegen om de bug te testen voordat deze wordt opgelost.

Table driven tests kunnen een waardevolle toevoeging zijn aan je gereedschapskist, maar zorg ervoor dat je noodzaak voor de extra ruis in de tests echt nodig hebt. Ze zijn zeer geschikt wanneer je verschillende implementaties van een interface wilt testen, of als de data die aan een functie wordt doorgegeven veel verschillende vereisten heeft die getest moeten worden.

Laten we dit allemaal demonstreren door een andere vorm toe te voegen en te testen: een driehoek.

## Schrijf eerst je test

Het toevoegen van een nieuwe test voor onze nieuwe vorm is heel eenvoudig. Voeg gewoon `{Triangle{12, 6}, 36.0}` toe aan onze lijst.

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		shape Shape
		want  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Circle{10}, 314.1592653589793},
		{Triangle{12, 6}, 36.0},
	}

	for _, tt := range areaTests {
		got := tt.shape.Area()
		if got != tt.want {
			t.Errorf("got %g want %g", got, tt.want)
		}
	}

}
```

## Voer de test uit

Vergeet niet dat je de test moet blijven proberen en dat de compiler je naar een oplossing moet leiden.

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

`./shapes_test.go:25:4: undefined: Triangle`

We hebben `Triangle` nog niet gedefinieerd

```go
type Triangle struct {
	Base   float64
	Height float64
}
```

Probeer het opnieuw

```
./shapes_test.go:25:8: cannot use Triangle literal (type Triangle) as type Shape in field value:
    Triangle does not implement Shape (missing Area method)
```

Het vertelt ons dat we een Triangle niet als vorm kunnen gebruiken omdat deze geen `Area()`-methode heeft, dus voeg een lege implementatie toe om de test werkend te krijgen

```go
func (t Triangle) Area() float64 {
	return 0
}
```

Uiteindelijk compileert de code en krijgen we onze foutmelding te zien

`shapes_test.go:31: got 0.00 want 36.00`

## Schrijf genoeg code om de test te laten slagen

```go
func (t Triangle) Area() float64 {
	return (t.Base * t.Height) * 0.5
}
```

En onze tests slagen!

## Refactor

En opnieuw is de implementatie prima, maar onze tests zouden nog wel wat verbetering kunnen gebruiken.

Wanneer je dit ziet:

```
{Rectangle{12, 6}, 72.0},
{Circle{10}, 314.1592653589793},
{Triangle{12, 6}, 36.0},
```

is het niet meteen duidelijk wat alle getallen voorstellen. Zorg er daarom voor dat je tests gemakkelijk te begrijpen zijn.

Tot nu toe hebben we alleen de syntaxis getoond voor het maken van instanties van de structs `MyStruct{val1, val2}`, maar je kunt de velden optioneel een naam geven.

Laten we eens kijken hoe dat eruit ziet

```
        {shape: Rectangle{Width: 12, Height: 6}, want: 72.0},
        {shape: Circle{Radius: 10}, want: 314.1592653589793},
        {shape: Triangle{Base: 12, Height: 6}, want: 36.0},
```

In [Test-Driven Development by Example](https://app.gitbook.com/u/0vinjXPm7bXI716vVEHCE1JF7q02), refactort Kent Beck enkele tests tot een bepaald punt en beweert:

> De test spreekt ons duidelijker aan, alsof het een bewering van de waarheid is, **en geen reeks handelingen**.

(de nadruk in het citaat is van mij)

Onze tests (of beter gezegd de lijst met testcases) doen nu uitspraken over de waarheid van vormen en hun oppervlakken.

## Wees er zeker van dat je test uitkomsten behulpzaam zijn.

Weet je nog toen we `Triangle` implementeerden en de test mislukte? Het gaf `shapes_test.go:31: got 0.00 want 36.00`.

We wisten dat dit betrekking had op `Triangle`, omdat we er net mee werkten. Maar wat als er in een van de twintig cases in de tabel een bug in het systeem sluipt? Hoe weet een ontwikkelaar dan welke case mislukt is? Dit is niet prettig voor de ontwikkelaar; hij of zij moet handmatig alle cases doornemen om te achterhalen welke case daadwerkelijk mislukt is.

We kunnen onze foutmelding wijzigen in `%#v got %g want %g`. De `%#v`-opmaakstring print onze struct met de waarden in het veld, zodat de ontwikkelaar in één oogopslag kan zien welke eigenschappen worden getest.

Om de leesbaarheid van onze testcases verder te vergroten, kunnen we het `want`-veld hernoemen naar iets meer beschrijvend, zoals `hasArea`.

Een laatste tip voor table driven tests is om `t.Run` te gebruiken en de testcases een naam te geven.

Door elk geval in een `t.Run` te wikkelen, krijg je een duidelijker testresultaat bij fouten, omdat de naam van het geval wordt afgedrukt.

```
--- FAIL: TestArea (0.00s)
    --- FAIL: TestArea/Rectangle (0.00s)
        shapes_test.go:33: main.Rectangle{Width:12, Height:6} got 72.00 want 72.10
```

En je kunt specifieke tests binnen de tabel uitvoeren met `go test -run TestArea/Rectangle`.

Hier is onze laatste testcode die dit vastlegt

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		name    string
		shape   Shape
		hasArea float64
	}{
		{name: "Rectangle", shape: Rectangle{Width: 12, Height: 6}, hasArea: 72.0},
		{name: "Circle", shape: Circle{Radius: 10}, hasArea: 314.1592653589793},
		{name: "Triangle", shape: Triangle{Base: 12, Height: 6}, hasArea: 36.0},
	}

	for _, tt := range areaTests {
		// using tt.name from the case to use it as the `t.Run` test name
		t.Run(tt.name, func(t *testing.T) {
			got := tt.shape.Area()
			if got != tt.hasArea {
				t.Errorf("%#v got %g want %g", tt.shape, got, tt.hasArea)
			}
		})

	}

}
```

## Samenvattend

Dit was meer een TDD-oefening, waarbij we door onze oplossingen voor eenvoudige wiskundige problemen heen itereerden en nieuwe taalkenmerken leerden, gemotiveerd door onze tests.

* Het declareren van structs om je eigen gegevenstypen te creëren waarmee je gerelateerde gegevens kunt bundelen en de bedoeling van je code duidelijker kunt maken
* Interfaces declareren zodat je functies kunt definiëren die door verschillende typen kunnen worden gebruikt ([parametrisch polymorfisme](https://en.wikipedia.org/wiki/Parametric_polymorphism))
* Methoden toevoegen zodat je functionaliteit aan je gegevenstypen kunt toevoegen en interfaces kunt implementeren
* Table driven tests om je beweringen duidelijker te maken en je testsuites eenvoudiger uit te breiden en te onderhouden

Dit was een belangrijk hoofdstuk, omdat we nu beginnen met het definiëren van onze eigen typen. In static typed talen zoals Go is het kunnen ontwerpen van je eigen typen essentieel voor het bouwen van software die gemakkelijk te begrijpen, samen te stellen en te testen is.

Interfaces zijn een geweldig hulpmiddel om complexiteit te verbergen voor andere delen van het systeem. In ons geval hoefde onze testhelpe&#x72;_&#x63;ode_ niet te weten op welke vorm hij precies een claim legde, alleen hoe hij om de oppervlakte ervan moest "vragen".

Naarmate je meer vertrouwd raakt met Go, zul je de echte kracht van interfaces en de standaardbibliotheek gaan zien. Je leert over interfaces die in de standaardbibliotheek zijn gedefinieerd en overal worden gebruikt. Door ze te implementeren op je eigen typen, kun je heel snel veel geweldige functionaliteit hergebruiken.

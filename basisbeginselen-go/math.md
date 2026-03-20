# Wiskunde

[**Je kunt hier alle code van dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/math)

Ondanks alle kracht van moderne computers om razendsnel enorme sommen te berekenen, gebruikt de gemiddelde ontwikkelaar zelden wiskunde om zijn werk te doen. Maar vandaag zal anders zijn! Vandaag gebruiken we wiskunde om een ​&#x200B;_&#x65;cht_ probleem op te lossen. En geen saaie wiskunde, we gaan goniometrie, vectoren en allerlei andere dingen gebruiken waarvan je altijd zei dat je ze na de middelbare school nooit meer zou hoeven gebruiken.

## Het probleem

Je wilt een SVG maken van een klok. Geen digitale klok, dat zou veel te makkelijk zijn, maar een analoge klok, met wijzers. Je zoekt niets bijzonders, gewoon een handige functie die een tijd uit het `Time` pakket haalt en een SVG van een klok genereert met alle wijzers: uur, minuut en seconde, die in de juiste richting wijzen. Hoe moeilijk kan dat zijn?

Eerst hebben we een SVG van een klok nodig om mee te spelen. SVG's zijn een fantastisch afbeeldingsformaat om programmatisch te bewerken, omdat ze geschreven zijn als een reeks vormen, beschreven in XML. Dus deze klok:

![een svg van een klok](../.gitbook/assets/example_clock.svg)

is als volgt beschreven:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">

  <!-- bezel -->
  <circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>

  <!-- hour hand -->
  <line x1="150" y1="150" x2="114.150000" y2="132.260000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>

  <!-- minute hand -->
  <line x1="150" y1="150" x2="101.290000" y2="99.730000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>

  <!-- second hand -->
  <line x1="150" y1="150" x2="77.190000" y2="202.900000"
        style="fill:none;stroke:#f00;stroke-width:3px;"/>
</svg>
```

Het is een cirkel met drie lijnen. Elke lijn begint in het midden van de cirkel (x=150, y=150) en eindigt op enige afstand.

Wat we dus gaan doen is het bovenstaande op een of andere manier reconstrueren, maar dan veranderen we de lijnen zodat ze in de juiste richting wijzen voor een bepaalde tijd.

## Een acceptatie test

Voordat we er te diep op ingaan, laten we eerst eens nadenken over een acceptatietest.

Wacht even, je weet nog niet wat een acceptatietest is. Laat me het proberen uit te leggen.

Laat me je eens het volgende vragen: hoe ziet winnen eruit? Hoe weten we dat we klaar zijn met werken? TDD biedt een goede manier om te weten wanneer je klaar bent: wanneer de test slaagt. Soms is het fijn, eigenlijk bijna altijd, om een ​​test te schrijven die aangeeft wanneer je klaar bent met het schrijven van de hele bruikbare feature. Niet alleen een test die aangeeft dat een bepaalde functie werkt zoals je verwacht, maar een test die aangeeft dat alles wat je probeert te bereiken, de 'feature'. voltooid is.

Deze tests worden soms 'acceptatietests' of 'feature tests' genoemd. Het idee is dat je een test op hoog niveau schrijft om te beschrijven wat je probeert te bereiken: een gebruiker klikt bijvoorbeeld op een knop op een website en ziet een complete lijst met de gevangen Pokémon. Zodra we die test hebben geschreven, kunnen we meer tests schrijven, de unit tests, die toewerken naar een werkend systeem dat de acceptatietest zal doorstaan. In ons voorbeeld kunnen deze tests bijvoorbeeld gaan over het weergeven van een webpagina met een knop, het testen van route handlers op een webserver, het uitvoeren van database-opzoekingen, enzovoort. Al deze dingen worden tests volgens de TDD-aanpak en dragen allemaal bij aan het slagen van de oorspronkelijke acceptatietest.

Iets als deze _klassieke_ afbeelding van Nat Pryce en Steve Freeman

![Van buiten naar binnen feedbackloops in TDD](../.gitbook/assets/TDD-outside-in.jpg)

Hoe dan ook, laten we proberen die acceptatietest te schrijven, de test die ons laat weten wanneer we klaar zijn.

We hebben een voorbeeldklok, dus laten we eens nadenken over de belangrijke parameters.

```
<line x1="150" y1="150" x2="114.150000" y2="132.260000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>
```

Het middelpunt van de klok (de attributen `x1` en `y1` voor deze lijn) is voor elke wijzer hetzelfde. De getallen die voor elke wijzer moeten veranderen, de parameters voor de SVG, zijn de attributen `x2` en `y2`. We hebben een X en een Y nodig voor elke wijzer van de klok.

Ik zou aan meer parameters kunnen denken, de straal van de cirkel van de wijzerplaat, de grootte van de SVG, de kleuren van de wijzers, hun vorm, etc. Maar het is beter om te beginnen met het oplossen van een eenvoudig, concreet probleem met een eenvoudige, concrete oplossing, en dan parameters toe te voegen om de oplossing te generaliseren.

Laten we dus van het volgende uitgaan:

* iedere klok heeft een middelpunt op (150, 150)
* de uren wijzer is 50 lang&#x20;
* de minutenwijzer is 80 lang
* de secondenwijzer is 90 lang.

Let op: SVG's hebben een punt nodig: de oorsprong, punt (0,0), bevindt zich in de _linkerbovenhoek_, niet _linksonder_ zoals we misschien zouden verwachten. Het is belangrijk om dit te onthouden wanneer we bepalen waar we welke getallen in onze lijnen moeten invoeren.

Tot slot bepaal ik nog niet hoe we de SVG gaan maken, we zouden een sjabloon uit het  [`text/template`](https://golang.org/pkg/text/template/) pakket kunnen gebruiken, of we zouden bytes gewoon naar een `bytes.Buffer` of een andere `writer` kunnen sturen. Maar we weten dat we die getallen nodig hebben, dus laten we ons concentreren op het testen van iets dat ze genereert.

### Schrijf eerst je test

Mijn eerste test ziet er dus zo uit:

```go
package clockface_test

import (
	"projectpath/clockface"
	"testing"
	"time"
)

func TestSecondHandAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 - 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

Weet je nog hoe SVG's hun coördinaten vanuit de linkerbovenhoek weergeven? Om de secondewijzer op middernacht te plaatsen, verwachten we dat deze niet van het midden van de wijzerplaat op de X-as is verplaatst, nog steeds 150, en dat de Y-as de lengte van de wijzer 'omhoog' vanaf het midden is; 150 min 90.

### Probeer de test uit te voeren

Hiermee worden de verwachte fouten rondom de ontbrekende functies en typen verholpen:

```
--- FAIL: TestSecondHandAtMidnight (0.00s)
./clockface_test.go:13:10: undefined: clockface.Point
./clockface_test.go:14:9: undefined: clockface.SecondHand
```

Dus een `Point` waar de punt van de secondewijzer moet komen, en een functie om dat te krijgen.

### Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

Laten we deze typen implementeren om de code te laten compileren

```go
package clockface

import "time"

// A Point represents a two-dimensional Cartesian coordinate
type Point struct {
	X float64
	Y float64
}

// SecondHand is the unit vector of the second hand of an analogue clock at time `t`
// represented as a Point.
func SecondHand(t time.Time) Point {
	return Point{}
}
```

en nu krijgen we:

```
--- FAIL: TestSecondHandAtMidnight (0.00s)
    clockface_test.go:17: Got {0 0}, wanted {150 60}
FAIL
exit status 1
FAIL	learn-go-with-tests/math/clockface	0.006s
```

### Schrijf genoeg code om de test te laten slagen

Wanneer we de verwachte fout krijgen, kunnen we de retourwaarde invullen van `SecondHand`:

```go
// SecondHand is the unit vector of the second hand of an analogue clock at time `t`
// represented as a Point.
func SecondHand(t time.Time) Point {
	return Point{150, 60}
}
```

Ziehier, een geslaagd examen.

```
PASS
ok  	    clockface	0.006s
```

### Refactor

Geen reden te refactoren, er is nauwelijks genoeg code!

### Herhaal voor nieuwe eisen

Waarschijnlijk moeten we hier wat werk doen dat niet alleen bestaat uit het terugzetten van een klok die op elk tijdstip middernacht aangeeft...

### Schrijf eerst je test

```go
func TestSecondHandAt30Seconds(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 30, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 + 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

Hetzelfde idee, maar nu wijst de secondewijzer naar _beneden_. Daarom _voegen_ we de lengte _toe_ aan de Y-as.

Dit zal compileren... maar hoe zorgen we ervoor dat het lukt?

## Denken in tijd

Hoe gaan we dit probleem oplossen?

Elke minuut doorloopt de secondewijzer dezelfde 60 standen en wijst hij in 60 verschillende richtingen. Bij 0 seconden wijst hij naar de bovenkant van de wijzerplaat, bij 30 seconden naar de onderkant. Zo simpel is het.

Dus als ik wilde nadenken over de richting waarin de secondewijzer wees, bijvoorbeeld 37 seconden, zou ik de hoek tussen 12 uur en 37/60e van een volledige rotatie willen. In graden is dit `(360/60) * 37 = 222`, maar het is makkelijker om gewoon te onthouden dat het `37/60`e van een volledige rotatie is.

Maar de hoek is slechts het halve verhaal; we moeten de X- en Y-coördinaat weten waar de punt van de secondewijzer naar wijst. Hoe kunnen we dat berekenen?

## Wiskunde

Stel je een cirkel voor met een straal van 1, getekend rond de oorsprong, de coördinaat `0, 0`.

![picture of the unit circle](../.gitbook/assets/unit_circle.png)

Dit wordt de 'eenheidscirkel' genoemd, omdat... de straal 1 eenheid is!

De omtrek van de cirkel bestaat uit punten op het raster, of coördinaten. De x- en y-componenten van elk van deze coördinaten vormen een driehoek, waarvan de schuine zijde altijd 1 is (d.w.z. de straal van de cirkel).

![picture of the unit circle with a point defined on the circumference](../.gitbook/assets/unit_circle_coords.png)

Met trigonometrie kunnen we nu de lengtes van X en Y voor elke driehoek berekenen, mits we de hoek kennen die ze met de oorsprong maken. De X-coördinaat is dan cos(a) en de Y-coördinaat sin(a), waarbij a de hoek is tussen de lijn en de (positieve) x-as.

![afbeelding van de eenheidscirkel met de x- en y-elementen van een straal gedefinieerd als respectievelijk cos(a) en sin(a), waarbij a de hoek is die de straal maakt met de x-as](../.gitbook/assets/unit_circle_params.png)

(Als je dit niet gelooft, [neem dan een kijkje op Wikipedia...](https://en.wikipedia.org/wiki/Sine#Unit_circle_definition))

Nog een laatste verandering: omdat we de hoek vanaf 12 uur willen meten in plaats van vanaf de X-as (3 uur), moeten we de assen omdraaien; nu is x = sin(a) en y = cos(a).

![unit circle ray defined from by angle from y axis](../.gitbook/assets/unit_circle_12_oclock.png)

Nu weten we hoe we de hoek van de secondewijzer (1/60e van een cirkel voor elke seconde) en de X- en Y-coördinaten kunnen bepalen. We hebben functies nodig voor zowel `sin` als `cos`.

## `math`

Gelukkig heeft het Go-`math`pakket beide, met één klein minpuntje waar we even aan moeten wennen. Kijk maar naar de beschrijving van `math.Cos`:

> Cos retourneert de cosinus van het radiant argument x.

De hoek moet in radialen zijn. Dus wat is een radiaal? In plaats van de volledige omwenteling van een cirkel te definiëren als 360 graden, definiëren we een volledige omwenteling als 2π radialen. Er zijn goede redenen om dit te doen, maar daar gaan we nu niet op in.

Nu we wat gelezen, geleerd en nagedacht hebben, kunnen we de volgende toets schrijven.

### Schrijf eerst je test

Al deze wiskunde is moeilijk en verwarrend. Ik weet niet zeker of ik begrijp wat er aan de hand is, dus laten we een test maken! We hoeven de hele opgave niet in één keer op te lossen, laten we beginnen met het berekenen van de juiste hoek, in radialen, voor de secondewijzer op een bepaald moment.

Ik ga de acceptatietest waar ik aan werkte, even buiten beschouwing laten terwijl ik aan deze tests werk. Ik wil niet afgeleid worden door die test terwijl ik deze test laat slagen.

### Een samenvatting van packages

Onze acceptatie tests bevinden zich momenteel in het `clockface_test`-pakket. Onze tests kunnen ook buiten het `clockface`-pakket worden uitgevoerd, zolang hun naam eindigt op `_test.go`.

Ik ga deze radialentests schrijven _binnen_ het `clockface`-pakket; ze worden mogelijk nooit geëxporteerd en mogelijk verwijderd (of verplaatst) zodra ik meer inzicht heb in wat er echt gebeurt. Ik hernoem mijn acceptatie test bestand naar `clockface_acceptance_test.go`, zodat ik een nieuw bestand met de naam `clockface_test` kan maken om seconden in radialen te testen.

```go
package clockface

import (
	"math"
	"testing"
	"time"
)

func TestSecondsInRadians(t *testing.T) {
	thirtySeconds := time.Date(312, time.October, 28, 0, 0, 30, 0, time.UTC)
	want := math.Pi
	got := secondsInRadians(thirtySeconds)

	if want != got {
		t.Fatalf("Wanted %v radians, but got %v", want, got)
	}
}
```

Hier testen we dat 30 seconden over de minuut de secondewijzer halverwege de klok zou moeten zetten. En dit is ons eerste gebruik van het `math`pakket! Als een volledige omwenteling van een cirkel 2π radialen is, weten we dat halverwege gewoon π radialen zou moeten zijn. `math.Pi` geeft ons een waarde voor π.

### Probeer de test uit te voeren

```
./clockface_test.go:12:9: undefined: secondsInRadians
```

### Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func secondsInRadians(t time.Time) float64 {
	return 0
}
```

```
clockface_test.go:15: Wanted 3.141592653589793 radians, but got 0
```

### Schrijf genoeg code om de test te laten slagen

```go
func secondsInRadians(t time.Time) float64 {
	return math.Pi
}
```

```
PASS
ok  	clockface	0.011s
```

### Refactor

Er hoeft nog niets gerefactored te worden

### Herhaal voor nieuwe eisen

Nu kunnen we de test uitbreiden met een paar extra scenario's. Ik 'spoel' even vooruit en laat wat reeds gerefactoriseerde testcode zien, het zou duidelijk genoeg moeten zijn hoe ik op mijn gewenste punt ben gekomen.

```go
func TestSecondsInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 0, 30), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(0, 0, 45), (math.Pi / 2) * 3},
		{simpleTime(0, 0, 7), (math.Pi / 30) * 7},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondsInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

Ik heb een paar hulpfuncties toegevoegd om het schrijven van deze tabel gebaseerde test wat minder saai te maken. `testName` converteert een tijd naar een digitaal horlogeformaat (UU:MM:SS), en `simpleTime` construeert een `time.Time` met alleen de onderdelen die we echt belangrijk vinden (opnieuw uren, minuten en seconden). Hier zijn ze:

```go
func simpleTime(hours, minutes, seconds int) time.Time {
	return time.Date(312, time.October, 28, hours, minutes, seconds, 0, time.UTC)
}

func testName(t time.Time) string {
	return t.Format("15:04:05")
}
```

Deze twee functies moeten het schrijven en onderhouden van deze tests (en toekomstige tests) een stuk eenvoudiger maken.

Dit levert ons een mooi testresultaat op:

```
clockface_test.go:24: Wanted 0 radians, but got 3.141592653589793

clockface_test.go:24: Wanted 4.71238898038469 radians, but got 3.141592653589793
```

Tijd om alle wiskundige zaken waar we het hierboven over hadden in de praktijk te brengen:

```go
func secondsInRadians(t time.Time) float64 {
	return float64(t.Second()) * (math.Pi / 30)
}
```

Wacht even is (2π / 60) radialen... streep de 2 weg en we krijgen π/30 radialen. Vermenigvuldig dat met het aantal seconden (als een `float64`) en dan zouden alle tests moeten slagen...

```
clockface_test.go:24: Wanted 3.141592653589793 radians, but got 3.1415926535897936
```

Wacht even, wat gebeurt hier?

### Floats zijn verschikkelijk

Rekenen met komma getallen is [notoir onnauwkeurig](https://0.30000000000000004.com/). Computers kunnen eigenlijk alleen gehele getallen verwerken, en tot op zekere hoogte ook rationale getallen. Decimale getallen worden onnauwkeurig, vooral wanneer we ze ontbinden in factoren, zoals in de functie `secondenInRadians`. Door `math.Pi` te delen door 30 en vervolgens te vermenigvuldigen met 30, _hebben we een getal gekregen dat niet langer hetzelfde is als `math.Pi`_.

Er zijn twee manieren om hiermee om te gaan:

1. Leer ermee leven
2. Herschrijf onze functie door onze vergelijking te herstructureren

Nu lijkt (1) misschien niet zo aantrekkelijk, maar het is vaak de enige manier om komma-gelijkheid te laten werken. Een onnauwkeurigheid van een oneindig klein deel maakt eerlijk gezegd niet uit voor het tekenen van een wijzerplaat, dus we zouden een functie kunnen schrijven die een 'voldoende nauwkeurige' gelijkheid definieert voor onze hoeken. Maar er is een eenvoudige manier om de nauwkeurigheid terug te krijgen: we herschikken de vergelijking zodat we niet langer hoeven te delen en vervolgens te vermenigvuldigen. We kunnen het allemaal doen door simpelweg te delen.

Dus in plaats van

```
numberOfSeconds * π / 30
```

kunnen we het volgende schrijven

```
π / (30 / numberOfSeconds)
```

wat eigenlijk precies hetzelfde doet.

In Go:

```go
func secondsInRadians(t time.Time) float64 {
	return (math.Pi / (30 / (float64(t.Second()))))
}
```

En alle testen slagen

```
PASS
ok      clockface     0.005s
```

Je code zou er nu [ongeveer zo uit moeten zien](https://github.com/quii/learn-go-with-tests/tree/main/math/v3/clockface).

### Een opmerking over delen door nul

Computers vinden delen door nul vaak niet prettig, omdat oneindigheid een beetje vreemd is.

Als je in Go expliciet door nul probeert te delen, krijg je een compilatiefout.

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println(10.0 / 0.0) // fails to compile
}
```

Uiteraard kan de compiler niet altijd voorspellen dat je door nul deelt, zoals bij onze `t.Second()`

Probeer dit maar eens:

```go
func main() {
	fmt.Println(10.0 / zero())
}

func zero() float64 {
	return 0.0
}
```

Dit geeft `+Inf` (oneindig) terug. Delen door +Inf lijkt nul te zijn, en dat zien we terug in het volgende:

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println(secondsinradians())
}

func zero() float64 {
	return 0.0
}

func secondsinradians() float64 {
	return (math.Pi / (30 / (float64(zero()))))
}
```

### Herhaal voor de nieuwe eisen

We hebben het eerste deel dus al behandeld: we weten in welke hoek de secondewijzer in radialen zal wijzen. Nu moeten we de coördinaten bepalen.

Laten we het zo simpel mogelijk houden en alleen met de _eenheidscirkel_ werken; de cirkel met een straal van 1. Dit betekent dat al onze wijzers een lengte van 1 hebben, maar aan de andere kant betekent dit dat de berekening voor ons makkelijker te begrijpen is.

### Schrijf eerst je test

```go
func TestSecondHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 0, 30), Point{0, -1}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondHandPoint(c.time)
			if got != c.point {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
./clockface_test.go:40:11: undefined: secondHandPoint
```

### Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func secondHandPoint(t time.Time) Point {
	return Point{}
}
```

```
clockface_test.go:42: Wanted {0 -1} Point, but got {0 0}
```

### Schrijf genoeg code om de test te laten slagen

```go
func secondHandPoint(t time.Time) Point {
	return Point{0, -1}
}
```

```
PASS
ok  	clockface	0.007s
```

### Herhaal voor nieuwe eisen

```go
func TestSecondHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 0, 30), Point{0, -1}},
		{simpleTime(0, 0, 45), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondHandPoint(c.time)
			if got != c.point {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
clockface_test.go:43: Wanted {-1 0} Point, but got {0 -1}
```

### Schrijf genoeg code om de test te laten slagen

Herinner je zich de afbeelding met de eenheidscirkel nog?

![afbeelding van de eenheidscirkel met de x- en y-elementen van een straal gedefinieerd als respectievelijk cos(a) en sin(a), waarbij a de hoek is die de straal maakt met de x-as](../.gitbook/assets/unit_circle_params.png)

Bedenk ook dat we de hoek willen meten vanaf 12 uur, de Y-as, in plaats van vanaf de X-as. We willen de hoek meten tussen de secondewijzer en 3 uur.

![unit circle ray defined from by angle from y axis](../.gitbook/assets/unit_circle_12_oclock.png)

We willen nu de vergelijking die X en Y oplevert. Laten we deze in seconden opschrijven:

```go
func secondHandPoint(t time.Time) Point {
	angle := secondsInRadians(t)
	x := math.Sin(angle)
	y := math.Cos(angle)

	return Point{x, y}
}
```

Nu krijgen we

```
clockface_test.go:43: Wanted {0 -1} Point, but got {1.2246467991473515e-16 -1}

clockface_test.go:43: Wanted {-1 0} Point, but got {-1 -1.8369701987210272e-16}
```

Wacht, wat (alweer)? Het lijkt erop dat we weer vervloekt zijn door de floats, beide onverwachte getallen zijn infinitesimaal, tot ver in de 16e decimaal. Dus we kunnen er weer voor kiezen om de precisie te verhogen, of gewoon te zeggen dat ze ongeveer gelijk zijn en verder te gaan met ons leven.

Een optie om de nauwkeurigheid van deze hoeken te vergroten, zou zijn om het rationale type `Rat` uit het `math/big`-pakket te gebruiken. Maar aangezien het doel is om een ​​SVG te tekenen en niet om op de maan te landen, denk ik dat we wel met wat vaagheid kunnen leven.

```go
func TestSecondHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 0, 30), Point{0, -1}},
		{simpleTime(0, 0, 45), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}

func roughlyEqualFloat64(a, b float64) bool {
	const equalityThreshold = 1e-7
	return math.Abs(a-b) < equalityThreshold
}

func roughlyEqualPoint(a, b Point) bool {
	return roughlyEqualFloat64(a.X, b.X) &&
		roughlyEqualFloat64(a.Y, b.Y)
}
```

We hebben twee functies gedefinieerd om de geschatte gelijkheid tussen twee `Points` te definiëren. Ze werken als de X- en Y-elementen binnen 0,0000001 van elkaar liggen. Dat is nog steeds behoorlijk nauwkeurig.

En nu krijgen we:

```
PASS
ok  	clockface	0.007s
```

### Refactor

Ik ben nog steeds behoorlijk blij met de code.

Hier is [hoe het eruit ziet op dit moment](https://github.com/quii/learn-go-with-tests/tree/main/math/v4/clockface)

### Herhaal voor nieuwe eisen

Nou, _nieuw_ zeggen is niet helemaal waar, wat we nu echt kunnen doen is die acceptatietest laten slagen! Laten we die test er nog eens bij pakken en bekijken hoe deze eruitziet:

```go
func TestSecondHandAt30Seconds(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 30, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 + 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

### Probeer de test uit te voeren

```
clockface_acceptance_test.go:28: Got {150 60}, wanted {150 240}
```

### Schrijf genoeg code om de test te laten slagen

We moeten drie dingen doen om onze eenheidsvector om te zetten in een punt op de SVG:

1. Naar de lengte van de wijzer schalen
2. Spiegelen over de X-as om rekening te houden met het feit dat de SVG een oorsprong heeft in de linkerbovenhoek
3. Vertaal het naar de juiste positie (zodat het afkomstig is van een oorsprong van (150,150))

Laat de echte lol beginnen!

```go
// SecondHand is the unit vector of the second hand of an analogue clock at time `t`
// represented as a Point.
func SecondHand(t time.Time) Point {
	p := secondHandPoint(t)
	p = Point{p.X * 90, p.Y * 90}   // scale
	p = Point{p.X, -p.Y}            // flip
	p = Point{p.X + 150, p.Y + 150} // translate
	return p
}
```

Schaal, spiegel en vertaal in precies die volgorde. Hoera, wiskunde!

```
PASS
ok  	clockface	0.007s
```

### Refactor

Er zijn hier een paar magische getallen die als constanten moeten worden gebruikt, dus laten we dat doen:

```go
const secondHandLength = 90
const clockCentreX = 150
const clockCentreY = 150

// SecondHand is the unit vector of the second hand of an analogue clock at time `t`
// represented as a Point.
func SecondHand(t time.Time) Point {
	p := secondHandPoint(t)
	p = Point{p.X * secondHandLength, p.Y * secondHandLength}
	p = Point{p.X, -p.Y}
	p = Point{p.X + clockCentreX, p.Y + clockCentreY} //translate
	return p
}
```

## Teken de klok

Nou ja... de secondewijzer in ieder geval...

Laten we dit doen, want er is niets erger dan geen waarde te leveren terwijl het er maar ligt te wachten om de wereld in te gaan om mensen te verbazen. Laten we eens een secondewijzer tekenen!

We gaan een nieuwe map toevoegen onder onze hoofdmap voor het `clockface`-pakket, genaamd (verwarrend genoeg) `clockface`. Daarin plaatsen we het `main`-pakket dat het binaire bestand voor een SVG-bestand aanmaakt:

```
|-- clockface
|       |-- main.go
|-- clockface.go
|-- clockface_acceptance_test.go
|-- clockface_test.go
```

In `main.go` begin je met deze code, maar wijzig je de import voor het clockface-pakket zodat deze naar uw eigen versie verwijst:

```go
package main

import (
	"fmt"
	"io"
	"os"
	"time"

	"learn-go-with-tests/math/clockface" // REPLACE THIS!
)

func main() {
	t := time.Now()
	sh := clockface.SecondHand(t)
	io.WriteString(os.Stdout, svgStart)
	io.WriteString(os.Stdout, bezel)
	io.WriteString(os.Stdout, secondHandTag(sh))
	io.WriteString(os.Stdout, svgEnd)
}

func secondHandTag(p clockface.Point) string {
	return fmt.Sprintf(`<line x1="150" y1="150" x2="%f" y2="%f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
}

const svgStart = `<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">`

const bezel = `<circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>`

const svgEnd = `</svg>`
```

O jee, ik probeer met deze puinhoop geen prijzen te winnen voor mooie code, maar het doet zijn werk. Het schrijft een SVG naar `os.Stdout`, string voor string tegelijk.

Bouw deze code

```
go build
```

en voer het uit, waarbij de uitvoer naar een bestand wordt verzonden

```
./clockface > clock.svg
```

Je zou iets moeten zien als hieronder:

![een klok met alleen een secondewijzer](../.gitbook/assets/clock.svg)

En dit is [hoe de code eruit ziet](https://github.com/quii/learn-go-with-tests/tree/main/math/v6/clockface).

### Refactor

Dit stinkt. Nou ja, het stinkt niet echt, maar ik ben er niet blij mee.

1. Die hele `SecondHand`-functie is super gekoppeld aan het feit dat het een SVG is... zonder SVG's te noemen of daadwerkelijk een SVG te produceren...
2. ... terwijl ik tegelijkertijd geen enkele SVG-code test.

Ja, ik denk dat ik een fout heb gemaakt. Dit voelt verkeerd. Laten we proberen het te herstellen met een meer SVG-gerichte test.

Wat zijn onze opties? We zouden kunnen testen of de tekens die uit de `SVGWriter` komen, dingen bevatten die lijken op het soort SVG-tag dat we voor een bepaalde tijd verwachten. Bijvoorbeeld:

```go
func TestSVGWriterAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	var b strings.Builder
	clockface.SVGWriter(&b, tm)
	got := b.String()

	want := `<line x1="150" y1="150" x2="150" y2="60"`

	if !strings.Contains(got, want) {
		t.Errorf("Expected to find the second hand %v, in the SVG output %v", want, got)
	}
}
```

Maar is dat echt een verbetering?

De test slaagt niet alleen als ik geen geldige SVG produceer (er wordt immers alleen getest of een tekenreeks in de uitvoer voorkomt), maar mislukt ook als ik een kleine, onbelangrijke wijziging in die tekenreeks aanbreng, bijvoorbeeld als ik een extra spatie tussen de kenmerken toevoeg.

De _grootste_ valkuil is dat ik een datastructuur (XML)​ test door de representatie ervan te bekijken als een reeks tekens, als een string. Dit is nooit een goed idee, want het levert problemen op zoals die ik hierboven heb geschetst: een test die te kwetsbaar en niet gevoelig genoeg is. Een test die het verkeerde test!

De enige oplossing is dus om de uitvoer als `XML` te testen. En daarvoor moeten we het parsen.

## Parsen van XML

[`encoding/xml`](https://pkg.go.dev/encoding/xml) is het Go-pakket dat alles kan afhandelen wat te maken heeft met eenvoudige XML-parsing.

De functie [`xml.Unmarshal`](https://pkg.go.dev/encoding/xml#Unmarshal) neemt een `[]byte` aan XML-gegevens en een aanwijzer naar een struct waarin deze moet worden gedeserialiseerd.

We hebben dus een structuur nodig om onze XML in te deserialiseren. We zouden wat tijd kunnen besteden aan het uitzoeken van de juiste namen voor alle knooppunten en attributen, en hoe we de juiste structuur moeten schrijven, maar gelukkig heeft iemand een programma voor [`zek`](https://github.com/miku/zek) geschreven dat al dat zware werk voor ons automatiseert. Sterker nog, er is een online versie op [https://xml-to-go.github.io/](https://xml-to-go.github.io/). Plak de SVG van boven in het bestand in één vak en - bam - daar verschijnt:

<pre class="language-go"><code class="lang-go"><strong>type Svg struct {
</strong>	XMLName xml.Name `xml:"svg"`
	Text    string   `xml:",chardata"`
	Xmlns   string   `xml:"xmlns,attr"`
	Width   string   `xml:"width,attr"`
	Height  string   `xml:"height,attr"`
	ViewBox string   `xml:"viewBox,attr"`
	Version string   `xml:"version,attr"`
	Circle  struct {
		Text  string `xml:",chardata"`
		Cx    string `xml:"cx,attr"`
		Cy    string `xml:"cy,attr"`
		R     string `xml:"r,attr"`
		Style string `xml:"style,attr"`
	} `xml:"circle"`
	Line []struct {
		Text  string `xml:",chardata"`
		X1    string `xml:"x1,attr"`
		Y1    string `xml:"y1,attr"`
		X2    string `xml:"x2,attr"`
		Y2    string `xml:"y2,attr"`
		Style string `xml:"style,attr"`
	} `xml:"line"`
}
</code></pre>

We kunnen hier indien nodig aanpassingen aan doen (zoals de naam van de struct wijzigen naar `SVG`), maar het is zeker goed genoeg om mee te beginnen. Plak de struct in het bestand `clockface_acceptance_test` en laten we er een test mee schrijven:

```go
func TestSVGWriterAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	b := bytes.Buffer{}
	clockface.SVGWriter(&b, tm)

	svg := Svg{}
	xml.Unmarshal(b.Bytes(), &svg)

	x2 := "150"
	y2 := "60"

	for _, line := range svg.Line {
		if line.X2 == x2 && line.Y2 == y2 {
			return
		}
	}

	t.Errorf("Expected to find the second hand with x2 of %+v and y2 of %+v, in the SVG output %v", x2, y2, b.String())
}
```

We schrijven de uitvoer van `clockface.SVGWriter` naar een `bytes.Buffer` en `Unmarshal` deze vervolgens naar een `SVG`. Vervolgens bekijken we elke `Line` in de `SVG` om te zien of deze de verwachte `X2`- en `Y2`-waarden heeft. Als we een match vinden, keren we eerder terug (en slagen we voor de test); zo niet, dan falen we met een (hopelijk) informatief bericht.

```sh
./clockface_acceptance_test.go:41:2: undefined: clockface.SVGWriter
```

Het lijkt erop dat we `SVGWriter.go` moeten creëren...

```go
package clockface

import (
	"fmt"
	"io"
	"time"
)

const (
	secondHandLength = 90
	clockCentreX     = 150
	clockCentreY     = 150
)

// SVGWriter writes an SVG representation of an analogue clock, showing the time t, to the writer w
func SVGWriter(w io.Writer, t time.Time) {
	io.WriteString(w, svgStart)
	io.WriteString(w, bezel)
	secondHand(w, t)
	io.WriteString(w, svgEnd)
}

func secondHand(w io.Writer, t time.Time) {
	p := secondHandPoint(t)
	p = Point{p.X * secondHandLength, p.Y * secondHandLength} // scale
	p = Point{p.X, -p.Y}                                      // flip
	p = Point{p.X + clockCentreX, p.Y + clockCentreY}         // translate
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%f" y2="%f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
}

const svgStart = `<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">`

const bezel = `<circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>`

const svgEnd = `</svg>`
```

De mooiste SVG-schrijf oplossing? Nee. Maar hopelijk doet hij zijn werk...

```
clockface_acceptance_test.go:56: Expected to find the second hand with x2 of 150 and y2 of 60, in the SVG output <?xml version="1.0" encoding="UTF-8" standalone="no"?>
    <!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
    <svg xmlns="http://www.w3.org/2000/svg"
         width="100%"
         height="100%"
         viewBox="0 0 300 300"
         version="2.0"><circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/><line x1="150" y1="150" x2="150.000000" y2="60.000000" style="fill:none;stroke:#f00;stroke-width:3px;"/></svg>
```

Oeps! De `%f`-opmaakrichtlijn drukt onze coördinaten af ​​met het standaardnauwkeurigheidsniveau van zes decimalen. We moeten expliciet aangeven welk nauwkeurigheidsniveau we voor de coördinaten verwachten. Laten we zeggen drie decimalen.

```go
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
```

En nadat we onze verwachtingen in de test hebben bijgewerkt

```go
	x2 := "150.000"
	y2 := "60.000"
```

Krijgen we:

```
PASS
ok  	clockface	0.006s
```

Nu kunnen we onze `main` functie inkorten:

```go
package main

import (
	"os"
	"time"

	"learn-go-with-tests/math/clockface"
)

func main() {
	t := time.Now()
	clockface.SVGWriter(os.Stdout, t)
}
```

Dit is hoe [de code er nu uit zou moeten zien](https://github.com/quii/learn-go-with-tests/tree/main/math/v7b/clockface).

En we kunnen een test voor een ander tijdstip schrijven volgens hetzelfde patroon, maar niet voordat...

### Refactor

Drie dingen vallen op:

1. We testen niet echt of alle informatie aanwezig is. Hoe zit het bijvoorbeeld met de `x1`-waarden?
2. En die attributen voor `x1` etc. zijn toch geen `strings`? Het zijn getallen!
3. Maakt het mij echt uit hoe de `stijl` van de wijzer is? Of, wat dat betreft, de lege `Text`node die door `zak` is gegenereerd?

Dat kunnen we beter. Laten we een paar aanpassingen doen aan de `SVG`-structuur en de tests om alles scherper te maken.

```go
type SVG struct {
	XMLName xml.Name `xml:"svg"`
	Xmlns   string   `xml:"xmlns,attr"`
	Width   string   `xml:"width,attr"`
	Height  string   `xml:"height,attr"`
	ViewBox string   `xml:"viewBox,attr"`
	Version string   `xml:"version,attr"`
	Circle  Circle   `xml:"circle"`
	Line    []Line   `xml:"line"`
}

type Circle struct {
	Cx float64 `xml:"cx,attr"`
	Cy float64 `xml:"cy,attr"`
	R  float64 `xml:"r,attr"`
}

type Line struct {
	X1 float64 `xml:"x1,attr"`
	Y1 float64 `xml:"y1,attr"`
	X2 float64 `xml:"x2,attr"`
	Y2 float64 `xml:"y2,attr"`
}
```

Hier heb ik

* De belangrijke onderdelen van de structuur de `Line` en de `Circle` apart benoemd
* De numerieke kenmerken omgezet naar `float64`'s in plaats van `string`s.
* De ongebruikte attributen `Style` en `Text` verwijderd
* `Svg` hernoemd naar `SVG` omdat dit _het juiste was om te doen_

Hiermee kunnen we nauwkeuriger bepalen welke regel we zoeken:

```go
func TestSVGWriterAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)
	b := bytes.Buffer{}

	clockface.SVGWriter(&b, tm)

	svg := SVG{}

	xml.Unmarshal(b.Bytes(), &svg)

	want := Line{150, 150, 150, 60}

	for _, line := range svg.Line {
		if line == want {
			return
		}
	}

	t.Errorf("Expected to find the second hand line %+v, in the SVG lines %+v", want, svg.Line)
}
```

Ten slotte kunnen we een voorbeeld nemen aan de tabellen van de unit-tests en een hulpfunctie schrijven die `containsLine(line Line, lines []Line) bool` toelaat om deze tests echt te laten schitteren:

```go
func TestSVGWriterSecondHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(0, 0, 0),
			Line{150, 150, 150, 60},
		},
		{
			simpleTime(0, 0, 30),
			Line{150, 150, 150, 240},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the second hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}

func containsLine(l Line, ls []Line) bool {
	for _, line := range ls {
		if line == l {
			return true
		}
	}
	return false
}
```

Hier zie hoe de [code eruit ziet](https://github.com/quii/learn-go-with-tests/tree/main/math/v7c/clockface)

Dit noem ik nog eens een acceptatietest!

### Schrijf eerst je test

Zo, dat is de secondewijzer. Laten we nu beginnen met de minutenwijzer.

```go
func TestSVGWriterMinuteHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(0, 0, 0),
			Line{150, 150, 150, 60},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the minute hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
clockface_acceptance_test.go:87: Expected to find the minute hand line {X1:150 Y1:150 X2:150 Y2:70}, in the SVG lines [{X1:150 Y1:150 X2:150 Y2:60}]
```

We kunnen beter beginnen met het bouwen van andere wijzers. Net zoals we de tests voor de secondewijzer hebben gemaakt, kunnen we itereren om de volgende set tests te produceren. We zullen onze acceptatietest opnieuw uit-commentariëren terwijl we dit werkend krijgen:

```go
func TestMinutesInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 30, 0), math.Pi},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minutesInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
./clockface_test.go:59:11: undefined: minutesInRadians
```

### Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func minutesInRadians(t time.Time) float64 {
	return math.Pi
}
```

### Herhaal voor de nieuwe eisen

Oké, laten we nu eens wat echt werk doen. We zouden de minutenwijzer kunnen modelleren alsof hij elke hele minuut beweegt, zodat hij 'springt' van 30 naar 31 minuten, zonder tussendoor te bewegen. Maar dat zou er een beetje raar uitzien. Wat we willen is dat hij elke seconde een klein beetje beweegt. Laten we daar de test voor schrijven.

```go
func TestMinutesInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 30, 0), math.Pi},
		{simpleTime(0, 0, 7), 7 * (math.Pi / (30 * 60))},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minutesInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

Hoeveel is dat kleine beetje? Nou...

* Zestig seconden in een minuut
* dertig minuten in een halve draai van de cirkel (`math.Pi` radialen)
* dus `30 * 60` seconden in een halve draai.
* Dus als de tijd 7 seconden na het uur is ...
* ... verwachten we de minutenwijzer te zien op `7 * (math.Pi / (30 * 60))` radialen voorbij de 12.

### Probeer de test uit te voeren

```
clockface_test.go:62: Wanted 0.012217304763960306 radians, but got 3.141592653589793
```

### Schrijf genoeg code om de test te laten slagen

In de onsterfelijke woorden van Jennifer Aniston: [Here comes the science bit](https://www.youtube.com/watch?v=29Im23SPNok)

```go
func minutesInRadians(t time.Time) float64 {
	return (secondsInRadians(t) / 60) +
		(math.Pi / (30 / float64(t.Minute())))
}
```

In plaats van helemaal opnieuw uit te rekenen hoe ver de minutenwijzer per seconde rond de wijzerplaat moet worden gedraaid, kunnen we hier gewoon de functie `secondsInRadians` gebruiken. Voor elke seconde beweegt de minutenwijzer 1/60e van de hoek die de secondewijzer beweegt.

```go
secondsInRadians(t) / 60
```

Vervolgens voegen we de beweging voor de minuten toe, vergelijkbaar met het beweging van de secondewijzer.

```go
math.Pi / (30 / float64(t.Minute()))
```

En...

```
PASS
ok  	clockface	0.007s
```

Dat valt mee toch!? Dit is hoe de code er [nu uit zou moeten zien](https://github.com/quii/learn-go-with-tests/tree/main/math/v8/clockface/clockface_acceptance_test.go)

### Herhaal voor nieuwe eisen

Moet ik meer gevallen toevoegen aan de `minutesInRadians`-test? Op dit moment zijn er maar twee. Hoeveel gevallen heb ik nodig voordat ik verder ga met het testen van de `minuteHandPoint`-functie?

Een van mijn favoriete TDD-citaten, vaak toegeschreven aan Kent Beck, is

> Schrijf tests totdat angst omslaat in verveling.

En eerlijk gezegd ben ik het beu om die functie te testen. Ik weet wel hoe het werkt, dus op naar de volgende.

### Schrijf eerst de test

```go
func TestMinuteHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 30, 0), Point{0, -1}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minuteHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
./clockface_test.go:79:11: undefined: minuteHandPoint
```

### Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func minuteHandPoint(t time.Time) Point {
	return Point{}
}
```

```
clockface_test.go:80: Wanted {0 -1} Point, but got {0 0}
```

### Schrijf genoeg code om de test te laten slagen

```go
func minuteHandPoint(t time.Time) Point {
	return Point{0, -1}
}
```

```
PASS
ok  	clockface	0.007s
```

### Herhaal voor nieuwe eisen

En nu wat echt werk

```go
func TestMinuteHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 30, 0), Point{0, -1}},
		{simpleTime(0, 45, 0), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minuteHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

```
clockface_test.go:81: Wanted {-1 0} Point, but got {0 -1}
```

### Schrijf genoeg code om de test te laten slagen

Een snelle kopie en plak van de `secondHandPoint`-functie met een paar kleine aanpassingen zou voldoende moeten zijn...

```go
func minuteHandPoint(t time.Time) Point {
	angle := minutesInRadians(t)
	x := math.Sin(angle)
	y := math.Cos(angle)

	return Point{x, y}
}
```

```
PASS
ok  	clockface	0.009s
```

### Refactor

We hebben zeker wat herhaling in de `minuteHandPoint` en `secondHandPoint`, dat weten we omdat we de ene gewoon hebben gekopieerd en geplakt om de andere te maken. Laten we het opdrogen met een functie.

```go
func angleToPoint(angle float64) Point {
	x := math.Sin(angle)
	y := math.Cos(angle)

	return Point{x, y}
}
```

en we kunnen `minuteHandPoint` en `secondHandPoint` herschrijven als one-liners:

```go
func minuteHandPoint(t time.Time) Point {
	return angleToPoint(minutesInRadians(t))
}
```

```go
func secondHandPoint(t time.Time) Point {
	return angleToPoint(secondsInRadians(t))
}
```

```
PASS
ok  	clockface	0.007s
```

Nu kunnen we de acceptatietest uit commentaar halen en aan de slag gaan met het tekenen van de minutenwijzer.

### Schrijf genoeg code om de test te laten slagen

De `minuteHand`-functie is een kopieer-en-plakbewerking van `secondHand` met enkele kleine aanpassingen, zoals het declareren van een `minuteHandLength`:

```go
const minuteHandLength = 80

//...

func minuteHand(w io.Writer, t time.Time) {
	p := minuteHandPoint(t)
	p = Point{p.X * minuteHandLength, p.Y * minuteHandLength}
	p = Point{p.X, -p.Y}
	p = Point{p.X + clockCentreX, p.Y + clockCentreY}
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#000;stroke-width:3px;"/>`, p.X, p.Y)
}
```

En een aanroep ervan in onze `SVGWriter`-functie:

```go
func SVGWriter(w io.Writer, t time.Time) {
	io.WriteString(w, svgStart)
	io.WriteString(w, bezel)
	secondHand(w, t)
	minuteHand(w, t)
	io.WriteString(w, svgEnd)
}
```

Nu zouden we moeten zien dat `TestSVGWriterMinuteHand` het volgende resultaat geeft:

```
PASS
ok  	clockface	0.006s
```

Maar het bewijs van de pudding zit in het eten. Als we nu ons `clockface` programma compileren en uitvoeren, zouden we iets moeten zien als

![een klok met seconde- en minutenwijzers](<../.gitbook/assets/clock (1).svg>)

### Refactor

Laten we de duplicatie uit de functies `secondHand` en `minuteHand` verwijderen en alle logica voor schaal, flip en vertaling op één plek onderbrengen.

```go
func secondHand(w io.Writer, t time.Time) {
	p := makeHand(secondHandPoint(t), secondHandLength)
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
}

func minuteHand(w io.Writer, t time.Time) {
	p := makeHand(minuteHandPoint(t), minuteHandLength)
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#000;stroke-width:3px;"/>`, p.X, p.Y)
}

func makeHand(p Point, length float64) Point {
	p = Point{p.X * length, p.Y * length}
	p = Point{p.X, -p.Y}
	return Point{p.X + clockCentreX, p.Y + clockCentreY}
}
```

```
PASS
ok  	clockface	0.007s
```

Dit is [waar we nu zijn aangekomen](https://github.com/quii/learn-go-with-tests/tree/main/math/v9/clockface).

Zo... nu hoef je alleen nog maar de kleine wijzer te doen!

### Schrijf eerst je test

```go
func TestSVGWriterHourHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(6, 0, 0),
			Line{150, 150, 150, 200},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the hour hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
clockface_acceptance_test.go:113: Expected to find the hour hand line {X1:150 Y1:150 X2:150 Y2:200}, in the SVG lines [{X1:150 Y1:150 X2:150 Y2:60} {X1:150 Y1:150 X2:150 Y2:70}]
```

Laten we dit nog even buiten beschouwing laten totdat we wat dekking hebben met de tests op lager niveau:

### Schrijf de eerste test

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
./clockface_test.go:97:11: undefined: hoursInRadians
```

### Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func hoursInRadians(t time.Time) float64 {
	return math.Pi
}
```

```
PASS
ok  	clockface	0.007s
```

### Herhaal voor nieuwe eisen

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
clockface_test.go:100: Wanted 0 radians, but got 3.141592653589793
```

### Schrijf genoeg code om de test te laten slagen

```go
func hoursInRadians(t time.Time) float64 {
	return (math.Pi / (6 / float64(t.Hour())))
}
```

### Herhaal voor nieuwe eisen

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(21, 0, 0), math.Pi * 1.5},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
clockface_test.go:101: Wanted 4.71238898038469 radians, but got 10.995574287564276
```

### Schrijf genoeg code om de test te laten slagen

```go
func hoursInRadians(t time.Time) float64 {
	return (math.Pi / (6 / (float64(t.Hour() % 12))))
}
```

Bedenk wel dat dit geen 24-uursklok is. Om de rest van het huidige uur te berekenen, gedeeld door 12, moeten we de restoperator gebruiken.

```
PASS
ok  	learn-go-with-tests/math/clockface	0.008s
```

### Schrijf eerst je test

Laten we nu proberen de kleine wijzer op de klok te draaien, gebaseerd op de minuten en seconden die verstreken zijn.

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(21, 0, 0), math.Pi * 1.5},
		{simpleTime(0, 1, 30), math.Pi / ((6 * 60 * 60) / 90)},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
clockface_test.go:102: Wanted 0.013089969389957472 radians, but got 0
```

### Schrijf genoeg code om de test te laten slagen

Opnieuw is er nu wat denkwerk nodig. We moeten de uurwijzer een klein beetje verplaatsen voor zowel de minuten als de seconden. Gelukkig hebben we al een hoek voor de minuten en de seconden bij de hand: die van `minutenInRadians`. Die kunnen we hergebruiken!

De enige vraag is dus met welke factor we die hoek moeten verkleinen. Een volledige draai is één uur voor de minutenwijzer, maar voor de urenwijzer is dat twaalf uur. Dus delen we de hoek die `minutesInRadians` oplevert gewoon door twaalf:

```go
func hoursInRadians(t time.Time) float64 {
	return (minutesInRadians(t) / 12) +
		(math.Pi / (6 / float64(t.Hour()%12)))
}
```

en zie:

```
clockface_test.go:104: Wanted 0.013089969389957472 radians, but got 0.01308996938995747
```

De komma berekening slaat weer toe.

Laten we onze test bijwerken om grofweg `EqualFloat64` te gebruiken voor de vergelijking van de hoeken.

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(21, 0, 0), math.Pi * 1.5},
		{simpleTime(0, 1, 30), math.Pi / ((6 * 60 * 60) / 90)},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if !roughlyEqualFloat64(got, c.angle) {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

```
PASS
ok  	clockface	0.007s
```

### Refactor

Als we `rouglyEqualFloat64` gaan gebruiken in een van onze radialentests, moeten we het waarschijnlijk voor _alle_ tests gebruiken. Dat is een mooie en simpele refactoring, waardoor [het er zo uitziet](https://github.com/quii/learn-go-with-tests/tree/main/math/v10/clockface).

## Uurwijzer

Oké, het is tijd om te berekenen waar de kleine wijzer naartoe gaat door de eenheidsvector te berekenen.

### Schrijf eerst je test

```go
func TestHourHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(6, 0, 0), Point{0, -1}},
		{simpleTime(21, 0, 0), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hourHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

Wacht, ga ik twee testcases tegelijk schrijven? Is dit geen slechte TDD?

### Over TDD-fanatisme

Test Driven Development is geen religie. Sommige mensen doen misschien alsof dat wel zo is (meestal mensen die geen TDD doen) maar graag op Twitter of Dev.to klagen dat het alleen door fanatici wordt gedaan en dat ze 'pragmatisch' zijn als ze geen tests schrijven. Maar het is geen religie. Het is een tool.

Ik weet welke twee tests het worden. Ik heb twee andere wijzers op exact dezelfde manier getest. En ik weet ook al wat mijn implementatie wordt. Ik heb een functie geschreven voor het algemene geval waarbij een hoek in de minutenwijzeriteratie in een punt wordt veranderd.

Ik ga de TDD-ceremonie niet zomaar doorploegen. TDD is een techniek die me helpt de code die ik schrijf, en de code die ik ga schrijven, beter te begrijpen. TDD geeft me feedback, kennis en inzicht. Maar als ik die kennis al heb, ga ik de ceremonie niet zomaar doorploegen. Noch tests, noch TDD zijn een doel op zich.

Mijn zelfvertrouwen is toegenomen, dus ik heb het gevoel dat ik grotere stappen vooruit kan zetten. Ik ga een paar stappen 'overslaan', want ik weet waar ik ben, ik weet waar ik naartoe ga en ik heb deze weg al eerder bewandeld.

Maar let op: ik sla het schrijven van de tests niet helemaal over, ik schrijf ze nog steeds eerst. Ze verschijnen alleen in minder gedetailleerde stukjes.

### Probeer de test uit te voeren

```
./clockface_test.go:119:11: undefined: hourHandPoint
```

### Schrijf genoeg code om de test te laten slagen

```go
func hourHandPoint(t time.Time) Point {
	return angleToPoint(hoursInRadians(t))
}
```

Zoals ik al zei, ik weet waar ik ben en ik weet waar ik naartoe ga. Waarom zou ik anders doen alsof? De tests zullen me binnenkort vertellen of ik het mis heb.

```
PASS
ok  	learn-go-with-tests/math/clockface	0.009s
```

## Teken de uur wijzer

En tot slot tekenen we de uren wijzer. We kunnen die acceptatietest invoeren door hem in te schakelen:

```go
func TestSVGWriterHourHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(6, 0, 0),
			Line{150, 150, 150, 200},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the hour hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}
```

### Probeer de test uit te voeren

```
clockface_acceptance_test.go:113: Expected to find the hour hand line {X1:150 Y1:150 X2:150 Y2:200},
    in the SVG lines [{X1:150 Y1:150 X2:150 Y2:60} {X1:150 Y1:150 X2:150 Y2:70}]
```

### Schrijf genoeg code om de test te laten slagen

En nu kunnen we onze laatste aanpassingen maken aan de SVG-schrijfconstanten en -functies:

```go
const (
	secondHandLength = 90
	minuteHandLength = 80
	hourHandLength   = 50
	clockCentreX     = 150
	clockCentreY     = 150
)

// SVGWriter writes an SVG representation of an analogue clock, showing the time t, to the writer w
func SVGWriter(w io.Writer, t time.Time) {
	io.WriteString(w, svgStart)
	io.WriteString(w, bezel)
	secondHand(w, t)
	minuteHand(w, t)
	hourHand(w, t)
	io.WriteString(w, svgEnd)
}

// ...

func hourHand(w io.Writer, t time.Time) {
	p := makeHand(hourHandPoint(t), hourHandLength)
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#000;stroke-width:3px;"/>`, p.X, p.Y)
}

```

En dus...

```
ok  	clockface	0.007s
```

Laten we dit controleren door ons `clockface` programma te compileren en uit te voeren.

![een klok](<../.gitbook/assets/clock (2).svg>)

### Refactor

Kijkend naar `clockface.go`, zie je een paar 'magische getallen' rondzweven. Ze zijn allemaal gebaseerd op het aantal uren/minuten/seconden dat er in een halve draai rond een wijzerplaat zit. Laten we de betekenis ervan eens herstructureren.

```go
const (
	secondsInHalfClock = 30
	secondsInClock     = 2 * secondsInHalfClock
	minutesInHalfClock = 30
	minutesInClock     = 2 * minutesInHalfClock
	hoursInHalfClock   = 6
	hoursInClock       = 2 * hoursInHalfClock
)
```

Waarom doen we dit? Nou, het maakt duidelijk wat elk getal in de vergelijking _betekent_. Als we later terugkomen bij deze code, zullen deze namen ons helpen te begrijpen waar de waarden voor staan.

Bovendien, mochten we ooit echt heel VREEMDE klokken willen maken (bijvoorbeeld met 4 uur voor de uurwijzer en 20 seconden voor de secondewijzer) dan zouden deze constanten gemakkelijk parameters kunnen worden. We helpen die deur open te houden (zelfs als we er nooit doorheen gaan).

## Samenvattend

Moeten we nog iets anders doen?

Laten we onszelf eerst een schouderklopje geven: we hebben een programma geschreven dat een SVG-wijzerplaat maakt. Het werkt en het is geweldig. Het maakt maar één soort wijzerplaat, maar dat is prima! Misschien wil je maar één soort wijzerplaat. Er is niets mis met een programma dat een specifiek probleem oplost en niets anders.

### Een programma... en een bibliotheek

Maar de code die we hebben geschreven, lost wel een meer algemene reeks problemen op die te maken hebben met het tekenen van een wijzerplaat. Omdat we tests hebben gebruikt om elk klein onderdeel van het probleem afzonderlijk te bekijken, en omdat we die isolatie met functies hebben vastgelegd, hebben we een zeer redelijke kleine API gebouwd voor wijzerplaatberekeningen.

We kunnen aan dit project werken en het omzetten in iets algemeners: een bibliotheek voor het berekenen van hoeken en/of vectoren van wijzerplaten.

Het is eigenlijk een _heel goed idee_ om de bibliotheek samen met het programma aan te bieden. Het kost ons niets, maar vergroot wel de bruikbaarheid van ons programma en helpt bij het documenteren van de werking ervan.

> API's zouden met programma's meegeleverd moeten worden, en vice versa. Een API waarvoor je C-code moet schrijven en die niet eenvoudig vanaf de opdrachtregel kan worden aangeroepen, is moeilijker te leren en te gebruiken. En omgekeerd is het een enorme opgave om interfaces te hebben waarvan de enige open, gedocumenteerde vorm een ​​programma is, waardoor je ze niet eenvoudig vanuit een C-programma kunt aanroepen.
>
> \-- Henry Spencer, uit _The Art of Unix Programming_

In mijn [uiteindelijke versie van dit programma](https://github.com/quii/learn-go-with-tests/tree/main/math/vFinal/clockface) heb ik de niet-geëxporteerde functies in `clockface` omgezet naar een openbare API voor de bibliotheek, met functies om de hoek en eenheidsvector voor elke wijzer te berekenen. Ik heb het SVG-generatiegedeelte ook opgesplitst in een eigen pakket, `svg`, dat vervolgens rechtstreeks door het `clockface`-programma wordt gebruikt. Uiteraard heb ik elk van de functies en pakketten gedocumenteerd.

Over SVG's gesproken...

### De meest waardevolle test

Ik weet zeker dat je hebt gemerkt dat het meest geavanceerde stukje code voor het verwerken van SVG's helemaal niet in onze applicatiecode zit; het zit in de testcode. Moeten we ons hier ongemakkelijk bij voelen? Zouden we niet zoiets moeten doen als:

* een template van `text/template` gebruiken?
* een XML-bibliotheek gebruiken (zoals we in onze test doen)?
* een SVG-bibliotheek gebruiken?

We zouden onze code kunnen refactoren om al deze dingen te doen, en dat kunnen we doen omdat het niet uitmaakt _hoe_ we onze SVG produceren; wat belangrijk is, is _wat_ we produceren: een SVG. Het deel van ons systeem dat het meest over SVG's moet weten (en dus het meest strikt moet zijn over wat een SVG is) is de test voor de SVG-uitvoer: het moet voldoende context en kennis hebben over wat een SVG is, zodat we er zeker van kunnen zijn dat we een SVG uitgeven. Het _wat_ van een SVG zit in onze tests; het _hoe_ in de code.

We vonden het misschien vreemd dat we zoveel tijd en moeite in die SVG-tests staken, het importeren van een XML-bibliotheek, het parsen van XML, het refactoren van de structs,  maar die testcode is een waardevol onderdeel van onze codebase. Mogelijk zelfs waardevoller dan de huidige productiecode. Het helpt garanderen dat de output altijd een geldige SVG is, ongeacht wat we ervoor gebruiken.

Tests zijn geen tweederangsburgers. Het is geen 'wegwerpcode'. Goede tests gaan veel langer mee dan de versie van de code die ze testen. Je moet nooit het gevoel hebben dat je 'te veel tijd' besteedt aan het schrijven van je tests. Het is een investering.

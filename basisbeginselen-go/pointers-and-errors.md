# Pointers & errors

[**Je kunt hier alle code voor dit hoofdstuk vinden**](https://github.com/quii/learn-go-with-tests/tree/main/arrays)

In de vorige sectie hebben we geleerd over structuren. Hiermee kunnen we een aantal waarden vastleggen die verband houden met een concept of onderwerp.

Op een gegeven moment wil je wellicht structuren gebruiken om de status te beheren. Je wilt dan methoden beschikbaar stellen waarmee gebruikers de status op een door jou gecontroleerde manier kunnen wijzigen.

**Fintech is dol op Go** en eh, bitcoins? Laten we eens kijken wat voor een geweldig banksysteem we kunnen maken.

Laten we een `Wallet`-structuur maken waarmee we `Bitcoin` kunnen storten.

## Schrijf eerst je test

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(10)

	got := wallet.Balance()
	want := 10

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

In het [vorige voorbeeld](../bouw-een-applicatie/app-intro.md) hadden we rechtstreeks toegang tot velden met de veldnaam, maar in onze zeer veilige wallet willen we onze interne status niet blootstellen aan de rest van de wereld. We willen de toegang beheren via methoden.

## Voer de test uit

`./wallet_test.go:7:12: undefined: Wallet`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

De compiler weet niet wat een `Wallet` is, dus laten we het hem vertellen.

```go
type Wallet struct{}
```

Nu we onze `Wallet` hebben gemaakt, proberen we de test opnieuw uit te voeren

```
./wallet_test.go:9:8: wallet.Deposit undefined (type Wallet has no field or method Deposit)
./wallet_test.go:11:15: wallet.Balance undefined (type Wallet has no field or method Balance)
```

We moeten deze methoden definiëren.

Vergeet niet om alleen voldoende te doen om de tests uit te voeren. We moeten ervoor zorgen dat onze test correct mislukt met een duidelijke foutmelding.

```go
func (w Wallet) Deposit(amount int) {

}

func (w Wallet) Balance() int {
	return 0
}
```

Als deze syntaxis onbekend is, lees dan de sectie '[structs](structs-methods-and-interfaces.md)'.

De tests zouden nu moeten compileren en draaien.

`wallet_test.go:15: got 0 want 10`

## Schrijf genoeg code om test te laten slagen

We hebben een soort _balan&#x73;_&#x76;ariabele nodig in onze structuur om de status op te slaan

```go
type Wallet struct {
	balance int
}
```

Als een symbool in Go (variabelen, typen, functies en dergelijke) begint met een kleine letter, dan is het privé en alleen toegankelijk voor het pakket waarin het is gedefinieerd.

In ons geval willen we dat onze methoden deze waarde kunnen manipuleren, maar niemand anders.

Vergeet niet dat we toegang hebben tot het interne `balance`veld in de struct via de variabele "receiver".

```go
func (w Wallet) Deposit(amount int) {
	w.balance += amount
}

func (w Wallet) Balance() int {
	return w.balance
}
```

Nu we onze carrière in fintech hebben veiliggesteld, voeren we de testsuite uit en genieten we van de geslaagde test

`wallet_test.go:15: got 0 want 10`

### Dat klopt niet helemaal

Dit is verwarrend, maar onze code lijkt te werken. We tellen het nieuwe bedrag op bij ons saldo en de balance-methode zou de huidige status ervan moeten retourneren.

**Wanneer je in Go een functie of methode aanroept, worden de argumenten&#x20;**_**gekopieerd**_**.**

Wanneer je `func(w Wallet) Deposit(amount int)` aanroept, is de `w` een kopie van de methode waarmee we de methode hebben aangeroepen.

Zonder al te veel in de computerwetenschap te duiken: wanneer je een waarde creëert (zoals een wallet) wordt deze ergens in het geheugen opgeslagen. Je kunt het _adres_ van dat stukje geheugen achterhalen met `&myVal`.

Experimenteer door enkele afdrukken aan je code toe te voegen

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(10)

	got := wallet.Balance()

	fmt.Printf("address of balance in test is %p \n", &wallet.balance)

	want := 10

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

```go
func (w Wallet) Deposit(amount int) {
	fmt.Printf("address of balance in Deposit is %p \n", &w.balance)
	w.balance += amount
}
```

De tijdelijke aanduiding `%p` geeft geheugenadressen weer in basis 16-notatie met voorafgaande `0x`s. Het escape-teken `\n` geeft een nieuwe regel weer. Merk op dat we de pointer (het geheugenadres) van iets verkrijgen door een `&`-teken aan het begin van het symbool te plaatsen.

Voer nu de test opnieuw uit

```
address of balance in Deposit is 0xc420012268
address of balance in test is 0xc420012260
```

Je ziet dat de adressen van de twee balansen verschillend zijn. Wanneer we de waarde van de balans in de code wijzigen, werken we dus met een kopie van wat er uit de test komt. De balans in de test blijft dus ongewijzigd.

We kunnen dit oplossen met _pointers_. [Pointers](https://gobyexample.com/pointers) laten ons naar bepaalde waarden verwijzen en deze vervolgens wijzigen. Dus in plaats van een kopie van de hele wallet te maken, gebruiken we een pointer naar die wallet, zodat we de oorspronkelijke waarden erin kunnen wijzigen.

```go
func (w *Wallet) Deposit(amount int) {
	w.balance += amount
}

func (w *Wallet) Balance() int {
	return w.balance
}
```

Het verschil is dat het type ontvanger `*Wallet` is in plaats van `Wallet`, wat je kunt lezen als "een verwijzing naar een wallet".

Probeer de tests opnieuw uit te voeren. Ze zouden moeten slagen.

Nu vraag je je misschien af: waarom zijn ze geslaagd? We hebben de pointer in de functie niet gederefereerd, zoals hier:

```go
func (w *Wallet) Balance() int {
	return (*w).balance
}
```

en schijnbaar rechtstreeks naar het object verwezen. Sterker nog, de bovenstaande code met `(*w)` is absoluut geldig. De makers van Go vonden deze notatie echter omslachtig, dus de taal staat ons toe om `w.balance` te schrijven, zonder expliciete naar deze pointer te gaan. Deze verwijzingen naar structs hebben zelfs een eigen naam: _struct pointers_, en Go volgt deze [automatisch voor je](https://golang.org/ref/spec#Method_values).

Technisch gezien hoef je `Balance` niet te wijzigen om een ​​pointer-ontvanger te gebruiken, aangezien een kopie van de balans prima is. Uit gewoonte is het echter verstandig om de ontvangertypen van je methoden hetzelfde te houden voor consistentie.

## Refactor

We zeiden dat we een Bitcoin-wallet zouden maken, maar we hebben ze tot nu toe niet genoemd. We gebruiken int omdat ze een goed type zijn om dingen te tellen!

Het lijkt me wat overdreven om hiervoor een `struct` te maken. `int` werkt prima, maar is niet beschrijvend.

Met Go kun je nieuwe typen maken op basis van bestaande typen.

De syntax is `type MyName OriginalType`

```go
type Bitcoin int

type Wallet struct {
	balance Bitcoin
}

func (w *Wallet) Deposit(amount Bitcoin) {
	w.balance += amount
}

func (w *Wallet) Balance() Bitcoin {
	return w.balance
}
```

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(Bitcoin(10))

	got := wallet.Balance()

	want := Bitcoin(10)

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

Om `Bitcoin` te maken, gebruik je gewoon de syntaxis `Bitcoin(999)`.

Door dit te doen, maken we een nieuw type aan en kunnen we er _methods_ voor declareren. Dit kan erg handig zijn wanneer je domeinspecifieke functionaliteit wilt toevoegen aan bestaande typen.

Laten we [Stringer](https://golang.org/pkg/fmt/#Stringer) op Bitcoin implementeren

```go
type Stringer interface {
	String() string
}
```

Deze interface is gedefinieerd in het `fmt`-pakket en laat je definiëren hoe je type wordt afgedrukt wanneer je de opmaakreeks `%s` gebruikt.

```go
func (b Bitcoin) String() string {
	return fmt.Sprintf("%d BTC", b)
}
```

Zoals je ziet, is de syntaxis voor het maken van een methode op een typedeclaratie hetzelfde als op een struct.

Vervolgens moeten we de opmaakstrings van onze test bijwerken, zodat ze `String()` gebruiken.

```go
	if got != want {
		t.Errorf("got %s want %s", got, want)
	}
```

Om dit in actie te zien, moeten we de test opzettelijk fout laten gaan

`wallet_test.go:18: got 10 BTC want 20 BTC`

Hierdoor wordt duidelijker wat er gebeurt tijdens onze test.

De volgende vereiste is een `Withdraw`-functie.

## Schrijf eerst je test

Eigenlijk het tegenovergestelde van `Deposit()`

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}

		wallet.Deposit(Bitcoin(10))

		got := wallet.Balance()

		want := Bitcoin(10)

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	})

	t.Run("withdraw", func(t *testing.T) {
		wallet := Wallet{balance: Bitcoin(20)}

		wallet.Withdraw(Bitcoin(10))

		got := wallet.Balance()

		want := Bitcoin(10)

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	})
}
```

## Voer de test uit

`./wallet_test.go:26:9: wallet.Withdraw undefined (type Wallet has no field or method Withdraw)`

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func (w *Wallet) Withdraw(amount Bitcoin) {

}
```

`wallet_test.go:33: got 20 BTC want 10 BTC`

## Schrijf genoeg code om test te laten slagen

```go
func (w *Wallet) Withdraw(amount Bitcoin) {
	w.balance -= amount
}
```

## Refactor

Er zit wat duplicatie in onze tests. Laten we dat eens aanpassen.

```go
func TestWallet(t *testing.T) {

	assertBalance := func(t testing.TB, wallet Wallet, want Bitcoin) {
		t.Helper()
		got := wallet.Balance()

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	}

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw", func(t *testing.T) {
		wallet := Wallet{balance: Bitcoin(20)}
		wallet.Withdraw(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

}
```

Wat moet er gebeuren als je probeert meer op te nemen dan er op de rekening staat? Voorlopig gaan we ervan uit dat er geen sprake is van een roodstand.

Hoe signaleren we een probleem bij het gebruik van `Withdraw`?

Als je in Go een fout wilt aangeven, is het standard dat je functie een `err` retourneert, zodat de aanroepende functie deze kan controleren en er actie op kan ondernemen.

Laten we dit proberen in een test.

## Schrijf eerst je test

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertBalance(t, wallet, startingBalance)

	if err == nil {
		t.Error("wanted an error but didn't get one")
	}
})
```

We willen dat `Withdraw` een foutmelding geeft als je meer probeert op te nemen dan je hebt, terwijl het saldo hetzelfde moet blijven.

Vervolgens controleren we of er een fout is geretourneerd door de test te laten mislukken als deze `nil` is.

`nil` is synoniem met `null` uit andere programmeertalen. Fouten kunnen `nil` zijn omdat het retourtype van `Withdraw` `error` is, wat een interface is. Als je een functie ziet die argumenten accepteert of waarden retourneert die interfaces zijn, kunnen deze nillable zijn.

Net als bij `null` zal een **runtime-panic** optreden als je probeert toegang te krijgen tot een waarde die `nil` is. Dit is niet goed! Controleer daarom altijd op nils.

## Probeer de test uit te voeren

`./wallet_test.go:31:25: wallet.Withdraw(Bitcoin(100)) used as value`

De formulering is misschien wat onduidelijk, maar onze eerdere bedoeling met `Withdraw` was om het gewoon aan te roepen: het retourneert nooit een waarde. Om dit te laten compileren, moeten we het aanpassen zodat het een retourtype heeft.

## Schrijf de minimale hoeveelheid code om de test te laten uitvoeren en de falende test output te controleren

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {
	w.balance -= amount
	return nil
}
```

Nogmaals, het is erg belangrijk om net genoeg code te schrijven om de compiler tevreden te stellen. We corrigeren onze `Withdraw`-methode om een ​​`error` te retourneren en voor nu moeten we _iets_ retourneren, dus laten we gewoon `nil` retourneren.

## Schrijf genoeg code om de test te laten slagen

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return errors.New("oh no")
	}

	w.balance -= amount
	return nil
}
```

Vergeet niet om de `errors` package in je code te importeren.

`errors.New` creëert een nieuwe `error` met een bericht naar keuze.

## Refactor

Laten we een snelle testhulp maken voor onze foutcontrole om de leesbaarheid van de test te verbeteren

```go
assertError := func(t testing.TB, err error) {
	t.Helper()
	if err == nil {
		t.Error("wanted an error but didn't get one")
	}
}
```

And in our test

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertError(t, err)
	assertBalance(t, wallet, startingBalance)
})
```

Ik hoop dat je, toen je de foutmelding "oh nee" terugkreeg, dacht dat we daarop konden itereren, omdat het niet zo nuttig lijkt om terug te keren.

Ervan uitgaande dat de fout uiteindelijk aan de gebruiker wordt gemeld, passen we onze test aan zodat deze een foutmelding weergeeft in plaats van alleen het bestaan ​​van een fout.

## Schrijf eerst je test

Werk onze helper bij met een `string` waarmee we kunnen vergelijken.

```go
assertError := func(t testing.TB, got error, want string) {
	t.Helper()

	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got.Error() != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Zoals je ziet, kunnen `errors` worden omgezet naar een string met de `.Error()`-methode. Dit doen we om de fout te vergelijken met de gewenste string. We zorgen er ook voor dat de fout niet `nil` is, zodat we `.Error()` niet aanroepen op `nil`.

En dan passen we de aanroeper aan

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertError(t, err, "cannot withdraw, insufficient funds")
	assertBalance(t, wallet, startingBalance)
})
```

We hebben `t.Fatal` geïntroduceerd, dat de test stopt als deze wordt aangeroepen. Dit doen we omdat we geen verdere beweringen willen doen over de geretourneerde fout als er geen is. Zonder deze bewering zou de test doorgaan naar de volgende stap en in paniek raken vanwege een nil-pointer.

## Probeer de test uit te voeren

`wallet_test.go:61: got err 'oh no' want 'cannot withdraw, insufficient funds'`

## Schrijf genoeg code om de test te laten slagen

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return errors.New("cannot withdraw, insufficient funds")
	}

	w.balance -= amount
	return nil
}
```

## Refactor

De foutmelding komt zowel in de testcode als in de `Withdraw`-code voor.

Het zou echt vervelend zijn als de test mislukt als iemand de fout opnieuw zou willen formuleren, en het is gewoon te gedetailleerd voor onze test. Het maakt ons niet zoveel uit wat de exacte formulering is, zolang er maar een zinvolle fout rond het intrekken wordt geretourneerd onder bepaalde voorwaarden.

In Go zijn fouten waarden, dus we kunnen deze omzetten in een variabele en er één bron van waarheid voor hebben.

```go
var ErrInsufficientFunds = errors.New("cannot withdraw, insufficient funds")

func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return ErrInsufficientFunds
	}

	w.balance -= amount
	return nil
}
```

Met het sleutelwoord `var` kunnen we waarden definiëren die globaal zijn voor het pakket.

Dit is op zich al een positieve verandering, want nu ziet onze `Withdraw`-functie er heel overzichtelijk uit.

Vervolgens kunnen we onze testcode refactoren om deze waarde te gebruiken in plaats van specifieke strings.

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw with funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		wallet.Withdraw(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw insufficient funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(100))

		assertError(t, err, ErrInsufficientFunds)
		assertBalance(t, wallet, Bitcoin(20))
	})
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
	t.Helper()
	got := wallet.Balance()

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}

func assertError(t testing.TB, got, want error) {
	t.Helper()
	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

En nu is de test ook gemakkelijker te volgen.

Ik heb de helpers uit de hoofd-testfunctie gehaald, zodat wanneer iemand een bestand opent, hij of zij eerst onze test beweringen kan lezen in plaats van een aantal helpers.

Een andere nuttige eigenschap van tests is dat ze ons helpen het _werkelijke_ gebruik van onze code te begrijpen, zodat we duidelijke code kunnen maken. We zien hier dat een ontwikkelaar simpelweg onze code kan aanroepen, een equals-check kan uitvoeren op `ErrInsufficientFunds` en daar naar kan handelen.

### Ongegecontroleerde fouten

Hoewel de Go-compiler je veel helpt, zijn er soms toch nog dingen die je over het hoofd ziet en de foutafhandeling kan soms lastig zijn.

Er is één scenario dat we nog niet hebben getest. Om dit te vinden, voer je het volgende uit in een terminal om `errcheck` te installeren, een van de vele linters die beschikbaar zijn voor Go.

`go install github.com/kisielk/errcheck@latest`

Voer vervolgens in de directory met je code het volgende uit: `errcheck .`

Je zou zoiets moeten krijgen als

`wallet_test.go:17:18: wallet.Withdraw(Bitcoin(10))`

Wat dit ons vertelt, is dat we de fout die op die regel code wordt geretourneerd, niet hebben gecontroleerd. Die regel code op mijn computer komt overeen met ons normale opnamescenario, omdat we niet hebben gecontroleerd of er geen fout wordt geretourneerd als `Withdraw` succesvol is.

Hier is de definitieve testcode die hiermee rekening houdt.

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))

		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw with funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(10))

		assertNoError(t, err)
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw insufficient funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(100))

		assertError(t, err, ErrInsufficientFunds)
		assertBalance(t, wallet, Bitcoin(20))
	})
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
	t.Helper()
	got := wallet.Balance()

	if got != want {
		t.Errorf("got %s want %s", got, want)
	}
}

func assertNoError(t testing.TB, got error) {
	t.Helper()
	if got != nil {
		t.Fatal("got an error but didn't want one")
	}
}

func assertError(t testing.TB, got error, want error) {
	t.Helper()
	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got != want {
		t.Errorf("got %s, want %s", got, want)
	}
}
```

## Samenvattend

### Pointers

* Go kopieert waarden wanneer je ze doorgeeft aan functies/methoden. Als je dus een functie schrijft waarvan de status moet worden gewijzigd, moet deze een pointer hebben naar datgene wat je wilt wijzigen.
* Het feit dat Go een kopie van waarden maakt, is vaak nuttig, maar soms wil je niet dat je systeem een ​​kopie van iets maakt. In dat geval moet je een referentie doorgeven. Voorbeelden hiervan zijn het verwijzen naar zeer grote datastructuren of dingen waarbij slechts één instantie nodig is (zoals databaseverbindingspools).

### nil

* Pointers kunnen _nil_ zijn
* Wanneer een functie een aanwijzer naar iets retourneert, moet je controleren of deze waarde nil is. Anders loop je het risico dat er een runtime-uitzondering ontstaat. De compiler kan je hier niet bij helpen.
* Handig als je een waarde wilt beschrijven die mogelijk ontbreekt

### Errors

* Errors zijn de manier om aan te geven dat er een fout is opgetreden bij het aanroepen van een functie/methode.
* Door naar onze tests te luisteren, concludeerden we dat het controleren op een string in een fout zou resulteren in een onbetrouwbare test. Daarom hebben we onze implementatie gerefactored om in plaats daarvan een betekenisvolle waarde te gebruiken. Dit resulteerde in eenvoudiger te testen code en concludeerden dat dit ook voor gebruikers van onze API eenvoudiger zou zijn.
* Dit is niet het einde van het verhaal over foutverwerking. Je kunt geavanceerdere dingen doen, maar dit is slechts een introductie. Latere secties zullen meer strategieën behandelen.
* [Controleer niet alleen op fouten, maar handel ze ook op een correcte manier af](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

### Nieuwe typen maken van bestaande typen

* Handig om meer domeinspecifieke betekenis aan waarden toe te voegen
* Geeft je de mogelijkheid om interfaces te implementeren

Pointers en errors vormen een belangrijk onderdeel van het schrijven van Go en je moet er vertrouwd mee raken. Gelukkig helpt de compiler je _meestal_ als je iets verkeerd doet; neem gewoon de tijd en lees de fout.

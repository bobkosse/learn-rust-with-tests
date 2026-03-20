# Revisiting HTTP Handlers

[**Je vindt alle code voor dit hoofdstuk hier**](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/http-handlers-revisited)

Dit boek bevat al een hoofdstuk over [het testen van een HTTP-handler](../bouw-een-applicatie/http-server.md), maar dit zal een bredere bespreking van het ontwerp ervan bevatten, zodat ze eenvoudig te testen zijn.

We bekijken een praktijkvoorbeeld en hoe we het ontwerp ervan kunnen verbeteren door principes toe te passen zoals het principe van één verantwoordelijkheid en scheiding van belangen. Deze principes kunnen worden gerealiseerd met behulp van [interfaces](../basisbeginselen-go/structs-methods-and-interfaces.md) en [dependency injection](../basisbeginselen-go/dependency-injection.md). Hiermee laten we zien dat het testen van handlers eigenlijk heel triviaal is.

![Veelgestelde vraag in de Go-community geïllustreerd](../.gitbook/assets/amazing-art.png)

Het testen van HTTP-handlers lijkt een terugkerende vraag te zijn in de Go-community, en ik denk dat het wijst op een breder probleem: mensen begrijpen niet hoe ze deze moeten ontwerpen.

De problemen die mensen met testen ondervinden, komen vaak voort uit het ontwerp van hun code in plaats van het daadwerkelijk schrijven van de tests. Zoals ik zo vaak in dit boek benadruk:

> Als je tests je problemen bezorgen, luister dan naar dat signaal en denk na over het ontwerp van je code.

## Een voorbeeld

[Santosh Kumar tweeted me](https://twitter.com/sntshk/status/1255559003339284481)

> Hoe test ik een http-handler die afhankelijk is van Mongodb?

Hier is de code

```go
func Registration(w http.ResponseWriter, r *http.Request) {
	var res model.ResponseResult
	var user model.User

	w.Header().Set("Content-Type", "application/json")

	jsonDecoder := json.NewDecoder(r.Body)
	jsonDecoder.DisallowUnknownFields()
	defer r.Body.Close()

	// check if there is proper json body or error
	if err := jsonDecoder.Decode(&user); err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// Connect to mongodb
	client, _ := mongo.NewClient(options.Client().ApplyURI("mongodb://127.0.0.1:27017"))
	ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
	err := client.Connect(ctx)
	if err != nil {
		panic(err)
	}
	defer client.Disconnect(ctx)
	// Check if username already exists in users datastore, if so, 400
	// else insert user right away
	collection := client.Database("test").Collection("users")
	filter := bson.D{{"username", user.Username}}
	var foundUser model.User
	err = collection.FindOne(context.TODO(), filter).Decode(&foundUser)
	if foundUser.Username == user.Username {
		res.Error = UserExists
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	pass, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
	if err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}
	user.Password = string(pass)

	insertResult, err := collection.InsertOne(context.TODO(), user)
	if err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// return 200
	w.WriteHeader(http.StatusOK)
	res.Result = fmt.Sprintf("%s: %s", UserCreated, insertResult.InsertedID)
	json.NewEncoder(w).Encode(res)
	return
}
```

Laten we eens kijken wat deze ene functie allemaal moet doen:

1. HTTP-reacties schrijven, headers, statuscodes, enz. versturen.
2. De body van het verzoek decoderen naar een 'Gebruiker'.
3. Verbinding maken met een database (en alle details daaromheen).
4. De database bevragen en afhankelijk van het resultaat business logic toepassen.
5. Een wachtwoord genereren.
6. Een record invoegen.

Dit is te veel.

## Wat is een HTTP-handler en wat moet hij doen?

Ik vergeet even de specifieke details van Go, maar ongeacht de taal waarin ik heb gewerkt, heeft het me altijd goed van pas gekomen om na te denken over de [scheiding van belangen](https://en.wikipedia.org/wiki/Separation_of_concerns) en het [principe van individuele verantwoordelijkheid](https://en.wikipedia.org/wiki/Single-responsibility_principle).

Dit kan best lastig zijn om toe te passen, afhankelijk van het probleem dat je oplost. Wat _is_ precies een verantwoordelijkheid?

De grenzen kunnen vervagen, afhankelijk van hoe abstract je denkt en soms klopt je eerste gok niet.

Gelukkig heb ik met HTTP-handlers een redelijk goed idee wat ze moeten doen, ongeacht aan welk project ik heb gewerkt:

1. Een HTTP-verzoek accepteren, parsen en valideren.
2. Roep `ServiceThing` aan om `ImportantBusinessLogic` uit te voeren met de gegevens die ik uit stap 1 heb gehaald.
3. Stuur een passend `HTTP`-antwoord, afhankelijk van wat `ServiceThing` retourneert.

Ik zeg niet dat elke HTTP-handler ooit ongeveer deze vorm zou moeten hebben, maar 99 van de 100 keer lijkt dat voor mij het geval te zijn.

Wanneer je deze aandachtspunten scheidt:

* Het testen van handlers wordt een fluitje van een cent en richt zich op een klein aantal aandachtspunten.
* Belangrijk: het testen van `ImportantBusinessLogic` hoeft zich niet langer bezig te houden met `HTTP`; je kunt de bedrijfslogica schoon testen.
* Je kunt `ImportantBusinessLogic` in andere contexten gebruiken zonder het te hoeven aanpassen.
* Als `ImportantBusinessLogic` verandert wat het doet, hoef je je handlers niet te wijzigen, zolang de interface hetzelfde blijft.

## Go's Handlers

[`http.HandlerFunc`](https://golang.org/pkg/net/http/#HandlerFunc)

> Het type HandlerFunc is een adapter die het gebruik van gewone functies als HTTP-handlers mogelijk maakt.

`type HandlerFunc func(ResponseWriter, *Request)`

Lezer, haal even adem en bekijk de bovenstaande code. Wat valt je op?

**Het is een functie die argumenten accepteert**

Er zit geen frameworkmagie in, geen annotaties, geen magic beans, niets.

Het is gewoon een functie, _en we weten hoe we functies moeten testen_.

Het sluit mooi aan bij het bovenstaande commentaar:

* Het accepteert een [`http.Request`](https://golang.org/pkg/net/http/#Request), wat gewoon een bundel data is die we kunnen inspecteren, parsen en valideren. \* > [Een `http.ResponseWriter`-interface wordt door een HTTP-handler gebruikt om een ​​HTTP-respons samen te stellen.](https://golang.org/pkg/net/http/#ResponseWriter)

### Supereenvoudige voorbeeldtest

```go
func Teapot(res http.ResponseWriter, req *http.Request) {
	res.WriteHeader(http.StatusTeapot)
}

func TestTeapotHandler(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/", nil)
	res := httptest.NewRecorder()

	Teapot(res, req)

	if res.Code != http.StatusTeapot {
		t.Errorf("got status %d but wanted %d", res.Code, http.StatusTeapot)
	}
}
```

Om onze functie te testen, _roepen_ we deze aan.

Voor onze test geven we een `httptest.ResponseRecorder` mee als ons `http.ResponseWriter`-argument, en onze functie gebruikt deze om het `HTTP`-antwoord te schrijven. De recorder neemt op (of _bespioneert_) wat er is verzonden, waarna we onze beweringen kunnen doen.

## Een `ServiceThing` aanroepen in onze handler

Een veelgehoorde klacht over TDD-tutorials is dat ze altijd "te simpel" en niet "realistisch genoeg" zijn. Mijn antwoord daarop is:

> Zou het niet fijn zijn als al je code eenvoudig te lezen en te testen was, zoals de voorbeelden die je noemt?

Dit is een van de grootste uitdagingen waar we voor staan, maar waar we naar moeten blijven streven. Het _is mogelijk_ (hoewel niet per se gemakkelijk) om code te ontwerpen, dus het kan eenvoudig te lezen en te testen zijn als we goede software engineering-principes oefenen en toepassen.

Samenvattend wat de handler van eerder doet:

1. HTTP-reacties schrijven, headers, statuscodes, enz. versturen.
2. De body van het verzoek decoderen naar een 'User'.
3. Verbinding maken met een database (en alle details daaromheen).
4. De database bevragen en afhankelijk van het resultaat bedrijfslogica toepassen.
5. Een wachtwoord genereren.
6. Een record invoegen.

Uitgaande van een meer ideale scheiding van belangen, zou ik het meer als volgt willen:

1. De body van het verzoek decoderen naar een 'User'.
2. Een 'UserService.Register(user)' aanroepen (dit is onze 'ServiceThing').
3. Als er een fout optreedt (het voorbeeld stuurt altijd een '400 BadRequest', wat ik niet juist vind), gebruik ik voorlopig gewoon een '500 Internal Server Error'-handler. Ik moet benadrukken dat het retourneren van '500' voor alle fouten een vreselijke API oplevert! Later kunnen we de foutafhandeling geavanceerder maken, bijvoorbeeld met [error types](error-types.md).
4. Als er geen fout is, `201 Created` met de ID als antwoordbody (wederom vanwege de beknoptheid/luiheid).

Om het kort te houden, zal ik het gebruikelijke TDD-proces niet bespreken; raadpleeg de andere hoofdstukken voor voorbeelden.

### Nieuw ontwerp

```go
type UserService interface {
	Register(user User) (insertedID string, err error)
}

type UserServer struct {
	service UserService
}

func NewUserServer(service UserService) *UserServer {
	return &UserServer{service: service}
}

func (u *UserServer) RegisterUser(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()

	// request parsing and validation
	var newUser User
	err := json.NewDecoder(r.Body).Decode(&newUser)

	if err != nil {
		http.Error(w, fmt.Sprintf("could not decode user payload: %v", err), http.StatusBadRequest)
		return
	}

	// call a service thing to take care of the hard work
	insertedID, err := u.service.Register(newUser)

	// depending on what we get back, respond accordingly
	if err != nil {
		//todo: handle different kinds of errors differently
		http.Error(w, fmt.Sprintf("problem registering new user: %v", err), http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusCreated)
	fmt.Fprint(w, insertedID)
}
```

Onze `RegisterUser`-methode komt overeen met de vorm van `http.HandlerFunc`, dus we kunnen aan de slag. We hebben het als methode gekoppeld aan een nieuw type `UserServer`, dat een afhankelijkheid bevat van een `UserService`, die wordt vastgelegd als een interface.

Interfaces zijn een fantastische manier om ervoor te zorgen dat onze `HTTP`-zorgen losgekoppeld zijn van een specifieke implementatie; we kunnen de methode gewoon aanroepen op de afhankelijkheid en hoeven ons niet druk te maken over _hoe_ een gebruiker wordt geregistreerd.

Als je deze aanpak na TDD verder wilt verkennen, lees dan het hoofdstuk [Dependency Injection](../basisbeginselen-go/dependency-injection.md) en het hoofdstuk [HTTP Server in de sectie "Een applicatie bouwen"](../bouw-een-applicatie/http-server.md).

Nu we ons hebben losgekoppeld van specifieke implementatiedetails rondom registratie, is het schrijven van de code voor onze handler eenvoudig en volgt het de eerder beschreven verantwoordelijkheden.

### De tests!

Deze eenvoud zie je terug in onze tests.

```go
type MockUserService struct {
	RegisterFunc    func(user User) (string, error)
	UsersRegistered []User
}

func (m *MockUserService) Register(user User) (insertedID string, err error) {
	m.UsersRegistered = append(m.UsersRegistered, user)
	return m.RegisterFunc(user)
}

func TestRegisterUser(t *testing.T) {
	t.Run("can register valid users", func(t *testing.T) {
		user := User{Name: "CJ"}
		expectedInsertedID := "whatever"

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return expectedInsertedID, nil
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusCreated)

		if res.Body.String() != expectedInsertedID {
			t.Errorf("expected body of %q but got %q", res.Body.String(), expectedInsertedID)
		}

		if len(service.UsersRegistered) != 1 {
			t.Fatalf("expected 1 user added but got %d", len(service.UsersRegistered))
		}

		if !reflect.DeepEqual(service.UsersRegistered[0], user) {
			t.Errorf("the user registered %+v was not what was expected %+v", service.UsersRegistered[0], user)
		}
	})

	t.Run("returns 400 bad request if body is not valid user JSON", func(t *testing.T) {
		server := NewUserServer(nil)

		req := httptest.NewRequest(http.MethodGet, "/", strings.NewReader("trouble will find me"))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusBadRequest)
	})

	t.Run("returns a 500 internal server error if the service fails", func(t *testing.T) {
		user := User{Name: "CJ"}

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return "", errors.New("couldn't add new user")
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusInternalServerError)
	})
}
```

Nu onze handler niet meer gekoppeld is aan een specifieke implementatie van storage, is het voor ons triviaal om een `MockUserService` te schrijven waarmee we eenvoudige, snelle unittests kunnen schrijven om de specifieke verantwoordelijkheden uit te voeren.

### Hoe zit het met de databasecode? Je speelt vals!

Dit is allemaal heel bewust gedaan. We willen niet dat HTTP-handlers zich bezighouden met onze bedrijfslogica, databases, verbindingen, enz.

Door dit te doen, hebben we de handler bevrijd van rommelige details en hebben we het _ook_ gemakkelijker gemaakt om onze persistentielaag en bedrijfslogica te testen, omdat deze ook niet langer gekoppeld is aan irrelevante HTTP-details.

Het enige wat we nu nog hoeven te doen, is onze `UserService` implementeren met behulp van de database die we willen gebruiken.

```go
type MongoUserService struct {
}

func NewMongoUserService() *MongoUserService {
	//todo: pass in DB URL as argument to this function
	//todo: connect to db, create a connection pool
	return &MongoUserService{}
}

func (m MongoUserService) Register(user User) (insertedID string, err error) {
	// use m.mongoConnection to perform queries
	panic("implement me")
}
```

We kunnen dit apart testen en zodra we tevreden zijn in `main` kunnen we deze twee units aan elkaar koppelen voor onze werkende applicatie.

```go
func main() {
	mongoService := NewMongoUserService()
	server := NewUserServer(mongoService)
	http.ListenAndServe(":8000", http.HandlerFunc(server.RegisterUser))
}
```

### Een robuuster en uitbreidbaarder ontwerp met weinig moeite

Deze principes maken ons leven niet alleen gemakkelijker op de korte termijn, ze maken het systeem ook gemakkelijker uit te breiden in de toekomst.

Het zou niet verwonderlijk zijn als we bij verdere iteraties van dit systeem de gebruiker een registratiebevestiging per e-mail zouden willen sturen.

Met het oude ontwerp zouden we de handler _en_ de omliggende tests moeten aanpassen. Dit is vaak de reden waarom delen van de code niet meer te onderhouden zijn, er sluipt steeds meer functionaliteit in omdat het al zo _ontworpen_ is; zodat de "HTTP-handler"... alles kan afhandelen!

Door aandachtspunten te scheiden met behulp van een interface hoeven we de handler _helemaal_ niet te bewerken, omdat deze zich niet bezighoudt met de bedrijfslogica rondom registratie.

## Samenvattend

Het testen van Go's HTTP-handlers is niet uitdagend, maar het ontwerpen van goede software kan dat wel zijn!

Mensen maken de fout te denken dat HTTP-handlers speciaal zijn en gooien goede software engineering practices overboord bij het schrijven ervan, wat het testen ervan vervolgens lastig maakt.

Nogmaals: **Go's HTTP-handlers zijn gewoon functies**. Als je ze schrijft zoals je andere functies zou schrijven, met duidelijke verantwoordelijkheden en een goede scheiding van taken, zul je geen moeite hebben met het testen ervan en zal je codebase er gezonder door worden.

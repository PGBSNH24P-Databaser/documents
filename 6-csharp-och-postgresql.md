# C# och PostgreSQL

När man arbetar med databaser i C# så behöver man ett sätt att kommunicera med databasen. För PostgreSQL används Npgsql, vilket är ett paket som installeras via NuGet eller dotnet. När det är installerat kan vi börja med grunderna - att ansluta till databasen.

Kör följande kommando för att installera `Npgsql`:
`dotnet add package Npgsql`

En anslutning till PostgreSQL kräver en "connection string", vilket är en text som innehåller information om var databasen finns och hur vi ska logga in:

```csharp
var connectionString = "Host=localhost;Username=your_username;Password=your_password;Database=your_database";
```

Låt oss säga att vi har två tabeller i vår databas - en för husdjur och en för deras ägare:

```sql
CREATE TABLE pets (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    favoriteFood VARCHAR(100),
    type VARCHAR(50)
);

CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    petId INTEGER REFERENCES pets(id)
);
```

Dessa används för att kunna visa exempel med kod.

För att arbeta mot databasen genom kod så används flera klasser och metoder. Här är ett exempel där vi vill ta reda på vad en specifik kunds husdjur heter:

```csharp
// Skapa anslutningen
using var connection = new NpgsqlConnection(connectionString);

// Öppna anslutningen till databasen
connection.Open();

// Skapa vår SQL-query
var sql = @"
    SELECT 
        pets.name as PetName,
        pets.type as PetType,
        pets.favoriteFood as FavoriteFood
    FROM pets 
    INNER JOIN customers ON pets.id = customers.petId 
    WHERE customers.name = @customerName";

// Skapa ett command-objekt som innehåller vår SQL
using var cmd = new NpgsqlCommand(sql, connection);

// Lägg till parametern på ett säkert sätt
cmd.Parameters.AddWithValue("customerName", "Amanda");

// Kör query:n och läs resultatet
using var reader = cmd.ExecuteReader();

// Om vi hittar något resultat
if (reader.Read())
{
    var petName = reader.GetString(0);      // Hämta första kolumnen (PetName)
    var petType = reader.GetString(1);      // Hämta andra kolumnen (PetType)
    var favFood = reader.GetString(2);      // Hämta tredje kolumnen (FavoriteFood)
    
    Console.WriteLine($"Amandas husdjur heter {petName}");
    Console.WriteLine($"Det är en {petType}");
    Console.WriteLine($"Den gillar att äta {favFood}");
}
```

Detta exempel visar flera viktiga koncept:

1. **Connection** - Vi skapar en anslutning till databasen med `NpgsqlConnection`
2. **Command** - Vi skapar ett kommando med vår SQL-query
3. **Parameters** - Vi använder parametrar (@customerName) för att skicka in värden, och även skydda mot SQL-injection
4. **Reader** - Vi läser resultatet rad för rad

När vi kör denna kod mot vår databas så kommer den hitta att Amanda har en häst som heter Ben som gillar att äta bananer.

Ofta vill vi dock göra mer än att bara läsa data. Säg att vi vill lägga till en ny kund med ett husdjur. Detta kräver att vi först lägger till husdjuret (för att få dess ID) och sedan kunden. Följande är ett exempel som visar hur detta kan ske - med hjälp av en transaktion:

```csharp
using var connection = new NpgsqlConnection(connectionString);
connection.Open();

// Starta en transaktion - om något går fel så ångras alla ändringar
using var transaction = connection.BeginTransaction();

try
{
    // Först lägger vi till husdjuret
    var petSql = @"
        INSERT INTO pets (name, type, favoriteFood) 
        VALUES (@name, @type, @food) 
        RETURNING id";  // RETURNING id ger oss det nya ID:t

    using var petCmd = new NpgsqlCommand(petSql, connection, transaction);
    petCmd.Parameters.AddWithValue("name", "Luna");
    petCmd.Parameters.AddWithValue("type", "cat");
    petCmd.Parameters.AddWithValue("food", "tuna");

    // ExecuteScalar används när vi förväntar oss ett enda värde tillbaka
    var newPetId = (int)petCmd.ExecuteScalar();

    // Sen lägger vi till kunden
    var customerSql = @"
        INSERT INTO customers (name, email, petId) 
        VALUES (@name, @email, @petId)";

    using var customerCmd = new NpgsqlCommand(customerSql, connection, transaction);
    customerCmd.Parameters.AddWithValue("name", "Maria");
    customerCmd.Parameters.AddWithValue("email", "maria@mail.com");
    customerCmd.Parameters.AddWithValue("petId", newPetId);

    // ExecuteNonQuery används när vi inte förväntar oss något resultat
    customerCmd.ExecuteNonQuery();

    // Om vi kommer hit har allt gått bra och vi kan spara ändringarna
    transaction.Commit();
    Console.WriteLine("Ny kund och husdjur har lagts till!");
}
catch (Exception ex)
{
    // Om något går fel så ångrar vi alla ändringar
    transaction.Rollback();
    Console.WriteLine($"Något gick fel: {ex.Message}");
}
```

I detta exempel ser vi flera nya koncept:

1. **Transaktioner** - Används för att göra flera ändringar som en enhet (referera till dokumentet om transaktioner för mer information)
2. **ExecuteScalar** - Används när vi förväntar oss ett enda värde tillbaka
3. **ExecuteNonQuery** - Används när vi inte förväntar oss något resultat
4. **Error handling** - Try-catch block för att hantera fel

Om vi vill söka efter flera husdjur samtidigt, till exempel alla hundar, så kan vi göra så här:

```csharp
using var connection = new NpgsqlConnection(connectionString);
connection.Open();

var sql = @"
    SELECT 
        pets.name,
        customers.name as owner,
        pets.favoriteFood
    FROM pets 
    INNER JOIN customers ON customers.petId = pets.id
    WHERE pets.type = @petType";

using var cmd = new NpgsqlCommand(sql, connection);
cmd.Parameters.AddWithValue("petType", "dog");

using var reader = cmd.ExecuteReader();

// Read() går till nästa rad, returnerar false när det inte finns fler rader
// Detta är en typ av iterator
while (reader.Read())
{
    var petName = reader.GetString(0);
    var ownerName = reader.GetString(1);
    var favoriteFood = reader.GetString(2);
    
    Console.WriteLine($"{petName} ägs av {ownerName} och älskar {favoriteFood}");
}
```

Detta kommer skriva ut alla hundar, deras ägare och favoritmat.

När du arbetar med databaser i C# är det viktigt att tänka på:

1. **Använd alltid parametrar** - Bygg aldrig SQL-strängar genom att sätta ihop text, det kan leda till säkerhetshål
2. **Stäng dina resurser** - Använd `using` för att se till att anslutningar stängs, eller stäng dem manuellt
3. **Använd transaktioner** för uppdateringar som hör ihop
4. **Hantera fel** - Använd try-catch för att fånga och hantera databasproblem

Om man vill så kan man skapa .sql filer och ladda in queries från dem. Detta är ofta bra om man har många, och stora, queries.
# SQL API vejledninger til visning og editering af GeoFA-data

## Indholdsfortegnelse

- [Vilkår for anvendelse af data](#vilkår-for-anvendelse-af-data)
- [Åbent API uden brug af tokens](#åbent-api-uden-brug-af-tokens)
- [Udvidet adgang til data gennem token](#udvidet-adgang-til-data-gennem-token)
- [Hent data som SQL query via API-endpoint](#hent-data-som-sql-query-via-api-endpoint)
- [Specifikationen](#specifikationen)
- [Metadata om relationer mm](#metadata-om-relationer-mm)
- [Editering](#editering)
  		- [Select](#select)
 		 - [Insert](#insert)
  		- [Update](#update)
  	- [Delete](#delete)
- [Example: Recreational cycle network](#example-recreational-cycle-network)
  	- [Themes and network topology](#themes-and-network-topology)
  	- [SQL examples for cycle network data](#sql-examples-for-cycle-network-data)


## Vilkår for anvendelse af data
Generelt stilles GeoFA data til fri afbenyttelse (under reglerne for Creative Commons CCBY). 

Du kan generelt finde mere vejledning om information om GeoFA på:  
https://www.geodanmark.dk/home/vejledninger/geofa/ 

## Åbent API uden brug af tokens
Hvis du blot skal læse offentlig tilgængelige temaer, kan du få adgang uden brug af tokens. Det token-løse API endpoint hedder `https://geofa.geodanmark.dk/api/v4/sql/database/fkg` og anvendes på samme måde, som det med tokens (se nedenfor). Du kan blot lade være sætte `Authorization` headeren i dine requests.

## Udvidet adgang til data gennem token
For at kunne se data skjult for offentligheden (f.eks. data som kun må ses af myndigheder) eller hvis du vil editere data gennem SQL API, skal du oprettes som bruger i GeoFA, hvis du ikke allerede er det. 
Send en anmodning om brugeroprettelse til: support@geopartner.dk
Hvis du allerede har adgang til at editere data via GeoFA-webkortet eller GeoFA-editoren, så skal du benytte det brugernavn og password du bruger til dette.
Når du har din bruger, kan du med din bruger få et access token og et refresh token. Access token har 1 times brug og er derefter ugyldigt. Du kan få et nyt enten via brug af refresh token eller ved at anmode igen. 
Når du skal anmode om dine tokens, så skal du bruge denne url: 
https://geofa.geodanmark.dk/api/v4/oauth
Det kan anbefales at bruge Postman til at tjekke, om det virker. Her er et eksempel på en anmodning:

![image](https://github.com/user-attachments/assets/1853bd4f-abe8-458f-adbb-61287e775c74)

 
Username og password skal selvfølgelig erstattes med din egen bruger.
Derudover er det vigtigt, at du sætter ”Content-Type” til ”application/json” og ”Accept” til ”application/json: charset=utf-8”.  I Postman ser det sådan her ud: 

![image](https://github.com/user-attachments/assets/73930077-4e0f-4ad5-924d-642933bf4a09)

 
Med det kan du få dine tokens og komme i gang med at hente data.

Udenfor Postman vil et request for at få access token se sådan ud i http header:

```http
POST https://geofa-test.geodanmark.dk/api/v4/oauth
Content-Type: application/json
Accept: application/json; charset=utf-8

{
  "grant_type": "password",
  "username": "kom851",
  "password": "xxxxx",
  "database": "fkg",
  "client_id": "geofa"
}
```

## Hent data som SQL query via API-endpoint
Nu har du dit access token og kan derfor anmode om data fra databasen, API’et til databasen er: 
https://geofa.geodanmark.dk/api/v4/sql
For at teste, at det hele virker, kan man igen bruge Postman. Først skal man sætte sine headers op som på billedet: 

![image](https://github.com/user-attachments/assets/2326cd03-a148-4d8c-b153-436483915a5b)

 
Under ”Authorization” indsætter du dine egen Access token value efter ”Bearer”. Du skal herefter lave et SQL-statement, som passer til de data, du gerne vil hente. Her er et eksempel på hentning af Hundeskov/Hundepark/frit løbsareal som punkter:

![image](https://github.com/user-attachments/assets/e83dd7a2-c25b-412d-8458-22234461790e)

 
For at finde ud af hvad der skal stå i din SQL query, så kig nærmere i GeoFA-specifikationen.

### Specifikationen
I specifikationen kan du finde alle tabeller og kolonner, så du nemt kan komme i gang. Du finder den her:
https://confluence.kds.dk/display/GS/GeoFA-specifikation  
Et eksempel på en tabel i specifikationen:

![image](https://github.com/user-attachments/assets/749f468d-ec0e-40e0-b7eb-7059b86889d3)

 
Her kan du se, hvordan datamodellen er bygget og dermed hvilke data, du kan kalde. F.eks. kan du finde alle registrerede vandreruter (rute_t_k = 5). Eller måske er du kun interesseret i en særlig type af vandreruter. Så kan du finde dette, og f.eks. kun vise Hærvejen (rute_uty_k = 12).

 ![image](https://github.com/user-attachments/assets/3d5ee240-e864-41d5-aef5-d26adbaae49b)





### Metadata om relationer mm
Meta APIet returnerer information om en eller flere relationer. Her kan fx ses felter og disses typer, herunder enums. Dette kan være meget anvendeligt til udformning af SQL statements:

```http
GET https://geofa-test.geodanmark.dk/api/v3/meta/fkg.t_5607_ladefacilitet
Content-Type: application/json
Accept: application/json; charset=utf-8
Authorization: Bearer {{token}}
```

Den sidste del af url kan have flere relationer separeret med komma. Alle udgivne relationer kan hentes således:

```http
GET https://geofa-test.geodanmark.dk/api/v3/meta/tag:udgivet
Content-Type: application/json
Authorization: Bearer {{token}}
Accept: application/json; charset=utf-8
```

**Bemærk**: De relationer, der anvendes til hentning og ændring af data er *database views*. I views kan constraints ikke ses, fx om et felt er primary key eller er nullable. 


### Editering
Data kan hentes og ændres gennem SQL APIet. Systemet har en regelbaseret adgangskontrol, som omskriver den sendte SQL statement med `where` clauses eller blokkerer (eller giver lov) til de pågældende relationer.

Den anvendte database engine er PostgreSQL med PostGIS extension.

#### Select
En statememt skal sendes som en string literal. En statement kan parametriseres, og ved insert/update/delete kan der sendes flere sæt parametre for en statement.

Her er en simple **select** statement uden parametre:  

```http
POST https://geofa-test.geodanmark.dk/api/v4/sql
Content-Type: application/json
Accept: application/json; charset=utf-8
Authorization: Bearer {{token}}

{
  "q": "select * from fkg.t_5607_ladefacilitet where kommunekode=851 limit 10"
}
```

Tilsidst i repsonsen ses property `auth_check.statement`, som indeholder den statement, der faktisk blev kørt. Fx som user kom851 ser den såleds ud:

```sql
select *
  from fkg.t_5607_ladefacilitet
  where kommunekode = 851
    and (off_kode in (1, 3) or kommunekode = 851)
limit 10
```

Det er muligt at parametrisere statement strengen:

```http
POST https://geofa-test.geodanmark.dk/api/v4/sql
Content-Type: application/json
Accept: application/json; charset=utf-8
Authorization: Bearer {{token}}

{
  "q": "select * from fkg.t_5607_ladefacilitet where kommunekode=:kode limit 10",
  "params": {"kode":  851},
  "srs": 25832
}
```
Geometry leveres som default i EPSG:4326. Man ændre dette ved at angive den ønskede EPSG kode med en `srs` property.

#### Insert

Inserts kan også laves med eller uden parametre. Men her kan der være en stor fordel at anvende parametre. Specielt hvis der skal indsættes flere objekter på en gang. Her kan der nemlig sendes én statement og et vilkårlig antal sæt parametre. Dette er langt mere effektivt end at sende hver enkelt insert i sin egen request. Alle inserts bliver også behandlet i en transaktion (enten bliver alle indsat eller ingen bliver det).

Hvis man indsætter objekter, som ikke overholder reglerne for den pågældende bruger, vil der blive returneret en LIMIT error.

Her er et **insert** eksemple med to objekter. Bemærk at PostGIS funktionen `st_geomfromgeojson` anvendes til geometrien. Andre funktioner kan også anvendes fx `st_geomfromtext` eller `st_point`. Bemærk også, at statementen har `returning objekt_id` tilsidst. Derved leveres de to nye `objekt_id` tilbage. Flere felter kan angives separeret med komma eller `*`, som giver de nye objekter tilbage med alle felter:

```http
POST https://geofa-test.geodanmark.dk/api/v4/sql
Content-Type: application/json
Accept: application/json; charset=utf-8
Authorization: Bearer {{token}}

{
  "q": "insert into fkg.t_5607_ladefacilitet (cvr_kode,bruger_id,oprindkode,statuskode,off_kode,ladefacilitet_type_kode,geometri) values(:cvr_kode,:bruger_id,:oprindkode,:statuskode,:off_kode,:ladefacilitet_type_kode,st_geomfromgeojson(:geometri)) returning objekt_id",
  "params": [
    {
      "cvr_kode": 29188378,
      "bruger_id": "Clever",
      "oprindkode": 0,
      "statuskode": 3,
      "off_kode": 1,
      "ladefacilitet_type_kode": 1,
      "geometri": {
        "type": "MultiPoint",
        "crs": {
          "type": "name",
          "properties": {
            "name": "EPSG:25832"
          }
        },
        "coordinates": [
          [
            555870.780456494,
            6320180.089515179
          ]
        ]
      }
    },
    {
      "cvr_kode": 29188378,
      "bruger_id": "Clever",
      "oprindkode": 0,
      "statuskode": 3,
      "off_kode": 1,
      "ladefacilitet_type_kode": 1,
      "geometri": {
        "type": "MultiPoint",
        "crs": {
          "type": "name",
          "properties": {
            "name": "EPSG:25832"
          }
        },
        "coordinates": [
          [
            555170.780456494,
            6320480.089515179
          ]
        ]
      }
    }
  ]
}
```

#### Update

Ved updates kan der ligeledes anvendes parametre. Update statements bliver typisk omskrevet i regelsystemet, hvor der bliver sat en `where` clause på. Endvidre bliver der kørt et post-tjek på de ændrede objekter for at tjekke, at de stadig overholder regelsættet for den pågældende bruger efter ændringen (fx kan man ikke flytte et objekt fra sin egen kommune til en anden). Hvis ikke, vil en LIMIT error blive sendt og i det tilfælde vil ingen objekter blive ændret.   

Hvis reglerne bevirker, at der ikke foretages ændringer i mindst et objekt (fx alle objekterne ligger uden for kommunegrænsen fra starten) vil en COUNT 0 error blive sendt. Dette gøres for at oplyse brugeren om, at intet blev ændret. Men bemærk, at denne fejl kun bliver sendt hvis intet skete. Fx hvis to objekter forsøges ændret og det ene ligger udenfor for kommunegrænsen og det andet inde, vil transationen blive gennemført med én ændring. Det kan være en god ide, at tjekke `affected_rows` i responsen, som angiver hvor mange objekter, der blev ændret.   

Her et **update** eksempel, der ændrer to objekter:

```http
POST https://geofa-test.geodanmark.dk/api/v4/sql
Content-Type: application/json
Accept: application/json; charset=utf-8
Authorization: Bearer {{token}}

{
  "q": "update fkg.t_5607_ladefacilitet set off_kode=:off_kode,statuskode=:statuskode where objekt_id=:objekt_id",
  "params": [
    {
      "objekt_id": "6427cf5c-db9f-11ee-bb57-bb467bf148f9",
      "off_kode": 1,
      "statuskode": 3
    },
    {
      "objekt_id": "642adbb6-db9f-11ee-bb57-bb467bf148f9",
      "off_kode": 1,
      "statuskode": 2
    }
  ]
}
```

#### Delete

Delete kan også laves med og uden paremtre. Her er et eksempel på en **delete** statement, som sletter et enkelt objekt. Som ved update, vil en kørt delete statement, der ikke ændrer noget resultere i en LIMIT 0 error. Og ligeledes vil regelsystemet typisk sætte en `where` clause på en delete statement:

```http
POST https://geofa-test.geodanmark.dk/api/v4/sql
Content-Type: application/json
Accept: application/json; charset=utf-8
Authorization: Bearer {{token}}

{
  "q": "delete from fkg.t_5607_ladefacilitet where objekt_id=:objekt_id",
  "params": [
    {
      "objekt_id": "6427cf5c-db9f-11ee-bb57-bb467bf148f9"
    }
  ]
}
```

## Example: Recreational cycle network

For general API access, authentication, SQL requests and data editing, use the instructions above.

### Themes and network topology

| Theme | Table | Description |
|---|---|---|
| 5608 | `fkg.t_5608_cykelkrydspunkter` | Numbered cycle network nodes (point geometry). |
| 5609 | `fkg.t_5609_cykelkrydspunktsstraekninger` | Cycle network stretches connecting nodes (line geometry). |

Theme 5609 does not contain explicit `from_node` or `to_node` columns. To derive connections between stretches and nodes, match the start and end vertices of a stretch to nearby points in theme 5608. Use an appropriate distance tolerance when performing this spatial matching.

### SQL examples for cycle network data

Send the SQL as the `q` value using the SQL API request format documented above. They are reference examples, not independently verified queries.

**1. Up to 500 cycle nodes in a municipality**

```sql
SELECT krydspunktsnummer, geometri
FROM fkg.t_5608_cykelkrydspunkter
WHERE kommunekode = 573  -- Varde
LIMIT 500
```

**2. Up to 500 stretches in the same municipality, with length**

```sql
SELECT objectid, ST_Length(geometri) AS length_m, geometri
FROM fkg.t_5609_cykelkrydspunktsstraekninger
WHERE kommunekode = 573
ORDER BY length_m DESC
LIMIT 500
```

**3. Nodes inside a bounding box**

```sql
SELECT krydspunktsnummer, geometri
FROM fkg.t_5608_cykelkrydspunkter
WHERE ST_Within(geometri, ST_MakeEnvelope(8.0, 55.4, 9.2, 56.2, 4326))
```

**4. Count nodes per municipality**

```sql
SELECT kommunekode, COUNT(*) AS node_count
FROM fkg.t_5608_cykelkrydspunkter
GROUP BY kommunekode
ORDER BY node_count DESC
```

**5. The 10 longest stretches nationally**

```sql
SELECT objectid, ST_Length(geometri) AS length_m
FROM fkg.t_5609_cykelkrydspunktsstraekninger
ORDER BY length_m DESC
LIMIT 10
```

**Note:** The examples presents these queries with a GET endpoint (`/api/v4/sql/fkg`) and describes its own API parameters and defaults. The original GeoFA guide above documents a different request format. The examples are therefore provided as SQL only, without assuming that the GET endpoint or its parameter defaults work identically. In particular, confirm the `objectid` field name and stored geometry coordinate system before relying on the length and bounding box examples.

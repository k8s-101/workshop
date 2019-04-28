Docker, en konteinerteknologi
=============================
_Her kommer første del av workshopen, hvor deltakterne dockerizerer de ulike applikasjonene. teksten er foreløpig bare notater._

### For å skjønne hvordan Docker funker, må man først skjønne bygg og terminal
Her kan vi starte med å si litt generelt om Docker, og at man jobber med det mye som om man skriver et terminal-script.

Så bør man si litt om hvordan man bygger .NET Core, siden det er det vi skal bygge her. Dekk restore, build og publish, og at publish gjør alt sammen siden v2.1.

Avslutter kanskje med å leke at man er docker i to mapper på maskinen?

**Hvorfor holder det ikke å bygge til release?**
Kan si to ord fra [denne posten](https://stackoverflow.com/questions/27320374/the-difference-between-build-and-publish-in-vs)
og litt fra [MSDN](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish?tabs=netcore21).

### Litt om context og sånt

**Hvordan sender man in en context?**
Den bestemmes av hva man sender inn når man bygger. Under har vi sagt at den skal være `.`, dvs. den mappen vi står i når vi kjører kommandoen, dvs. roten av repoet.

```shell
&> docker build -f Dockerfile -t speed-test-logger:0.0.1 .
```

Vi skriver en Dockerfile!
-------------------------
### Litt om base-imager
Man starter ikke helt på scratch, finner noe som har [.NET Core installert fra før](https://hub.docker.com/_/microsoft-dotnet-core). [Her](https://docs.docker.com/engine/examples/dotnetcore/) er det også litt nyttig informasjon.

_Motivasjonen er at vi trenger en datamaskin som har .NET Core SDK installert. Her kan det være mye rart avhengig av hva man skal bygge og kjøre._

### WORKDIR
Vi bruker denne kommandoen for å fortelle hvilken mappe vi skal jobbe i inne i kontaineren.

### COPY
Her henter vi inn filer fra context til Docker-kontaineren.

Burde ha en tegning litt som:
```
  ////////////////      /////////////////////////      ////////////////////
 /Kode på maskin/  =>  /Context til docker bygg/  =>  /COPY til kontainer/
////////////////      /////////////////////////      ////////////////////
```
Det er gjerne her ting går galt, vanskelig å huske helt hvilken filbaner man skal sette opp hvis man ikke konsentrerer.
Her er det greit å si to ord om `./` og `.`

### RUN
Så skal vi kjøre en kommando i image. I dette tilfellet `dotnet publish` med litt argumenter.

Greit å si to ord om å bruke `\` for å dele opp lange kommandoer i flere linjer. De er lettere å diffe i git etterpå.

### ENTRYPOINT
Vi kjører kanskje alt som single-stage bygg først? I så tilfelle er det ENTRYPOINT som skal kjøres nå. Etter det kan vi teste bygget.

### Mulit-stage bygg
Typisk vil vi ikke bruke samme base-image for å bygge å kjøre. Kjøre-imaget trenger ikke hele .NET SDK, bare runtime.

Vi utvider til multi-stage bygg her.

### To ord om LABEL?
Vi kan merke imagene våre med ting. Er ikke alltid vi bruker det, men kan være verdt å vite om.

### Et halvt ord om ENV?
speedtest-web kan nok ha behov for å ta inn miljøvariabler, så da kan vi kanskje introdusere dette.

### Bruk av .dockerignore for å unngå å hente inn flere filer enn man trenger
Det tar tid å laste inn data fra repoet til kontaineren. Typisk har man bygget kontaineren på forhånd, så da kan det være at man har /bin og /obj -mapper. Disse trenger man ikke, da man byger koden i kontaineren.

Her ser vi at størrelsen på dataene som lastes over til containeren er 328.2kB.

```shell
$> docker build -f Dockerfile -t speed-test-logger:0.0.1 .
Sending build context to Docker daemon  328.2kB <- Det er her vi ser
Step 1/8 : FROM microsoft/dotnet:2.1-sdk AS build
 ---> 7a0ebb4aec36
Step 2/8 : COPY /SpeedTestLogger /SpeedTestLogger
 ---> a3e1ca330732
Step 3/8 : WORKDIR /SpeedTestLogger
 ---> Running in 6ef6cbad47f3
Removing intermediate container 6ef6cbad47f3
 ---> 4561f24c40a6
Step 4/8 : RUN dotnet publish     -o /PublishedApp     -c Release
 ---> Running in ccc543b40f34
Microsoft (R) Build Engine version 16.0.450+ga8dc7f1d34 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 3.52 sec for /SpeedTestLogger/SpeedTestLogger.csproj.
  SpeedTestLogger -> /SpeedTestLogger/bin/Release/netcoreapp2.1/SpeedTestLogger.dll
  SpeedTestLogger -> /PublishedApp/
Removing intermediate container ccc543b40f34
 ---> 042fa539cfc3
Step 5/8 : FROM microsoft/dotnet:2.1-aspnetcore-runtime
 ---> 80cac97eddad
Step 6/8 : LABEL repository="github.com/k8s-101/speedtest-logger"
 ---> Using cache
 ---> e4ce98c8d68b
Step 7/8 : COPY --from=build /PublishedApp .
 ---> 29b45d4af568
Step 8/8 : ENTRYPOINT ["dotnet", "Detektor.Host.dll"]
 ---> Running in 1557bb960f60
Removing intermediate container 1557bb960f60
 ---> b969b16366a1
Successfully built b969b16366a1
Successfully tagged speed-test-logger:0.0.1
```

Så legger vi inn en .dockerignore på /bin og /obj. Da går størrelsen ned til 80.9kB. Det er fire ganger mindre!

```shell
✔ ~/k8s-101/speedtest-logger [master|✚ 1…1]
23:23 $ docker build -f Dockerfile -t speed-test-logger:0.0.1 .
Sending build context to Docker daemon   80.9kB <- Sjekk strørrelsen!
Step 1/8 : FROM microsoft/dotnet:2.1-sdk AS build
 ---> 7a0ebb4aec36
Step 2/8 : COPY /SpeedTestLogger /SpeedTestLogger
 ---> a702713cc2d1
Step 3/8 : WORKDIR /SpeedTestLogger
 ---> Running in 8bc101a7b00f
Removing intermediate container 8bc101a7b00f
 ---> 6d65f3050d27
Step 4/8 : RUN dotnet publish     -o /PublishedApp     -c Release
 ---> Running in b602360d2d0d
Microsoft (R) Build Engine version 16.0.450+ga8dc7f1d34 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 3.29 sec for /SpeedTestLogger/SpeedTestLogger.csproj.
  SpeedTestLogger -> /SpeedTestLogger/bin/Release/netcoreapp2.1/SpeedTestLogger.dll
  SpeedTestLogger -> /PublishedApp/
Removing intermediate container b602360d2d0d
 ---> 58b641a7e341
Step 5/8 : FROM microsoft/dotnet:2.1-aspnetcore-runtime
 ---> 80cac97eddad
Step 6/8 : LABEL repository="github.com/k8s-101/speedtest-logger"
 ---> Using cache
 ---> e4ce98c8d68b
Step 7/8 : COPY --from=build /PublishedApp .
 ---> Using cache
 ---> 29b45d4af568
Step 8/8 : ENTRYPOINT ["dotnet", "Detektor.Host.dll"]
 ---> Using cache
 ---> b969b16366a1
Successfully built b969b16366a1
Successfully tagged speed-test-logger:0.0.1
```

### Docker cacher steg i bygget
Her har vi et eksmpel på en kjøring hvor alle steg er cachet. Dette gikk super-raskt.

```shell
23:30 $ docker build -f Dockerfile -t speed-test-logger:0.0.1 .
Sending build context to Docker daemon   80.9kB
Step 1/8 : FROM microsoft/dotnet:2.1-sdk AS build
 ---> 7a0ebb4aec36
Step 2/8 : COPY /SpeedTestLogger /SpeedTestLogger
 ---> Using cache
 ---> a702713cc2d1
Step 3/8 : WORKDIR /SpeedTestLogger
 ---> Using cache
 ---> 6d65f3050d27
Step 4/8 : RUN dotnet publish     -o /PublishedApp     -c Release
 ---> Using cache
 ---> 58b641a7e341
Step 5/8 : FROM microsoft/dotnet:2.1-aspnetcore-runtime
 ---> 80cac97eddad
Step 6/8 : LABEL repository="github.com/k8s-101/speedtest-logger"
 ---> Using cache
 ---> e4ce98c8d68b
Step 7/8 : COPY --from=build /PublishedApp .
 ---> Using cache
 ---> 29b45d4af568
Step 8/8 : ENTRYPOINT ["dotnet", "Detektor.Host.dll"]
 ---> Using cache
 ---> b969b16366a1
Successfully built b969b16366a1
Successfully tagged speed-test-logger:0.0.1
```

Vi kunne også brukt "lag" for å få tli bedre caching. Et eksempel er å kjøre restore-steget på et eget lag.
Dermed kunne vi potensielt unngått å kjøre restore i kontaineren hver gang vi bygger den.

```dockerfile
COPY /SpeedTestLogger/SpeedTestLogger.csproj ./
RUN dotnet restore
```

Dette betyr at `dotnet restore` bare kjøres hvis det er en endring i SpeedTestLogger.csproj.

**Er dette egentlig noe poeng?**
Typisk får vi ikke så mye ut av dette, da bygg-serveren ikke cacher imager som er bygget der. Hvis ikke den gjør det, som f.eks. Azure DevOps/VSTS, så vil dette bare gjøre bygge tregere :-/

Nå kan vi litt om Docker, og kan dockerisere de andre repoene
-------------------------------------------------------------
Starter med speedtest-logger, og tar resten etter det. Bruker følgende kommando for å starte:

```shell
&> docker run -it speed-test-logger:0.0.1
```

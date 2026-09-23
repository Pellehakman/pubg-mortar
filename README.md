# Eldledning — mortarkalkylator för PUBG

Zoombar karta där du sätter en avfyrningsplats och sedan klickar runt på mål.
Räknar ut avstånd, bäring och vilken räckvidd du ska ställa in på mortaren när
målet ligger högre eller lägre än du.

## Kom igång

1. Lägg kartbilderna i `public/`:

   ```
   public/erangel.webp
   public/rondo.webp
   public/taego.webp
   ```

2. Installera och starta:

   ```bash
   npm install
   npm run dev
   ```

Fungerar installationen inte på grund av paketversioner: kör
`npm create vite@latest` och välj Svelte, kopiera sedan in `src/`, `index.html`,
`vite.config.js` och `svelte.config.js` från det här projektet.

## Användning

| Handling | Gör så här |
|---|---|
| Sätt avfyrningsplats | Klicka på kartan (vit prick) |
| Sätt mål | Klicka igen — orange cirkel visar spridningen |
| Byt mål | Klicka på nästa mål, avfyrningsplatsen ligger kvar |
| Ta bort avfyrningsplats | Klicka på den vita pricken |
| Panorera / zooma | Dra · scrolla · nyp |

Tangenter: `Esc` rensar målet, `Backspace` nollställer, `+` `−` `0` styr zoom.

På mobil fyller kartan skärmen och panelen ligger som ett draglakan nedtill.
Kollapsad visar den inställning, avstånd, bäring och höjd; dra upp handtaget för
kartval, höjdregel och banprofil.

## Skalan

Varje karta bär sin egen skala i `src/App.svelte`:

```js
{ id: 'erangel', label: 'Erangel', src: '/erangel.webp', pixels: 4096, meters: 8000 }
```

`pixels` är bildens bredd i pixlar, `meters` kartans verkliga bredd. Meter per
pixel räknas ut därifrån, så bilderna behöver inte ha samma upplösning.
**Stämmer inte `pixels` med filen blir alla avstånd fel med samma faktor.**

## Ballistikmodellen

Mortaren antas välja den höga banlösningen. Pipvinkeln för inställning `S`:

```
θ = 90° − ½·arcsin( (S / maxRange)^arcShape )
```

Farten väljs per inställning så att plan räckvidd blir exakt `S`. Det gör att
skott utan höjdskillnad alltid stämmer, oavsett hur parametrarna skruvas.
`arcShape = 1` ger konstant utgångsfart; lägre värden planar ut banan snabbare
med avståndet.

Höjdkompensationen löses numeriskt: hitta den inställning vars bana passerar
genom målets höjd på rätt avstånd. Höjden vid ett fast avstånd växer inte
monotont med inställningen — den toppar och faller sedan — så koden söker upp
toppen först och bisekterar på den växande delen.

### Kalibrering

Parametrarna i `src/App.svelte` är anpassade till mätningar i spelet:

| Kartavstånd | Höjdskillnad | Uppmätt inställning |
|---|---|---|
| 403 m | +130 m | 486 m |
| 503 m | +130 m | 612 m |

Nuvarande passform: `maxRange = 696`, `arcShape = 0.53`. Båda punkterna träffas
på centimetern.

**Oprövat:** båda mätningarna är uppför med samma höjdskillnad. Nedförsriktningen
och korta avstånd är inte validerade. Modellen säger 446 m för 503 m nedför
130 m — stämmer det håller formen rimligen åt båda hållen.

Nya mätningar justeras in genom att ändra `maxRange` och `arcShape` i
`src/App.svelte`. Högre `maxRange` ger mindre höjdkorrigering; lägre `arcShape`
ger flackare bana på långa håll och därmed större korrigering där.

## Filer

```
src/
├─ App.svelte              kartlista och kalibrerade parametrar
├─ app.css                 global layout
├─ main.js                 monteringspunkt
└─ lib/
   └─ MapViewer.svelte     allt: karta, ballistik, panel
```

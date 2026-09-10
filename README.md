# ¿A qué jugamos? — recomendador del armario de Iñigo Montoya

Página estática que habla con Claude y recomienda juegos **solo** de los que hay
en casa, filtrando por tiempo, número de jugadores y apetencia.

## Publicar en GitHub Pages

1. Repositorio nuevo, y dentro `index.html` + `juegos.json` en la raíz.
2. Settings → Pages → Source: `Deploy from a branch`, rama `main`, carpeta `/ (root)`.
3. En un par de minutos está en `https://<usuario>.github.io/<repo>/`.

No hay build, ni dependencias, ni backend.

## La clave de API

**No está en el código.** La página la pide la primera vez y la guarda en el
`localStorage` del navegador de cada persona.

Esto es a propósito. Si la clave estuviera escrita en `index.html`, cualquiera
con el repo público podría leerla, y los bots que rastrean GitHub la encontrarían
en cuestión de horas: el secret scanning de GitHub la revocaría, o alguien la
gastaría antes. Con este método la experiencia para ti es idéntica (la metes una
vez y ya) y el riesgo es cero.

Si quieres que la gente use la página **sin** tener clave propia, hace falta un
proxy: una función en Cloudflare Workers o Vercel con la clave en una variable de
entorno, y cambiar la URL del `fetch`. Son ~20 líneas, pero ya no es GitHub Pages.

## Regenerar los datos

Cuando entren juegos nuevos en el Excel:

```bash
curl -o bgg.csv    https://raw.githubusercontent.com/jalwz17/Board-Game-Data-Analysis/main/bgg_dataset.csv
curl -o rank22.csv https://raw.githubusercontent.com/albert-marrero/bgg-data/master/CSV/rankings/2022-07-13.csv
python3 enriquecer.py       # ajusta la ruta del .xlsx dentro del script
```

`enriquecer.py` cruza el inventario con los datasets **filtrando por los BGG ID
del inventario**, así que de las 20.000 filas del dataset original solo sobreviven
las ~90 que tenéis. El `juegos.json` resultante pesa 54 KB.

## Qué hay dentro y qué falta

De las 111 filas del Excel salen **95 juegos únicos**:

| | |
|---|---|
| Recomendables (con duración, jugadores, categorías y mecánicas) | **91** |
| Expansiones (excluidas: no son juego independiente) | 4 |
| Filas del Excel sin BGG ID → **fuera de la web** | 13 |

Los 13 excluidos son: La Comunidad Del Anillo (juego de bazas), Chao Pescao!,
Aeterna, Duelo Por Cardia, Cards against Downtime, Galaxia la conquista,
Mafia cosa di capo, Dagon, That's not a hat, Unlock, El espía que se perdió,
El señor de los anillos (parchís) y Bancarrota. Para meterlos hay que ponerles
su BGG ID en el Excel, o rellenarlos a mano en el diccionario `SIN_ID` del script.

### Fiabilidad de los datos

- **84 juegos** vienen del snapshot de BGG de febrero de 2021 (`fuente: bgg_dataset_2021`).
- **7 juegos** posteriores a 2021 (Ark Nova, Heat, Mindbug, Hive Pocket, Votes for
  Women, Ca$h 'n Gun$ Yakuzas, Sì Oscuro Signore) llevan datos **escritos a mano**
  y marcados con `fuente: "manual"` en el JSON. Están en el rango correcto, pero
  no proceden de un dataset verificado: si algo canta, corrígelo en el
  diccionario `MANUAL` de `enriquecer.py`.
- Las notas vienen del snapshot de julio de 2022, más reciente que el de 2021.

La API en vivo de BoardGameGeek no es accesible desde el entorno donde se generó
esto (mismo problema que ya documentaba la hoja «Fuentes» del Excel), por eso se
usan snapshots en GitHub en vez de datos del día.

### Por qué el JSON y no Google Drive en directo

El Excel de Drive **no tiene** duración, jugadores ni categorías: sin enriquecer,
no se puede recomendar por tiempo ni por apetencia. Como el enriquecimiento pasa
por un script, el resultado hay que guardarlo en algún sitio, y un JSON en el
mismo repo es más rápido y no depende de que Drive siga publicando el fichero.
La contrapartida honesta: si añades un juego al Excel, la web no se entera hasta
que vuelvas a correr `enriquecer.py`.

## Contra las invenciones

El modelo tiene prohibido escribir duraciones y números de jugadores en su texto.
Marca cada juego como `[[Nombre]]` y la página sustituye eso por una ficha con
los datos leídos del JSON. Si el modelo se inventa un juego que no existe, la
ficha sale en rojo diciendo que no está en la estantería, en vez de colártela.

# Ríos de Panamá ordenados

La capa de ríos y quebradas de Panamá (IGNTG, 2022) con los torrentes clasificados de pequeños a grandes. Se indican los valores usando varios sistemas (Strahler Shreve, Scheidegger y Drwal) según lo producido por el módulo v.stream.order de GRASS.

Esta es una versión bastante inicial, sin un control de calidad exhaustivo. **No doy absolutamente ninguna garantía sobre su nivel de precisión**.

Con esta capa puedes hacer mapas con distintos grosores según el órden del río:

![](/rios_azuero.jpeg)

Escríbeme a mir@almanaqueazul.org para dudas y quejas.

## Datos técnicos

Estas son unas notas que tomé sobre el proceso de clasificar estos datos usando GRASS, el cual en este caso me tomó dos semanas entre entender el problema y resolverlo. Están en inglés mezclado con español (quería hacer un tutorial) y un poco incompletas pero mejor aquí que en mi disco duro.

## Assigning stream order to a vector layer of rivers

Recently the government mapping office of Panama finally made digital data available to the public. Yes, in 2022 we now have an alternative to paper or PDF maps if you want official data. This happened as I am starting work on an ecological connectivity map of Panama, and it turns out the data are of very good quality. They did not release the entire river network vector data that you can see in the 1:25000 maps, but the main tributaries are there. However they are not ordered, i.e. there is no way to tell the larger from the smaller rivers and creeks.

I learned there are various systems to order stream networks, the most well known of which is the Strahler algorithm, but there are a bunch more. Basically you want each line of your river layer to have number like this:

Here we identify the initial streams in the watershed with a small number and further downstream the number increases. Having this is a good thing first of all to make your rivers look a lot better, but also for any kind of study where you need an estimate of flow or width of a river or how upstream or downstream your water body is.

### Automating river orders

There are ways to do this semi-automatically. I'm writing this basically because I found very little documentation online on how to do this and hopefully this can save someone a lot of time. The whole process is in the "advanced" level, you may want to move data between QGIS and Postgresql and GRASS for various parts of the process. But if you are a beginner -- you can learn to do all of this stuff online!

I used a module in GRASS GIS called v.stream.order that works on vector layers (hence the v.). There is one more free application (a QGIS plugin called H2roresO) but the GRASS one is the one I got to work. At some point I was desperate enough to ask a friend with an ArcGIS licence to do it for me, but turns out the ArcGIS Stream Order tool needs a raster elevation model as input (like the equivalent r.stream.order in GRASS).

I do not intend to provide a simple recipe from basic river line vector layer to fully ordered, simply because there are too many variables and you will need to do lots of manual work. But here are the basic things I learned.

### About GRASS

GRASS GIS es uno de los sistemas de información geográfica más antiguos pero sigue siendo usado por distintas industrias hoy en día. Cuando empecé a hacer mapas y por algún motivo necesité usar GRASS, lo tuve que hacer desde una línea de comando, ya hoy puedes operar GRASS desde su interfaz gráfico (un tanto peculiar) o desde dentro de QGIS, donde GRASS corre como un plugin. Para empezar a usarlo hará falta que hagas algunos pasos básicos de configuración -- busca en la documentación de GRASS.

I used a combination of GRASS and QGIS - mostly editing layers in QGIS and running commands in GRASS. You can open layers in both places, but you can only open a mapset --and run GRASS commands-- in one place (QGIS or GRASS) at the same time.

Una vez tengas GRASS funcionando y configurado para la región en la que vas a trabajar, tuve que instalar el addon v.stream.order, que no venía en mi instalación de GRASS (Linux Mint 21). Esto lo haces utilizando el comando g.extension, indicando v.stream.order en el nombre de la extensión. El comando (vale la pena que vayas aprendiendo a usar la línea de comando en el interfaz gráfico de GRASS) es

```
g.extension extension=v.stream.order operation=add
```

You need to feed two vector layers to v.stream.order:

1. A line layer of rivers
2. A point layer of outlets (where the river drains into the sea or a lake, i.e. where the river ends)

### Data in: rivers

The v.stream.order documentation as "the stream network map must be topologically correct". This means:

- rivers need to be separate features at intersection points
- river segments and tributaries need to meet at precisely the same point
- the last river segment that flows into the sea needs to have the right direction - into the sea. The direction of other parts of your river network does not seem to matter

If you want to make sure all pieces snap at exactly the same points, there are two tools I used:

1. Reduce the precision of your river layer, I used ST_ReducePrecision in Postgresql.
2. Enable snapping to vertex in QGIS. This will make your life a lot easier when correcting line segments that don't touch or if needing to split a line that escapes the procedure below.

The lines representing your rivers need to be broken up wherever they meet a tributary, as various parts of your river may have different order scores. This is not explained anywhere in the documentation and it took a while to realise this. So if your network is made of long lines of rivers from beginning to end this will not work.

In my case I needed to break up my lines. I followed these steps:

1. Make a points layer where river tributaries intersect. From QGIS (use the Processing Toolbox) upen "Line intersections" and set your river layer for both the input and the intersect layers. You will get a point layer in places where lines meet each other.

2. You now need to split your lines at these points. I used (this)[https://gis.stackexchange.com/questions/332363/problem-with-split-lines-by-points] answer in StackExchange.

   Use geometry by expression to create a temporary layer of lines. Use the split points as the input layer, and use a geometry expression like this:

   `make_line($geometry, make_point($x+1, $y))`

   The output should be a temporary layer of very short lines that start at the split points, and extend a very short distance due east.

   Check the output and make sure the lines don't cross any of your original lines more than once. If they do, re-do the first step with a smaller x-offset value (eg make_point($x+0.0001, $y)).

   Use these lines in the QGIS tool split with lines to split the original line layer.

Now you need to check the direction of the last segment of each of your rivers draining into the sea (or lake). In QGIS, add a "Marker line" symbol to your line layer and select "on last vertex only". This way you will have a line with a little dot at the end of it. This dot needs to be on the sea side of your river. Edit your layer and use the "Reverse line" tool to switch directions if it is wrong.

Now the first draft of your river layer is ready, you will be making small corrections (or big ones if you have a bad quality layer) as part of your trial and error.

### Data in: outlets

An outlet is the point(s) in a given watershed where the river drains out into the sea or another body of water. I used the "Line intersections" tool again, this time with a line layer of coastlines and your layer of rivers. I first used the polygon version of my coastlines and made a buffer of -10 m to make sure the rivers in my layer actually met the coastlines. In some cases this ended up making two intersections, which I had to fix by hand.

There can only be one outlet point in a river segment (the last one), but a given watershed may have more than one outlet in separate streams, the algorithm seemed to handle this well in most cases.

### Separating your watersheds

My GRASS installation is set up with the default SQLite database, but it turns out that this may not always work with complex algorithms -- I got an "sqlite3.OperationalError: database is locked" error. I tried to switch the database engine of GRASS to Postgresql, but it seems like there is a bug with the Psycopg2 library, I was getting a different error ("AttributeError: 'NoneType' object has no attribute 'fetchall'"). I decided it would take less work to make it work with SQLite, so I ended up breaking up my layer of rivers of Panama into 5 different layers, taking care to not separate watersheds between layers. Turns out the resulting incomplete layer from my first run already had an "outlet_cat" which mostly correctly identified separate river networks going to a given outlet, which came in handy when separating my layer into smaller ones. But be careful -- your incomplete layer may end up having duplicate lines and you need to clean this up ("Delete duplicate geometries") before trying again.

### Running the algorithm

After you are done, import your layers into GRASS. I saved them as geopackage and loaded the .gpkg file into GRASS using the v.in.ogr tool. You can also open a GRASS mapset in QGIS and load a temporary scratch layer or a file layer using v.in.ogr.qgis. But remember you need to close the mapset in QGIS before running the v.stream.order command from GRASS, it seems you can't run GRASS addons from QGIS.

You will end up going back and forth between QGIS and GRASS when doing small edits to your files as you go through your trial and error process. In some cases I could edit the GRASS layers directly from QGIS, with more complex edits I worked on the geopackage and reimported it into GRASS with v.in.ogr.

Run the v.stream.order module and fill in the data:

Name of input vector map: your line layer of rivers
Name of the input vector point map: your outlets
Name of the vector output map: the name to save your rivers with the order columns

I used a threshold of 2 and I also checked "Allow output files to overwrite existing files" and "Verbose module output". I also checked all four stream order algorithms: strahler, shreve, scheidegger and drwal. The command looks like this:

```
v.stream.order --overwrite --verbose input=rios_1@PERMANENT points=outlets_1@PERMANENT output=panama_orden_1 threshold=2 order=strahler,shreve,scheidegger,drwal
```

## Créditos

Mir Rodríguez Lombardo, Fundación Almanaque Azul

Basado en:
[MAPA*BASE_NACIONAL_25K\*\*\_DATOS_WFL1](https://services6.arcgis.com/9RMv4QT4fyJedc5h/ArcGIS/rest/services/MAPA_BASE_NACIONAL_25K*\*\*DATOS_WFL1/FeatureServer) (capas Quebradas y Rios)

Instituto Geográfico Nacional Tommy Guardia (IGNTG)
Esri Panamá

Shield: [![CC BY-NC 4.0][cc-by-nc-shield]][cc-by-nc]

Esta obra está bajo una
[Licencia Creative Commons Atribución/Reconocimiento-NoComercial 4.0 Internacional][cc-by-nc].

[![CC BY 4.0][cc-by-nc-image]][cc-by-nc]

[cc-by-nc]: https://creativecommons.org/licenses/by-nc/4.0/deed.es
[cc-by-nc-image]: https://i.creativecommons.org/l/by-nc/4.0/88x31.png
[cc-by-nc-shield]: https://img.shields.io/badge/License-CC%20BY%20NC%204.0-lightgrey.svg

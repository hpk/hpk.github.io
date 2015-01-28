---
layout:     post
title:      Building Interactive Maps with Polymaps, TileStache, and MongoDB
date:       2011-11-01
comments:   true
---

_Note: This article was written in November of 2011 and documents the process I took to build an interactive map for a now-defunct side project of mine called Goalfinch. I shut down Goalfinch recently, but plenty of people found this article very informative at the time, so I've resurrected it here._

I'd been toying around with ideas for cool ancillary features for Goalfinch for a while, and finally settled on creating an interactive map of weight loss goals. I knew what I wanted: a Google-maps-style, draggable, zoomable, slick-looking map, with the ability to combine raster images and style-able vector data. And I didn't want to use Flash. But as a complete geographic information sciences (GIS) neophyte, I had no idea where to start. Luckily there are some new technologies in this area that greatly simplified this project. I'm going to show you how they all fit together so you can create your own interactive maps for the browser.

### Overview ###

The main components of the weight loss goals map are:

1. Client-side Javascript that assembles the map from separate layers (using Polymaps)
2. Server-based application that provides the data for each layer (TileStache, MongoDB, PostGIS, Pylons)
3. Server-based Python code that runs periodically to search Twitter and update the weight loss goal data

I'll cover each component separately in upcoming posts, but I'll start with a high-level description of how the components work together for those of you who are new to web-based interactive maps.

Serving information-rich content to the browser requires programmers to think carefully about performance. For an interactive, detail-filled map of the globe, we could serve a single, very high-resolution image, but it would take a while to load. If we want our map to show up and be usable right away, we need a different strategy. That's why most online maps (such as [Google Maps](http://maps.google.com)) use a technique called [tiling](http://en.wikipedia.org/wiki/Tile_Map_Service). With tiling we load a series of smaller images (tiles) that cover the visible map area and dynamically load tiles covering other areas only as the user pans to them. Tiles can be images stitched together by the browser or vector data for a particular geographical region. This lets us display the map relatively quickly without having to wait for the non-visible images to load. Another advantage to tiling is that we can load different tiles for different zoom levels. So when the map initially appears zoomed all the way out we don't have to overload the browser with all the geographic complexities that won't even be discernable at this scale.

[Polymaps](http://polymaps.org) is a Javascript library that handles requesting image and vector tiles, stitching them together, and assembling the multiple layers. Using Polymaps, I was able to assemble a base layer of image tiles and two SVG layers for the county and state boundaries with a bit of Javascript.

So we have Polymaps assembling the map in the browser on the fly, but where is this data coming from? The short answer is: wherever we want. Here's the long answer.

For the image tiles, the conventional approach has been to collect a bunch of geographic data from somewhere like [OpenStreetMap](http://www.openstreetmap.org/), shove it into a database, and use that data to render PNG files for the various zoom levels. If you want complete control over how your images tiles look, this is the only way to go. I, however, only wanted a basic, monochrome gray map on which to overlay SVG, and found the perfect solution in the [CloudMade Maps API](http://cloudmade.com/products/web-maps-api) and their free developer account. So rather than building and hosting the map tiles myself, I was able to pull in map tiles from CloudMade's servers in my Polymaps code.

The vector data for state and county boundaries is served from my own server as [GeoJSON](http://geojson.org) using a combination of [TileStache](http://tilestache.org) -- a cache for image and vector map tiles -- and Postgresql/PostGIS. Integrating TileStache with my existing Pylons application was a breeze. Learning all about PostGIS, shapefiles, SRIDs, projections, and polygon simplification was quite a bit more pain for me, so hopefully my upcoming post on that will help other newcomers get these details right.

Finally, to get the data I was actually interested in, I wrote a Python script to repeatedly ask Twitter's search API for tweets related to weight loss in each county across the US, store the results in MongoDB, and do some simple natural language processing to determine how much weight each user wanted to lose. This made it possible to calculate the average weight loss goals of Twitter users on a per-location basis.

### Geography Data ###

Our map is assembled by rendering a layer of image tiles and overlaying one or more layers of vector data on top of it. In our case, we have a layer for state boundaries and another layer for county boundaries. When a portion of the map becomes visible (as a result of the page initially loading or the visitor scrolling or zooming to a new area) Polymaps asks the server for geography data contained by the region. The server responds with a chunk of [GeoJSON](http://geojson.org) that contains any features that should be drawn on the map.

The "raw" data for both layers can be found on the US Census Bureau's website ([county boundaries](http://www.census.gov/geo/www/cob/co2000.html), [state boundaries](http://www.census.gov/geo/www/cob/st2000.html)) in the shapefile format. Shapefiles are binary files that contain property data (name, identifier code, etc.) and geometric data (points, lines, polygons) about geographic features. The geometric data is either _unprojected_ -- that is, it represents real longitude/latitude positions of points on the globe -- or _projected_ -- where it has been passed through a transform and now represents points on a plane. Projected data must be displayed on maps that use the same projection, otherwise features won't line up. You can read more about map projections [here](http://en.wikipedia.org/wiki/Map_projection).

The shapefiles provided by the US Census Bureau are unprojected. Polymaps also expects the GeoJSON it renders to be in unprojected longitude/latitude format -- it handles transforming the data into the map's spherical mercator projection before drawing SVG polygons. So we're all set to go, right? Let's just find a utility to convert our shapefile into GeoJSON, make it availale as a static file on our server, and add it as a layer in Polymaps. Done!

Except now our visitor's browser has to download 30 megabytes of JSON before drawing anything on our map. Damn.

### TileStache ###

As I mentioned in the previous post, tiling is used to avoid having to download huge map images, and we can tile our GeoJSON as well. Enter [TileStache](http://tilestache.org), an excellent Python package that was originally built to serve image tiles and can serve GeoJSON tiles too. There may be other options for serving GeoJSON tiles out there, but I chose TileStache because its WSGI entry point made it easy to call from within my existing Pylons application. The TileStache server listens for requests for a URL like `/counties/5/7/9.json` where 5 is the zoom level, and 7 and 9 are the column and row of the tile, and returns a GeoJSON response describing any geographical features in that region:

{% highlight js %}
{
    "type": "FeatureCollection", 
    "features": [
        {
            "id": 10,
            "type": "Feature", 
            "geometry": {
                "type": "Polygon", 
                "coordinates": [
                    [
                        [-146.972095, 68.001122], 
                        [-145.994865, 68.001285],
                        ...
                        [-145.998214, 68.48987365138], 
                        [-149.231664, 64.358736]
                    ]
                ]
            }, 
            "properties": {
                "county": "068", 
                "state": "02", 
                "name": "Denali"
            }
        },
        ...
    ]
}
{% endhighlight %}

TileStache also caches these results so it doesn't have to perform the database lookup next time someone asks for the same URL.

At the time of this writing, TileStache can query either a PostGIS database or Solr for its geographic data. It uses a bounding box query to ask the database, "What do you have for me in _this_ region?" and doesn't know how to search a raw GeoJSON file or shapefile. So I had to get my shapefiles into one of these databases.

### PostGIS ###

I chose [PostGIS](http://postgis.refractions.net/) because it seems like a widely-used standard for this type of thing. PostGIS is an add-on for PostgreSQL, and getting the two installed on my Ubuntu Lucid server was as easy as 

{% highlight sh %}
sudo apt-get install postgresql
sudo apt-get install postgresql-8.4-postgis
{% endhighlight %}

For my Mac OS X development environment, I had good luck with [William Kyngesburye's packages](http://www.kyngchaos.com/software/postgres).

Now, this is where my lack of experience with GIS really started to hurt. Massaging geographic data from one format into another is a confusing process if you aren't familiar with the different types of projections, file formats, and cryptic option flags for the command line tools that come with PostgreSQL and PostGIS. Here's what finally worked for me to get my unprojected shapefiles imported into PostGIS with the proper projections and indexes.

#### Transform shapefile to spherical mercator projection ####

Yes, TileStache serves up unprojected GeoJSON, but for some reason it requires the data in PostGIS to be in the [Google Spherical Mercator projection](http://spatialreference.org/ref/sr-org/6627/). So before we load our shapefiles in, we transform them with `ogr2ogr`, a utility from [GDAL](http://www.gdal.org/):

{% highlight sh %}
ogr2ogr -s_srs EPSG:4326 -t_srs EPSG:900913 states_900913.shp st99_d00.shp
{% endhighlight %}

Here, `EPSG:4326` is the spatial reference identifier (SRID) for the source's longitude/latitude projection, `EPSG:900913` is the SRID for the spherical mercator projection we need, `st99_d00.shp` is our input shapefile and `states_900913.shp` is our output shapefile.

#### Convert shapefile to SQL file ####

Now we can use `shp2pgsql`, a utility in PostGIS, to create a SQL file that can be loaded into the database.

{% highlight sh %}
shp2pgsql -s 900913 -d -I -W LATIN1 states_900913 states > states.sql
{% endhighlight %}

We tell `shp2pgsql` the SRID of the projection we'll be using (`-s 900913`), that we want to drop the table and recreate it when this file is loaded (`-d`), tells PostGIS to create an index on our data (`-I`), that the input file's encoding is "LATIN1" (`-W LATIN1`, yours may be different), and that this data should be put into the `states` table. `states.sql` is our output file.

#### Create the database and load the files ####

{% highlight sh %}
createdb goalfinch_maps
createlang plpgsql goalfinch_maps
psql -d goalfinch_maps -f /usr/share/postgresql/8.4/contrib/postgis.sql
psql -d goalfinch_maps -f /usr/share/postgresql/8.4/contrib/spatial_ref_sys.sql
psql -f states.sql goalfinch_maps
psql -f counties.sql goalfinch_maps
{% endhighlight %}

This creates the database, installs the `plpgsql` languages on it, sets it up to use PostGIS, and loads in our state and county data. Depending on how you have your PostgreSQL users and permissions set up, you may need to use `-U postgres` or something similar for these commands.

Great! Now we have our county and state boundaries in a form that's queryable by TileStache. Let's get TileStache configured to serve up GeoJSON tiles from this database.

### Configuring TileStache ###

The [TileStache documentation](http://tilestache.org/doc) tells us how to configure our tile server to serve GeoJSON. Specifically, we can use the [PostGeoJSON Provider](http://tilestache.org/doc/TileStache.Goodies.Providers.PostGeoJSON.html) to respond to requests by searching PostGIS. Here's what my configuration looks like:

{% highlight js %}
// tilestache.cfg
{
    "cache": {
        "name": "Disk",
        "path": "/tmp/stache",
        "umask": "0000",
        "dirs": "portable"
    },
    "layers": {
        "counties": {
            "provider": {
                "class": "goalfinch.lib.geojson.SimplifyingGeoJSONProvider",
                "kwargs": {
                    "dsn": "dbname=<dbname> user=<user>",
                    "query": "SELECT gid, the_geom, state, county, name FROM counties WHERE the_geom && !bbox!",
                    "id_column": "gid", "geometry_column": "the_geom",
                    "indent": 0
                }
            }
        },
        "states": {
            "provider": {
                "class": "goalfinch.lib.geojson.SimplifyingGeoJSONProvider",
                "kwargs": {
                    "dsn": "dbname=<dbname> user=<user>",
                    "query": "SELECT gid, the_geom FROM states WHERE the_geom && !bbox!",
                    "id_column": "gid", "geometry_column": "the_geom",
                    "indent": 0
                }
            }
        }
    }
}
{% endhighlight %}

You'll notice that I have two layers set up, one for county boundaries and the other for state boundaries. Instead of the `PostGeoJSON.Provider`, however, I'm using `SimplifyingGeoJSONProvider`, which is a subclass I wrote to handle polygon simplification for different zoom levels. Why? Because when `PostGeoJSON.Provider` gets geographic features back from PostGIS, it converts them directly to GeoJSON and sends them along with every little detail intact. If we're at a low zoom level (we're viewing a large area of the map), those little details won't be visible and will just increase the size of the file that has to be transfered. We only want to show those details as we increase our zoom level. So the `SimplifyingGeoJSONProvider` checks what zoom level we're requesting and will perform more aggressive polygon simplification for lower zoom levels and will keep more details intact for higher zoom levels.

[Here's](https://github.com/migurski/TileStache/blob/master/TileStache/Goodies/Providers/PostGeoJSON.py) the source code for the original `PostGeoJSON` provider and here are the relevent bits of my `SimplifyingGeoJSONProvider`:

{% highlight python %}
# geojson.py
TOLERANCES = {
    0: 64000,
    1: 64000,
    2: 64000,
    3: 32000,
    4: 16000,
    5: 10000,
    6: 4000,
    7: 2000,
    8: 1000,
    9: 250,
    10: 125,
    11: 75,
    12: 40,
    13: 20,
    14: 10,
    15: 1
}

def zoom2tolerance(zoom):
    if zoom < 0:
        return 64000
    if zoom > 15:
        return 1
    return TOLERANCES[zoom]

def row2simpleFeature(row, id_field, geometry_field, tolerance):
    feature = {'type': 'Feature', 'properties': _copy(row)}

    geometry = feature['properties'].pop(geometry_field)
    simplified_geometry = _loadshape(_unhexlify(geometry)).simplify(tolerance)
    feature['geometry'] = simplified_geometry
    feature['id'] = feature['properties'].pop(id_field)

    return feature

class SimplifyingGeoJSONProvider(Provider):
    def renderTile(self, width, height, srs, coord):
        ...
        tolerance = zoom2tolerance(int(coord.zoom))
        ...
        for row in rows:
            feature = row2simpleFeature(row, self.id_field, self.geometry_field, tolerance)

            try:
                geom = shape2geometry(feature['geometry'], self.mercator, clip)
            except _InvisibleBike:
                # don't output this geometry because it's empty
                pass
            else:
                feature['geometry'] = geom
                response['features'].append(feature)
    
        return SaveableResponse(response, self.indent, self.precision)
{% endhighlight %}

So we look up the tolerance for the requested zoom level and then `simplify()` (part of the [Shapely](http://trac.gispython.org/lab/wiki/Shapely) package) the geometry with that tolerance before writing the coordinates out to JSON.

The final step is to call TileStache within my maps controller:

{% highlight python %}
# controllers/maps.py
...
from TileStache import WSGITileServer
tilestache = WSGITileServer('/path/to/tilestache.cfg')
...
class MapsController(BaseController):
    ...
    def tiles(self):
        return tilestache(request.environ, self.start_response)
    ...
{% endhighlight %}

And to get the URL paths working nicely, I had to add the following line in `config/routing.py`:

{% highlight python %}
map.connect('/maps/tiles/*path_info', controller='maps', action='tiles')
{% endhighlight %}

So requests to `/maps/tiles/counties/6/23/18.json` will get routed to the `maps/controller` action, but TileStache will see the request's `PATH_INFO` as if it was for just `/counties/6/23/18.json`.

_Now_ we're ready to serve GeoJSON tiles to Polymaps. I've shown how I use TileStache to serve GeoJSON tiles of US county and state boundary data for the map, and touched a bit on how Polymaps requests and assembles these tiles and displays them as polygons on top of standard image tiles. Now I'll delve more into Polymaps and the rest of the client-side code. I'll also show how I collect and parse weight loss goals from Twitter and display them on the map.

### Setting up Polymaps ###

First we need to include Polymaps and a couple of other scripts on our map page:

{% highlight html %}
<script type="text/javascript" src="/polymaps/polymaps.min.js"></script>
<script type="text/javascript" src="/maps/weightdata"></script>
<script type="text/javascript" src="/polymaps/lib/protovis/protodata.min.js"></script>
<script type="text/javascript" src="/weightmap.js"></script>
{% endhighlight %}

`polymaps.min.js` is the Polymaps library, `weightdata` is the average weight loss goal by county ID (more on this later), `protodata.min.js` is the [Protivis](http://vis.stanford.edu/protovis/) library, used to calculate quantiles for the data, and `weightmap.js` is the actual code used to create the map.

We also need a placeholder element that will contain our map once it's built:

{% highlight html %}
<div id="map" class="interactive-map"></div>
{% endhighlight %}

The Polymaps code is pretty straightforward. Here are the relevant bits from `weightmap.js`:

{% highlight js %}
// Get a reference to Polymaps
var po = org.polymaps;

// Compute the quantiles for our data
// averages is a map of county ID to average weight loss goal
var quantile = pv.Scale.quantile()
    .quantiles(9)
    .domain(pv.values(averages))
    .range(0, 8);

// Create the map, center it on the US, and add interaction (double click, scroll) callbacks
var map = po.map()
    .container(document.getElementById("map").appendChild(po.svg("svg")))
    .center({lat: 39, lon: -96})
    .zoom(4)
    .zoomRange([3,8])
    .add(po.interact());

// Add the base layer of image tiles.
map.add(po.image()
    .url(po.url("http://{S}tile.cloudmade.com"
    + "/f8fc8ca1c35a419996535a07f478225d" // Get your own darn API key: http://cloadmade.com/start
    + "/20760/256/{Z}/{X}/{Y}.png")
    .hosts(["a.", "b.", "c.", ""])));

// Add the county boundaries layer and register the onload callback (see below)
map.add(po.geoJson()
    .url("/maps/tiles/counties/{Z}/{X}/{Y}.json")
    .on('load', onload_counties)
    .id("county"));

// Add the state boundaries layer
map.add(po.geoJson()
    .url("/maps/tiles/states/{Z}/{X}/{Y}.json")
    .id("state"));

// Add the zoom buttons
map.add(po.compass()
    .pan("none"));
{% endhighlight %}

This sets up our map and adds the image and SVG layers to it. `averages` on line 8 is the Javascript object included in `weightdata` that contains a mapping of county ID to the average weight loss goal for that county. This creates a map that we can zoom and pan around in, but we aren't yet showing weight loss data for each county. This gets set up in the `onload_counties` function (line 29) that gets called each time we load new county boundaries from GeoJSON. Here's `onload_counties` and a couple other functions we use to show the tooltip when the mouse hovers over a county:

{% highlight js %}
function detailFollow(e) {
    $('#tooltip').css({
        top: (e.pageY) + "px",
        left: (e.pageX + 15) + "px"
    });
}

function onload_counties(e) {
    var count = e.features.length;

    if (!count) { return; }

    for (var i=0; i<count; i++) {
        var f = e.features[i];
        var cid = f.data.properties.state + f.data.properties.county;

        f.element.setAttribute('onmouseover', 'countyDetail("' + cid + '","' + f.data.properties.name + '");');
        f.element.setAttribute('onmouseout', 'hideCountyDetail();');
        f.element.onmousemove = detailFollow;

        num = averages[cid];
        if (!num) {
            f.element.setAttribute('class', 'no-quantile');
        } else {
            f.element.setAttribute('class', 'q' + quantile(num));
        }
    }
}

function countyDetail(cid, name) {
    var avg = averages[cid];
    if (!avg) {
        $('#tooltip').html('<strong>'+name+' County</strong><br/>No data available');
    } else {
        $('#tooltip').html('<strong>'+name+' County</strong><br/>Average weight loss goal: '+averages[cid]+' lbs');
    }
    $('#tooltip').show();
}

function hideCountyDetail() {
    $('#tooltip').hide();
}
{% endhighlight %}

Pretty self-explanatory. For each county polygon that we load, we assign a few event listeners that handle mouse events over that shape and style it according to the average weight loss goal in the region. If we have data for it, we give it a CSS class of `q0`-`q8`, with `q0` for the lowest weight loss goals and `q8` for the highest. Counties with no data get a `no-quantile` class. When the mouse enters a county (see `countyDetail`), we look up the average weight loss goal for the area and display it in a tooltip-like dialog. I'm using a bit of JQuery here because I already have it included on the rest of my site, but this could be rewritten without it.

### Twitter Weight Loss Goals ###

The actual data that we care about and want to show on this map is in the `/maps/weightdata` script. It looks like this:

{% highlight js %}
var averages = {
    "48441": 28, 
    "48117": 21, 
    "16079": 22, 
    "48113": 23, 
    "45029": 6, 
    "48111": 8,
    ...
}
{% endhighlight %}

`averages` is a Javascript object whose keys are the IDs of each county for which we have data (here it's the 2-digit FIPS state ID + 3 digit FIPS county ID) and whose values are the average weight loss goals for each county. I pulled this data from Twitter by using the [Search API](http://dev.twitter.com/doc/get/search) to find tweets mentioning phrases related to weight loss ("lose pounds", "drop lbs", and so on) within each county. Each tweet gets parsed to determine what the author's weight loss goal is, and the results are averaged together to form a single number for a county. I broke this process into two separate steps: The tweet search is a Python script that runs daily and adds new results to MongoDB. The tweet parsing and averaging happens in my Pylons application when the map actually makes a request for the data. Let's get straight into the nitty gritty.

A first cut at the search process would (if you were using Python, at least) look something like this:

{% highlight python %}
search_terms = ['lose pounds', 'drop pounds', ...]
locations = [ ... list of lat/lon pairs to search near ... ]
for term in search_terms:
    for location in locations:
        scrape_tweets(location, term)
{% endhighlight %}

Where `scrape_tweets` constructs a query to the Search API and make the request, parses the JSON results, and stores them in Mongo. At the highest level that's exactly what my script does, but there are some refinements that I needed to make to get things running efficiently and to keep Twitter from hating me.

#### Location Clustering ####

First, the locations. Since I wanted data for each individual county, the temptation was to make `locations` a list of the geographic center of every single county in the United States. According to the county details I pulled from the US Geological Survey's site, this would have resulted in 3219 requests for each search term I was interested in. With the Twitter Search API's rate limit, this would have taken forever. I needed to cut down the number of locations I was searching.

Fortunately, this was relatively easy to do. It's obvious that there are lots of Twitter users in metropolitan centers like New York and San Francisco and that Twitter users in rural Montana are relatively sparse. Consequently, most of the searches near locations in sparsely populated areas would return only a few results at most. Where population density was low, I could chunk counties together into larger regions and search across all of them. Where population density was high, I could fall back to searching individual counties. At the expense of reduced resolution in sparsely populated areas, this would cut the number of requests I had to make.

So that's what I did.

{% highlight python %}
county_coords = {} # Dictionary of county ID -> geographic center lat/lon
county_population = {} # Dictionary of county ID -> population
county_area = {} # Dictionary of county ID -> area in square miles
radius_feather = 1.25 # Expand our search radius by this factor

class QueryLocation(object):
    def __init__(self, countyid):
        self.county_ids = [countyid]

    def population(self):
        pop = 0
        for id in self.county_ids:
            pop += county_population[id]
        return pop

    def coordinates(self):
        if len(self.county_ids) == 1:
            return county_coords[self.county_ids[0]]
        else:
            num_pts = len(self.county_ids)
            lat_tot = 0.0
            lon_tot = 0.0
            for id in self.county_ids:
                c = county_coords[id]
                lat_tot += c[0]
                lon_tot += c[1]
            return [lat_tot / float(num_pts), lon_tot / float(num_pts)]

    def radius(self):
        total_area = 0
        for id in self.county_ids:
            total_area += county_area[id]
        r2 = float(total_area) / 3.14159
        r = math.sqrt(r2)
        return "%dmi" % int(r * radius_feather)

    def find_nearest(self, locations):
        min_dist = 100000000
        nearest = None
        for other in locations:
            if other == self : continue
            this_dist = distance(self.coordinates(), other.coordinates())
            if this_dist < min_dist:
                nearest = other
                min_dist = this_dist
        return nearest

    def append_nearest(self, locations):
        nearest = self.find_nearest(locations)
        for id in nearest.county_ids:
            self.county_ids += [id]
        locations.remove(nearest)

def load_county_data():
    ...
    # Load county_coords, county_population, and county_area
    # from files
    ...

    pop_cutoff = 50000
    locations = [QueryLocation(it) for it in county_coords.keys()]
    new_locations = [l for l in locations]
    did_combine = True

    print 'Starting with %d locations' % len(new_locations)
    while did_combine:
        did_combine = False
        for loc in locations:
            if not (loc in new_locations) : continue
            pop = loc.population()
            if pop < pop_cutoff:
                loc.append_nearest(new_locations)
                did_combine = True
        print 'Condensed to %d locations' % len(new_locations)
        locations = [l for l in new_locations]

    return new_locations
{% endhighlight %}

`QueryLocation` is a class that encapsulates a location and radius to search near. It could consist of a single county or a group of counties. In `load_county_data` we start out with a list of one `QueryLocation` for every county in the US. Then we look at each one, and if its total population is below our threshold we find the nearest neighboring `QueryLocation` and add it to this one. We repeat this process, combining neighboring regions, until all locations have a population above our threshold. Now we have a list (`new_locations`) of regions to search. For a location that consists of a single county, we use its geographic center as our search location and approximate our search radius as if the county were a perfect circle. For a location consisting of a group of counties, we calculate the average geographic center by looking at the coordinates for each county in the group. Our search radius is approximated by finding the total area of the group and assuming we're dealing with a circle with that area.

Obviously, assuming each region is a circle is not perfect. When we search for tweets near the center of the region and within a certain radius, we may miss tweets that are in some corners not covered by the bounding circle, and we may find tweets that belong to other `QueryLocation`s. To solve the first problem, I added `radius_feather`, which increases the search radii by a little bit in order to reach corners and other geographic features that extend far away from a region's center. I ignored the second problem by simply noting every `QueryLocation` that found a tweet; tweets found inside two regions count toward the average weight loss goal in both.

With this technique I was able to cut the number of requests I needed to make by 70%, while still retaining good resolution in more densely populated areas.

#### Intelligent Search Timing ####

Since this script is running on a daily basis, most of the results for each query will be the same, with only a few new tweets that happened since we last looked. So we don't need to perform the same set of searches every single day. To further limit the number of requests this script makes every day, it saves the last time it searched for a particular term near a particular location. The next time the script is run, it checks to see if the current request was made within the last _n_ days. If it was, the script skips the request. This is how it looks:

{% highlight python %}
class SearchTime(minimongo.model.Model):
    # locid
    # search_term
    # time
    mongo = minimongo.model.MongoCollection(database=dbname, collection='searchtimes')

def scrape_tweets(location, search_term, since_id):
    now = datetime.utcnow()
    last_search = SearchTime.collection.find_one({'locid': location.locid(), 'search_term': search_term})
    if last_search and (now - last_search.time).days < refresh_days:
        return

    ...
    # Perform the request and save the results...
    ...

    if not last_search:
        last_search = SearchTime({'locid': location.locid(), 'search_term': search_term})
    last_search.time = now
    last_search.save()
{% endhighlight %}

`SearchTime` is a [minimongo](https://github.com/slacy/minimongo) collection that saves the last time we searched for a term near a location. `refresh_days`, in my case, is 3, but this can be adjusted lower for searches with a higher rate of new tweets.

You'll also notice another parameter to `scrape_tweets`: `since_id`. This is the ID of the last tweet we found for this search term and location. We pass this along in the API request so that we only get new tweets that we haven't seen yet.

#### Calculating Average Weight Loss Goals ####

Now we have a collection of tweets sitting in MongoDB, categorized by search term and the county (or counties) that the tweet came from. Now we need to determine how much weight the author of each tweet wants to lose. When a visitor loads the map page and the browser requests `/maps/weightdata`, the Pylons controller calls this:

{% highlight python %}
@cache_region('map_data')
def weight_by_county_json():
    tweets = Tweet.collection.find()

    counts = {}
    sums = {}

    for tweet in tweets:
        if not hasattr(tweet, 'county_ids'):
            continue

        pounds = find_pounds(tweet)

        if pounds == None:
            continue
        for id in tweet.county_ids:
            if counts.has_key(id):
                counts[id] += 1
            else:
                counts[id] = 1

            if sums.has_key(id):
                sums[id] += pounds
            else:
                sums[id] = pounds

    averages = {}
    for id in sums.keys():
        averages[id] = int(round(float(sums[id]) / float(counts[id])))

    out = 'var averages = ' + json.dumps(averages) + ';'
    return out
{% endhighlight %}

We look at each tweet in the collection, determine how many pounds the author wants to lose (`find_pounds(tweet)`), and keep a running sum for every county. Then we find the average for each county and return the result as JSON. Finally, the result of the calculation gets cached using [Beaker](http://beaker.groovie.org/index.html)'s `cache_region` decorator. Subsequent requests will use the cached value rather than calculating the averages every time. Of course, when the Twitter search script runs and adds new tweets to the collection it invalidates the cache so the averages get refreshed.

The `find_pounds()` function is a bit of special sauce, and I won't get into too many details there. I initially went down the road of using [NLTK](http://www.nltk.org) to parse, tag, and stem the text content of each tweet and use that to find subjects and direct objects that quantified weight. Unfortunately I couldn't get this natural language processing-based approach to recognize even half of the weight loss goals. Twitter users do all sorts of crazy stuff that confused NLTK: They shorten words and phrases to stay within 140 characters, sprinkle hashtags and URLs everywhere, and don't bother to spell check. So I settled on a more brute force approach. Roughly:

* Remove URLs, extraneous punctuation, and other non-English tokens
* Find every token in the tweet that could be interpreted as a number, such as "5", "five", "a few", and so on.
* Mark locations of each part of the search term (i.e. "lose", "pounds") within the text
* Look for patterns such as "lose _x_ pounds", "I have _x_ more pounds to lose", and lots more

If a tweet can't be definitively parsed, it's skipped. This pattern-based approach does much better: it's able to figure out the number of pounds an author is referring to about 90% of the time.

### Conclusion ###

So that's that. I covered pretty much every layer and component used to create an interactive map entirely without flash. I had a lot of fun with this project, so I'll be building more of this type of interactive map in the future. Hopefully the details in this series of posts will help future developers and hackers get up to speed on similar projects as well. If you have any questions about this project or your own efforts, don't hesitate to get in touch. Thanks again for reading!


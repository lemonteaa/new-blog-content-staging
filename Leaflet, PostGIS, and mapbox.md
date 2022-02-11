# Leaflet, PostGIS, and mapbox

In this tutorial, we will code a simple Map/GIS Application with both frontend and backend.

The demo repos are given for reference:

- Backend: [lemonteaa/gisdemo-backend](https://github.com/lemonteaa/gisdemo-backend)
- Frontend: [lemonteaa/gisdemo-frontend](https://github.com/lemonteaa/gisdemo-frontend)

# PostGIS Extension

## Installation

First start up a PostgreSQL DB using docker. Now run the following to install the extension that will enable GIS capabilities:

```sql
CREATE DATABASE gisdb;
\connect gisdb;
CREATE EXTENSION postgis;
```

Note that there are many more related extensions for even more functionalities, some of which we may play with in future tutorials.

## Usage

The PostGIS Extension provide us with two extra datatype: `GEOMETRY` and `GEOGRAPHY`. As an example, consider the following:

```sql
CREATE TABLE regions (
    region_id SERIAL PRIMARY KEY,
    region_name VARCHAR(50),
    region_polygon GEOMETRY(POLYGON)
);
```

Note that we can specify a subtype using bracket. In the example above it is `POLYGON`. A more basic example would be `POINT`.

These datatype have been specially designed with efficient computational geometric algorithms in mind. Unfortunately a side effect is that the native representation of these data are not human-readable. Instead PostGIS give us a whole bunch of functions to work with these data, including conversion/read/write/processing, etc.

Now let's insert a record:

```sql
INSERT INTO regions (region_name, region_polygon) 
VALUES 
('Test reg', 
 ST_POLYGON(ST_GeomFromText('LINESTRING(75 29,77 29,77 29, 75 29)'), 4326));
```

Here [ST_GeomFromText]() takes a textual representation of the data and convert it into binary format. [ST_POLYGON]() is the construction for the `POLYGON` subtype. Notice the number `4326` as the second argument here. The reason is because TODO.

The textual format is referred to as WKT (Well Known Text), while the binary is referred to WKB (Well Known Binary) in PostGIS.

For some feature we will implement in the future, we also need to know the functions for the `POINT`, `RECTANGLE` subtype. The answer is [ST_POINT]() for constructing a point, [ST_SetSRID]() for the correction in general; [ST_X]() and [ST_Y]() for getting the coordinates of a `POINT`. [ST_MakeEnvelope]() creates a rectangular box out of the coordinates of the bottom left and upper right corner. Finally, [ST_Intersects]() perform the intersection operation.

## GeoJSON Format

GeoJSON is a standardized format for encoding geographic information in JSON. The format have provision for including metadata to geographic features.

PostGIS provide function for outputing data in GeoJSON too: [ST_AsGeoJSON](). Try it out with some SQLs on simple table yourself to see the effect.

## JSON Processing in SQL

Base PostgreSQL already provided functions to work with JSON directly inside SQL. This open up the opportunity to push everything onto SQL so that the backend doesn't need any special processing.

Our key here is two functions:

* `json_build_object(key1, value1, key2, value2, ...)` builds a JSON object with the specified keys and values pair. (`jsonb_build_object` are similar except that `json` is replaced with JSONb.)
* `json_agg(entities)` group individual objects together into a collection, similar to `GROUP BY` in SQL.

# A simple backend

# Serving Tiles using Tileserver

Aside from the geographic informations, we also need the images for the map itself. This consists of a collection of tiles (at different zoom/scale), each covering a part of the global map.

Previously, there are free cloud services that provide these, such as the Google Map API. Unfortunately, Google started charging for it in 2018.

Because of this, we're going to try self-hosting using [Tileserver](https://github.com/maptiler/tileserver-gl). However, do note that this project has a dependencies on `@mapbox/mapbox-gl-native` at version `5.0.2`. Mapbox have been a long time, major competitor to Google Map, who also offer a SaaS cloud service. It used to be the case that their core libraries are open source, however they've since [changed into a paid license](https://news.ycombinator.com/item?id=25347310) for both the Mapbox GL JS and the native library in 2020. The community in the mean time have responded with [open source fork](https://www.maptiler.com/news/2021/01/mapbox-gl-open-source-fork/) of the last open source version of the code base. (See also this [issue](https://github.com/maptiler/tileserver-gl/issues/542))

We also need to have the map tile data. For this we will use [MapTiler](https://data.maptiler.com/downloads/asia/china/hong-kong/) (due to convinence) and select Hong Kong region. Choose the Open Street Map Vector Tile, then choose either "evaluation and education purpose" or "non-commercial personal project". Choose the free version (which contains older data from a few years ago), login (create an account if you don't have one), then download.

We will be using the docker images provided since we're deploying to a kubernetes cluster. We will need to provide command line argument to point it to the data file, for this see the [doc](https://tileserver.readthedocs.io/en/latest/usage.html#getting-started).



# Some kubernetes setup

## Secrets

## Jobs

# Frontend with react-leaflet

Now let's create a UI for our map app. We'll use React as the framework, Chakra for the UI, and react-leaflet as a react wrapper of leaflet (since leaflet itself is a traditional, imperative JS library).

Our main app look like this at the moment:

```js
export default function App() {
  const center = {
    lat: 22.3327,
    lng: 114.1709
  };
  const [position, setPosition] = useState(center);
  return (
    <ChakraProvider theme={theme}>
      <DrawerExample className="test-btn" position={position} setPosition={setPosition} />
      <Box>
        <MapContainer center={[22.3327, 114.1709]} zoom={12} scrollWheelZoom={true}>
          <TileLayer
            attribution='&copy; <a href="https://stadiamaps.com/">Stadia Maps</a>, &copy; <a href="https://openmaptiles.org/">OpenMapTiles</a> &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors'
            url="https://maptiler-gislab-lemonteaa.cloud.okteto.net/styles/basic-preview/{z}/{x}/{y}.png"
          />
          <DraggableMarker position={position} setPosition={setPosition} />
          <LoadShops />
        </MapContainer>
      </Box>
    </ChakraProvider>
  );
}
```

Note a few things:

- `<ChakraProvider>` is used to inject the theme - and all Chakra UI elements/components should live under one to be styled.
- `position` is a shared/global state for a draggable marker that we will use when creating new Shop/Location.
- For the draggable marker we copy the example from the [official doc](https://react-leaflet.js.org/docs/example-draggable-marker/).
- The basic usage of react-leaflet is to have a `<MapContainer>` component, then insert various layers/object as child. For example, the `<TileLayer>` is used to load tiles from URL.
- Notice the coordinate here is `/{z}/{x}/{y}.png` to make it fit the convention of Tile-server.

## Rendering GeoJSON data

In the simplest case, we can simply use the `<GeoJSON>` component as a child of `<MapContainer>`: `<GeoJSON data={geoData} />`, where `geoData` holds the data. We should re-render each time the user move or zoom the map by sending a request to our backend for the GeoJSON of all entities/shops inside a bound we specify. This can be done like this:

```js
const map = useMapEvent("moveend", () => {
    const sw = map.getBounds().getSouthWest();
    const ne = map.getBounds().getNorthEast();

    const box = {
      sw: [sw.lat, sw.lng],
      ne: [ne.lat, ne.lng]
    };
    //...
});
```

There is a complication though. We want the Markers to actually be clickable - on click, a popup should show displaying some basic information about the shop.



# Reference

https://learnsql.com/blog/getting-started-with-postgis-your-first-steps-with-the-geography-data-type/

https://www.bostongis.com/PrinterFriendly.aspx?content_name=postgis_tut01

https://blog.patricktriest.com/game-of-thrones-map-node-postgres-redis/

https://stevebennett.me/tag/openmaptiles/

https://react-leaflet.js.org/docs/api-map/

https://www.qed42.com/insights/coe/javascript/building-interactive-maps-leaflet-and-react


https://postgis.net/docs/ST_AsGeoJSON.html



# TerraStories

**TerraStories** is an application designed to help communities map and access their own place-based storytelling. The project to develop this application was initiated by the [**Amazon Conservation Team**](http://amazonteam.org) (ACT), an organization who partners with indigenous and other traditional communities in the Amazon rainforest to help them protect their ancestral lands and traditional culture. The application is developed to be entirely open source and offline-compatible, so that it can be used by communities in the most remote locations of the world. It is a Dockerized Rails App that uses [**Mapbox**](https://mapbox.com) to help users locate content geographically on an interactive map. A team is attempting to finish this app at **Ruby for Good 2018**: http://rubyforgood.org/2018

*Note: TerraStories is a work in progress.*

## Prerequisites
Install docker. On linux, you may have to install docker-compose separately.
 - https://docs.docker.com/install/
 - https://docs.docker.com/compose/install/

## Setup
Docker will automatically build images as needed when running `docker-compose up`,
but to confirm everything builds correctly, run the following and check that the
output ends with something like this.
```
$ docker-compose build
  ...
> mariadb uses an image; skipping
  ...
> Successfully built 0123456789
> Successfully tagged terrastories/tilebuilder:latest
> tileserver uses an image; skipping
> rails uses an image; skipping
```

## Build The Map Tiles
The tilebuilder service will need to be run once to populate the `mbtiles`
shared volume that the tileserver will read from. The tilebuilder does not need
to stay running along with the other services. Building map tiles may take quite
a long time, but it should show progress similar to the following and eventually
get to 100%, exiting with code 0.
```
$ docker-compose run tilebuilder
...
> wwww features, xxxx bytes of geometry, yyyy bytes of separate metadata,
zzzz bytes of string pool
> 99.9% 11/2222/3333
```

Any time the shapefiles change and require regenerating the mbtiles file,
this service will need to be run again and the tileserver restarted once the
tilebuilder finishes (just run `docker-compose restart tileserver`).

## Make It Go
In `docker-compose.yml`, the tileserver and rails database are listed as
dependencies for the rails service. So to start the whole thing up (omitting
tilebuilder, which only needs to run once) just run the following. Omit the
`-d` flag if you prefer to see all of the rails server output.
```
$ docker-compose up -d nginx
```

To spin all the services back down run the following.
```
$ docker-compose down
```

## Tidying Up
Docker likes to accumulate cached containers and images, some of which
are bound to be superfluous and quite large. Here's how to clean those up.
```
$ docker container prune -f
$ docker rmi $(docker images | grep "^<none>" | awk "{print $3}")
```

The last one may look a little risky, but basically relies on the idea that any
docker image without a useful tag is unlikely to be needed. Untagged images
show up as `<none>` when in the list provided by `docker images`. If you don't
trust that batch removal command, you can manually review the listed images and
delete by referencing the image id as something like `docker image rm abc123f`.

With any of these steps, there's really no risk of removing anything that can't
be restored with a `docker build` or `docker pull` later.

## Updating the Tileserver Map
### Step 1: preparing content in Mapbox Studio

Terrastories is designed to render a basemap as designed and styled in Mapbox Studio. There are two different components: shapefiles (the spatial data without any styling properties) and styles (the look and feel of the map, as designed in Mapbox Studio, exported in json format). The basic workflow is as follows: 
1) upload their shapefile content to [Mapbox Studio](https://www.mapbox.com/mapbox-studio/), and use the Studio interface to lay out the map. You have to have a Mapbox account to use Mapbox Studio (creating and designing maps using Mapbox Studio is free up to certain file size limitations.) To learn how to use Mapbox Studio, you can refer to the manuals and tutorials made available by Mapbox [here](https://www.mapbox.com/help/studio-manual-tutorials/) or other resources on the web.

2) download the style.json from Mapbox Studio via the Mapbox Studio styles interface [here](https://www.mapbox.com/studio/styles/)

3) copy both the shapefiles and style.json into the respective directories on Terrastories. 
Note: the user must provide their own shapefile content. It is not possible to use any of the standard OpenStreetMap (OSM) content used in the standard styles made available by Mapbox, unless the user first downloads that OSM content and converts it to shapefile first.

### Step 2: adding new or updating shapefiles to Terrastories

To add new shapefiles or update existing shapefiles, there are two steps:

1) Include the new files in your `shapefile` directory (\tilebuilder\shapefiles])

2) You will need to re-run the tilebuilder, following the instructions here: https://github.com/rubyforgood/terrastories/blob/master/tilebuilder/README.md

### Step 3: adding or updating style to the Map

To add or update the map style, 

1) download the style.json from Mapbox Studio via the Mapbox Studio styles interface [here](https://www.mapbox.com/studio/styles/)

2) at this point, we have to edit the style.json a little. When you upload shapefiles to Mapbox Studio, it actually adds on an additional six alphanumeric characters preceded by a dash (-), which is called "hash." For example, a shapefile called "South_America" might be called "South_America-a2027z" in Mapbox Studio. And then in style.json file, all of the names for this layer will have "–a2027z" added to it. This is a problem because there is a discrepancy between the names of the shapefile you added in Step 2, which does no include "-a2027z." So, you have to go into the json and look for "source-layer": "South_America-a2027z", and take out the "-a2027z", and do the same for each layer. 
In the future, we will create an automatic script that will take care of this process.

3) copy the style into your `styles` directory (tileserver\data\styles])

4) make sure that config.json in \tileserver\data\ is pointing to the right style file.

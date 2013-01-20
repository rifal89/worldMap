# WorldMap

## Summary

WorldMap was a small Project I have done for „Clientside Web Engineering“ at university of applied sciences Salzburg. It utilizes the power of [D3.js](http://d3js.org/) to generate SVG paths representing the shapes of countries.

These paths are later transformed to [THREE.js](http://mrdoob.github.com/three.js/) Meshes and rendered in 3d space. You can zoom in to continents and hover over countries to see their name on the bottom. 

### Demo
For a live preview, see <http://oi.qoop.io/fhs/clientside>

## How it was done

When I began working on this project I realized, that it is impossible to render map/geo data directly in THREE.js. D3 is much better suited for such things and accepts geoJSON as input format. Fortunately, there is a project named [d3-threeD](https://github.com/asutherland/d3-threeD) which aims to integrate D3.js in THREE.js. With this library it is possible to render SVG elements from D3.js directly in a THREE.js scene. Because I wanted to control how the country shapes were rendered I only used the path transformation functionality and did the rendering on my own. After I was able to render the Map correctly I wanted to make the countries hoverable, which was done via raycasting. Finally I added camera animations to be able to "zoom" to a continent. I used [Bootstrap](http://twitter.github.com/bootstrap/) to make everything look pretty.

* get map/geo data of countries
* transform this data to geoJSON
* generate SVG paths with D3.js
* transform these paths to THREE.js meshes
* render these meshes in a 3D scene
* make country meshes hover-able
* add zooming
* make everything look pretty

### Where can I get shape files of all countries?

After some research I discovered the [Natural Earth](http://www.naturalearthdata.com/) Database. It offers free vector and raster map data at 1:10m, 1:50m, and 1:110m scales. Data themes are available in three levels of detail. For this project I used the 1:110m Cultural Country Vectors which you can download [here](http://www.naturalearthdata.com/downloads/110m-cultural-vectors/).

### What is geoJSON?

GeoJSON is an open format for encoding a variety of geographic data structures. Spatial data format types supported in GeoJSON include points, polygons, multipolygons, features, geometry collections, and bounding boxes, which are stored along with feature information and attributes.

### How to convert shapefiles (.shp) to geoJSON?

First I tried [QuantumGIS](http://www.qgis.org/) but this is a rather big software package and overkill for our purpose. I ended up in using two online tools which worked out pretty well.

First I used [MapShaper](http://mapshaper.com/test/MapShaper.swf) to reduce the amount of details in the shapefile because the 1:110m Cultural Country Vectors from Natural Earth converted to geoJSON were unnecessarily big.

Then I used an online [Vector Converter](http://converter.mygeodata.eu/vector) to convert everything to geoJSON.


### Initialising D3.js

```
init_d3: function() {

    geoConfig = function() {
        
        this.mercator = d3.geo.equirectangular();
        this.path = d3.geo.path().projection(this.mercator);
        
        var translate = this.mercator.translate();
        translate[0] = 500;
        translate[1] = 0;
        
        this.mercator.translate(translate);
        this.mercator.scale(200);
    }

    this.geo = new geoConfig();
}
```
In `init_d3` I store the D3.js configuration in the `geo` object for later use. The configuration includes the `d3.geo.equirectangular()` mercator and some translation and scaling to move everything on the right place. You can read more about D3 geo-projections (mercators) [here](https://github.com/mbostock/d3/wiki/Geo-Projections).

### Initialising THREE.js

```
init_tree: function() {

    if( Detector.webgl ){
        this.renderer = new THREE.WebGLRenderer({
            antialias : true
        });
        this.renderer.setClearColorHex( 0xBBBBBB, 1 );
    } else {
        this.renderer = new THREE.CanvasRenderer();
    }

    this.renderer.setSize( this.WIDTH, this.HEIGHT );

    this.projector = new THREE.Projector();
    
    this.scene = new THREE.Scene();

    this.camera = new THREE.PerspectiveCamera(this.VIEW_ANGLE, this.WIDTH / this.HEIGHT,
                                              this.NEAR, this.FAR);
    this.camera.position.x = this.CAMERA_X;
    this.camera.position.y = this.CAMERA_Y;
    this.camera.position.z = this.CAMERA_Z;
    this.camera.lookAt( { x: this.CAMERA_LX, y: 0, z: this.CAMERA_LZ} );
    this.scene.add(this.camera);

    $("#worldmap").append(this.renderer.domElement);
}
```
In `init_tree` THREE.js is initilised. First of all I use [Detector.js](https://github.com/mrdoob/three.js/blob/master/examples/js/Detector.js) to determine if the current browser supports WebGL. If so I can use `THREE.WebGLRenderer`. If not the fallback `THREE.CanvasRenderer` is used. After that I create a new `THREE.Scene` and a `THREE.PerspectiveCamera`. The `THREE.Projector` is needed for the raycasting later on. At the end the renderer is appended to the DOM.

### Adding countries to the scene
```
add_countries: function(data) {

    var countries = [];
    var i, j;

    // convert to threejs meshes
    for (i = 0 ; i < data.features.length ; i++) {
        var geoFeature = data.features[i];
        var properties = geoFeature.properties;
        var feature = this.geo.path(geoFeature);

        // we only need to convert it to a three.js path
        var mesh = transformSVGPathExposed(feature);

        // add to array
        for (j = 0 ; j < mesh.length ; j++) {
              countries.push({"data": properties, "mesh": mesh[j]});
        }
    }
    ...
```
In `add_countries` all the features out of the geoJSON are converted to THREE.js meshes. First we create a `D3.Geo.Path` out of every geoJSON feature. This features are afterwards converted to a `THREE.Mesh` via `transformSVGPathExposed`. These meshes, together with some data about the current country are pushed to a countries array.
```
    ...
    // extrude paths and add color
    for (i = 0 ; i < countries.length ; i++) {

        // create material color based on average		
        var material = new THREE.MeshPhongMaterial({
            color: this.getCountryColor(countries[i].data), 
            opacity:0.5
        }); 

        // extrude mesh
        var shape3d = countries[i].mesh.extrude({
            amount: 1, 
            bevelEnabled: false
        });

        // create a mesh based on material and extruded shape
        var toAdd = new THREE.Mesh(shape3d, material);
 
        ...

        this.scene.add(toAdd);
    }
}
```
Now we create a new `THREE.MeshPhongMaterial` with a calculated colour and opacity of 0.5 for every country. Afterwards we extrude the mesh by one so that it is no longer flat and create the final `THREE.Mesh` which we now can add to our scene.

### Get a hex colour out of country codes
```
getCountryColor: function(data) {
    var multiplier = 0;

    for(i = 0; i < 3; i++) {
        multiplier += data.iso_a3.charCodeAt(i);
    }

    multiplier = (1.0/366)*multiplier;
    return multiplier*0xffffff;
}
```
In `getCountryColor` I add up the results of `charCodeAt` for each char in an iso_a3 country code. Then I normalize the result and multiply it to `0xFFFFFF` which results in (mostly) different colours for each country.

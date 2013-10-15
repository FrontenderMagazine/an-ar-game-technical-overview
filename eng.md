<article class="post">
[An AR Game][1] is the winning entry for the [May 2013][2] [Dev Derby][3]. It
is an[augmented reality][4] game, the objective being to transport rolling play
pieces from a 2D physics world into a 3D space. The game is
[playable on GitHub][5], and [demonstrated on YouTube][6]. The objective of
this article is to describe the underlying approaches to the game’s design and 
engineering.

Technically the game is a simple coupling of four sophisticated open source
technologies:[WebRTC][7], [JSARToolkit][8], [ThreeJS][9], and [Box2D.js][10].
This article describes each one, and explains how we weaved them together. We 
will work in a stepwise fashion, constructing the game from the ground up. The 
code discussed in this article is available on[github][11], with a [tag][12]
and[live][13] link for each tutorial step. Specific bits of summarized source
will be referenced in this document, with the full source available through the
‘diff’ links. Videos demonstrating application behaviour are provided where 
appropriate.

`git clone https://github.com/abrie/devderby-may-2013-technical.git<br>`

This article will first discuss the AR panel ([realspace][14]), then the 2D
panel
([flatspace][15]), and conclude with a description of their [coupling][16].

## Panel of Realspace {#realspace}

Realspace is what the camera sees — overlaid with augmented units.

### Begin with a Skeleton

`git checkout example_0`  
[live][17], [diff][18], [tag][19] 

We will organize our code into modules using [RequireJS][20]. The starting
point is a main module with two skeletal methods common to games. They are
`initialize()` to invoke startup, and `tick()` for rendering every frame.
Notice that the gameloop is driven by repeated calls to
[requestAnimationFrame][21]:

| requirejs([], function() {
     
        // Initializes components and starts the game loop
        function
initialize
() {
        }
     
        // Runs one iteration of the game loop
        function tick() {
            // Request another iteration of the
gameloop
            window.requestAnimationFrame(tick);
        }
     
        // Start the application
        initialize();
        tick();
    }); |
||

The code so far gives us an application with an empty loop. We will build up
from this foundation.

### Give the Skeleton an Eye

`git checkout example_1`  
[live][22], [diff][23], [tag][24] 

AR games require a realtime video feed: [HTML5][25]‘s [WebRTC][26] provides
this through access to the[camera][27], thus AR games are possible in modern
browsers like[Firefox][28]. Good documentation concerning WebRTC and 
[getUserMedia][29] may be found on [developer.mozilla.org][30], so we won’t
include the basics here.

A camera library is provided in the form of a [RequireJS][31] module named 
[webcam.js][32], which we’ll incorporate into our example.

First the camera must be initialized and authorized. The webcam.js module 
[invokes a callback on user consent][33], then for each tick of the gameloop 
[a frame is copied from the `video` element to a `canvas` context][34]. This is
important because it makes the image data accessible. We’ll use it in subsequent
sections, but for now our application is simply a`canvas` updated with a video
frame at each tick.

### Something Akin to a Visual Cortex {#\"jsartoolkit\"}

`git checkout example_2`  
[live][35], [diff][36], [tag][37] 

JSARToolkit is an augmented reality engine. It identifies and describes the
orientation of[fiducial markers][38] in an image. Each marker is uniquely
associated with a number. The markers recognized by JSARToolkit are available
[here][39] as PNG images named according to their ID number (although as of
this writing the lack of PNG extensions confuses Github.) For this game we will 
use #16 and #32, consolidated onto a single page:

![][40]

JSARToolkit found its beginnings as [ARToolkit][41], which was written in 
[C++][42] at the [Univeristy of Washington][43]‘s [HITLab][44] in [Seattle][45]
[Java][46], then from Java to [Flash][47], and finally from Flash to [JS][8].
This ancestry causes some idiosyncrasies and inconsistent naming, as we’ll see.

Let’s take a look at the distilled functionality:

| // The raster object is the canvas to which we are copying video frames.

var JSARRaster = NyARRgbRaster_Canvas2D(canvas
);



var JSARParameters = new FLARParam(canvas.width, canvas.height
);



var JSARDetector = new FLARMultiIdMarkerDetector(FLARParameters, 120
);

JSARDetector.setContinueMode(true
);



var threshold = 64;

var count = JSARDetector.detectMarkerLite(JSARRaster, threshold
); |
||

Once a frame has been processed by `JSARDetector.detectMarkerLite()`, the
JSARDetector object contains an index of detected markers.
`JSARDetector.getIdMarkerData(index)` returns the ID number, and 
`JSARDetector.getTransformMatrix(index)` returns the spatial orientation. Using
these methods is somewhat complicated, but we’ll[wrap them in][48] 
[usable helper methods][49] and call them from a loop like this:

| var markerCount = JSARDetector.detectMarkerLite(JSARRaster, 90); 
     
    for( var index = ; index &lt; markerCount; index++ ) {
        // Get
the ID number of the detected marker.
        var id =
getMarkerNumber(index
);
     
        // Get the transformation matrix of the detected marker.
        var
matrix = getTransformMatrix(index
);
    } |
||

Since the detector operates on a per-frame basis it is our responsibility to
maintain marker state between frames. For example, any of the following may 
occur between two successive frames:

*   a marker is first detected
*   an existing marker’s position changes
*   an existing marker disappears from the stream.

The state tracking [is implemented using ardetector.js][50]. To use it we
instantiate a copy with the`canvas` receiving video frames:

| // create an AR Marker detector using the canvas as the data source
    var
detector = ardetector.create( canvas
); |
||

And with each tick the canvas [image is scanned by the detector][51], 
[triggering callbacks as needed][52]:

| // Ask the detector to make a detection pass. 
    detector.detect(
onMarkerCreated, onMarkerUpdated, onMarkerDestroyed
); |
||

As can be deduced from the code, our application [now detects markers][51] and
[writes its discoveries to the console.][52]

### Reality as a Plane {#\"threejs\"}

`git checkout example_3`  
[live][53], [diff][54], [tag][55] 

An augmented reality display consists of a *reality* view overlaid with 3D
models. Rendering such a display normally consists of two steps. The first is to
render the*reality* view as captured by the camera. In the previous examples 
[we simply copied that image to a `canvas`][56]. But we want to augment the
display with 3D models, and that requires a[WebGL][57] canvas. The complication
is that a WebGL canvas has no context into which we can copy an image. Instead 
we render[a textured plane][58] into the WebGL scene, using images from the
webcam as the texture. ThreeJS can use a`canvas` as a texture source, 
[so we can feed the `canvas`][59] [receiving the video frames][60] into it:

| // Create a texture linked to the canvas.
    var texture = new THREE.Texture(
canvas
); |
||

ThreeJS [caches][61] textures, therefore each time a video frame is copied to
the`canvas` a flag must be set to indicate that the texture cache should be
updated:

| // We need to notify ThreeJS when the texture has changed.
    function update
() {
        texture.
needsUpdate = true;
    } |
||

![][62]

This results in an application which, from the perspective of a user, is no
different than[example_2][63]. But behind the scenes it’s all WebGL; the next
step is to augment it!

### Augmenting Reality

`git checkout example_4`  
[live][64], [diff][65], [tag][66], [movie][67] 

We’re ready to add augmented components to the mix: these will take the form
of 3D models aligned to markers captured by the camera. First we must allow the 
ardector and ThreeJS to communicate, and then we’ll be able to build some models
to augment the fiducial markers.

#### Step 1: Transformation Translation

Programmers familiar with 3D graphics will know that the rendering process
requires two matrices: the model matrix
([transformation][68]) and a camera matrix ([projection][69]). These are
supplied by the[ardetector][70] we implemented earlier, but they cannot be used
as is — the matrix arrays provided by ardetector are incompatible with ThreeJS. 
For example, the helper method[`getTransformMatrix()`][71] returns a 
[Float32Array][72], which ThreeJS does not accept. Fortunately the conversion
is straightforward and easily done through a prototype extension, also known as
[monkey patching][73]:

| // Allow Matrix4 to be set using a Float32Array
    THREE.Matrix4.prototype.
setFromArray = function(m
) {
     return this.set(
      m[], m[4], m[8],  m[12],
      m[1], m[5], m[9],  m[13],
      m[2], m[6], m[10], m[14],
      m[3], m[7], m[11], m[15]
     );
    } |
||

This allows us to set the transformation matrix, but in practice we’ll find
that updates have no effect. This is because of ThreeJS’s[caching][61]. To
accommodate such changes we construct a container object and
[set the `matrixAutoUpdate` flag to `false`][74]. Then for each update to the
matrix we[set `matrixWorldNeedsUpdate` to `true`][75].

#### Step 2: Cube Marks the Marker

Now we’ll use our monkey patches and container objects to display colored
cubes as augmented markers. First we make a cube mesh, sized to fit over the 
fiducial marker:

| function createMarkerMesh(color) {
        var geometry = new THREE.
CubeGeometry( 100,100,100
);
        var material = new THREE.
MeshPhongMaterial( {color:color, side:THREE.DoubleSide
} );
     
        var mesh = new THREE.Mesh( geometry, material
);                      
     
        //Negative half the height makes the object appear "on top" of the
AR Marker.
        mesh.position.z = -50; 
     
        return mesh;
    } |
||

Then we enclose the mesh in [the container object][76]:

| function createMarkerObject(params) {
        var modelContainer =
createContainer
();
     
        var modelMesh = createMarkerMesh(params.color);
        modelContainer.
add( modelMesh
);
     
        function transform(matrix) {
            modelContainer.
transformFromArray( matrix
);
        }
    } |
||

Next we generate marker objects, each one corresponding to a marker ID number
:

| // Create marker objects associated with the desired marker ID.
        var
markerObjects
= {
            16
: arobject.createMarkerObject({color:0xAA0000}), // Marker #16, red.
            32
: arobject.createMarkerObject({color:0x00BB00}), // Marker #32, green.
        }; |
||

The `ardetector.detect()` callbacks 
[apply the transformation matrix to the associated marker][77]. For example,
here the`onCreate` handler adds the transformed model to the arview:

| // This function is called when a marker is initally detected on the stream












![][78]

Our application is now a functioning example of augmented reality!

### Making Holes

In An AR Game the markers are more complex than coloured cubes. They are “
warpholes”, which appear to go -into- the marker page. The effect requires a bit
of trickery, so for the sake of illustration we’ll construct the effect in three
steps.

#### Step 1: Open the Cube

`git checkout example_5`  
[live][79], [diff][80], [tag][81], [movie][82] 

First we remove the top face of the cube to create an open box. This is
accomplished by[setting the face’s material][83] to [be invisible][84]. The
open box is positioned*behind*/*underneath* the marker page 
[by adjusting the Z coordinate to half of the box height][85].

![][86]

The effect is interesting, but unfinished — and perhaps it is not immediately
clear why.

#### Step 2: Cover the Cube in Blue

`git checkout example_6`  
[live][87], [diff][88], [tag][89], [movie][90] 

So what’s missing? We need to hide the part of the box which juts out from
‘behind’ the marker page. We’ll accomplish this by first enclosing the box in a
[slightly larger box][91]. This box will be called an “occluder”, and in 
[step 3][92] it will become an invisibility cloak. For now we’ll leave it 
[visible and colour it blue][93], as a visual aid.

The [occluder objects][94] and the [augmented objects][95] are 
[rendered into the same context, but in separate scenes][96]:

| function render() {
        // Render the reality scene
        renderer.render(reality.scene,
reality.camera
);
     
        // Render the occluder scene
        renderer.render( occluder.scene,
occluder.camera
);
     
        // Render the augmented components on top of the reality scene.

renderer.render(virtual.scene, virtual.camera
);

||

![][97]

This blue jacket doesn’t yet contribute much to the “warphole” illusion.

#### Step 3: Cover the Cube In Invisibility {#\"step3\"}

`git checkout example_7`  
[live][98], [diff][99], [tag][100], [movie][101] 

The illusion requires that the blue jacket be invisible while retaining its
occluding ability — it should be an invisible occluder. The trick is to
[deactivate the colour buffers][102], thereby rendering only to the depth
buffer. The[`render()` method now becomes][103]:

| function render() {
        // Render the reality scene
        renderer.render(reality.scene,
reality.camera
);
     
        // Deactivate color and alpha buffers, leaving only depth buffer
active.
        renderer.context.colorMask(false,false,false,false);
     
        // Render the occluder scene
        renderer.render( occluder.scene,
occluder.camera
);
     
        // Reactivate color and alpha buffers.
        renderer.context.
colorMask(true,true,true,true
);
     
        // Render the augmented components on top of the reality scene.

renderer.render(virtual.scene, virtual.camera
);

||

![][104]

This results in a much more convincing illusion.

### Selecting Holes

`git checkout example_8`  
[live][105], [diff][106], [tag][107] 

An AR Game allows the user to select which warphole to open by positioning the
marker underneath a targeting reticule. This is a core aspect of the game, and 
it is technically known as*object picking*. ThreeJS makes this a fairly simple
thing to do. The key classes are[`THREE.Projector()`][108] and 
[`THREE.Raycaster()`][109], but there is a [caveat][110]: despite the key
method having a name of[`Raycaster.intersect<i>Object</i>()`][111], it
actually takes a`THREE.Mesh` as the parameter. Therefore we 
[add a mesh named “hitbox”][112] to `createMarkerObject()`. In our case 
[it is an invisible geometric plane][113]. Note that we are not explicitly
setting a position for this mesh, leaving it at the default (0,0,0), relative to
the`markerContainer` object. This places it at the mouth of the warphole object
, in the plane of the marker page, which is where the face we removed would be 
if we hadn’t removed it.

Now we have a testable hitbox, we make a class called [`Reticle`][114] to
handle intersection detection and state tracking. Reticle notifications are 
incorporated into the arview by including a callback when we add an object with
[`arivew.add()`][115]. This callback will be invoked whenever the object is
selected, for example:

| view.add( object, function(isSelected) {
        onMarkerSelectionChanged(
marker.id, isSelected
);
    }); |
||

The player is now able to select augmented markers by positioning them at the
center of the screen.

## Refactoring

`git checkout example_9`  
[live][116], [diff][117], [tag][118] 

Our augmented reality functionality is essentially complete. We are able to
detect markers in webcam frames and align 3D objects with them. We can also 
detect when a marker has been selected. We’re ready to move on to the second key
component of An AR Game: the flat 2D space from which the player transports play
pieces. This will require a fair amount of code, and some preliminary 
refactoring would help keep everything neat. Notice that a lot of AR 
functionality is currently in the main application.js file. Let’s excise it and 
place it into a dedicated module named[realspace.js][119], leaving our 
[application.js][120] file much cleaner.

## Panel of Flatspace[&dash;][121]  {#flatspace}

`git checkout example_10`  
[live][122], [diff][123], [tag][124] 

In An AR Game the player’s task is to transfer play pieces from a 2D plane to
a 3D space. The realspace module implemented earlier serves as the the 3D space.
Our 2D plane will be managed by a module named flatspace.js,
[which begins as a skeletal pattern][125] similar to those of 
[application.js][126] and [realspace.js][119].

### The Physics

`git checkout example_11`  
[live][127], [diff][128], [tag][129] 

The [physics][130] of the realspace view comes free with [nature][131]. But the
flatspace pane uses simulated 2D physics, and that requires physics middleware. 
We’ll use a JavaScript[transpilation][132] of the famous [Box2D][133] engine
named[Box2D.js][134]. The JavaScript version is born from the original 
[C++][135] via [LLVM][136], processed by [emscripten][137]. 

Box2D is a rather complex piece of software, but well [documented][138] and
well[described][138]. Therefore this article will, for the most part, refrain
from repeating what is already well-documented in other places. We will instead 
describe the common issues encountered when using Box2D, introduce a solution in
the form of a module, and describe its integration into flatspace.js.

First we build [a wrapper for the raw Box2D.js world engine][139] and name it
boxworld.js. This is then[integrated into flatspace][140].

This does not yield any outwardly visible affects, but in reality we are now
simulating an empty space.

### The Visualization

It would be helpful to be able to see what’s happening. Box2D thoughtfully
provides debug rendering, and Box2D.js facilitates it through something like
[virtual functions][141]. The functions will draw to a `canvas` context, so we
’ll need to create a canvas and then supply the VTable with draw methods.

#### Step 1: Make A Metric Canvas

`git checkout example_12`  
[live][142], [diff][143], [tag][144] 

The `canvas` will map a Box2D world. A canvas uses pixels as its unit of
measurement, whereas Box2D describes its space using meters. We’ll need methods 
to convert between the two, using a[pixel-to-meter ratio][145]. The conversion
methods use this constant to convert from[pixels to meters][146], and from 
[meters to pixels][147]. We also [align the coordinate origins][148]. These
methods are associated with a canvas and all are wrapped into the
[boxview.js module][149]. This makes it easy to 
[incorporate it into flatspace][150]: 

It is instantiated during initialization, its canvas then added to the DOM:

| view = boxview.create({
        width:640, 
        height:480,
        pixelsPerMeter:13,
    });
     
    document.getElementById("flatspace").appendChild( view.canvas ); |
||

There are now two canvases on the page ‐ the flatspace and the realspace. A
bit of CSS in[application.css][151] puts them side-by-side:

| #realspace {
        overflow:hidden;
    }
     
    #flatspace {
        float:left;
    } |
||

#### Step 2: Assemble A Drafting Kit

`git checkout example_13`  
[live][152], [diff][153], [tag][154] 

As mentioned previously, Box2D.js provides hooks for drawing a debug sketch of
the world. They are accessed via a[VTable][155] through the 
[`customizeVTable()`][156] method, and subsequently invoked by 
[`b2World.DrawDebugData()`][157]. We’ll take the draw methods from 
[kripken’s description][156], and wrap them in a module called 
[boxdebugdraw.js][158]. 

Now we can draw, but have nothing to draw. We need to jump through a few hoops
first!

### The Bureaucracy

A Box2D world is populated by entities called Bodies. Adding a body to the
boxworld subjects it to the laws of physics, but it must also comply with the 
rules of the game. For this we create a set of governing structures and methods 
to manage the population. Their application simplifies body creation, collision 
detection, and body destruction. Once these structures are in place we can begin
to implement the game logic, building the system to be played.

#### Creation

`git checkout example_14`  
[live][159], [diff][160], [tag][161] 

Let’s liven up the simulation with some creation. Box2D Body construction is
somewhat verbose, involving fixtures and shapes and physical parameters. So we’
ll stow our body creation methods in a module named[boxbody.js][162]. To create
a body we pass a[boxbody method][163] to [`boxworld.add()`][164]. 
[For example:][165]

| function populate() {
        var ball = world.add(
            boxbody.ball,
            {
                x:,
                y:8,
                radius:10
            }
        );
    } |
||

This yields an undecorated ball in midair experiencing the influence of gravity
. Under contemplation it may bring to mind a[particular whale][166].

#### Registration

`git checkout example_15`  
[live][167], [diff][168], [tag][169] 

We must be able to keep track of the bodies populating flatworld. Box2D
provides access to a body list, but it’s a bit too low level for our purposes. 
Instead we’ll use a field of`b2Body` named `userData`. To this we assign a
unique ID number subsequently used as an index to a registry of our own design. 
It is implemented in boxregistry.js, and is a key aspect of the flatspace 
implementation. It enables the association of bodies with decorative entities (
such as sprites), simplifies collision callbacks, and facilitates the removal of
bodies from the simulation. The implementation details won’t be described here, 
but interested readers can refer to the repo to see how
[the registry is instantiated][170] in boxworld.js, and how 
[the `add()` method][171] [returns wrapped-and-registered bodies.][172]

#### Collision

`git checkout example_16`  
[live][173], [diff][174], [tag][175] 

Box2D collision detection is complicated because the native callback simply
gives two fixtures, raw and unordered, and all collisions that occur in the 
world are reported, making for a lot of conditional checks. The boxregistry.js 
module avails itself to managing the data overload. Through it we assign
[an onContact callback][176] to registered objects. When a Box2D collision
handler is triggered[we query the registry for the associated objects][177] and
check for the[presence of a callback][178]. If the object has a defined
callback then we know its collision activity is of interest. To use this 
functionality in flatspace.js, we simply need to
[assign a collision callback to a registered object][179]:

| function populate() {
        var ground = world.add(
            boxbody.edge,
            {
                x:,
                y:-15,
                width:20,
                height:,
            }
        );
     
        var ball = world.add(
            boxbody.ball,
            {
                x:,
                y:8,
                radius:10
            }
        );
     
        ball.onContact = function(object) {
            console.log("The ball
has contacted:", object
);
        };
    } |
||

#### Deletion

`git checkout example_17`  
[live][180], [diff][181], [tag][182] 

Removing bodies is complicated by the fact that Box2D does not allow calls to
`b2World.DestroyBody()` from within `b2World.Step()`. This is significant
because usually you’ll want to delete a body because of a collision, and 
collision callbacks occur during a simulation step: this is a conundrum! One 
solution is to queue bodies for deletion, then process the queue outside of the 
simulation step. The`boxregistry` addresses the problem by furnishing a flag
, [`isMarkedForDeletion`][183], for each object. The collection of registered
objects[is iterated and listeners are notified of the deletion request][184].
The iteration[happens after a simulation step][185], so the 
[deletion callback cleanly destroys the bodies][186]. Perceptive readers may
notice that we are now
[checking the isMarkedForDeletion flag before invoking collision callbacks][187]

This happens transparently as far as flatspace.js is concerned, so all we need
to do is[set the deletion flag for a registered object][188]:

| ball.onContact = function(object) {
        console.log("The ball has
contacted:", object
);
        ball.isMarkedForDeletion = true;
    }; |
||

Now the body is deleted on contact with the ground.

#### Discerning

`git checkout example_18`  
[live][189], [diff][190], [tag][191] 

When a collision is detected An AR Game needs to know what the object has
collided with. To this end we add an[`is()` method for registry objects][192],
used for comparing objects. We will now adda conditional deletion to our game
: 

| ball.onContact = function(object) {
        console.log("The ball has
contacted:", object
);
        if( object.is( ground ) ) {
            ball.isMarkedForDeletion = true
;
        }
    }; |
||

### A 2D Warphole

`git checkout example_19`  
[live][193], [diff][194], [tag][195] 

We’ve already discussed the realspace warpholes, and now we’ll implement
their flatspace counterparts. The flatspace warphole is simply a body consisting
of a Box2D sensor. The ball should pass*over* a closed warphole, but *through*
an open warphole. Now imagine an edge case where a ball is over a closed 
warphole which is then opened up. The problem is that Box2D’s`onBeginContact`
handler behaves true to its name, meaning that we detected warphole contact 
during the closed state but have since opened the warphole. Therefore the ball 
is not warped, and we’re left with a bug. Our fix is to use a cluster of sensors.
With a cluster there will be a series of`BeginContact` events as the ball moves
*across* the warphole. Thus we can be confident that opening a warphole while
the ball is over it will result in a warp. The sensor cluster generator is named
[`hole` and is implemented in boxbody.js][196]. The generated cluster looks
like this:

![][197]

## The Conduit {#conduit}

At this point we’ve made JSARToolkit and Box2D.js into usable modules. We’
ve used them to create warpholes in realspace and flatspace. The objective of An
AR Game is to transport pieces from flatspace to the realspace, so it is 
necessary that the warpholes communicate. Our approach is as follows:

1.  `git checkout example_20`  
    [live][198], [diff][199], [tag][200] 
    Notify the application when a realspace warphole’s state changes.

2.  `git checkout example_21`  
    [live][201], [diff][202], [tag][203] 
    Set flatspace warphole states according to realspace warphole states.

3.  `git checkout example_22`  
    [live][204], [diff][205], [tag][206] 
    Notify the application when a ball transits an open flatspace warphole.

4.  `git checkout example_23`  
    [live][207], [diff][208], [tag][209] 
    Add a ball to realspace when the application receives a notification of a
    transit.
   

## Conclusion

This article has shown the technical underpinnings of An AR Game. We have
constructed two panes of differing realities and connected them with warpholes. 
A player may now entertain him or herself by transporting a ball from flatspace 
to realspace. Technically this is interesting, but generally it is not fun!

There is still much to be done before this application becomes a game, but they
are outside the scope of this article. Among the remaining tasks are:

*   Sprites and animations.
*   Introduce multiple balls and warpholes.
*   Provide a means of interactively designing levels.

Thanks for reading! We hope this has inspired you to delve into this topic more
!</article>

 [1]: https://developer.mozilla.org/fr/demos/detail/an-ar-game
 [2]: https://developer.mozilla.org/en/demos/devderby/2013/may
 [3]: https://developer.mozilla.org/en/demos/devderby
 [4]: http://en.wikipedia.org/wiki/Augmented_reality
 [5]: http://abrie.github.io/devderby-may-2013
 [6]: http://www.youtube.com/watch?v=Pa_EwQ0DoWk
 [7]: http://dev.w3.org/2011/webrtc/editor/getusermedia.html
 [8]: https://github.com/kig/JSARToolKit
 [9]: https://github.com/mrdoob/three.js/
 [10]: https://github.com/kripken/box2d.js
 [11]: https://github.com/abrie/devderby-may-2013-technical
 [12]: https://github.com/abrie/devderby-may-2013-technical/tags
 [13]: http://abrie.github.io/devderby-may-2013-technical/

 [14]: http://hacks.mozilla.org/2013/10/an-ar-game-technical-overview/#realspace

 [15]: http://hacks.mozilla.org/2013/10/an-ar-game-technical-overview/#flatspace
 [16]: http://hacks.mozilla.org/2013/10/an-ar-game-technical-overview/#conduit

 [17]: http://abrie.github.io/devderby-may-2013-technical/example_0/application.html

 [18]: https://github.com/abrie/devderby-may-2013-technical/commit/bef8bf5c6505e30a3fc9ad0f2f4564df060c4296
 [19]: https://github.com/abrie/devderby-may-2013-technical/tree/example_0
 [20]: http://requirejs.org/

 [21]: https://developer.mozilla.org/en-US/docs/Web/API/window.requestAnimationFrame

 [22]: http://abrie.github.io/devderby-may-2013-technical/example_1/application.html

 [23]: https://github.com/abrie/devderby-may-2013-technical/commit/4d7f2faa18b794be4bd0a10d4fa0c6dfa306e519
 [24]: https://github.com/abrie/devderby-may-2013-technical/tree/example_1
 [25]: http://en.wikipedia.org/wiki/HTML5
 [26]: http://en.wikipedia.org/wiki/Webrtc
 [27]: http://en.wikipedia.org/wiki/Webcam
 [28]: http://firefox.com/
 [29]: https://developer.mozilla.org/en-US/docs/Web/API/Navigator.getUserMedia
 [30]: https://developer.mozilla.org/
 [31]: http://requirejs.org/docs/api.html#define

 [32]: https://github.com/abrie/devderby-may-2013-technical/blob/master/webcam.js

 [33]: https://github.com/abrie/devderby-may-2013-technical/commit/4d7f2faa18b794be4bd0a10d4fa0c6dfa306e519/#L0R32

 [34]: https://github.com/abrie/devderby-may-2013-technical/commit/4d7f2faa18b794be4bd0a10d4fa0c6dfa306e519/#L0R25

 [35]: http://abrie.github.io/devderby-may-2013-technical/example_2/application.html

 [36]: https://github.com/abrie/devderby-may-2013-technical/commit/388378463230b28c333b4954154f087919d7a3c2
 [37]: https://github.com/abrie/devderby-may-2013-technical/tree/example_2
 [38]: http://en.wikipedia.org/wiki/Fiduciary_marker
 [39]: https://github.com/kig/JSARToolKit/tree/master/demos/markers
 [40]: img/AR-Game-16_and_32.png
 [41]: http://www.hitl.washington.edu/artoolkit/
 [42]: http://en.wikipedia.org/wiki/C%2B%2B
 [43]: http://en.wikipedia.org/wiki/University_of_Washington
 [44]: http://www.hitl.washington.edu/home/
 [45]: http://en.wikipedia.org/wiki/Seattle
 [46]: http://nyatla.jp/nyartoolkit/wp/?page_id=198
 [47]: http://www.libspark.org/wiki/saqoosha/FLARToolKit/en

 [48]: https://github.com/abrie/devderby-may-2013-technical/blob/example_2/ardetector.js#L10

 [49]: https://github.com/abrie/devderby-may-2013-technical/blob/example_2/ardetector.js#L24

 [50]: https://github.com/abrie/devderby-may-2013-technical/blob/example_2/ardetector.js#L65

 [51]: https://github.com/abrie/devderby-may-2013-technical/blob/example_2/application.js#L34

 [52]: https://github.com/abrie/devderby-may-2013-technical/blob/example_2/application.js#L47

 [53]: http://abrie.github.io/devderby-may-2013-technical/example_3/application.html

 [54]: https://github.com/abrie/devderby-may-2013-technical/commit/c9de6aaf6805c760bbab6671126c323ac4227dad
 [55]: https://github.com/abrie/devderby-may-2013-technical/tree/example_3

 [56]: https://github.com/abrie/devderby-may-2013-technical/blob/example_2/application.js#L19
 [57]: https://developer.mozilla.org/en/WebGL "WebGL"

 [58]: https://github.com/abrie/devderby-may-2013-technical/blob/example_3/arview.js#L11

 [59]: https://github.com/abrie/devderby-may-2013-technical/blob/example_3/arview.js#L14

 [60]: https://github.com/abrie/devderby-may-2013-technical/blob/example_3/application.js#L31
 [61]: http://en.wikipedia.org/wiki/Cache_(computing)
 [62]: img/AR-Game-example_3.png

 [63]: http://hacks.mozilla.org/2013/10/an-ar-game-technical-overview/#example_2

 [64]: http://abrie.github.io/devderby-may-2013-technical/example_4/application.html

 [65]: https://github.com/abrie/devderby-may-2013-technical/commit/e16eb6c510d3f0e38b66057befa51406b516e38c
 [66]: https://github.com/abrie/devderby-may-2013-technical/tree/example_4
 [67]: http://youtu.be/g4n9lebq_Kg
 [68]: http://en.wikipedia.org/wiki/Transformation_matrix
 [69]: http://en.wikipedia.org/wiki/Camera_matrix

 [70]: https://github.com/abrie/devderby-may-2013-technical/blob/master/ardetector.js

 [71]: https://github.com/abrie/devderby-may-2013-technical/blob/example_4/ardetector.js#L24

 [72]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Typed_arrays/Float32Array
 [73]: http://en.wikipedia.org/wiki/Monkey_patch

 [74]: https://github.com/abrie/devderby-may-2013-technical/blob/example_4/arobject.js#L21

 [75]: https://github.com/abrie/devderby-may-2013-technical/blob/example_4/arobject.js#L16

 [76]: https://github.com/abrie/devderby-may-2013-technical/blob/example_4/arobject.js#L19

 [77]: https://github.com/abrie/devderby-may-2013-technical/blob/example_4/application.js#L57
 [78]: img/AR-Game-example_4.png

 [79]: http://abrie.github.io/devderby-may-2013-technical/example_5/application.html

 [80]: https://github.com/abrie/devderby-may-2013-technical/commit/1c28aa84b7559cdd46f04d31ef94abb7b7136854
 [81]: https://github.com/abrie/devderby-may-2013-technical/tree/example_5
 [82]: http://youtu.be/50nYnIBJtTg

 [83]: https://github.com/abrie/devderby-may-2013-technical/blob/example_5/arobject.js#L35

 [84]: https://github.com/abrie/devderby-may-2013-technical/blob/example_5/arobject.js#L29

 [85]: https://github.com/abrie/devderby-may-2013-technical/blob/example_5/arobject.js#L38
 [86]: img/AR-Game-example_5.png

 [87]: http://abrie.github.io/devderby-may-2013-technical/example_6/application.html

 [88]: https://github.com/abrie/devderby-may-2013-technical/commit/0e4f2d5b9058425cbf61befabb510df20d5563ac
 [89]: https://github.com/abrie/devderby-may-2013-technical/tree/example_6
 [90]: http://youtu.be/BtAxElxQiFk

 [91]: https://github.com/abrie/devderby-may-2013-technical/blob/example_6/arobject.js#L44
 [92]: http://hacks.mozilla.org/2013/10/an-ar-game-technical-overview/#step3

 [93]: https://github.com/abrie/devderby-may-2013-technical/blob/example_6/arobject.js#L46

 [94]: https://github.com/abrie/devderby-may-2013-technical/blob/example_6/arview.js#L78

 [95]: https://github.com/abrie/devderby-may-2013-technical/blob/example_6/arview.js#L75

 [96]: https://github.com/abrie/devderby-may-2013-technical/blob/example_6/arview.js#L85
 [97]: img/AR-Game-example_6.png

 [98]: http://abrie.github.io/devderby-may-2013-technical/example_7/application.html

 [99]: https://github.com/abrie/devderby-may-2013-technical/commit/81b4a4ae2c44a910b319eff43edd7b1f8ea1651c
 [100]: https://github.com/abrie/devderby-may-2013-technical/tree/example_7
 [101]: http://youtu.be/GM5eu3NBMZY

 [102]: https://github.com/abrie/devderby-may-2013-technical/blob/example_7/arview.js#L90

 [103]: https://github.com/abrie/devderby-may-2013-technical/blob/example_7/arview.js#L85
 [104]: img/AR-Game-example_7.png

 [105]: http://abrie.github.io/devderby-may-2013-technical/example_8/application.html

 [106]: https://github.com/abrie/devderby-may-2013-technical/commit/3f236ed4f79704be9733cba55efd4f4c47a35ef5
 [107]: https://github.com/abrie/devderby-may-2013-technical/tree/example_8

 [108]: https://github.com/abrie/devderby-may-2013-technical/blob/example_8/arview.js#L92

 [109]: https://github.com/abrie/devderby-may-2013-technical/blob/example_8/arview.js#L95
 [110]: http://en.wiktionary.org/wiki/caveat

 [111]: https://github.com/abrie/devderby-may-2013-technical/blob/example_8/arview.js#L100

 [112]: https://github.com/abrie/devderby-may-2013-technical/blob/example_8/arobject.js#L78

 [113]: https://github.com/abrie/devderby-may-2013-technical/blob/example_8/arobject.js#L43

 [114]: https://github.com/abrie/devderby-may-2013-technical/blob/example_8/arview.js#L62

 [115]: https://github.com/abrie/devderby-may-2013-technical/blob/example_8/arview.js#L183

 [116]: http://abrie.github.io/devderby-may-2013-technical/example_9/application.html

 [117]: https://github.com/abrie/devderby-may-2013-technical/commit/005f87d6c40322309ad9a68d4b2627cc6599684d
 [118]: https://github.com/abrie/devderby-may-2013-technical/tree/example_9

 [119]: https://github.com/abrie/devderby-may-2013-technical/blob/example_9/realspace.js

 [120]: https://github.com/abrie/devderby-may-2013-technical/blob/example_9/application.js
 [121]: http://en.wikipedia.org/wiki/Flatland

 [122]: http://abrie.github.io/devderby-may-2013-technical/example_10/application.html

 [123]: https://github.com/abrie/devderby-may-2013-technical/commit/1ae5a00a63d0a6def6aeeab1f6d6fa7552f9d488
 [124]: https://github.com/abrie/devderby-may-2013-technical/tree/example_10

 [125]: https://github.com/abrie/devderby-may-2013-technical/blob/example_10/flatspace.js

 [126]: https://github.com/abrie/devderby-may-2013-technical/blob/example_0/application.js

 [127]: http://abrie.github.io/devderby-may-2013-technical/example_11/application.html

 [128]: https://github.com/abrie/devderby-may-2013-technical/commit/23e97233b1da08b4aebedd96917e406613286c1a
 [129]: https://github.com/abrie/devderby-may-2013-technical/tree/example_11
 [130]: http://en.wikipedia.org/wiki/Motion_(physics)
 [131]: http://en.wikipedia.org/wiki/Nature
 [132]: http://en.wikipedia.org/wiki/Source-to-source_compiler
 [133]: http://en.wikipedia.org/wiki/Box2d
 [134]: https://github.com/kripken/box2d.js/
 [135]: https://code.google.com/p/box2d/
 [136]: http://en.wikipedia.org/wiki/LLVM
 [137]: https://github.com/kripken/emscripten
 [138]: http://box2d.org/manual.pdf

 [139]: https://github.com/abrie/devderby-may-2013-technical/blob/example_11/boxworld.js

 [140]: https://github.com/abrie/devderby-may-2013-technical/commit/23e97233b1da08b4aebedd96917e406613286c1a#diff-19f884bd80981aedf202452cae4fc0f0
 [141]: http://en.wikipedia.org/wiki/Virtual_function

 [142]: http://abrie.github.io/devderby-may-2013-technical/example_12/application.html

 [143]: https://github.com/abrie/devderby-may-2013-technical/commit/dfa4fee9ccb3a40d17b7678adfdeb0d87a63420b
 [144]: https://github.com/abrie/devderby-may-2013-technical/tree/example_12

 [145]: https://github.com/abrie/devderby-may-2013-technical/blob/example_12/boxview.js#L9

 [146]: https://github.com/abrie/devderby-may-2013-technical/blob/example_12/boxview.js#L24

 [147]: https://github.com/abrie/devderby-may-2013-technical/blob/example_12/boxview.js#L31

 [148]: https://github.com/abrie/devderby-may-2013-technical/blob/example_12/boxview.js#L12

 [149]: https://github.com/abrie/devderby-may-2013-technical/blob/example_12/boxview.js

 [150]: https://github.com/abrie/devderby-may-2013-technical/commit/dfa4fee9ccb3a40d17b7678adfdeb0d87a63420b#diff-19f884bd80981aedf202452cae4fc0f0

 [151]: https://github.com/abrie/devderby-may-2013-technical/blob/7d2c4de27aa3ed5b04ab1b02c0e79643a51e17f2/application.css

 [152]: http://abrie.github.io/devderby-may-2013-technical/example_13/application.html

 [153]: https://github.com/abrie/devderby-may-2013-technical/commit/806d65ceca9707463a1cdfebc9d8feef433d85eb
 [154]: https://github.com/abrie/devderby-may-2013-technical/tree/example_13
 [155]: http://en.wikipedia.org/wiki/Virtual_method_table
 [156]: https://github.com/kripken/box2d.js/#using-debug-draw

 [157]: https://github.com/abrie/devderby-may-2013-technical/blob/example_13/boxdebugdraw.js#L181

 [158]: https://github.com/abrie/devderby-may-2013-technical/blob/example_13/boxdebugdraw.js

 [159]: http://abrie.github.io/devderby-may-2013-technical/example_14/application.html

 [160]: https://github.com/abrie/devderby-may-2013-technical/commit/b7117c1914941514855e01ee1a9573b853f4811a
 [161]: https://github.com/abrie/devderby-may-2013-technical/tree/example_14

 [162]: https://github.com/abrie/devderby-may-2013-technical/blob/example_14/boxbody.js

 [163]: https://github.com/abrie/devderby-may-2013-technical/blob/example_14/boxbody.js#L5

 [164]: https://github.com/abrie/devderby-may-2013-technical/blob/example_14/boxworld.js#L13

 [165]: https://github.com/abrie/devderby-may-2013-technical/blob/example_14/flatspace.js#L33

 [166]: http://en.wikipedia.org/wiki/List_of_minor_The_Hitchhiker's_Guide_to_the_Galaxy_characters#Whale

 [167]: http://abrie.github.io/devderby-may-2013-technical/example_15/application.html

 [168]: https://github.com/abrie/devderby-may-2013-technical/commit/04a86c1b28b642d876dd7e80947f361718c120d6
 [169]: https://github.com/abrie/devderby-may-2013-technical/tree/example_15

 [170]: https://github.com/abrie/devderby-may-2013-technical/blob/example_15/boxworld.js#L8

 [171]: https://github.com/abrie/devderby-may-2013-technical/blob/example_15/boxworld.js#L14

 [172]: https://github.com/abrie/devderby-may-2013-technical/blob/example_15/boxworld.js#L16

 [173]: http://abrie.github.io/devderby-may-2013-technical/example_16/application.html

 [174]: https://github.com/abrie/devderby-may-2013-technical/commit/d6a6657060d040bec2a002deac34d03e81122d25
 [175]: https://github.com/abrie/devderby-may-2013-technical/tree/example_16

 [176]: https://github.com/abrie/devderby-may-2013-technical/blob/example_16/boxregistry.js#L20

 [177]: https://github.com/abrie/devderby-may-2013-technical/blob/example_16/boxworld.js#L38

 [178]: https://github.com/abrie/devderby-may-2013-technical/blob/example_16/boxworld.js#L41

 [179]: https://github.com/abrie/devderby-may-2013-technical/blob/example_16/flatspace.js#L52

 [180]: http://abrie.github.io/devderby-may-2013-technical/example_17/application.html

 [181]: https://github.com/abrie/devderby-may-2013-technical/commit/9d9c2b91c7c7f55610d0b8459eccc76c4381c554
 [182]: https://github.com/abrie/devderby-may-2013-technical/tree/example_17

 [183]: https://github.com/abrie/devderby-may-2013-technical/blob/example_17/boxregistry.js#L22

 [184]: https://github.com/abrie/devderby-may-2013-technical/blob/example_17/boxregistry.js#L50

 [185]: https://github.com/abrie/devderby-may-2013-technical/blob/example_17/boxworld.js#L26

 [186]: https://github.com/abrie/devderby-may-2013-technical/blob/example_17/boxworld.js#L18

 [187]: https://github.com/abrie/devderby-may-2013-technical/blob/example_17/boxworld.js#L48

 [188]: https://github.com/abrie/devderby-may-2013-technical/blob/example_17/flatspace.js#L54

 [189]: http://abrie.github.io/devderby-may-2013-technical/example_18/application.html

 [190]: https://github.com/abrie/devderby-may-2013-technical/commit/682aa2b215e6d918393d03dd103f3338c59a2be0
 [191]: https://github.com/abrie/devderby-may-2013-technical/tree/example_18

 [192]: https://github.com/abrie/devderby-may-2013-technical/blob/example_18/boxregistry.js#L23

 [193]: http://abrie.github.io/devderby-may-2013-technical/example_19/application.html

 [194]: https://github.com/abrie/devderby-may-2013-technical/commit/736662188f65cbdb6aae7572ffcc46d1ced38253
 [195]: https://github.com/abrie/devderby-may-2013-technical/tree/example_19

 [196]: https://github.com/abrie/devderby-may-2013-technical/blob/example_19/boxbody.js#L42
 [197]: img/sensor-cluster.png

 [198]: http://abrie.github.io/devderby-may-2013-technical/example_20/application.html

 [199]: https://github.com/abrie/devderby-may-2013-technical/commit/ada13bcf70b633a06afbad065d2cdea6b86baeb2
 [200]: https://github.com/abrie/devderby-may-2013-technical/tree/example_20

 [201]: http://abrie.github.io/devderby-may-2013-technical/example_21/application.html

 [202]: https://github.com/abrie/devderby-may-2013-technical/commit/b93aab3db72cdcd004d1048c9a7e8b097ced547f
 [203]: https://github.com/abrie/devderby-may-2013-technical/tree/example_21

 [204]: http://abrie.github.io/devderby-may-2013-technical/example_22/application.html

 [205]: https://github.com/abrie/devderby-may-2013-technical/commit/7d1c2a2ce676f19a1970ce254eeecbeced299a34
 [206]: https://github.com/abrie/devderby-may-2013-technical/tree/example_22

 [207]: http://abrie.github.io/devderby-may-2013-technical/example_23/application.html

 [208]: https://github.com/abrie/devderby-may-2013-technical/commit/05b6bf16fa2da753ddfd9a777a5523e2272df693
 [209]: https://github.com/abrie/devderby-may-2013-technical/tree/example_23
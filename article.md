# AFrame Game Blog	

Today I’m going to show you how to build a simple game with AFrame and some models from the Sketchup design challenge.  I hope this will inspire you to submit to the current  challenge we are running with sketchfab. There’s still time to enter. All submissions must be in by [date].

Unlike with my previous tutorials, in this one we will walk through creating the entire application. Not just the basic interaction, but adding and scaling models, creating a landscape with rock**s, adding sounds and lighting to make the player feel immersed in the environment, and finally interaction tweaks for different form factors.


To start off we will begin with an empty HTML file with aframe included.

``` html
<html>
  <head>
    <!-- aframe itself -->
    <script src="https://aframe.io/releases/0.7.0/aframe.min.js"></script>
  </head>
  <body>
  </body>
</html>
```


At first we won’t make the scene fancy at all. We just want to prove that our concept will work, so we won’t add any lighting, models, or sound effects. We will keep it simply until the underlying concept is proven, then we will make it pretty.

Let’s start off with a scene with stats turns on, then add a camera with look controls at a height of 1.5 m, which is a good camera height for VR interaction (roughly corresponding to the average eye height of most humans).
```
<a-scene stats>
    <a-entity camera look-controls position="0 1.5 0">
        <a-cursor></a-cursor>
    </a-entity>
</a-scene>
```
 
Notice the `a-cursor` inside of the camera. This will draw a little circular cursor, which is important for displays that don’t have controllers, such as cardboard.

Our simple game will have an object that pops up from a cauldron then falls back down as gravity takes hold. The player will have a paddle or stick or other weapon to hit the object. If the player misses then it should fall on the ground. For now let’s represent the object with a sphere and the ground with a simple plane. Put this code inside of the `a-scene`.

```
    <a-entity id='ball'
              position="0 1 -4"
              material="color:green;"
              geometry="primitive:sphere; radius: 0.5;"
    ></a-entity>

    <a-plane color='red' rotation="-90 0 0" 
				width="100" height="100"></a-plane>
```

	* no lighting, just the default
	* put objects where we want them to be
	* no extra decorations.
	* the goal is to prove our app works and would be reasonably fun
	* no navigation. Already where you need to be. Just hit the ball as it pops up.
	* simplest possible interaction, paddle attached to the camera. Just move back and forth to hit it. A little awkward in desktop but great for testing.
	* show stats

Note that I’m using the long syntax of a-entity for the ball rather than a-sphere. That’s because later we will switch the geometry 
to an externally loaded model. However, the plane will always be a plane, so I’ll the use the shorter syntax for that one.

Now we have an object to hit but nothing to hit it with. Now add a box for the paddle.  Instead of swinging using a controller 
we will start with the simplest possible interaction: put the box inside of the camera. Then you can swing it by just turning 
your head (or dragging the scene camera w/ the mouse on desktop). A little awkward but it works well enough for now.

Also note that I placed the paddle box at z -3. If I left it at the default position it would seem to disappear, but it’s 
actually still there. It’s just too close to the camera to see. If I look down at my feet I can see it though.  Whenever 
you are working with aframe and your object won’t show up, first check if it’s behind you or too close to the camera.

```
        <a-entity position="0 0 -3" id="weapon">
            <a-box color='blue' 
                   width='0.25' height='0.5' depth='3'></a-box>
        </a-entity>
```


Great. Now all of the elements of our scene are here. If you followed along you should have a scene on your desktop that looks like this.

![screenshot](./images/Firefox%20NightlyScreenSnapz021.png)

If you play with this demo you’ll see that you can move your head and the paddle moves with it, but trying to hit the ball won’t do anything. 
That’s because we just have geometry. The computer knows how our objects look but nothing about how they behave. For that we need physics.

Physics engines can be complicated, but fortunately *[name]* has created AFrame bindings for the excellent open source physics framework *[name]*. We just need to include his `aframe-extras` library to start playing with physics.

Add this to the `head` of the html page

```
    <!-- physics and other extras -->
    <script src="//cdn.rawgit.com/donmccurdy/aframe-extras/v3.13.1/dist/aframe-extras.min.js"></script>
```

Now we can turn on physics by adding `physics="debug:true;”` to the `a-scene`.

Of course merely turning on the physics engine won’t do anything. We have to tell it which objects in the scene should be affected by gravity and other forces.  We do this with dynamic and static bodies.  A dynamic body is an object with full physics. It can transmit force and be affected by other forces, including gravity. A static body can transmit force when something hits it, but is otherwise unaffected by forces.  Generally you will use a static body for something that doesn’t move, like the ground or a wall, and a dynamic body for things which do move around the scene, like our ball.

Let’s make the ground be a static body and the ball be dynamic by adding dynamic-body and static-body to their components:

```
    <a-entity id='ball'
              position="0 1 -4"
              material="color:green;"
              geometry="primitive:sphere; radius: 0.5;"
              dynamic-body
    ></a-entity>

    <a-plane color='red'
             static-body
             rotation="-90 0 0" width="100" height="100"></a-plane>
```

Great. Now when you reload the page the ball will fall to the ground. You may also see grid lines or dots on the ball or plane. These are debugging information from the physics engine to let us see the edges of our objects from a physics perspective. It is possible to have the physics engine use a different size or shape for our objects than the real drawn geometry. Sounds crazy but it’s actually quite useful, as we will see later.

Now we need to make the paddle able to hit the ball. Since the paddle moves you might think we should use a dynamic-body, but really we want *our code* (and the camera) to control the position of the paddle, *not* the physics engine. We just want the paddle to be there for exerting forces on the ball, not the other way around, so we will use a *static-body*.

```
    <a-entity camera look-controls position="0 1.5 0">
        <a-cursor></a-cursor>
        <a-entity position="0 0 -3" id='weapon'>
            <a-box color='blue' width='0.25' height='0.5' depth='3'
                   static-body></a-box>
        </a-entity>
    </a-entity>
```

Now we can move the camera to swing the paddle and hit the ball. If you hit it hard then it will fly off to the side instead of rolling, exactly what we want!
You might ask why not just turn on physics for everything. The reason we don’t is because for two reasons. First, physics takes CPU type, and the more objects with physics the more will use up CPU. Secondly, many things in the scene we don’t actually want physics on. If I have a tree in my scene I don’t want it to fall down just because it’s a millimeter above the ground. I don’t want the moon to fall from the sky just because it’s above the ground.  Only turn on physics for things that need it for your application.

## Collisions
	* simple query functions for event handling.
Moving the ball by hitting it is fun, but to be a real game we need to track when the paddle hits the ball to increase the player’s score and also to reset the ball back to the middle. We will do this with collisions. The physics engine emits a `collide` event each time an object hits another object. By listening to this event we can find out when something was hit, what it is, can manipulate it.

First let’s make some utility functions for accessing DOM elements. I’ve put these at the top of the page so they will be available to code everywhere.

```
    <script>
        $ = (sel) => document.querySelector(sel)
        $$ = (sel) => document.querySelectorAll(sel)
        on = (elem, type, hand) => elem.addEventListener(type,hand)
    </script>
```

Let’s talk about the functions we need. First, we want to reset the ball after the player has hit it or if they missed and a certain number of seconds went by. Resetting means moving the ball back to the center, setting the forces back to zero, and initializing a timeout.  Let’s create the `resetBall` function to do this:
```
    let hit = false
    let resetId = 0
    const resetBall = () => {
        clearTimeout(resetId)
        $("#ball").body.position.set(0, 0.6,-4)
        $("#ball").body.velocity.set(0, 5,0)
        $("#ball").body.angularVelocity.set(0, 0,0)
        hit = false
        resetId = setTimeout(resetBall,6000)
    }
```

In the above code I’m using the `$` function with a selector to find the ball element in the page. The physics engine adds a `body` property to the element containing all of the physics attributes. We can reset the position, velocity, and angularVelocity from here. The code above also sets a timeout to call resetBall again after six seconds, if nothing else happens. 

There’s two things to note here. First, I’m setting `body.position` rather than the regular `position` component that all AFrame entities have. That’s because the physics engine is in charge of this object so we need to tell the *physics engine* about the changes, not aframe.  The second thing to note is that the velocity is not reset to zero. Instead it’s set to the vector `0,5,0`.  This means zero velocity in the x and z directions, but *5* in the y direction. This will give the ball an initial velocity vertically, shooting it up. Of course gravity will start to affect it as soon as the ball jumps so the velocity will quickly slow down. If I wanted to make the game harder I could increase the initial velocity here, or point the vector in a random direction. Lots of opportunities for improvements.

Now we need to know when the collision actually happens so we can increment the score and trigger the reset. We’ll do this by handling the `collide` event on the `#weapon` entity.

```
    on($("#weapon"),'collide',(e)=>{
        const ball = $("#ball")
        if(e.detail.body.id === ball.body.id && !hit) {
            hit = true
            score = score + 1
            clearTimeout(resetId)
            resetId = setTimeout(resetBall,2000)
        }
    })

    setTimeout(resetBall,3000)
```

The code above checks if the collision event is for the ball by comparing the body ids. It also makes sure the play didn’t already hit the ball, otherwise they could hit the ball over and over again before we reset it.
If it was hit, then set hit to true, clear the reset timeout, and schedule a new one for two seconds in the future. 

Great, now we can launch the ball over and over and keep track of score.  Of course a score isn’t very useful if we can’t see it. Let’s add a text element *inside* of the camera, so it is always visible. This is called a Heads Up Display or HUD.

```
<a-entity camera ....
        <a-text id="score" value="Score" position="-0.2 -0.5 -1" color="red" width="5" anchor="left"></a-text>
</a-entity>
```

Now we need to update the score text whenever the score changes. Let’s add this to the end of the  `collide` event handler.

```
    on($("#weapon"),'collide',(e)=>{
        const ball = $("#ball")
        if(e.detail.body.id === ball.body.id && !hit) {
...
            $("#score").setAttribute('text','value','Score '+score)
        }
    })
```
Now we can see the score on screen. It should look like this:
[image:1520B632-C985-45C3-90FC-E58AE030B775-1025-00000264A11748BC/Firefox NightlyScreenSnapz022.png]


## Adding Models

Now we have a basic running game. The player can hit the ball with a paddle and get points.  It’s time to make this look better with real 3D models.
The last [real title] resulted in tons of great 3D scenes built around the theme of Low Poly Medieval Fantasy.  Many of these have already been split up into individual assets and tagged with [medievalfantasyassets](https://sketchfab.com/tags/medievalfantasyassets)
For this project I chose to use this [imp model](https://sketchfab.com/models/e4dd9c9d9ad14976a4ce0d1f6049292e?ref=related) as the ball and this [staff mode](https://sketchfab.com/models/9d96aab0ddc34aae9f7bb5ae8dfc8bd6#) for the paddle.  Since we are going to be loading lots of models let’s load them as assets.  Assets are large chunks of data (images, sounds, models) that are pre-loaded and cached automatically when the game starts. Put this at the top of the scene and adjust the `src` urls to point to wherever you downloaded the models.

```
    <a-assets>
        <a-asset-item id="imp" src="models/imp/scene.gltf"></a-asset-item>
        <a-asset-item id="staff" src="models/staff/scene.gltf"></a-asset-item>
    </a-assets>
```

Now we can swap the sphere with the imp and the paddle box for the staff. Update the weapon element like this
```
        <a-entity position="0 0 -3" id="weapon">
            <a-entity gltf-model="#staff"></a-entity>
        </a-entity>
```
And the ball element like this:
```
    <a-entity id='ball'
              position="0 1 -4"
              dynamic-body
    >
        <a-entity id='imp-model' gltf-model="#imp"></a-entity>
    </a-entity>
```


We can see the imp but the staff is missing. What happened?
[image:9DCF1460-8C46-4F3F-A1E8-DD4262A35B37-29782-0003508C9AD58EE7/Firefox NightlyScreenSnapz024.png]

The problem is the staff model itself.  The imp model is centered inside of it’s coordinate system, so it is visually positioned where we put it. However the staff model is not. It’s center is significantly off from the center of it’s coordinate system; roughly 15 to 20 meters away.  This is common with models you find online.  To fix it we need to translate the model’s position to account for the offset.  After playing around with the staff model I found that an offset of 2.3, -2.7, -16.3 did the trick. I also had to rotate it 90 degrees to make it horizontal and shift it forward by four meters so that it’s visible. Wrap the model with an additional entity to apply the translation and rotation.

```
        <a-entity id=“weapon”  rotation="-90 0 0" position="0 0 -4">
            <a-entity position="2.3 -2.7 -16.3" 
                      gltf-model="#staff"
                      static-body></a-entity>
        </a-entity>
```


Now we can see the staff but we still have a problem. The staff is not a simple geometric shape, it’s a full 3d model. The physics engine can’t work directly with a full mesh. Instead it needs to know which primitive object to use.  We could use a box like we did originally, but I chose to go with a /sphere/ centered at the end of the staff. That’s the part that the player should actually use to hit the imp, and by making it larger than the staff’s diameter we can make the game easier than it would be in real life. We also need to move the static-body definition to the outer entity so that it isn’t affected by the model offset.

```
        <a-entity rotation="-90 0 0" position="0 0 -4" id='weapon'
                  static-body="shape:sphere; sphereRadius: 0.3;">
            <a-entity position="2.3 -2.7 -16.3" 
                      gltf-model="#staff" ></a-entity>
        </a-entity>
```

[image:E4B36331-CE8C-4B28-999F-56B7D3274520-29782-0003531D9B0D8719/Firefox NightlyScreenSnapz025.png]

# Adding scenery
Now that we have the core game mechanics working correctly with the new models, let’s add some decorations. I grabbed more models from SketchFab for a [moon](https://sketchfab.com/models/83c39af4bc5c478296501b80ef993855#download), a [cauldron](https://sketchfab.com/models/e6eb5febb0ef4b63b3a90206a8b96135#download), a [rock](https://sketchfab.com/models/6a0fa67b84a44d51b929d8450fb1ec48), and [two](https://sketchfab.com/models/4ddc25af08b24509aa9aba53c5365967) different [trees](https://sketchfab.com/models/71c0756bf001437ebab9955c12f10d77).  Place them in the scene  ad different positions. 
```
    <a-assets>
        <a-asset-item id="imp" src="models/imp/scene.gltf"></a-asset-item>
        <a-asset-item id="staff" src="models/staff/scene.gltf"></a-asset-item>
        <a-asset-item id="tree1" src="models/arbol1/scene.gltf"></a-asset-item>
        <a-asset-item id="tree2" src="models/arbol2/scene.gltf"></a-asset-item>
        <a-asset-item id="moon" src="models/moon/scene.gltf"></a-asset-item>
        <a-asset-item id="cauldron" src="models/cauldron/scene.gltf"></a-asset-item>
        <a-asset-item id="rock1" src="models/rock1/scene.gltf"></a-asset-item>
    </a-assets>
...
    <!-- cauldron -->
    <a-entity  position="1.5 0 -3.5" gltf-model="#cauldron"></a-entity>
    <!-- the moon -->
    <a-entity gltf-model="#moon"></a-entity>

    <!-- trees -->
    <a-entity gltf-model="#tree2" position="38 8.5 -10"></a-entity>
    <a-entity gltf-model="#tree1" position="33 5.5 -10"></a-entity>
    <a-entity gltf-model="#tree1" position="33 5.5 -30"></a-entity>
```

Now this is starting to look like a real scene!
[image:537E9552-53C7-4A57-B3C2-153D9BBD2266-29782-000353EA7BFD6CA8/Firefox NightlyScreenSnapz027.png]

The cauldron has bubbles which appeared to animate on SketchFab but they aren’t animating here.  The animation is stored inside the model but it isn’t automatically played without an additional component.  Just add `animation-mixer` to the entity for the cauldron.

The final game has rocks scattered around the field. However, we really don’t want to manually position fifty different rocks. Instead we can write a component to randomly position them for us.

The aframe docs explain how to create a component so I won’t recount it all here, but the gist of it is this:  a component has some input properties and then executes code when init() is called. In this case, we want to accept the source of a model, some variables controlling how to distribute the model around the scene, then have a function which will create N copies of the model.  

Below is the code. I know it looks intimidating but it’s actually pretty simple. We’ll go through it step by step.

```
    <!-- alternate random number generator -->
    <script src="js/random.js"></script>
    <!-- our `distribute` component -->
    <script>
        AFRAME.registerComponent('distribute', {
            schema: {
                src: {type:'string'},
                jitter: {type:'vec3'},
                centerOffset: {type:'vec3'},
                radius: {type:'number'}
            },
            init: function() {
                const rg = new Random(Random.engines.mt19937().seed(10))
                const center = new THREE.Vector3(this.data.centerOffset.x, this.data.centerOffset.y, this.data.centerOffset.z)
                const jx = this.data.jitter.x
                const jy = this.data.jitter.y
                const jz = this.data.jitter.z
                if($(this.data.src).hasLoaded) {
                    const s = this.data.radius
                    for(let i = -s; i<s; i++) {
                        for(let j=-s; j<s; j++) {
                            const el = document.createElement('a-entity')
                            el.setAttribute('gltf-model', this.data.src)
                            const offset = new THREE.Vector3(i*s + rg.real(-jx,jx), rg.real(-jy,jy), j*s - rg.real(-jz,jz));
                            el.setAttribute('position', center.clone().add(offset));
                            el.setAttribute('rotation',{x:0, y:rg.real(-45,45)*Math.PI/180, z:0})
                            const scale = rg.real(0.5,1.5)
                            el.setAttribute('scale',{x:scale,y:scale,z:scale})
                            $('a-scene').appendChild(el)
                        }
                    }
                }
            }
        })
    </script>
```

First I import random.js. This is a random number generator from this project[link]. We could use the Math.random() function built into Javascript, but I want to ensure that the rocks are always positioned the same way every time the game is run. The [name] generator lets us provide a seed.   In the first line of the init() code you can see that I used the seed 10. I actually tried several different seeds until I found one that I liked the look of.  If I /did/ actually want each load to be different, say different levels of the game, then I could provide a different seeds for each level.

The core of the distribute component is the nested for loops. It creates a grid of entities, each attached to the same model. For each copy we will translate it from the natural center point of the original model (the modelCenter parameter) , adding a random offset using the `jitter` parameter.  Jitter represents the maximum amount the rock should move from that grid point.  Using `0 0 0` would be no jitter.  Using `0 10 0` would make the rocks go vertically anywhere between -10 and 10, but not move at all in the horizontal plane.  For this game I used `2 0.5 2` to move them around mostly horizontally but move up and down a tiny bit.  The loop code also gives the rocks a random scale and rotation around the Y axis, just to make the scene look a bit more organic.

This is the final result.

[image:1E26D2F5-778B-457F-A758-8F3A308491ED-29782-00035523B023C926/Firefox NightlyScreenSnapz028.png]


# Lighting

Our game works now and has some nice landscaping, but it still doesn’t feel very immersive. That’s because the lighting is all wrong. The sky is pure white and the ground is pure red. The trees don’t have shadows and there is no firelight coming from the cauldron. The moon is out so it must be night time, but we don’t see any reflections of moonlight anywhere. A-Frame has given us default lighting but it no longer meets our needs. Let’s add our own lighting.

First change the color of the ground to something more ground-like, a dark green.

``` html
    <a-plane color="#52430e" ...
```

Add a dark twilight sky
``` html
    <!-- background sky -->
    <a-sky color="#270d2c"></a-sky>
```

I did try adding fog for extra mood, but it simply blocked the sky, so I took it out.

For the moonlight we will use a directional light. This means the light comes from a particular direction but is positioned infinitely far away so that the light hits all surfaces from the same angle. For something like the moon this is what we want.
``` html
    <a-entity light="type: directional; color: #ffffff; intensity: 0.5;"
              position="31 80 -50"></a-entity>
```

Here’s what it looks like now.
[image:0F6BF042-36CC-4F5B-A50A-EB26EBA22F4E-29782-000355CDB6F8E066/Firefox NightlyScreenSnapz029.png]

Hmm. We are getting there but it’s still not quite right.  The moonlight certainly is realistic but the bottoms of the rocks and trees are too dark to see.  While this might be a realistic scene it doesn’t feel like a place that has been lived in. A common trick in movies when they want to shoot a night scene is to have a colored light shining up to illuminate the undersides of objects without making the scene so bright that the illusion of night time is ruined.  We can do this with a hemisphere light. This light gives us one color above and one below. I used white for the upper and a sort of purplish dark blue for the lower, at an intensity of 0.4, however feel free to experiment.

```html
    <!-- hemisphere light going from white to dark blue -->
    <a-entity light="type: hemisphere; color: white; groundColor: #5424ff; intensity: 0.4"></a-entity>
```

Now just one more thing. The fire under the cauldron should emit a warm red glow and the near by rock should reflect this glow.
```html
    <a-entity light="type: point; intensity: 1.6; distance: 5; decay: 2; color: red" position="0.275 -0.32 -3.77"></a-entity>
```
This is a red colored point light, meaning it has a specific position and will decay with distance. I set the decay to 2 and the intensity to 1.6. It is positioned just slightly offset from the bottom of the cauldron so that we get a nice red reflection. I also set the distance to 5 so that only the very closest rocks will get any of the red light.

Here’s what it looks like now. I think we finally have a cool looking scene. It feels like a place where stuff is happening, with secrets to explore.

[image:5C5CC684-2245-4B7B-81C3-3EFA41F7BC13-29782-0003562A757FD861/Firefox NightlyScreenSnapz030.png]


# Shadows
There’s just one more piece of lighting work to do. We need some shadows.  Shadows are expensive computationally speaking, so we only want to turn them on for objects whose shadows we actually care about.  
First we need to enable casting shadows from the light that will create them, the moonlight. Simply add `castShadow:true` to the light attribute.
```html
    <a-entity light="type: directional; color: #ffc18f; intensity: 0.5; castShadow: true; "
    position="31 80 -50"></a-entity>
```

Now add `shadow="receive:true"` to the ground. All of the objects will now automatically cast shadows onto the ground. 
```html
    <!-- the ground --->
    <a-plane color="#52430e"
             static-body
             rotation="-90 0 0" width="100" 
             height="100"  shadow="receive:true"></a-plane>
```
Now it’s starting to feel like a real place.

[image:5213F4D9-E23D-4B8D-BB27-493977282591-30154-000356CE985671C4/Firefox NightlyScreenSnapz031.png]

In an effort to save cycles the shadows will only be cast in an area called the light’s shadow frustum. To see this area set `shadowCameraVisible` to true. [link to aframe docs]

# Audio
Just a few more things to polish up our game.  First some audio.  Lived in worlds aren’t silent. A summer night should have crickets or wind, the bubbling of the cauldron, and of course when we hit the imp it probably will complain. To liven things up I found a few useful sounds at freesound.org.  

First up is the night time sounds of crickets and other creatures.  I found a sound called [name] by free sound user [name] here [link]. Since this is background sounds I don’t want it to be positional. The player should be able to hear it from anywhere, and it should loop over and over.   To make this happen I put the sound on the scene it self using the `sound` attribute. 

```html
<a-scene
...
        sound="src: url(./audio/octobernight2.mp3); loop:true; autoplay:true; volume:0.5;"
>

```
Notice that I set the volume to 50% so it won’t drown out the sound effects.

Next we need a sound for the bubbles in the cauldron. I’m using this sound called [name] by [name] from here [link].

```html
<!-- cauldron -->
<a-entity  gltf-model="#cauldron" ...
   sound="src: url(./audio/boilingwater-loop.mp3); autoplay: true; loop:true;"
></a-entity>
```

Again I have set the audio to loop, but because it’s attached to the cauldron’s entity instead of the scene the sound will appear to come from the cauldron itself.  Positional audio really enhances the immersion of virtual scenes.  Granted, this boiling water effect is *overkill* for the cauldron. In real life the cauldron wouldn’t bubble as quickly or loudly, but we want immersion, not realism.

Finally we need a sound on the imp.
```html
<a-entity id='imp-model' ...
    sound="src: url(./audio/gah.mp3); autoplay: false; loop: false;"
></a-entity>
```

Both autoplay and loop are set to false because we only want the sound to play when the imp is hit with the weapon. Go down to the `collide` event handler and add this line to play the sound on every collision.
```javascript
   $("#imp-model").components.sound.playSound();
```

The original files from freesound.org are in wav format, which is completely uncompressed. If you plan to edit the sound files then this is what you want, but for distribution on the web we want something far smaller. Be sure to convert them to MP3s first, which gives a 90% file size savings. On Mac and Linux you can use the `ffmpeg` tool to convert them like this:

```shell
ffmpeg -i boilingwater-loop.wav boilingwater-loop.mp3
```

# A little more polish

Creating the basic code is the first 90% of building a game. Polishing the experience is the second 90%.   After I first built Whack and Imp I realized it would get boring really fast. The only thing the player can do is wait until the imp jumps out and hit it. It would be  more interesting if every now and then something popped out that the player *shouldn’t* hit.   Let’s add a dragon’s egg.

Inside the ball entity we have a model for the imp. Next to it add another entity called egg-model, this time using a slightly distorted sphere.
``` html
    <!-- the ball contains two models that we swap -->
    <a-entity id='ball'
              position="0 0.1 -4"
              rotation="0 0 0"
              dynamic-body="shape:sphere; sphereRadius: 0.3; mass:4"
    >
        <a-entity id='imp-model' gltf-model="#imp" position="0 -0.4 0"
                  sound="src: url(./audio/gah.mp3); autoplay: false; loop: false;"
        ></a-entity>
        <a-sphere id='egg-model' radius="0.25" segments-height="8" segments-width="8"
                  scale="1 0.6 0.8"
                  material="color: purple; flatShading:true; emissive:red; emissiveIntensity:0.2"
                  sound="src: url(./audio/cowbell.mp3); autoplay: false; loop: false;"
        ></a-sphere>

    </a-entity>
```

To make the sphere look more magical I gave it a purple colored material with flat shading, but also set the emissive color to red. Normally a material  only reflects light that comes from a light source, but  the emissive color lets the material produce it’s own light, even in the dark. In effect it glows.  I also added a cowbell sound for when the player hits the egg. 
Note in the code above that I moved the `dynamic-body` from the `imp` to the surrounding `ball` entity. This is because we want the same physics behavior regardless of which object is hit.  However, the imp model is slightly offset and will stick outside of the sphere bounds, so I adjusted the position by -0.4 in the y direction.

Now we need to update the `resetBall` event handler with a boolean indicating if we should show the imp or the dragon’s egg.

``` javascript
    let showImp = true
    const resetBall = () => {
        clearTimeout(resetId)
        $("#ball").body.position.set(0, 0.6,-4)
        $("#ball").body.velocity.set(0, 5,0)
        $("#ball").body.angularVelocity.set(0, 0,0)
        showImp = (Math.floor(Math.random()*4)!==0)
        $("#imp-model").setAttribute('visible',showImp);
        $("#egg-model").setAttribute('visible',!showImp);
        hit = false
        resetId = setTimeout(resetBall,6000)
    }
```

We also need to make the collide handler play the correct sound and decrement the score if you accidentally hit the egg.

```javascript
    on($("#weapon"),'collide',(e)=>{
        const ball = $("#ball")
        if(e.detail.body.id === ball.body.id && !hit) {
            hit = true
            score = score + 1
            $("#score").setAttribute('text','value','Score '+score)
            if(showImp) {
                $("#imp-model").components.sound.playSound();
                score = score + 1
            } else {
                $("#egg-model").components.sound.playSound();
                score = score - 10
            }
            clearTimeout(resetId)
            resetId = setTimeout(resetBall,2000)
        }
    })
```


As one final detail let’s make the score be white so we can see it in the dark.
[image:76CBB95E-6DB6-4BA2-981B-BA1BB76EEB3E-30154-00035908C5D36949/Firefox NightlyScreenSnapz032.png]

Our game is complete. It’s time to test it out.  We already know it works on the desktop. Here it is on my phone.

[image from FF ios]

Now let’s try it out on a real VR headset

[image from FF on windows / MR]

The image and positional audio works quite well. I definitely have a feeling of being present. However, the interaction feels very awkward because the staff is attached to my head. Instead, I want to use the staff with my actual 6dof controller.  We can do this with an extra element.


* improve the interaction for other devices
	* testing in 3dof
	* testing in 6dof
	* testing on mobile
	* tweaks for desktop

bubble sound: https://freesound.org/people/Euphrosyyn/sounds/370217/
cowbell: https://freesound.org/people/pj1s/sounds/34272/
crickets: https://freesound.org/people/sagetyrtle/sounds/62489/
gah: https://freesound.org/people/metekavruk/sounds/348270/



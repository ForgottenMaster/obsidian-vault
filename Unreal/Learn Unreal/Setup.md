In order to be able to develop with Unreal engine, we first need to get it set up and learn how to navigate the editor before starting to actually make anything in it. This post is just a quick overview of where to install it and basic controls.

---
# Installation #

The Epic games launcher is required to be able to download versions of the Unreal engine which can be found [HERE](https://store.epicgames.com/en-US/download). It should detect your OS and present the appropriate download option for you.

We then need to log in with an Epic games account, or make one with a social login. Once logged into the launcher we want to click on the "Unreal Engine" tab, and then "Library", then finally on the little plus icon next to "Engine Versions" - this should present us with the latest version of the engine and we can click "Install"

![installation](Installation.png)

After installation we can go ahead and launch the editor with the handy button

![launching](Launching.png)

---
# Creating a New Project #

When we launch Unreal editor for the first time, we'll be greeted with the create a project page. Here we can pick from various presets depending on the type of project we're wanting to make, but for getting to grips with the editor, we pick the third-person game option.

The project location and name can be selected, but things should look something like this - then click create

![creating](Creating.png)

---
# Navigating the Viewport #

Once loaded, we're greeted with the following scene in the editor

![scene](Scene.png)

In order to move around the scene, we can hold down the right mouse button, and then moving the mouse will rotate the camera. We can use W, A, S, and D keys to move the camera position around. Both of these in conjunction will allow us to fly around the scene fairly easily.

---
# Playmode #

In order to enter playmode, we can click the green button to enter or use the **ALT + P** shortcut which will put us into the level with the camera behind the character

![playmode](Playmode.png)

When in playmode, we can still rotate the camera with the mouse, but W, A, S, and D will move the character along with the camera. In addition, pressing the space bar will cause them to jump.

Pressing escape will exit playmode and return to the editor again.

---
# Manipulating Actors #

Unreal's terminology for game objects is "actors" which are any individual object in the world. We can click these in either the **outliner** or directly in the viewport in order to select them.

We know if we've selected something because it'll have the gizmos visible to manipulate the transform.

![selecting](Selecting.png)

We can manipulate the position when the translation gizmos are displayed (the directional arrows as in the image above). But we can also select to rotate or scale the actor as well.

We can select the rotate gizmos by using the button at the top as shown

![rotate](Rotate.png)

And scaling with the button shown here

![scale](Scale.png)

Clicking and holding an axis of one of these gizmos, and then dragging in one of the axes will allow translating, rotating, and scaling in that axis.

---
# Quickly Adding an Actor #

We can quickly add a new actor to the scene by using this button in the editor. There are a variety of different types of actor to add, and we can just drag and drop one into the scene to spawn it there.

For example we can add a new cube into the scene

![spawnmenu](Spawn%20Menu.png)

By dragging and dropping the cube into the world, and then using the manipulation gizmos we are able to create platforms for example and allow our character to jump onto them. An example of what can be set up is shown here

![platforms](Platforms.png)

And clicking play allows us to navigate to, and jump onto the platforms with the character as it does with any of the existing pieces of geometry

![jumping onto platforms](Jumping%20Onto%20Platforms.png)
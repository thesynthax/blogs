In this article, I’ll be writing about [my portfolio website](https://thesynthax.space), one that you do not scroll through, but walk through. I’ll discuss the history behind the project, my motivation, the technical journey, and the challenges I faced along the way.

%[https://youtu.be/oxmAx-vxwZw] 

## Background

Since I started programming, I have loved the concept of inculcating creativity, aesthetics, and visualizations in software development. I grew up watching YouTube channels like 3Blue1Brown and Sebastian Lague, who I consider the pioneers of using programming to make math and systems visual, intuitive, and beautiful.

Even in mathematics, concepts like linear algebra and calculus often lose people because they’re taught abstractly. We’re visual learners by nature: we remember what we see, not what we’re told. That realization shaped how I view programming: not just as a technical skill, but as a **creative medium**.

In 2022, while watching videos about insane portfolio websites, I stumbled upon [Bruno Simon's Portfolio](https://bruno-simon.com). It was fascinating and creative. Even though I loved visuals, I didn’t really click with front-end development. I’d rather be a designer than a front-end developer. I wanted to create something similar, but not one of those boring React and Vue websites. I wanted something more tactile, more me*.* Something that blended design, logic, and storytelling, without dealing with HTML, CSS, or JavaScript.

That’s when I thought: why not build it in Unity?

## The Concept

The idea was simple but ambitious: an **interactive 3D environment** that reflects both my creative and technical sides.

The viewer can walk around, explore, and interact with elements to discover my journey. Nature would represent creativity: flowing, spontaneous, free, while the house would represent logic, structure, and engineering.

Clicking the **river** reveals my creative work: music, videos, and experiments. Entering the **house** reveals my technical projects, experience, and skills. It’s a literal metaphor for my thinking: art outside, logic inside.

I went for a **low-poly aesthetic** because it’s clean, expressive, and nostalgic, and fits perfectly with the idea of handcrafted simplicity.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1761570381287/fb82a6c3-68c0-4ffd-9e5e-deacb1e412a2.png align="center")

## The Technicalities

After three years of procrastination, I finally got to work on this. I fired up Unity 6. I created a new URP (Universal Render Pipeline) project that will be built on WebGL. I had some assets that I had collected over the years.

The first few days just went towards building the scene. If I may be honest with you, I didn’t even sketch the environment you eventually see on the website; it was all in my mind. To get started, I even asked ChatGPT to sketch a rough concept of what I had in mind. It was not perfect, but it was enough to visualize the vibe.

Anyway, those four days in which I was building the outside environment, experimenting with the lighting, building the house and the interiors, experimenting with indoor lighting, camera effects, auto-exposure, all of it was extremely tedious. I had basically done both landscape and interior design. Then, I added the trees, rocks, river, and the leaf particle effects.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1761570623423/bd24aac6-4daf-43bd-9812-c127c9b744ea.png align="center")

Once the environment felt right, I started working on the camera logic. I implemented smooth ease-outs for all transitions to make movement feel cinematic. First, I wrote the camera look script, which will react to the mouse movement. Next, I created several waypoints for the camera to move through when the user clicks a specific object. For example, if the user clicks the house, they can go inside it, or if they click the stairs, they can climb up them.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1761570608157/b07c97a2-45f4-4752-914e-aeaa9801a338.png align="center")

Initially, camera movement was linear and robotic, so I added **Centripetal Catmull-Rom Splines** to generate curved trajectories, giving each transition a natural, fluid arc. Next, I added clickable colliders that triggered these movements. I wanted to keep it as intuitive as possible so that no instructions are required to make the user do something.

The next part was the shaders. I wanted the user to feel feedback when hovering over an interactable object. I built a custom shader that gave an outline of the bounding box of the collider that the mouse is pointing at. This was definitely not as straightforward as it sounded. So many messy things were going on with the graphics API. Initially, it only rendered the front side of the outlines; sometimes it did not render at all, and there were Z-flighting issues. When everything worked in the game engine, it did not work in the WebGL build.

Eventually, I figured it out, and we moved to the next most boring part, adding the actual content. I placed several props where my details were mentioned, and the user had to click on them to view. I had many problems with the world-space canvas and UI system, messing with the actual world-space mechanics I had programmed. I had added buttons, which were not working, so I did a very unconventional thing. I added colliders on the buttons and did a physical raycast rather than a UI system raycast, but anyway, everything worked eventually. You know what they say: if it works, don’t touch it.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1761570666912/186ccf4d-bfc8-49c1-8d21-4ccf5c739fad.png align="center")

Now with everything working, I baked the lighting, fixed the player and build settings, and exported the WebGL build. I deployed the project via **GitHub Pages** and pointed my custom domain to it.

And just like that, my website was live. You can experience it here → [thesynthax.space](https://thesynthax.space)

## Conclusion

This was one of my dream projects, and now it exists.

I wanted to show the world that programming can be deeply artistic and expressive, and not just the nerdy, mechanical task most non-tech people imagine it to be. I’ve always been drawn to visuals, geometry, and aesthetics, and I’ve found programming to be one of the most powerful ways to express those ideas: to turn logic into something alive.

Thank you for reading this article. See you in the next one!

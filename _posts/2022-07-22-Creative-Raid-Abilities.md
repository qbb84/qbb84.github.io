---
layout: post
author: qbb84
tags: [Raid, Quadratic Equations, Math, Game Development]
---

If you enjoy MMOs, and visual wizardry, this should be of interest to you.

I had an idea about creating a 'raid' minigame for Minecraft, similar to MMOs like WOW, FFXIV, Destiny 2, and then I thought it'd be even more fun to create raid abilities, as I'd be able to flesh out the abilities much faster, and a lot of people enjoy seeing cool animations.

Here I'm going to discuss one of the abilities I called "Torture", and show early development footage.

## Torture (raid ability)

Torture is a high mana ability due to it's power, use carefully.

This was the initial adage I had in mind when creating, but first some Math.

### Why Quadratic Equations

Quadratic equations are mathematical expressions of the form `ax^2+bx+c=0`, representing parabolic curves. In Minecraft, they can be useful for visual effects by simulating realistic projectile motion, creating aesthetic designs, and adding dynamic elements to abilities or spells. The parabolic nature of quadratic equations suits the trajectory of particles, enhancing the visual experience in the game.

Knowing this, I incorporated a few equations to calculate certain parameters related to trajectory, positions, and timing of particle effects.

### Particle Effects

This can be achieved using NMS (Net Minecraft Server), which is used to access the server code of Minecraft, allowing the sending of packets with bundled data for more control. Here's what a basic packet of data relating to particle effects might look like:

```java
PacketPlayOutWorldParticles packet = new PacketPlayOutWorldParticles(particleType, true,
                particleX, particleY, particleZ, offsetX, offsetY, offsetZ, speed, count);

((CraftPlayer) player).getHandle().playerConnection.sendPacket(packet);
```

### The Design

I initially drew a sketch of the ability, it was going to be used on an entity. When used the blocks around the entity within a radius rise, and then the blocks that rise are chosen pseudo-randomly to fly and cause destruction where the ability was used.

There also contains a circle for the ability location, and a square inside the circle for where most the damage is taken.

This was my original design, and there were iterations which contained a large square which was a region of containment, inside the region there would be totems that rise from the ground to inhibit damage from an explicit range using ray-tracing, this became the design of another ability.

### Early Development Footage

<iframe width="560" height="315" src="https://www.youtube.com/embed/u-O7jM90qtQ?si=b2Vs4xYRW4AixWD2" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

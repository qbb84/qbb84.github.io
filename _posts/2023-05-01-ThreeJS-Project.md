---
layout: post
author: qbb84
tags: [Three.JS, Full-Stack, JavaScript, Typescript, R3F, React, HTML, CSS]
---

ThreeJS-powered website meticulously crafted using Javascript, Typescript, React, and React Three Fiber. As the creator, I delved into the intricate technicalities, making strategic decisions to enhance performance and ensure code elegance.

## Multiple Objects are interactable

At the heart of this project lies a meticulously designed 3D mesh representation of my bedroom, complemented by a creative loading screen that sets the stage for an immersive experience. Engage with the interactive portfolio, where the computer monitor reveals insights about the creator and showcases a dynamic portfolio screen.

## Beyond the Ordinary

- Chessboard Challenge: You are able to immerse yourself in a game of chess on a thoughtfully designed chessboard.
- Mirror Reflection: A mirror that reflects more than just a visual representation.
- First Person Controls: A first-person perspective that puts you in the center of this digital adventure.

<hr>

## Improving Performance

Here will be a glimpse of how I worked to improve performance, and reduce load times.

### How did I know when the performance was declining?

Firstly, it's important to understand that there can be a multitude of different reasons for ill-performance during development, especially game development, and performance issues are usually quite simple to track down.

I used a performance monitor to track the stress of the GPU/CPU of my scene - using this, I was able to easily discover issues I had made. Before I proceed, I'd like to note that the creation and manipulation of 3D objects are managed by the CPU, and the rendering and display of meshes occur on the GPU. Let's proceed.

Here is a basic component, similar to how I loaded my GLTF textures:

```javascript
export function Room(props: JSX.IntrinsicElements['group']) {
  const { nodes, materials } = useGLTF('/Portfolio_room.glb') as GLTFResult;
  const { isVisible } = useContext(VisibilityContext);

  const bedRef = useRef();
  const roomRef = useRef();

  return (
    <>
      <group {...props} dispose={null} ref={roomRef}>
        <group name="Bed" ref={bedRef}>
          <mesh
            name="Bed_Pillows"
            geometry={nodes.Box002_Baked.geometry}
            // material={materials.Objs2_Baked}
            position={[-6.907, 2.052, -0.094]}
            rotation={[-0.898, 0, 0]}
            material={materials.Objs2_Baked}
            userData={{ name: 'Bed_Pillows' }}
          />
      </group>
        </group>
    </>
  );

```

This component was called inside my another component like this:

```javascript
export default function loading() {
  const [isRoomVisible, setRoomVisible] = useState(true);

  useEffect(() => {
    const animationFrameId = requestAnimationFrame(() => {
      setRoomVisible((prevVisible) => !prevVisible);
    });

    return () => cancelAnimationFrame(animationFrameId);
  });

  return <>{isRoomVisible && <Room />}</>;
}
```

The code above isn't correct, but why? Well, because the room is being set visible on every frame, which will cause strenuous load on the GPU and CPU - Imagine every time you walked downstairs, you ended up on top the stairs, after a while you're going to become tired, and as such, the website eventually will become unusable. Let's dive deeper.

## Utilizing Big O Notation

### What is the "Big O"?

Big O notation is a way to describe the efficiency of an algorithm or the performance of a piece of code. It provides a standardized method for expressing the upper bound of the time complexity or space complexity of an algorithm in relation to its input size.

In simpler terms, Big O notation helps us understand how the performance of an algorithm scales as the size of the input (e.g., the number of elements in a list) increases. It's a way to analyze and compare algorithms, enabling us to choose the most efficient solution for a given problem.

Let's say you use a loop to manipulate pixels in an 2D array of a large image like this:

```javascript
for (let i = 0; i < images.length; i++) {
  for (let j = 0; j < images.length; j++) {
    // Simulate manipulation of pixels inside an image
  }
}
```

This loop is running in quadratic time complexity O(n^2). Can we achieve faster performance?

Since our pixels are stored inside a 2D array, if we know the image width and height, we can linearly loop through the image width \* height, and get each pixel inside the array using some arithmetic like so:

```javascript
const imageWidth = 42;
const imageHeight = 21;

for (let i = 0; i < imageWidth * imageHeight; i++) {
  const row = Math.floor(i / imageWidth);
  const col = i % imageWidth;

  // Simulate manipulation of pixels inside an image
  const pixelValue = pixels[row][col];
  // Do something with the pixel value
}
```

It's important to be cognizant of little changes like these which improve the time complexity to linear O(n). This was a more simpler example of how I improved performance of various parts of code inside my codebase.

## Updates

- I expect to release the site this year, and at the time Im writing this post I'm getting close to a beta build.
- The Chess board uses a design similar to bitboard engines, we love bytes!
- Recently implemented working Computer Monitors

Here's a sneak peak of the loading scene (low fps gif):

 <img style="border-radius: 8px;" src="/images/sneakpeak.gif">

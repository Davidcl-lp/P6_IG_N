# Sistema Solar en Three.js

Este proyecto representa un **sistema solar 3D** desarrollado con [Three.js](https://threejs.org/).  
Permite explorar los planetas en dos modos de visualización:

- **Modo Orbital** (vista externa del sistema solar)
- **Modo Nave** (vuelo libre con una nave TIE Fighter)

El usuario puede cambiar entre modos presionando **Enter**, y los controles disponibles se muestran en pantalla.

---

## Tecnologías utilizadas

- **Three.js** – Motor 3D basado en WebGL.
- **GLTFLoader** – Para cargar modelos 3D en formato `.glb`.
- **OrbitControls** – Control orbital de cámara.
- **FlyControls** – Control de vuelo libre (modo nave).
- **JavaScript (ES6)**.

---

## Inicialización del sistema

En el archivo principal, importamos los módulos necesarios de Three.js:

```js
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { FlyControls } from "three/examples/jsm/controls/FlyControls.js";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
```

Luego definimos las variables globales que controlan la escena, cámara, renderizador, controles y parámetros del sistema solar:

```js
let scene, camera, renderer, camcontrols, ship;
let flycontrol = true;
let distance = 3;
let Planets = [];
let Moons = [];
let t0 = Date.now();
let accglobal = 0.001;
const clock = new THREE.Clock();
```

---

## Creación del sistema solar

La lista `SolarSystem` contiene la información de cada planeta, incluyendo su radio, velocidad orbital y distancia media al sol:

```js
let SolarSystem = [
  { name: "sun", rad: 695, orbitSpeed: 0 },
  { name: "mercury", rad: 2.4397 * 2, orbitSpeed: 4.787, perihelio: 46 * 3, afelio: 69.8 * 3 },
  { name: "venus", rad: 6.0518 * 2, orbitSpeed: 3.502, perihelio: 107 * 1.2, afelio: 109 * 1.2 },
  ...
];
```

Cada planeta se crea mediante funciones específicas que generan geometrías esféricas, aplican texturas y añaden órbitas elípticas. Además se ha ajustado cada radio de forma individual de los planetas y el sol además de sus orbitas para que el sistema se pueda ver comodamente en su totalidad por motivos didacticos.

---

## Creación del Sol

```js
function createSun(distance, rad) {
  const texture = new THREE.TextureLoader().load("./planetsTextures/sun.jpg");
  const geometry = new THREE.SphereGeometry(rad, 50, 50);
  const material = new THREE.MeshBasicMaterial({ map: texture });
  const star = new THREE.Mesh(geometry, material);

  const sunLight = new THREE.PointLight(0xffffff, 5);
  sunLight.position.set(0, 0, 0);
  scene.add(star, sunLight);
}
```

El sol funciona como fuente de luz principal (`PointLight`) para iluminar los planetas.

---

## Creación de planetas y lunas

### Planetas
Cada planeta tiene su textura y su órbita elíptica:

```js
function createPlanets(distance, color, rad, speed, f1, f2, texturePath) {
  const texture = new THREE.TextureLoader().load(texturePath);
  const geometry = new THREE.SphereGeometry(rad, 32, 32);
  const material = new THREE.MeshStandardMaterial({ map: texture });
  const planet = new THREE.Mesh(geometry, material);

  planet.userData = { dist: distance, speed, f1, f2 };
  scene.add(planet);
  Planets.push(planet);

  // Dibuja la órbita
  const curve = new THREE.EllipseCurve(0, 0, distance * f1, distance * f2);
  const points = curve.getPoints(100);
  const orbit = new THREE.Line(
    new THREE.BufferGeometry().setFromPoints(points),
    new THREE.LineBasicMaterial({ color: 0xffffff })
  );
  scene.add(orbit);
}
```

### Lunas
Las lunas giran alrededor de sus planetas mediante un pivote (`THREE.Object3D`):

```js
function createMoon(planet, rad, dist, speed, col) {
  const pivot = new THREE.Object3D();
  planet.add(pivot);

  const moon = new THREE.Mesh(
    new THREE.SphereGeometry(rad, 16, 16),
    new THREE.MeshStandardMaterial({ color: col })
  );
  moon.position.x = dist;
  moon.userData = { dist, speed };
  pivot.add(moon);
  Moons.push({ moon, pivot });
}
```

---

## Carga de la nave

Se utiliza `GLTFLoader` para cargar un modelo 3D `.glb` (TIE Fighter):

```js
function loadShip() {
  const loader = new GLTFLoader();
  loader.load("./shipModels/star_wars_tie_fighter.glb", (gltf) => {
    ship = gltf.scene;
    ship.scale.set(0.5, 0.5, 0.5);
    ship.position.set(0, -1, -10);
    camera.add(ship);
    scene.add(camera);
  });
}
```

---

## Modos de cámara: Orbital y Nave

La función `view()` inicializa la cámara según el modo activo (`flycontrol`):

```js
function view() {
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(0, 0, 100);

  if (flycontrol) {
    camcontrols = new FlyControls(camera, renderer.domElement);
    camcontrols.movementSpeed = 100;
  } else {
    camcontrols = new OrbitControls(camera, renderer.domElement);
    camcontrols.enableDamping = true;
  }
}
```

El modo se cambia presionando **Enter**:

```js
function changeView() {
  window.addEventListener("keydown", (event) => {
    if (event.key === "Enter") {
      flycontrol = !flycontrol;
      if (camcontrols.dispose) camcontrols.dispose();
      view();
      updateHUD();
    }
  });
}
```

---

## HUD (Interfaz informativa)

```js
const hud = document.createElement("div");
hud.style.position = "fixed";
hud.style.bottom = "10px";
hud.style.right = "20px";
hud.style.color = "white";
hud.style.backgroundColor = "rgba(0, 0, 0, 0.5)";
hud.style.padding = "8px 12px";
hud.style.borderRadius = "10px";
document.body.appendChild(hud);

function updateHUD() {
  if (flycontrol) {
    hud.innerHTML = `
      <b>Modo Nave</b><br>
      WASD: Moverse<br>
      Q/E: Rotar<br>
      R/F: Subir-Bajar<br>
      <br><i>Presiona Enter para cambiar la vista</i>
    `;
  } else {
    hud.innerHTML = `
      <b>Modo Orbital</b><br>
      Arrastra con el ratón<br>
      Zoom con la rueda<br>
      <br><i>Presiona Enter para cambiar la vista</i>
    `;
  }
}
```

---

## Bucle de animación

```js
function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  const timestamp = (Date.now() - t0) * accglobal;

  for (let planet of Planets) {
    planet.position.x = Math.cos(timestamp * planet.userData.speed) * planet.userData.f1 * planet.userData.dist;
    planet.position.y = Math.sin(timestamp * planet.userData.speed) * planet.userData.f2 * planet.userData.dist;
  }

  for (let item of Moons) item.pivot.rotation.y += item.moon.userData.speed;

  camcontrols.update(delta);
  renderer.render(scene, camera);
}
```

---

## Controles

| Acción | Modo Nave | Modo Orbital |
|--------|------------|--------------|
| **W / A / S / D** | Avanzar / Girar | — |
| **Q / E** | Rotar nave | — |
| **R / F** | Subir / Bajar | — |
| **Ratón** | Mirar alrededor | Rotar cámara |
| **Rueda del ratón** | — | Zoom |
| **Enter** | Cambiar vista | Cambiar vista |

---

## Ejecución

```bash
git clone git@github.com:Davidcl-lp/P6_IG_N.git
cd P6_IG_N
npm install
npx vite
```

Luego abre la dirección local que te indique vite (por ejemplo `http://localhost:5173`).

Además el link al codesandbox es https://codesandbox.io/p/sandbox/practica-6-ky63rk

---

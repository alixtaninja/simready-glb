---
license: cc-by-4.0
pretty_name: SimReady GLB
tags:
- 3d
- gltf
- glb
- web
- three.js
- simready
- nvidia
size_categories:
- 1K<n<10K
configs:
- config_name: default
  data_files:
  - split: train
    path:
    - "metadata.jsonl"
    - "engine/*/*.glb"
---

# SimReady GLB

<video src="https://huggingface.co/datasets/codepawl/simready-glb/resolve/main/assets/preview.mp4" autoplay loop muted playsinline width="100%"></video>

![A grid of SimReady models rendered from this dataset](assets/hero.webp)

**[Open the 3D viewer](https://huggingface.co/spaces/codepawl/simready-glb-viewer)** to search all 2,595 models and
orbit any of them in the browser.

Web-ready glTF binary (`.glb`) copies of the 3D models in
[nvidia/simready-assets](https://huggingface.co/datasets/nvidia/simready-assets). Each file loads directly in
three.js, Babylon.js, model-viewer or any glTF 2.0 viewer.

Models by NVIDIA SimReady, licensed CC BY 4.0. This dataset converts and compresses them. It does not change what
they show. If you use these files, credit NVIDIA as the original author and link the source dataset.

| | Blender export | This dataset |
|---|---|---|
| Assets | 2,595 | 2,595 |
| Total size | 228.1 GB | 3.2 GB |
| Triangles | 821M | 415M |
| Largest texture edge | 4096 px | 1024 px |

Example: the V8 engine (`engineAssembly_v8_engine_a01`) is an 888 MB glb out of Blender and
7.4 MB here. Its first frame in Chrome took 9.4 s before and 0.57 s after.

## Use it in 67js

The files load in [67js](https://github.com/codepawl/67.js), a WebGL renderer, through its glTF addon. The geometry is
meshopt compressed, so pass the meshopt decoder to `loadGltf`.

```bash
npm install 67js@npm:@codepawlhq/67js meshoptimizer
```

```js
import { Renderer67 } from '67js';
import { loadGltf } from '67js/addons/gltf';
import { MeshoptDecoder } from 'meshoptimizer';

const url = 'https://huggingface.co/datasets/codepawl/simready-glb/resolve/main/glb/fireExtinguisher_abc_a03.glb';
const asset = await loadGltf(url, { decoders: { meshopt: MeshoptDecoder } });
const renderer = new Renderer67(document.querySelector('canvas'), asset.parts, { samples: 4 });
renderer.render({ eye: [0, 0.7, 1.8], target: [0, 0.35, 0] });
```

The banner at the top of this card is rendered with 67js from six models of this dataset.

## What is in it

- `glb/<asset>.glb`: one file per SimReady asset folder, named like the folder in the source dataset.
- `thumbs/<asset>.webp`: a 512 px render of each glb, with `thumbs/metadata.jsonl` holding its category, size,
  triangles and draw calls.
- `metadata.jsonl`: the same table with `file_name` set to the `engine/` file, which the Dataset Viewer above opens
  in 3D and `load_dataset("codepawl/simready-glb")` returns as a trimesh scene.
- `manifest.jsonl`: one line per asset with size, triangles, draw calls and textures before and after optimization,
  plus the number of hulls, concave parts and drives.
- `newton_loader/`: the loader for NVIDIA Newton.
- `glb_ktx2/<asset>.glb`: the same models with KTX2 textures for web games and WebGPU, plus `gpu_variants.jsonl`.
- `lods/<asset>/<asset>_lod1.glb` to `_lod3.glb`: simpler levels with smaller KTX2 textures.
- `mobile/<asset>/<asset>.glb`: a quarter of the triangles and KTX2 textures up to 512 px, for phones; it loads like `glb_ktx2/`.
- `engine/<asset>/<asset>.glb`: no meshopt or quantization and JPEG textures, for Unreal, Godot and other
  engine importers.

## Download

All files, about 10.9 GB (3.2 GB of it is `glb/`):

```bash
hf download codepawl/simready-glb --repo-type dataset --local-dir simready-glb
```

One category, for example every food model:

```bash
hf download codepawl/simready-glb --repo-type dataset --include "glb/food_*" --local-dir simready-glb
```

One file from Python:

```python
from huggingface_hub import hf_hub_download

path = hf_hub_download("codepawl/simready-glb", "glb/engineAssembly_v8_engine_a01.glb", repo_type="dataset")
```

## Loading in three.js

The geometry uses `EXT_meshopt_compression` and the textures use `EXT_texture_webp`, so set the meshopt decoder:

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';

const loader = new GLTFLoader();
loader.setMeshoptDecoder(MeshoptDecoder);
loader.load('engineAssembly_v8_engine_a01.glb', (gltf) => scene.add(gltf.scene));
```

## Physics and moving parts

2,587 files carry the rigid bodies, mass and physics materials of their SimReady USD. 929 of
them also carry joints, 5,550 in all: door hinges, drawer slides, buttons, wheels and lids. Each file holds
this data in two forms:

- `KHR_physics_rigid_bodies`, the Khronos glTF physics extension. Each part has a `motion` with its mass, center of
  mass and inertia. Each mesh has a convex hull `collider` with friction and restitution. Each joint has its limits.
  Loaders that do not read the extension ignore it.
- Plain node extras, which three.js puts in `object.userData`. A moving part has
  `joints: [{ type, parent, pivot, axis, min, max, unit }]`, with the pivot and axis in model space at rest.
  Revolute limits are in degrees and prismatic limits are in meters. Parts also have `mass` in kg and `density` in
  kg/m³.

Open a door or a drawer in three.js from its rest pose (the part's parent is at the model origin):

```js
import * as THREE from 'three';

function movePart(part, value) {
  const { type, pivot, axis } = part.userData.joints[0];
  const direction = new THREE.Vector3(...axis);
  if (type === 'prismatic') {
    part.position.addScaledVector(direction, value);
    return;
  }
  const turn = new THREE.Quaternion().setFromAxisAngle(direction, THREE.MathUtils.degToRad(value));
  const center = new THREE.Vector3(...pivot);
  part.position.sub(center).applyQuaternion(turn).add(center);
  part.quaternion.premultiply(turn);
}

movePart(gltf.scene.getObjectByName('door_01_obj_00'), 90);
```

The 3D viewer shows a slider for each moving part. The files from before the physics data are kept at the `v1` tag:
add `--revision v1` to the download command.

## Simulating in NVIDIA Newton

Each file with physics data also carries the data that a simulator needs and a viewer does not:

- 17,507 collision hulls in `extras.collisionHulls` of each body node. 2,852 of the 8,496 parts are concave.
  A concave part gets up to 8 hulls, so a cabinet or a bin stays hollow. A hull has at most 48 points, or 32 in a concave part.
- 2,145 joint drives in 333 files, in the `drives` array of the joint in `KHR_physics_rigid_bodies`.
- Inertia of 256 small parts in 67 files, which the earlier rounding had set to zero.

These additions make the set 0.17% larger: 3,180.2 MB to 3,185.5 MB, with a median file of 0.416 MB. Meshes, materials and
textures are byte-identical to the earlier files in all 2,595 files. The hulls sit after the old binary data, so
existing loaders read the files as before.

Half of the files have hulls within 3.9 mm of the part's own convex shape. The figure is 12 mm at the 95th percentile
and 177 mm in the worst file. A hull side that is nearly flat along an axis is then made exactly flat, which moves its
points outward by up to 2 cm. Newton's contact code turns an uneven bottom into false depth, which kicks the part upward.

`newton_loader/simready_newton` adds a file to a Newton model. It needs Newton 1.7, SciPy and meshoptimizer.

```python
import newton
import newton.solvers
from simready_newton import add_simready_glb

builder = newton.ModelBuilder()  # Z-up by default; the loader stands each Y-up file upright for any up axis
builder.add_ground_plane()
add_simready_glb(builder, "glb/bin_waste_a01.glb", position=(0.0, 0.0, 0.5))
model = builder.finalize()
solver = newton.solvers.SolverXPBD(model, iterations=8, joint_linear_relaxation=0.3, joint_angular_relaxation=0.2)
```

Use these two relaxation values with `SolverXPBD`. The default linear value of 0.7 went NaN on
`scaffold_rolling_steel_a03` within a few frames. The default angular value of 0.4 went NaN on
`climate_shopFan_pedestal_a01` when every joint was pushed at once. The loader read all 2,595 files with no failures.
8 files have no rigid bodies, so it adds nothing for them.

In a file with joints, the loader raises any part's inertia below 1e-4 kg·m² to that value. A part's rotation is slower
than the source says, and it no longer shakes or goes NaN. Set `jointed_part_inertia_floor` to change the value. Files
with one rigid body use 1e-6 kg·m² (`inertia_floor`).

A part with a mass but no inertia in the file gets its inertia from its hulls, scaled to that mass. This applies to
2,026 parts. As in Newton's own USD importer, the file's center of mass replaces the hulls' center without shifting the
inertia.

Each part starts in the pose the file shows, in every solver. Some source joints hold a gap across a locked axis, from
1 mm to 1.8 m (a bottle cap 21 cm above its bottle) or up to 37° (a forklift's brake button), which would pull the part
out of place. The loader closes these gaps on the first joint frame. A free axis keeps its rest offset as the joint's
starting coordinate in `builder.joint_q`, so MuJoCo and Featherstone, which place parts from the coordinates, start
there too. The forklift's seat headrest rests 13.5° outside its own limits, so the limit pushes it back when the
simulation starts.

The loader reads only the file's JSON and the hull bytes, never the textures, so most files load in a few milliseconds.

43 files across the categories were simulated at 60 fps with 8 substeps. 42 of them were compared with their SimReady USD:

| Check | Result |
| --- | --- |
| Mass, center of mass, joint type, pivot, axis, limits, friction, restitution | Match |
| Dropped from 20 cm stays finite | 43 of 43 |
| Dropped from 20 cm is at rest after 3 s | 31 of 43 |
| Dropped from 20 cm is at rest after 9 s (the 12 files left) | 10 of 12 |
| Every joint pushed at once on a pinned base stays finite | 43 of 43 |

At rest means under 0.05 m/s and 0.5 rad/s. The push test stops pushing a free axle above 200 rad/s. `conveyor_belt_a61`
stayed finite at 400 rad/s and went NaN at 800 rad/s, so keep free axles below that at 480 substeps per second.

## For web games and WebGPU: KTX2 and LODs

A WebP texture is decoded to full RGBA on the GPU, and most models here carry several 1024 px maps. `glb_ktx2/` has
the same 2,595 models with KTX2 textures (`KHR_texture_basisu`, full mip chains), which stay block compressed in
video memory. Base colour and emissive maps are ETC1S in sRGB, and metallic-roughness and occlusion maps are ETC1S in
linear. Normal maps are UASTC with RDO, because ETC1S smears normals. `lods/<asset>/` adds three simpler levels of each model
as separate files, because neither three.js nor 67js reads `MSFT_lod`.

Every level keeps the node names, the joint extras and the `KHR_physics_rigid_bodies` data of the file in `glb/`. A
collider points at its mesh node, so on a LOD the convex hull is built from the simpler mesh. The files in `glb/` did
not change.

| Level | File | Triangles kept | Texture edge | VRAM, median of 20 | Set size |
|---|---|---|---|---|---|
| WebP set | `glb/<asset>.glb` | 100% | up to 1024 px, WebP | 33.6 MiB | 3.19 GB |
| lod0 | `glb_ktx2/<asset>.glb` | 100% | up to 1024 px, KTX2 | 9.3 MiB | 4.30 GB |
| lod1 | `lods/<asset>/<asset>_lod1.glb` | 50% | half | 2.5 MiB | 2.02 GB |
| lod2 | `lods/<asset>/<asset>_lod2.glb` | 20% | a quarter | 0.69 MiB | 0.91 GB |
| lod3 | `lods/<asset>/<asset>_lod3.glb` | 6% | an eighth | 0.19 MiB | 0.43 GB |

Load time for one model, median of 20 models with 5 loads each:

| | WebP set | lod0 | lod1 | lod2 | lod3 |
|---|---|---|---|---|---|
| three.js WebGPURenderer, load | 25.2 ms | 28.7 ms | 12.0 ms | 7.2 ms | 5.5 ms |
| three.js WebGPURenderer, load and first frame | 42.5 ms | 40.4 ms | 20.2 ms | 13.2 ms | 11.1 ms |
| three.js WebGLRenderer, load | 26.3 ms | 30.0 ms | 12.2 ms | 7.2 ms | 5.3 ms |
| three.js WebGLRenderer, load and first frame | 59.0 ms | 51.3 ms | 22.7 ms | 16.2 ms | 13.5 ms |
| 67js, load | 69.6 ms | 86.5 ms | 39.1 ms | 22.5 ms | 16.8 ms |

lod0 loads about 3 ms slower than the WebP file, because each model transcodes several textures, and reaches the
first frame a little sooner. Its gain is video memory: 0.28 times the WebP set. The sample is 20 models spread over
the set by file size.

Measured on an RTX 3060 in headless Chrome (ANGLE D3D11), three.js r187 and 67js 0.15.0, files
from localhost. Every file loaded with no error in all three, and the textures transcoded to BC7. three.js drew all
20 models. 67js drew 17 and left three small, thin models blank, the WebP files too
([67.js#468](https://github.com/codepawl/67.js/issues/468)). Phones (ASTC or ETC2) and Safari were not measured.

three.js with WebGPU:

```js
import { WebGPURenderer } from 'three/webgpu';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';

const BASE = 'https://huggingface.co/datasets/codepawl/simready-glb/resolve/main';
const renderer = new WebGPURenderer({ antialias: true });
await renderer.init(); // call before detectSupport, so the loader knows the device's formats

const ktx2Loader = new KTX2Loader()
  .setTranscoderPath('https://cdn.jsdelivr.net/npm/three@0.187.0/examples/jsm/libs/basis/')
  .detectSupport(renderer);
const loader = new GLTFLoader().setKTX2Loader(ktx2Loader).setMeshoptDecoder(MeshoptDecoder);
const gltf = await loader.loadAsync(`${BASE}/glb_ktx2/handTool_wrench_torque_a01.glb`);
```

67js does not bundle the decoders, so pass the three.js KTX2 loader and meshopt decoder:

```js
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';
import { loadGltf, ktx2WorkerConfigFor } from '67js/addons/gltf';

const BASE = 'https://huggingface.co/datasets/codepawl/simready-glb/resolve/main';
const context = document.createElement('canvas').getContext('webgl2'); // with a Renderer67, use renderer.context
const ktx2 = new KTX2Loader().setTranscoderPath('https://cdn.jsdelivr.net/npm/three@0.187.0/examples/jsm/libs/basis/');
ktx2.workerConfig = ktx2WorkerConfigFor(context); // without it the loader throws

const asset = await loadGltf(`${BASE}/glb_ktx2/handTool_wrench_torque_a01.glb`, { decoders: { ktx2, meshopt: MeshoptDecoder } });
ktx2.dispose();
```

## For phones and game engines: mobile and engine

| Tier | File | Triangles kept | Textures | Median file | Set size |
|---|---|---|---|---|---|
| mobile | `mobile/<asset>/<asset>.glb` | 25% | up to 512 px, KTX2 | 0.18 MiB | 1.25 GB |
| engine | `engine/<asset>/<asset>.glb` | 100% | up to 1024 px, JPEG | 1.18 MiB | 14.6 GB |

The files in `glb/` and `glb_ktx2/` use `EXT_meshopt_compression`, which Unreal Engine 5.8's glTF importer does not
read ("Buffer from BIN chunk is too small"). The engine tier has plain float geometry and JPEG textures, so it
imports as is: 8 of 8 tested files imported into Unreal 5.8 with every mesh, texture and material of the file, and
Godot 4.7.2 imported a sample of 24. A spread sample of 200 assets loaded and drew in three.js WebGL,
WebGPURenderer and 67js in both tiers, and their engine files opened in Blender. Unity was not tested.

Both tiers keep the node names, the joint extras, `collisionHulls` and `KHR_physics_rigid_bodies` of the file in
`glb/`. In the Newton loader all 2,595 files of both tiers give the same masses, inertias and joint limits as `glb/`,
and body poses within 0.05 mm. Unreal and Godot do not read
`KHR_physics_rigid_bodies`, so in those engines the physics has to come from the node extras or be set up again.

## How the files were made

1. Download `simready_usd/` from each asset folder. `source_assets/` is skipped because it holds a second copy of
   the textures for MDL shaders.
2. Check that `.metadata/LICENSE.txt` is CC BY 4.0.
3. Import the USD in Blender 5.2 and remove the collision, proxy and grasp helper prims. Export a glb with the
   original textures.
4. Optimize with glTF Transform 4.5. Remove the USD instance prototypes Blender brings in next to the placed parts,
   remove duplicate mesh instances, then dedup, prune and weld.
5. Resize textures to 1024 px as WebP. Compress geometry with meshopt at its high level
   (`EXT_meshopt_compression` with octahedral normals and quantized attributes).
6. Read the rigid bodies, mass, physics materials and joints from the USD layers and write them into the glb JSON.
   The USD is Z-up, so joint frames and axes are turned to the glb's Y-up.
7. Cut each concave part into convex pieces and keep the hull points that stick out most. Make nearly flat hull sides
   exactly flat. Read the drives and the full inertia from the USD. Append the hulls after the binary data.

Assets with UDIM textures have each tile split into its own material before the export, with its UVs moved into 0-1.
Blender's own UDIM handling in the glTF exporter drops the faces of some tiles.

No mesh simplification is applied. The triangle drop comes from removed duplicates and helper geometry.

The conversion scripts are in
[codepawl/3d-builds](https://github.com/codepawl/3d-builds/tree/main/works/2026-10-28-simready-showcase/batch).

## Known gaps

SimReady collides with signed distance fields built from each render mesh. The `collider` of each mesh in
`KHR_physics_rigid_bodies` stays one convex hull, which every engine can build, so hollow parts collide as if filled
there. Use `extras.collisionHulls` to keep them hollow. `KHR_physics_rigid_bodies` is still a Khronos review draft, so
check that your engine reads it. The node extras work with any loader. USD simulation settings are not carried
over. Materials are converted to glTF PBR, so MDL-only shader effects are lost.

In Newton, parts of one file do not collide with each other, because SimReady's overlapping parts jam their hinges
otherwise. Nothing latches a door or a drawer, so use a drive to hold them closed. Two sample files do not stay still:

- `electrical_power_terminal_a01` is 5.1 m tall and 0.52 m deep, and it falls over when it stands free. Load it with
  `fixed_base=True`.
- `paint_supplies_brush_a05`, a 44 g brush, still rocks after 9 s. With `rigid_contact_relaxation=0.2` it settles in 9 s.

`structure_wall_entrance_a01` goes NaN at 8 substeps, even with a fixed base: its 184,601 kg wall is welded to 21 kg
barriers. It stays finite at 16 substeps.

Inertia differs from the USD where the loader raises it to its floor. Only 43 of the 2,595 files had these simulation
checks, so other files can have the same problems.

Two joints in the USD are left out because they hold nothing: the fixed joint of
`ladder_multiPurpose_aluminium_a05` points at a part with no rigid body, and one fixed joint of
`serviceTool_brakeCaliper_pistonWindBackKit_a01` joins a body to itself.

## Changelog

- 2026-11-06: Added `mobile/` (a quarter of the triangles, 512 px KTX2 textures, 1.25 GB) and `engine/` (no
  compression extensions, JPEG textures, 14.6 GB), which imports into Unreal 5.8 without decoding first. Both
  keep the physics of `glb/`. The files in `glb/`, `glb_ktx2/` and `lods/` do not change.
- 2026-11-03: The loader now stands files upright in a builder of any up axis, so Newton's default Z-up builder works.
  Parts start in their rest pose: joint gaps across locked axes are closed and a free axis's rest offset becomes the
  joint's starting coordinate. Before, 36 files moved a part out of place at q=0, by up to 1.86 m or 37°; now none do.
  Two more of 64 tested files come to rest after a drop, and no file got worse. Loading reads only the JSON and the
  hulls: all 2,595 files load in 23 s instead of 118 s (median 3.5 ms instead of 30 ms). The GLB files do not change.
- 2026-11-03: Fixed the loader's inertia for parts with no inertia in the file. qhull does not orient its hull
  triangles, so Newton's signed-volume integral cancelled most of the volume and the median part was 63 times off on
  one axis (reported by jayscambler in discussion #2). Triangles of these hulls now face outward; collision-only hulls
  are unchanged. These parts now also get the inertia floor, and flat parts with no hull volume (29 buttons and
  hinges) keep the file's mass instead of ending with zero mass and inertia. `checks.movable_joints` no longer recurses forever on the loop-closing joints of
  `instrument_adams_pliers_b01`. The GLB files do not change.
- 2026-11-01: Added collision hulls that keep concave parts hollow, joint drives and the inertia of small parts.
  Added a loader for NVIDIA Newton. The set grows by 0.17%. Meshes, materials and textures do not change.
- 2026-10-29: Added physics from the USD files: rigid bodies, mass, physics materials and 5,550 joints, as
  `KHR_physics_rigid_bodies` and as node extras. The viewer shows a slider for each moving part. The files before this
  change are at the `v1` tag.
- 2026-10-29: Geometry is encoded at meshopt's high level. The set went from 4.0 GB to 3.2 GB with the same
  triangles and no visible change in any of the 2,595 thumbnails.
- 2026-10-29: 52 files with UDIM textures were missing geometry, because Blender's glTF exporter dropped
  the faces of some texture tiles (for example crate_plywood_b01 kept only its metal frame). They are exported again
  with the tiles split before the export.
- 2026-10-29: 1,305 files carried a second copy of some parts at the origin, the USD instance prototypes
  Blender imports next to the placed parts (for example the eyeglasses showed two pairs of arms). Those copies are
  removed. Added thumbnails, the Dataset Viewer table and the 3D viewer Space.
- 2026-10-28: first release.

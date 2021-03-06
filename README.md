# Global Illumination Renderer
This repository contains a C++ program that uses multithreaded raytracing and photon mapping to render photorealistic images of 3D environments. The photon mapping technique was described in Henrik Wann Jensen's paper, ["Global Illumination using Photon Maps"][1], and implementation decisions were heavily influenced by [Jensen's SIGGRAPH course notes][2].

## Table of Contents
- [Global Illumination Renderer](#global-illumination-renderer)
  * [Table of Contents](#table-of-contents)
  * [Getting Started](#getting-started)
    + [A Note on Portability](#a-note-on-portability)
    + [Installing](#installing)
    + [Running the Program](#running-the-program)
  * [Program Input](#program-input)
    + [Provided Scenes](#provided-scenes)
    + [Adding a Custom Scene](#adding-a-custom-scene)
- [Implementation Details](#implementation-details)
  * [BRDF Sampling & Lighting](#brdf-sampling--lighting)
    + [The BRDF Function](#the-brdf-function)
    + [Importance Sampling the BRDF](#importance-sampling-the-brdf)
    + [2D Lights](#2d-lights)
  * [Direct Illumination](#direct-illumination)
    + [Shadows](#shadows)
    + [Soft Shadows](#soft-shadows)
  * [Reflection & Transmission](#reflection--transmission)
    + [Perfect Reflection](#perfect-reflection)
    + [Perfect Transmission](#perfect-transmission)
    + [Monte Carlo Path-Tracing](#monte-carlo-path-tracing)
  * [Photon Mapping](#photon-mapping)
    + [Photon Emission](#photon-emission)
    + [Photon Storage](#photon-storage)
    + [Radiance Sampling](#radiance-sampling)
  * [Global Illumination](#global-illumination)
    + [Caustics](#caustics)
    + [Indirect Illumination](#indirect-illumination)
  * [Additional Features](#additional-features)
    + [Pixel Integration (Anti-Aliasing)](#pixel-integration-anti-aliasing)
    + [Progress Bar & Statistics](#progress-bar--statistics)
    + [Optimized KdTree Implementation](#optimized-kdtree-implementation)
    + [Multithreading](#multithreading)
  * [Implementation Extensions](#further-implementation-extensions)
    + [Filtering](#filtering)
    + [Irradiance Caching](#irradiance-caching)
    + [Depth of Field](#depth-of-field)
- [Art Gallery](#art-gallery)
  * [The Rainbow Room](#the-rainbow-room)
  * [The Crystal Teapot](#the-crystal-teapot)
  * [Life out of Focus](#life-out-of-focus)
- [Future Features](#future-features)
    
- [Credits](#credits)
  * [Authors](#authors)
  * [References](#references)
  * [License](#license)
  * [Acknowledgments](#acknowledgments)

## Getting Started
Follow these instructions in order to run this program on your local machine.

### A Note on Portability
This program was only tested on Mac OS Mojave 10.14.2. Because C++ multithreading is not especially portable, it is possible that `photonmap` will not compile on Linux or Windows operating systems. If so, the author suggests replacing all instances of `__thread` with `thread_local`. If this does not work, next try removing the code that splits execution among child threads in both `src/render.cpp` and `src/photonmap.cpp` as well as any keywords that the compiler does not recognize (leaving a single threaded program).

### Installing
Change the directory to the root folder of the program and enter `make all` into the command line. If the program fails to compile, try changing the directory to `/src/` and enter `make clean && make all`.

### Running the Program
Once `photonmap` has been compiled, run it in the command line using the following arguments:

```
$ ./photonmap src.scn output.png [-FLAGS]
```

There are also provided shortcuts in the Makefile to expedite the scene-rendering process (as the number of flags can grow rather lengthy).

Here is a breakdown of the meaning of the arguments, as well as the avaliable flags:
* src.scn => file path to input scene image (required)
* output.png => file path to output (required)
* General flag arguments:
  * `-resolution <int X> <int Y>` => Sets output image dimensions to X by Y. Default is `X=1024 Y=1024`
  * `-v` => Enables verbose output, which prints rendering statistics to the screen. Off by default
  * `-threads <int N>` => Sets the number of threads (including main thread) used to trace photons and render the image. Default is `N=1`
  * `-aa <int N>` => Sets how many times the dimensions of the image should be doubled before downsampling (as a form of anti-aliasing) to the output image. To be more precise, there `4^N` rays sampled over an evenly-weighted grid per output pixel. Default is `N=2`
  * `-real` => Normalize the components of all materials in the scene such that they conserve energy. Off by default
  * `-no_fresnel` => Disables splitting transmissision into specular and refractive components based on angle of incident ray. Fresnel is enabled by default
  * `-ir <float N>` => Sets the refractive index of air. Default is `N=1.0`
Illumination flags:
  * `-no_ambient` => Disables ambient lighting in scene. Ambient lighting is enabled by default
  * `-no_direct` => Disables direct illumination (immediate illumination of surfaces by a light source) in scene. Direct illumination is enabled by default
  * `-no_transmissive` => Disables transmissive illumination (light carried by refraction through optical media) in scene. Transmissive illumination is enabled by default
  * `-no_specular` => Disables specular illumination (light bouncing off of surface) in scene. Specular illumination is enabled by default
  * `-no_indirect` => Disables indirect illumination (light that bounces diffusely off of at least one surface — this is the primary global illumination component) in scene. Indirect illumination is enabled by default
  * `-no_caustic` => Disables caustic illumination (light that is focused through mirror and optical media) in scene. Caustic illumination is enabled by default
  * `-photon_viz` => Enables direct radiance sampling of the global photon map for vizualization. This layer will (nearly) approach global illumination on its own if given large enough samples. Disabled by default
  * `-fast_global` => Enables a faster estimate of global illumination by combining direct lighting with direct radiance sampling of a version of the global photon map where photons are only stored after their first diffuse bounce. Disabled by default
  * `-cache` => Enables irradiance caching for the global map, which cuts accuracy for fast indirect illumination calculations without much noise. Irradiance caching is disabled by default.
* Monte Carlo flags:
  * `-no-monte` => Disables Monte Carlo path-tracing (used to compute specular and transmissive illumination). Monte Carlo is path-tracing is enabled by default
  * `-md <int N>` => Sets the max recursion depth of a Monte Carlo path-trace. Default is `N=128`
  * `-absorb <float N>` => Sets probability of a surface absorbing a photon. Default is `N=0.005` (0.5%)
  * `-no_rs` => Disables recursive shadows (shadow sampling from within a specular or transmissive raytrace). Recursive shadows are enabled by default
  * `-no_dt` => Disables distributed importance sampling of transmissive rays based on material shininess. For materials with low shininess but high transmision values, this creates a "frosted glass" effect. Distributed transmissive ray sampling is enabled by default
  * `-tt <int N>` => Sets the number of test rays that should be sent when sampling a transmissive surface. Default is `N=128`
  * `-no_ds` => Disables distributed importance sampling of specular rays based on material shininess. For materials with low shininess but high specular values, this creates a "glossy surface" effect. Distributed specualar ray sampling is enabled by default
* Photon Mapping flags:
  * `-global <int N>` => Sets the approximate number of photons that should be stored in the global map. Default is `N=2176`
  * `-caustic <int N>` => Sets the approximate number of photons that should be stored in the caustic map. Default is `N=10000000`
  * `-md <int N>` => Sets the max recursion depth of a Photon trace in the photon mapping step. Default is `N=128`
  * `-it <int N>` => Sets the number of test rays that should be sent when sampling the indirect illumination of a surface. Default is `N=256`
  * `-gs <int N>` => Sets the number of photons used in a radiance sample of the global photon map. Default is `N=50`
  * `-gd <float N>` => Sets the max radius of a radiance sample of the global photon map. Default is `N=2.5`
  * `-gf <"cone <float k>" | "gauss">` => Sets the filtering mechanism for the global photon map. The standard projected-sphere sample is used by default.
  * `-cs <int N>` => Sets the number of photons used in a radiance sample of the caustic photon map. Default is `N=225`
  * `-cd <float N>` => Sets the max radius of a radiance sample of the caustic photon map. Default is `N=0.225`
  * `-cf <"cone <float k>" | "gauss">` => Sets the filtering mechanism for the caustic photon map. The standard projected-sphere sample is used by default.
* Shadow Sampling flags:
  * `-no_shadow` => Disables shadows entirely. Shadows are enabled by default
  * `-no_ss` => Disables soft shadows. Soft shadows are enabled by default
  * `-lt <int N>` => Sets the number of occlusion + reflectance rays sent per light per sample. Used to compute both soft shadows and direct illumination by area light. Default is `N=128`
  * `-s <int N>` => Sets the number of occlusion (only) rays sent per light per sample. Used to take additional soft shadow estimates (on top of the number specified by the `-lt` flag). Default is `N=128`
* Depth of Field flag:
  * `-dof <int N> <float D> <float R>` => Enables depth of field for a camera with aperture radius `R` and focused on a plane at distance `D` from itself. `N` samples are sent through the aperture to approximate lense scattering. Depth of field is disabled by default.

## Program Input
### Provided Scenes
Sample scenes for testing out the program can be found in the `input/` folder. To ease commandline headaches, there are also provided `make` rules for almost all of these scenes in the Makefile that lives in the project's root directory.

### Adding a Custom Scene
It is easy to provide the rendering program with a scene of your own design. Currently, this program is only able to read a scenes in a very simple (custom) format. Documentation on the syntax of this format may be found [here](https://www.cs.princeton.edu/courses/archive/fall18/cos526/a3/scnformat.html).

Note that textures are not yet implemented in the program.

# Implementation Details
This section contains descriptions and examples of the rendering program's various features. The feature visualizations below (as opposed to renderings) were made with the provided `visualize` program.

## BRDF Sampling & Lighting
### The BRDF Function
This program uses two BRDFs depending on the sampling context. When sampling radiance (evaulating the integral directly), the original Phong BRDF is used in order to remain consistent with both the reflectance functions for the provided light classes as well as the provided input scenes. Since the Phong BRDF is not energy-conserving, it is necessary to use a normalized version of the Phong BRDF when importance sampling. This physically-based Phong model is borrowed from course notes by Jason Lawrence in ["Importance Sampling of the Phong Reflectance Model"][3], which in turn is sourced from Lafortune & William's 1994 paper, ["Using the modified Phong reflectance model for physically based rendering"][4].

### Importance Sampling the BRDF
In order to converge more quickly on the correct solution to the rendering equation, it is necessary to importance sample the BRDF when tracing a ray through a specular or diffuse bounce. In other words, rather than sampling all directions and weighting them according to the probability of a bounce heading in each direction, it is better to sample each direction at a frequency proportional to its probability, and then weight all bounces evenly when averaging.

#### Diffuse Importance Sampling
Under our BRDF model, the outgoing direction of a diffuse bounce is independent of the incident angle of the incoming ray (beyond determining the side of the surface off of which to bounce). Rather, its pdf is determined by a normalized cosine-weighted hemisphere along the surface normal. Borrowing the inverse mapping provided by Lawrence, the direction of the outgoing ray is sampled in spherical coordinates as `(θ, φ) = (arccos(sqrt(u)), 2πv)`, where `(u, v)` are uniformly-distributed random variables in the range `[0, 1)`, `θ` is the angle between the outgoing ray and the surface normal, and `φ` is the angle around the plane perpendicular to the surface normal.

##### Figure 1: Diffuse importance sampling of 500 rays at three angles and two viewpoints.
|| Viewpoint A | Viewpoint B |
|:----------------:|:----------------:|:----------------:|
| 90° | ![Fig 1a.i](/gallery/figures/fig_1a-i.png?raw=true) | ![Fig 1a.ii](/gallery/figures/fig_1a-ii.png?raw=true) |
| 45° | ![Fig 1b.i](/gallery/figures/fig_1b-i.png?raw=true) | ![Fig 1b.ii](/gallery/figures/fig_1b-ii.png?raw=true) |
| 5° | ![Fig 1c.i](/gallery/figures/fig_1c-i.png?raw=true) | ![Fig 1c.ii](/gallery/figures/fig_1c-ii.png?raw=true) |

#### Specular Importance Sampling
For specular importance sampling, the outgoing direction is sampled as a perturbance from the direction of perfect reflection of the incident ray. Again referencing Lawrence's note, we initially sample this direction as `(α, φ) = (arccos(pow(u,1/(n+1))), 2πv)`, where `(u, v)` are uniformly-distributed random variables in the range `[0, 1)`, `α` is the angle between the outgoing ray and the direction of perfect reflection, and `φ` is the angle around the plane perpendicular to the direction of perfect reflection. Finally, although this is not mentioned in the notes, in order to ensure the sampled outgoing ray is on the same side of the surface as the incoming ray, it is necessary to scale alpha from the range `[0, pi/2)` to `[0, θ)`, where `θ` is the angle between the direction of perfect reflection and the plane of the surface. Note that this rescaling is not a perfectly accurate model and somewhat inconsistent with our BRDF (rejection sampling would be a more accurate approach, but significantly more inefficient and not worth the cost), but it is still has a physical basis since glossy reflections become significantly sharper at increasingly grazing angles.

##### Figure 2: Specular importance sampling of 500 rays at three angles and two viewpoints for two materials with shininess of n = 100 and n = 1000 respectively.
|| Viewpoint A, n = 100| Viewpoint B, n = 100 | Viewpoint A, n = 1000 | Viewpoint B, n = 1000 |
|:----------------:|:----------------:|:----------------:|:----------------:|:----------------:|
| 90° | ![Fig 2a.i](/gallery/figures/fig_2a-i.png?raw=true) | ![Fig 2a.ii](/gallery/figures/fig_2a-ii.png?raw=true) | ![Fig 2a.iii](/gallery/figures/fig_2a-iii.png?raw=true) | ![Fig 2a.iv](/gallery/figures/fig_2a-iv.png?raw=true) |
| 45° | ![Fig 2b.i](/gallery/figures/fig_2b-i.png?raw=true) | ![Fig 2b.ii](/gallery/figures/fig_2b-ii.png?raw=true) | ![Fig 2b.iii](/gallery/figures/fig_2b-iii.png?raw=true) | ![Fig 2b.iv](/gallery/figures/fig_2b-iv.png?raw=true) |
| 5° | ![Fig 2c.i](/gallery/figures/fig_2c-i.png?raw=true) | ![Fig 2c.ii](/gallery/figures/fig_2c-ii.png?raw=true) | ![Fig 2c.iii](/gallery/figures/fig_2c-iii.png?raw=true) | ![Fig 2c.iv](/gallery/figures/fig_2c-iv.png?raw=true) |

### 2D Lights
The only physically-plausible light in the R3Graphics codebase is a circular area light. Because all example scenes in Jensen's Photon Mapping paper use a rectangular area light, the we added an R3RectLight class. Additionally, the original R3AreaLight class required modifications to its reflectance function implementation in order to more accurately reflect how light is emitted through diffuse surfaces.

#### R3RectLight
The syntax of the a rectanglar light object is as follows:
```
rect_light r g b    px py pz    a1x a1y a1z    a2x a2y a2z    len1 len2    ca la qa
```
This defines a rectangular light with radiance `r g b` (in Watts/sr/m^2) centered at `px py pz`. The surface of the light is defined by axes `a1` and `a2`, with lengths `len1` and `len2` respectively. Note that if `a1` and `a2` are not perpendicular, the light will be a parallelogram. Light is only emitted in the `a1 x a2` direction, and `ca la qa` define the light's attenuation constants.

##### Figure 3: A comparison of area lights. In Figure (3a) we see a Cornell Box illuminated by a circular area light. In Figure (3b) we see a Cornell Box illuminated by a rectangular area light.
| Circular Area Light | Rectangular Area Light |
|:----------------:|:----------------:|
| ![Fig 3a](/gallery/figures/fig_3a.png?raw=true) | ![Fig 3b](/gallery/figures/fig_3b.png?raw=true) |

#### Weighted Area Light Reflectance
Since area lights emit light diffusely — that is, according to the distribution of a cosine-weighted hemisphere — it was necessary to modify the area light reflectance implementation provided in R3AreaLight. When computing the illumination of a surface due to an area light (circular or rectangular), the intensity of the illumination doubled and then scaled by the cosine of the angle between the light normal and the vector spanning from the light to the surface. The doubling is necessary since we want to keep the power of the light consistent with the original implementation (the flux through an evenly-weighted hemisphere is 2π, whereas the flux through a cosine-weighted hemisphere is π).

##### Figure 4: A comparison of area light falloff. In Figure (4a) we see a Cornell Box illuminated by an area light that emits light evenly in all directions (the provided implemention). In Figure (4b) we see a Cornell Box illuminated by an area light that emits light according to a cosine-weighted hemisphere. Notice that this improvement alone already makes the box appear far more realistic and natural.
| No Light Falloff | Cosine Light Falloff |
|:----------------:|:----------------:|
| ![Fig 4a](/gallery/figures/fig_4a.png?raw=true) | ![Fig 4b](/gallery/figures/fig_4b.png?raw=true) |

## Direct Illumination
### Shadows
When sampling the illumination of a surface by a particular light, it is necessary to test if there are any objects in the scene that occlude the surface from that light (thereby casting a shadow). Details on how this is computed for each light follow.

#### Point Light & Spotlight Shadows
For point lights and spotlights, a ray is cast from the light's position to the surface sample. Then the first intersection of the ray with an object in the scene is computed, and if the distance between this intersection point and the light does not match up with the distance between the light and the surface sample, the surface sample is taken to be in shadow.

##### Figure 5: A demonstration of how point lights illuminate surfaces and cast shadows. In (5a) a sphere is illuminated on a box by a bright blue point light outside of the camera's view. In (5b) a sphere is illuminated by three brightly-colored point lights outside of the camera's view. Notice how the colors of the lights mix to form new colors.
| Figure 5a | Figure 5b |
|:----------------:|:----------------:|
| ![Fig 5a](/gallery/figures/fig_5a.png?raw=true) | ![Fig 5b](/gallery/figures/fig_5b.png?raw=true) |

##### Figure 6: A demonstration of how spotlights illuminate surfaces and cast shadows. In (6a) a sphere is illuminated on a box by a bright white spotlight (similar to a studio light) outside of the camera's view. In (6b) a sphere is illuminated by three brightly-colored spotlights outside of the camera's view that are focused on the sphere. Notice how only very faint shadows are cast in the scene because the cutoff angles of the spotlights are set such that the sphere "eclipses" the cones of light.
| Figure 6a | Figure 6b |
|:----------------:|:----------------:|
| ![Fig 6a](/gallery/figures/fig_6a.png?raw=true) | ![Fig 6b](/gallery/figures/fig_6b.png?raw=true) |

#### Directional Light Shadows
For directional lights, it is first necessary to find a point far outside the scene such that the vector from this point to the surface sample is colinear with the direction of the the directional light. Then a ray is cast from this point to the surface sample. The rest of the shadowing computation continues as before.

##### Figure 7: A demonstration of how directional lights illuminate surfaces and cast shadows. In (7a) a sphere is illuminated on a box by a bright blue directional light. In (7b) a sphere is illuminated by three brightly-colored directional lights. Notice the coloring of the shadows is subtractive (whereas the coloring on the ball is additive) because of the sharpness of directional light (i.e. there is no bleeding).
| Figure 7a | Figure 7b |
|:----------------:|:----------------:|
| ![Fig 7a](/gallery/figures/fig_7a.png?raw=true) | ![Fig 7b](/gallery/figures/fig_7b.png?raw=true) |

### Soft Shadows
For area lights, it is possible for a surface to be partially occluded from a light, causing a penumbra. In order to compute soft shadows, many random points on the surface of the light are sampled, and then a shadow ray (i.e. occlusion test) is sent from each of them to the potentially-occluded surface and the results are averaged to estimate the degree to which the object sits in shadow relative to the area light source.

#### An Illumination Optimization
Note that if the ray can make it all the way to the surface, then as an optimization, the surface illumination due to the ray is also sampled. For the R3AreaLight, this requires rewriting the provided reflection code to sit within the soft shadow loop. Note that this optimization is, remarkably, also a more accurate model since the illumination of a surface is now only sampled from unoccluded portions of area lights.

##### Figure 8: A comparison of soft shadow quality based the number of shadow rays sent per sample.
| No Soft Shadows | 1 Sample | 4 Samples | 16 Samples | 64 Samples | 256 Samples |
|:---:|:---:|:---:|:---:|:---:|:---:|
| ![Fig 8a](/gallery/figures/fig_8a.png?raw=true) | ![Fig 8b](/gallery/figures/fig_8b.png?raw=true) | ![Fig 8c](/gallery/figures/fig_8c.png?raw=true) | ![Fig 8d](/gallery/figures/fig_8d.png?raw=true) | ![Fig 8e](/gallery/figures/fig_8e.png?raw=true) | ![Fig 8f](/gallery/figures/fig_8f.png?raw=true) |

##### Figure 9: A comparison of hard shadowing to soft shadowing in two different scenes. Note that certain features that have not yet been discussed are disabled, hence the black spheres in (9b). For both scenes, which both measure 512x512, 512 shadow rays were sent for each pixel.
| Hard Shadows | Soft Shadows |
|:----------------:|:----------------:|
| ![Fig 9a-a](/gallery/figures/fig_9a-i.png?raw=true) | ![Fig 9a-ii](/gallery/figures/fig_9a-ii.png?raw=true) |
| ![Fig 9b-i](/gallery/figures/fig_9b-i.png?raw=true) | ![Fig 9b-ii](/gallery/figures/fig_9b-ii.png?raw=true) |

## Reflection & Transmission
### Perfect Reflection
To model perfect reflective behavior (a simplification of the physically-based reflective behavior that is implemented in this program), rays from the camera are reflected about the surface normal whenever they hit a reflective (specular) surface, and then the raytrace recurs on the reflected ray. This recursive raytracing process terminates when the ray bounces out of the scene, hits a non-specular (or transmissive) surface, or after the raytrace exceeds a maximum recursion depth.

##### Figure 10: A visualization of how light bounces around in a scene with a mirror sphere and mirror walls. Only light rays that bounces off of the sphere are shown, and traces are terminated either after eight bounces, or when their next bounce will carry the light out of the scene.
![Fig 10](/gallery/figures/fig_10.png?raw=true)

### Perfect Transmission
When light passes between optical media, it is necessary to apply to Snell's law in order to compute the correct angle of refraction. For most cases, this is a straightforward application of the refraction equations; however, when a ray attempts to pass via a sufficiently grazing angle into an optical medium with a lower index of refraction than that of its current optical medium, it is necessary to instead return a reflective bounce ("total internal reflection").

##### Figure 11: A visualization of how light is refracted through tramissive spheres with different indices of refraction (ir). Notice that the focus plane moves into the sphere from beyond the absorbing plane as the index of refraction grows. 
| ir = 1 | ir = 1.1 | ir = 1.2 | ir = 2 | ir = 16 |
|:---:|:---:|:---:|:---:|:---:|
| ![Fig 11a](/gallery/figures/fig_11a.png?raw=true) | ![Fig 11b](/gallery/figures/fig_11b.png?raw=true) | ![Fig 11c](/gallery/figures/fig_11c.png?raw=true) | ![Fig 11d](/gallery/figures/fig_11d.png?raw=true) | ![Fig 11e](/gallery/figures/fig_11e.png?raw=true) |

#### Fresnel Coefficients
For real dielectric surfaces (such as glass), full transmission of light never actually occurs, even if the material is fully transparent. Rather the incoming ray is split into specular and transmissive components, whose relative weightings depend on the Fresnel equations. Since evaluating these equations is relatively difficult to do quickly (e.g. as fast as a few floating-point operations), Schlick's Fresnel Approximation is used instead; this is a common substitution in physically-based renderers.

##### Figure 12: A comparison of how transparent objects appear with and without fresnel effects. Notice how the front of the Cornell Box is very softly reflected on the front of the glass sphere in the rendering with fresnel effects enabled.
| Fresnel Off | Fresnel On |
|:----------------:|:----------------:|
| ![Fig 12a](/gallery/figures/fig_12a.png?raw=true) | ![Fig 12b](/gallery/figures/fig_12b.png?raw=true) |


### Monte Carlo Path-Tracing
Although direct path-tracing is sufficient for estimating perfect reflection and perfect transmission, a stochastic path-tracing approach is needed in order to accurately estimate the integrals in the rendering equation that represent radiance due to reflection and transmission; this is because for a given backwards traced light ray that intersects a glossy surface (finite BRDF shininess value), the incoming light may be reflected in several direction in accordance to the specular lobe of the BRDF model.

Sampling all possible incoming directions and weighting them according to the specular pdf is not feasible, and so instead we use the Monte Carlo approach at each intersection to randomly select a diffuse, specular, or transmissive bounce (each chosen with probability proportional to their relative magnitude in the intersecting surface's material), importance sample the selected bounce, and then recur from there with adjusted weights. Although this approach proveably increases the variance of the sampling (thereby introducing noise into the rendering), it will converge to a fairly accurate estimate of reflected and transmitted radiance after a relatively small number of samples.

##### Figure 13: A visualization of several Monte Carlo paths traced from the camera into a Cornell Box with a glossy mirror sphere and a frosted transparent sphere. This visualization feature can be toggled by pressing `M` or `m` while in the visualization tool. The color of a ray corresponds to the pixel in the final render for which the ray was traced. Notice how specular importance sampling causes clusters of rays traced for the same pixel.
![Fig 13](/gallery/figures/fig_13.png?raw=true)

##### Figure 14: A comparison of noise introduced to reflective and transparent materials by the Monte Carlo method. For both spheres, the shininess value is `n = 1000`. Higher shininess constants will also reduce noise because there will be less variance in the specular importance sampling.
| 8 Samples | 32 Samples | 128 Samples |
|:---:|:---:|:---:|
| ![Fig 14a](/gallery/figures/fig_14a.png?raw=true) | ![Fig 14b](/gallery/figures/fig_14b.png?raw=true) | ![Fig 14c](/gallery/figures/fig_14c.png?raw=true) |

#### Glossy Reflection & Transmission
With our Monte Carlo path tracer, we are now able to accurately render specular and transmissive materials with low shininess parameters. When mirrors have low shininess values, the result is a glossy or nearly-diffuse reflection. When transmissive materials have low shininess values, the result is a "frosted glass" appearance.

##### Figure 15: A Cornell Box showcasing all specular and transmissive effects discussed in this section.
![Fig 15](/gallery/figures/fig_15.png?raw=true)

## Photon Mapping
### Photon Emission
As suggested in [Jensen's notes][2], photons are emitted with equal power from each light source, where the power of a color is taken to be the sum of its RGB channels. The number of photons emitted from a particular light source is proportional to that light source's contribution to the total power of all light in the scene. Once the emission phase is complete, the power of all stored photons is scaled down by the total power of light in the scene over the total number of photons emitted.

Note that the emission visualizations in this subsection can be toggled with either the `F` or `f` key from within the viewer, provided that the user has provided the appropriate arguments to generate a photon map beforehand (e.g. `-global <int N>`).

#### The Power of a Light Source
For 1D lights, the power of a light source is the flux of light into the scene due to the light source, and for 2D lights, the power of a light source is the area of the light multiplied by the flux of light into the scene due to any one point. In general, the area of a light is straightforward to compute, whereas the flux is a bit more tricky. Ignoring unrealistic light-attenuation, the flux due a point light is 4π (this follows from Gauss' Law). Conversly, the flux due to any point on a rectangular or circular light is just 2π since area lights only emit in one direction (there is only light flux through a single hemisphere around the point, as opposed to flux through a full sphere). For a given point on a directional light, the flux is only 1 since that point can only emit light along a single direction. Finally, the trickiest flux to compute is that of the spotlight, as it requires that we integrate isoenergetic rings around the spotlight's axis of emission. This works out to integrating `2π * sin(x) * cos(x)^n` with respect to `x` over the range `[0, α]` (where `α` is the cutoff angle and `n` is the dropoff rate). This works out to `Φ(Spotlight) = 2π / (n + 1.0) * (1.0 - pow(cos(α), n + 1.0))`.

#### The Emission Cycle
Because the user provides how many photons they would (roughly) like to store in their photon maps, it is necessary to emit light in several cycles so as to slowly approach the provided storage goal. This is efficiently achieved by first underestimating the number of photons to emit (e.g. assume each photon will be stored as many times as it is allowed to bounce before certain termination), distributing this number among the scene's lights for emittance, sampling the average stores per photon, and then using this average to arrive at the provided goal in three or four additional rounds (giving the average more time to converge lest we overshoot).

#### Point Light Photon Emission
Point lights emit photons uniformly in all directions. In order to achieve this, each photon leaves the point light in a direction chosen from a standard spherical point-picking process with rejection-sampling.

##### Figure 16: A visualization of photon emission from point lights. Note that the lines represent the vector of emission for each photon, and their color represents the carried power of the photon. In (16a) there is one point light with a power of 15, and in (16b) there are two point lights — both with a power of 15, but one tinted red and the other white. In both figures, 1000 photons were stored.
| Figure 16a | Figure 16b |
|:---:|:---:|
| ![Fig 16a](/gallery/figures/fig_16a.png?raw=true) | ![Fig 16b](/gallery/figures/fig_16b.png?raw=true) |

#### Spotlight Photon Emission
Spotlights emit photons in a distribution very similar to the specular lobe of the Phong BRDF. Therefore, we are able to recycle our specular importance sampling function to pick an emission vector for each photon emitted from a spotlight. As one modification, it is necessary restrict sampled vectors to fall the cutoff angle; this is again achieved with rejection sampling (however if enough samples are rejected, the program falls back to rescaling the displacement angle of the result by the cutoff angle).

##### Figure 17: A visualization of photon emission from spotlights with different dropoff values, sd. 
| sd = 100 | sd = 1000 |
|:---:|:---:|
| ![Fig 17a](/gallery/figures/fig_17a.png?raw=true) | ![Fig 17b](/gallery/figures/fig_17b.png?raw=true) |

#### Directional Light Photon Emission
To discuss emission from directional lights, we must first define our surface of emission. Although directional light is intended to simulate light emission from a extremely-bright and extremely-distant point (e.g. the sun), the same emission behavior can be achieved by emitting photons from a large disc that is oriented along the light's direction, placed sufficiently far outside the scene, and that has a diameter at least as wide as the diameter of the scene itself. With this established, emitting photons from a directional light is as simple as picking an emission point uniformly at random from the disc's surface and then sending that photon along the direction of the light.

#### Area Light Emission
For both rectangular and circular area lights, we first pick a point uniformly at random on their surface. Next, we use the diffuse important sampling function described in the first section to select a direction of emission from a cosine-weighted hemisphere along the light's normal.

##### Figure 18: A visualization of area light point-picking and normal directions for photon emission.
| Light Type | Point-Picking | Emission Normals |
|:------:|:---:|:---:|
| Directional| ![Fig 18a-i](/gallery/figures/fig_18a-i.png?raw=true) | ![Fig 18a-ii](/gallery/figures/fig_18a-ii.png?raw=true) |
| Circular | ![Fig 18b-i](/gallery/figures/fig_18b-i.png?raw=true) | ![Fig 18b-ii](/gallery/figures/fig_18b-ii.png?raw=true) |
| Rectangular | ![Fig 18c-i](/gallery/figures/fig_18c-i.png?raw=true) | ![Fig 18c-ii](/gallery/figures/fig_18c-ii.png?raw=true) |

### Photon Storage
In order to allow users to store hundreds of millions of photons in photon maps, it was necessary to compress the photon data structure as far as efficiently as possible (without losing accuracy). Using the compression suggestions from [Jensen][2], a storage size of only 30 Bytes per photon was achieved (this would be as low as 18 Bytes if single-precision floating-point values were used for position instead of double-precision).

#### The Photon Data Structure
A Photon object has three fields: `position`, `rgbe`, & `direction`. The `position` field holds an R3Point, which consists of three doubles; the `rgbe` field is an `unsigned char` array of length four that compactly stores RGB channels with single-precision floating-point values; the `direction` field is an integer value in the range `[0, 65536)` which maps to the incident direction of the photon. The 65536 possible directions are precomputed before the rendering step as an optimization.

#### The Photon Map Data Structure
A Photon Map is comprised of two objects: a global array of Photons, and a KdTree of Photons. Both of these data structures only hold pointers to Photons (which are stored in the heap). It is necessary to keep the original array of Photons even after the KdTree has been constructed because it is used for memory cleanup after rendering is complete. 

#### Photon Map Types
For certain renderings, it is necessary to store several types of photon maps. In particular, photon maps used to render caustic illumination are sampled directly and may have extremely fine features, so it is imperative that the map is of high quality (i.e. many photons). Conversely, photon maps used for indirect illumination are importance sampled, and so a small and lightweight map is strongly preferred to an accurate but slow photon map.

In this program, three types of photon maps are implemented: (1) the global map, which is used for indirect illumination and stores photons that bounced through `L{S|D}*D` paths; (2) the caustic map, which is used for caustic illumination and stores photons that bounced through `LS+D` paths; and finally, (3) the "fast-global" indirect map, which is a reduction of the global map to *exclude* photons traced through paths that match `LS+D` (caustics) and `LD` (direct illumination). This final map provides an approximate estimation of indirect illumination in a more efficient amount of time (since it does not require importance sampling a photon map, which is a slow process) at the cost of increased noise.

The following visualizations may be toggled from within the viewer by pressing either the `G` or `g` key for the global photon map, and either the `H` or `h` key for the caustic photon map. These maps will only show if the user has provided the appropriate arguments to generate a photon map beforehand (e.g. `-global <int N>` and/or `-caustic <int N>`).

##### Figure 19: A visualization of the photon maps of a Cornell Box that contains a point light and a transparent glass sphere.
| 500 Photons (Global Map) | 50,000 Photons (Global Map) | 5,000,000 Photons (Global Map) | 5,000 Photons (Caustic Map) |
|:------:|:---:|:---:|:---:|
|  ![Fig 19a](/gallery/figures/fig_19a.png?raw=true) | ![Fig 19b](/gallery/figures/fig_19b.png?raw=true) | ![Fig 19c](/gallery/figures/fig_19c.png?raw=true) | ![Fig 19d](/gallery/figures/fig_19d.png?raw=true) |

##### Figure 20: A visualization of the global photon map of a still-life scene.
| 5,000 Photons | 500,000 Photons | 
|:---:|:---:|
|  ![Fig 20a](/gallery/figures/fig_20a.png?raw=true) | ![Fig 20b](/gallery/figures/fig_20b.png?raw=true) |

##### Figure 21: A visualization of the global photon map of a teapot scene. 50,000 photons are shown.
![Fig 21](/gallery/figures/fig_21.png?raw=true)


### Radiance Sampling
Once the photon map is built, it is easy to use the map to estimate the radiance (due to photon paths traced in the map) at any point in the scene. First, as a preprocessing step, the photon map is fed into a data structure that can efficiently solve the K nearest neighbors problem, such as a KdTree. Then, for a given point, the K nearest photons to that point are found (up to a maximum distance), where K is a sampling parameter provided by the user. Finally, we use our BRDF to sample how much radiance each photon propagates along our incident ray, sum these values, and then normalize them by the area of a circle with a radius equal to the maximum distance between any one photon and the sample point, or, if fewer than K photons are found, the previously-discussed maximum distance parameter. The result of this computation is our radiance estimate.

##### Figure 22: A comparison of how radiance sample size effects the overall radiance estimate. In these figures, direct radiance estimates were made on a global photon map containing 5000 photons, and the maximum distance parameter is always the size of the box. Notice figures (22c) and (22d) approach accurate global illumination.
| 1 Photon per Sample | 8 Photons per Sample | 64 Photons per Sample | 128 Photons per Sample | 
|:---:|:---:|:---:|:---:|
|  ![Fig 22a](/gallery/figures/fig_22a.png?raw=true) | ![Fig 22b](/gallery/figures/fig_22b.png?raw=true) |  ![Fig 22c](/gallery/figures/fig_22c.png?raw=true) | ![Fig 22d](/gallery/figures/fig_22d.png?raw=true) |

##### Figure 23: A comparison of how the maximum radius of a radiance sample effects the overall radiance estimate. In these figures, direct radiance estimates were on made a global photon map containing 5000 photons, and the number of samples per estimate is always 64. The back wall of the Cornell Box is approximately 5x5.
| R = 0.05 | R = 0.25 | R = 0.5 | R = 1 | 
|:---:|:---:|:---:|:---:|
|  ![Fig 23a](/gallery/figures/fig_23a.png?raw=true) | ![Fig 23b](/gallery/figures/fig_23b.png?raw=true) |  ![Fig 23c](/gallery/figures/fig_23c.png?raw=true) | ![Fig 23d](/gallery/figures/fig_23d.png?raw=true) |

## Global Illumination
### Caustics
As mentioned above, caustics are computed by directly radiance-sampling an extremely large photon map that only stores photons that travel along `LS+D` paths. Because this restriction is relatively tight (specular or transparent objects in most scenes are often small and few in number), generating caustic maps often takes much longer than generating global photon maps of the same storage size since caustic maps will reject a majority of emitted photons. Currently, even this slowdown yields extremely fast photon-tracing times (such that building the KdTree is the true bottleneck), however caustic photon tracing could be further optimized by adding photon emission importance sampling, as described by [Jensen][2].

When rendering a scene, the caustic map is directly radiance-sampled for all intersections along our path-traces where shadowing is also computed (since they are both features caused by direct lighting). Although this is an accurate approach, not that this introduces a severe inefficiency into the rendering process: when Monte Carlo path-tracing reflections and refractions, integration allows us to reduce our shadow tests to one sample per recursive intersection; however we have no method — accurate or approximate — to reducing sample complexity of the caustic photon map (reducing radiance sampling parameters from within a Monte Carlo trace does not work because the same intersection point will still always give the same radiance; a stochastic solution combined with reduced parameters would be needed to optimize radiance-sampling in the same manner we optimized shadow rays). A potential solution to this problem would be to implement Ward's radiance caching algorithm, which would save us time wasted on both *new* radiance samplings and radiance samplings that had already been computed earlier! In its current state, this program does not implement Ward's radiance caching algorithm, but it does provide an irradiance computation speedup for global maps (the method is too inaccurate for caustic maps).

##### Figure 24: A comparison of three caustic photon maps for the same scene but each with different parameters. In (24a), there are 300 photons, with an estimate size of 10 and maximum distance of 1. In (24b) there are 300,000 photons, with an estimate size of 200 and maximum distance of 1. Finally, in (24c) there are 100,000,000 photons, with an estimate size of 500 and maximum distance of 0.5. Photon tracing took 0.04s, 1.52s, and 300s, respectively. Rendering the 512x512 images took 0.16s, 6.90s, and 178.8s, respectively. The intensity of the lights in the scene were increased to ease viewing, and the back wall of the Cornell Box is approximately 5x5.
| Figure 24a | Figure 24b | Figure 24c | 
|:---:|:---:|:---:|
|  ![Fig 24a](/gallery/figures/fig_24a.png?raw=true) | ![Fig 24b](/gallery/figures/fig_24b.png?raw=true) |  ![Fig 24c](/gallery/figures/fig_24c.png?raw=true) |

##### Figure 25: Merging the direct illumination layer of a Cornell Box with the caustic layer. The caustic photon map contains 10,000,000 photons, and radiance samples use estimates of 225 photons with maximum distances of 0.225. Rendering took 1564.3s for the last image. All images are of size 512x512, where each pixel is sampled four times.
| Direct | Caustic | Combined | 
|:---:|:---:|:---:|
|  ![Fig 25a](/gallery/figures/fig_25a.png?raw=true) | ![Fig 25b](/gallery/figures/fig_25b.png?raw=true) |  ![Fig 25c](/gallery/figures/fig_25c.png?raw=true) |

### Indirect Illumination
Indirect illumination is the illumination of surfaces by light that has bounced off a diffuse surface at least once. Using path notation, this is given by `L{S|D}*D(D|S)` paths. In this subsection, we present our two methods to compute indirect illumination, one of which is accurate but slow, the other of which is much faster at the cost of increased noise due to direct photon map sampling.

#### Directly Sampling the Global Map
The inaccurate method is to directly sample a modified version of the global photon map, just as we did for caustic sampling. Before we do this however, it is important to exclude certain paths — specifically, those that are sampled by other means. In particular, `LD` paths are computed by direct illumination, so we should never store a photon on its first bounce. Additionally, `LS+D` paths are already sampled through the caustic map, and so we conclude that we should only store photons at diffuse surfaces once they have already made at least one diffuse bounce, which fits the `L{S|D}*DD` path description of indirect illumination that terminates on diffuse surfaces.

##### Figure 26: A comparison of three "fast indirect illumination" global photon maps, each with different parameters. Note that these maps only store photons that have made at least one diffuse bounce. In (26a), there are 300 photons, with an estimate size of 10 and maximum distance of 1. In (26b) there are 300,000 photons, with an estimate size of 200 and maximum distance of 1. Finally, in (26c) there are 100,000,000 photons, with an estimate size of 500 and maximum distance of 0.5. Photon tracing took 0.02s, 1.52s, and 61.4s, respectively. Rendering the 512x512 images took 0.19s, 6.50s, and 205.4s, respectively. The back wall of the Cornell Box is approximately 5x5.
| Figure 26a | Figure 26b | Figure 26c | 
|:---:|:---:|:---:|
|  ![Fig 26a](/gallery/figures/fig_26a.png?raw=true) | ![Fig 26b](/gallery/figures/fig_26b.png?raw=true) |  ![Fig 26c](/gallery/figures/fig_26c.png?raw=true) |

##### Figure 27: Merging the direct illumination layer of a Cornell Box with the indirect layer using the fast visualization method. The global photon map contains 10,000,000 photons, and radiance samples use estimates of 225 photons with maximum distances of 0.225. Rendering took 1099.7s for the last image. All images are size 512x512, where each pixel is sampled four times.
| Direct | Indirect (Fast) | Combined | 
|:---:|:---:|:---:|
|  ![Fig 27a](/gallery/figures/fig_27a.png?raw=true) | ![Fig 27b](/gallery/figures/fig_27b.png?raw=true) |  ![Fig 27c](/gallery/figures/fig_27c.png?raw=true) |

#### Importance Sampling
The accurate method is to importance sample the photon map in order to compute the radiance at any point due to indirect illumination. Since the indirect illumination through photon maps is a diffuse interaction in the final layer (specular indirect illumination is handled seperately in the Monte Carlo path-tracing), we use diffuse importance sampling to shoot a numbern of rays into the scene. Each ray is path traced using the Monte Carlo process up until it makes a diffuse bounce, at which point the radiance at the bounce point is sampled from the global photon map. Finally, the results of all the path tracings are averaged into a final indirect illumination radiance. Because computing indirect illumination at a single point requires many path traces and many radiance samples, the global photon map is generally coarse and radiance estimates are made with relatively few photons because this will greatly increase the speed of the radiance sampling method.

Note that because we are importance sampling through a stochastic process, it is possible to optimize our sampling when it occurs within a Monte Carlo path trace for a reflection or a refraction just as we did with shadow rays. That is, if we normally take sample 1024 different direction when computing the indirect illumination at points that intersect eye rays, then we can only take 1 sample from within the Monte Carlo process, and rely on the integration to average out the noise.

To understand why this works, recall that indirect illumination is given by paths of light that follow `L{S|D}*D(D|S)` paths, where the final `(D|S)` is at the point that intersects with the eye ray. If the final bounce is specular, then our computation is handled by the Monte Carlo path tracer (which, as just discussed, will make indirect illumination samplings from the global photon map during diffuse bounces). Otherwise, in the diffuse case, the photons that hit the surface will have traveled along paths of `L{S|D}*D` before hitting, which is exactly what is stored in our global photon map!

##### Figure 28: A direct visualization of the global photon map that we importance sample for this scene in future figures. In the interest of sampling efficiency, this map only contains about 2048 photons, and radiance samples use estimates of 50 photons with maximum distances of 2.5.
![Fig 28](/gallery/figures/fig_28.png?raw=true)

##### Figure 29: A comparison of three indirect illumination layers, all of which importance sample the same global photon map, but each takes a different number of samples. Rendering (29a) took 3.95s, rendering (29b) took 51.8s, and rendering (29c) took 458.3s. Each image is 512x512 with a single sample per pixel. Compare these indirect estimates to those in figure 26.
| 8 Samples | 64 Samples | 1024 Samples | 
|:---:|:---:|:---:|
|  ![Fig 29a](/gallery/figures/fig_29a.png?raw=true) | ![Fig 29b](/gallery/figures/fig_29b.png?raw=true) |  ![Fig 29c](/gallery/figures/fig_29c.png?raw=true) |

##### Figure 30: Merging the direct illumination layer of a Cornell Box with the indirect layer using the importance sampling method. The global photon map contains about 2048 photons, and radiance samples use estimates of 50 photons with maximum distances of 2.5. The each indirect illumination radiance computation takes 320 importance-weighted samples. Rendering took 1099.7s for the last image. All images are of size 512x512, where each pixel is sampled four times (if there were no anti-aliasing, we would need to take 1,280 indirect samples for the same image quality).
| Direct | Indirect (Accurate) | Combined | 
|:---:|:---:|:---:|
|  ![Fig 30a](/gallery/figures/fig_30a.png?raw=true) | ![Fig 30b](/gallery/figures/fig_30b.png?raw=true) |  ![Fig 30c](/gallery/figures/fig_30c.png?raw=true) |

##### Figure 31: Merging all layers rendering layers into a full global illumination. The full image is 4096x4096 with four samples taken per pixel. Rendering time took about two days (169452.50 seconds), and required 67,108,864 Screen Rays, 21,510,009,344 Shadow Rays, 45,150,782,924 Monte Carlo Rays, 8,175,111,551 Transmissive Samples, 4,057,604,710 Specular Samples, 39,093,092,728 Indirect Samples, and 3,225,427,227 Caustic Samples. In total, this adds up to 121,279,137,348 total rays. Note that the BRDF model was adjusted since the merged rendering, so it might look slightly different than the ground truth. A new rendering is on its way.
| Direct | Transmissive | Specular | Caustic | Indirect | 
|:---:|:---:|:---:|:---:|:---:|
|  ![Fig 31a](/gallery/figures/fig_31a.png?raw=true) | ![Fig 31b](/gallery/figures/fig_31b.png?raw=true) |  ![Fig 31c](/gallery/figures/fig_31c.png?raw=true) |  ![Fig 31d](/gallery/figures/fig_31d.png?raw=true) |  ![Fig 31e](/gallery/figures/fig_31e.png?raw=true) |

![Fig 31f](/gallery/figures/fig_31f.png?raw=true)

## Additional Features
### Pixel Integration (Anti-Aliasing)
This program employs a straightforward technique in order to anti-alias its output. Given anti-aliasing factor `k` (specified by a user-provided parameter), an initial rendering is made for an output upscaled by `k`. Then, this image is downsampled to the final output resolution using an evenly-weighted grid filter.

##### Figure 32: A comparison of three renderings of the same scene, expept each is anti-aliased by a different parameter k. Each image is only 75x75 pixels.
| k = 0 | k = 1 | k = 2 | 
|:---:|:---:|:---:|
|  <img src="/gallery/figures/fig_32a.png" width="300"> | <img src="/gallery/figures/fig_32b.png" width="300"> | <img src="/gallery/figures/fig_32c.png" width="300"> |

### Progress Bar & Statistics
Because the photon tracing and rendering processes can take an extended amount of time for complex scenes, a real-time completion bar was implemented in order to ease impatience. If the verbose flag is provided by the user, applicable photon tracing and rendering statistics will be printed to the screen following program completetion. These statistics include number of photons stored shadow rays sent, and caustic randiance samples computed, among many others (including rendering time).

### Optimized KdTree Implementation
A program may make billions of radiance samples when rendering a scene with global illumination for high-resolution output. Therefore, it is of critical importance that our radiance-sampling function is extremely efficient. As pointed out by Nikhilesh Sigatapu, the provided R3KdTree's FindClosest() method uses linear-time lookup and quadratic-time insertion, which is very inefficient for radiance estimates that sample a large number of photons. As such, a new method was added called `FindClosestQuick()` which uses delayed heap construction, as suggested by [Jensen][2], to achieve a linearithmic solution to the k-closest points problem. Note that std::heap was used to maitain heap order.

Empirically, this improvement appeared to roughly halve the runtime of the Radiance sampling method.

### Multithreading
Both rendering and photon tracing are highly parallelizable processes. As such, as a major optimization, this program has full multithreading capabilities.

#### Multithreaded Randomness
One caveat of multithreading a distributed (stochastic) raytracer is that `rand()` and `drand48()` are not thread-safe because they use non-atomic global variables to store the PRNG state. As such, it was necessary to implement a thread-local PRNG, invoked through `RNThreadableRandomScalar()` using `std::mt19937` and `std::uniform_real_distribution`. 

#### Multithreaded Rendering
In order to fashion a single threaded raytracer into a multithreaded raytracer, very few adjustments were necessary. First, each thread (out of a total of `k` threads) is is responsible for sampling every `k`th pixel. The only shared variables among pixel samples are rendering statistics (e.g ray counts), and so it is necessary to fashion these into atomic variables through the `atomic.h` C++ library. Because atomic calls are slow, each thread maintains a thread-local count of the tracked statistics during the rendering process, and then the values are atomically added into the global statistics variables only once (right before each thread terminates).

#### Multithreaded Photon Mapping
Multithreading the photon mapper was significantly more challenging in part because photons themselves are stored in dynamic memory (the heap) and in part because pointers to photons are stored in a global resizing-array. It is necessary to store photons in the heap and their pointers in a global structure because we cannot know exactly how many photons will be stored in the map (the user only provides a rough storage target) and thread stack space is destroyed following thread termination. These requirements introduce two problems: (1) careless multithreaded heap-allocation can be slow due to heap contention, and (2) insertion into a resizing-array is not an atomic process.

Thankfully, both of these problems can be solved by giving each thread a very large local photon buffer on the stack, which is then flushed to dynamic and global memory when it reaches capacity. The entire photon-flushing step is made atomic through synchonization primatives, which both prevents heap contention (only one thread can allocate dynamic memory at once) and ensures that array insertion is thread-safe.

Empirically, using 8-threads on an 8-core (4 CPU) Intel i7 processor speeds up rendering and photon tracing by a factor of over four.

## Further Implementation Extensions
This section details several extensions that were made to the assignment. Already, the program offers a plethora of bells and whistles, including indirect-illumination acceleration techniques, multithreaded photontracing, multithreaded raytracing, and fresnel reflections. The features detailed below were added at a later date, hence why they are listed seperetely.

### Filtering
A standard 60-Watt lightbulb emits over `1x10^20` photons per second. Obviously, it is not feasible to store or compute this many interactions, and so we instead must make do with our coarse, and noisy estimates of Global illumination through the photon map. One major issue with the basic radiance gathering step is that the KdTree call searches for photons in 3D space, even though we presume them to all lie on a 2D surface for the final computation. As such, Jensen suggests in his paper several filtering methods that reduce raytracing noise, and smooth out photon maps.

For this submission, both cone filtering and weighted-gaussian filtering were implemented. Figure (33) demonstrates how the different filtering methods affect the caustic map (for which they are primarily intended due to the tradeoff between caustic detail and rendering time).

##### Figure 33: A comparision of photon map filtering techniques. The figures in the first row all use 300 photons, with an estimate size of 10 and maximum distance of 1. In the second row, there are 300,000 photons, with an estimate size of 200 and maximum distance of 1. Finally, in final row there are 30,000,000 photons per image, with an estimate size of 500 and maximum distance of 0.5.
|      | No Filter | Cone Filter (with k=1.25) | Gauss Filter |
|:---:|:------:|:---:|:---:|
| 300 Photons| ![Fig 33a-i](/gallery/figures/fig_33a-i.png?raw=true) | ![Fig 33a-ii](/gallery/figures/fig_33a-ii.png?raw=true) | ![Fig 33a-iii](/gallery/figures/fig_33a-iii.png?raw=true) |
| 300k Photons | ![Fig 33b-i](/gallery/figures/fig_33b-i.png?raw=true) | ![Fig 33b-ii](/gallery/figures/fig_33b-ii.png?raw=true) | ![Fig 33b-iii](/gallery/figures/fig_33b-iii.png?raw=true) |
| 3M Photons | ![Fig 33c-i](/gallery/figures/fig_33c-i.png?raw=true) | ![Fig 33c-ii](/gallery/figures/fig_33c-ii.png?raw=true) |![Fig 33c-iii](/gallery/figures/fig_33c-iii.png?raw=true) |

### Irradiance caching
As discussed earlier in this writeup, the most inefficient step of rendering by far is the radiance sample, but we observed that there is hope for improvement because many of these expensive computations sample the same point in space, which allows for caching. The techniqued used in this implementation is borrowed from Part III of the optimization section in Jensen's 2001 SIGGRAPH course notes. In this section, is it suggested that, as a preprocessing step, an irradiance sample is made at each traced photon. Then, the irradiance sample is placed into a KdTree so that taking radiance samples during the photontracing/gather stage reduces to just finding the single closest point in the tree and applying a BRDF. Note that non-lambertian surface behavior in necessarily lost in the simplification of indirect samplings from radiance estimates down to irradiance measures.


It is noted that are some noticible artificats along sharp edges, however it is likely possible to remove them either with filtering, or with sort sort of adaptive sampling — where rays that move relatively little between samples use full radiance estimates instead.

##### Figure 34: A comparision of indirect illumination techniques. Figure (34a) computes indirect illumination using expensive radiance estimations within large important sampling loops. It took over ten minutes to render. Conversely, Figure (34b) utilizes the accelerated approach of irradiance caching at the cost of accuracy and took about 3 minutes to render.
| Figure 34a | Figure 34b |
|:---:|:---:|
| ![Fig 34a](/gallery/figures/fig_34a.png?raw=true) | ![Fig 34b](/gallery/figures/fig_34b.png?raw=true) |

##### Figure 35: A visualization of the layers that are sampled for indirect illumination. Figure (34a) shows the layer for traditional approach, whereas Figure (34b) is the cached irradiance map.
| Figure 35a | Figure 35b |
|:---:|:---:|
| ![Fig 35a](/gallery/figures/fig_35a.png?raw=true) | ![Fig 35b](/gallery/figures/fig_35b.png?raw=true) |

### Depth of Field
Thus far, we have only rendered scenes taken from ideal cameras. In reality, photographic devices have relatively sizable apertures that will warp the motion of light. When this effect is exaggerated in photography and videography, the resulting effect of a heavily blurred background is referred to as "Depth of Field".

In order to implement this effect, a stochastic approach was used wherin many full ray traces were made for each pixel in the output. Rather than tracing all rays the same, however, each ray had its origin slightly perturbed along the plane of the aperture. This is what causes the depth of field in the first place because perturbed rays traced for the same pixel will diverge further (causing blur) as they race away from the camera.

Note that there are many other methods for rendering depth of field; in fact, this method is particularly inefficient because it requires a rerender of the image (more or less) for each pixel it samples. Nevertheless, this approach was chosen above others (for instance, the z-buffer blurring method) because it was the most straightforward to implement and also the most physically accurate.

##### Figure 36: On the left, a still life without depth of field. On the right, a still life with depth of field. Currently there is no way to adjust the focus plane, but this will come in a future version.
| No Depth of Field | Depth of Field |
|:---:|:---:|
| ![Fig 36a](/gallery/figures/fig_36a.png?raw=true) | ![Fig 36b](/gallery/figures/fig_36b.png?raw=true) |


# Art Gallery
This section contains several photorealistic renderings that were made using the entire feature-set of the global-illumination renderer.

## The Rainbow Room
This scene contains two spotlights and one pointlight behind a wall of multicolored glass columns. The "frostedness" of the glass decreases from left to right. Without caustics and indirect illumination, the glass would would fully occlude the three lights, rendering the scene black. Notice how each column casts a color of light, and how light transmitted from the focused purple column is scattered more sharply than light from the frosted red column.

<p align="center">
  <img width="1024" height="auto" src="/gallery/figures/fig_37.png?raw=true">
</p>

## The Crystal Teapot
This scene contains a teapot with similar material properties as diamond (slightly frosted and with a similar refractive index; dispersion has not been implemented here for full accuracy). Observe how fresnel effects cause reflections on the teapot, even though the material itself is fully transparent.

<p align="center">
  <img width="1024" height="auto" src="/gallery/figures/fig_38.png?raw=true">
</p>

## Life out of Focus
This scene contains a shiny purple teapot in front of objects from the still-life scene. Depth of field was used to focus on the teapot, thereby rendering the still-life objects blurry. Although somewhat difficult to decipher even here, notice how the bits of reflection of the still-life on the teapot are still out of focus. This is physically accurate behavior, however it is particularly difficult to achieve with traditional z-buffer depth of field implementations (since depth is stored at the plane of reflection and not for the objects in a reflection).

<p align="center">
  <img width="1024" height="auto" src="/gallery/figures/fig_39.png?raw=true">
</p>

# Future Features
- Importance Sampling
- Hemisphere Lights for Outdoor Scenes
- Dispersion Effects
- Volumetric Raytracing

# Credits
## Authors
* **Reilly Bova** - *Rendering Program and Examples* - [ReillyBova](https://github.com/ReillyBova)
* **Tom Funkhouser** - *C++ Graphics & Geometry Library*

See also the list of [contributors](https://github.com/ReillyBova/poisson/contributors) who participated in this project.

## References
1. [Henrik Wann Jensen. “Global illumination using photon maps”. Rendering Techniques ’96 (Proceedings of the Seventh Eurographics Workshop on Rendering), pages 21–30. Springer Verlag, 1996.][1]

2. [Henrik Wann Jensen, Frank Suykens, and Per Christensen. "A Practical Guide to Global Illumination using Photon Mapping". SIGGRAPH'2001 Course 38, Los Angeles, August 2001.][2]

3. [http://www.cs.princeton.edu/courses/archive/fall18/cos526/papers/importance.pdf "Jason Lawrence. "Importance Sampling of the Phong Reflectance Model". COS 526 Course Notes.][3]

4. [E. Lafortune and Y. Willems. "Using the modified Phong reflectance model for physically based rendering". Technical Report CW197, Dept. Comp. Sci., K.U. Leuven, 1994.][4]


[1]: http://graphics.ucsd.edu/~henrik/papers/photon_map/global_illumination_using_photon_maps_egwr96.pdf "Global illumination using photon maps"

[2]: http://www.cs.princeton.edu/courses/archive/fall18/cos526/papers/jensen01.pdf "A Practical Guide to Global Illumination using Photon Mapping"

[3]: http://www.cs.princeton.edu/courses/archive/fall18/cos526/papers/importance.pdf "Importance Sampling of the Phong Reflectance Model"

[4]: http://mathinfo.univ-reims.fr/IMG/pdf/Using_the_modified_Phong_reflectance_model_for_Physically_based_rendering_-_Lafortune.pdf "Using the modified Phong reflectance model for physically based rendering"

## License
This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details

## Acknowledgments
Thank you to Professor Szymon Rusinkiewicz and the rest of the Computer Graphics faculty at Princeton University for teaching me all the techniques I needed to champion this assignment in the Fall 2018 semester of COS 526: Advanced Computer Graphics.

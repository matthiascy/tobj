# `tobj` – Tiny OBJ Loader

[![Crate](https://img.shields.io/crates/v/tobj.svg)](https://crates.io/crates/tobj)
![Build Status](https://github.com/Twinklebear/tobj/workflows/CI/badge.svg)

Inspired by Syoyo’s excellent [`tinyobjloader`](https://github.com/syoyo/tinyobjloader).
Aims to be a simple and lightweight option for loading `OBJ` files.

Just returns two `Vec`s containing loaded models and materials.

## New Features in This Fork

Five new (optional) features, all behind flags in a settings struct passed
to the `load_obj*()` functions. Most of them arose as needs when trying to
use OBJ files in the wild with
[`opensubdiv-petite`](https://github.com/virtualritz/opensubdiv-petite).
* Merging of vertices that share a point in space (changes topology).
* Separate indices for normals & texture coordinates.
* Reordering of indices so they can be omitted completely.
* Filtering out of point-like and/or line-like faces (changes topology).
* Blowing up point-like and line-like faces to (degenerate) triangles (changes topology).

These have all now been merged and released in `tobj 3.x`.

## Triangulation

Meshes can be triangulated on the fly or left as-is.

Only polygons that are trivially convertible to triangle fans are supported.
Arbitrary polygons may not behave as expected. The best solution would be to
convert your mesh to solely consist of triangles in your modeling
software.

## Optional – Normals & Texture Coordinates

It is assumed that all meshes will at least have positions, but normals and
texture coordinates are optional.

If no normals or texture coordinates are found then the corresponding `Vec`s
for the `Mesh` will be empty.

## Flat Data

Values are stored packed as floats in flat `Vec`s.

For example, the `positions` member of a `Mesh` will contain `[x, y, z, x, y, z,
...]` which you can then use however you like.

## Indices

Indices are also loaded and may re-use vertices already existing in the mesh,
this data is stored in the `indices` member.

When a `Mesh` contains *per vertex per face* normals or texture coordinates,
positions can be duplicated to be *per vertex per face* too via the
`single_index` flag. This potentially changes the topology (faces may become
disconnected even though their vertices still share a position in space).

By default separate indices for normals and texture coordinates are created.
This also guarantees that the topology of the `Mesh` does *not* change when
either of the latter are specified *per vertex per face*.

## Materials

Standard `MTL` attributes are supported too. Any unrecognized parameters will be
stored in a `HashMap` containing the key-value pairs of the unrecognized
parameter and its value.

## Features

*  [`ahash`](https://crates.io/crates/ahash) – On by default. Use
   [`AHashMap`](https://docs.rs/ahash/latest/ahash/struct.AHashMap.html) for
   hashing when reading files and merging vertices. To disable and use the
   slower
   [`HashMap`](https://doc.rust-lang.org/std/collections/hash_map/struct.HashMap.html)
   instead, unset default features in `Cargo.toml`:

   ```toml
   [dependencies.tobj]
   default-features = false
   ```

* `merging` – Adds support for merging identical vertex positions on
   disconnected faces during import.

   **Warning:** this feature uses *const generics* and thus requires at least a
   `beta` toolchain to build.

* `reordering` – Adds support for reordering the normal- and texture coordinate
   indices.

## Documentation

Rust docs can be found [here](https://docs.rs/tobj/).

## Installation

Add the [crate](https://crates.io/crates/tobj) as a dependency in your
`Cargo.toml` and you’re all set!

## Example

The [print mesh example](examples/print_mesh.rs) (also below) loads an `OBJ`
file from the command line and prints out some information about its faces,
vertices, and materials.
```rust
fn main() {
    let obj_file = std::env::args()
        .skip(1)
        .next()
        .expect("A .obj file to print is required");

    let (models, materials) =
        tobj::load_obj(
            &obj_file,
            &tobj::LoadOptions::default()
        )
        .expect("Failed to OBJ load file");

    // Note: If you don't mind missing the materials, you can generate a default.
    let materials = materials.expect("Failed to load MTL file");

    println!("Number of models          = {}", models.len());
    println!("Number of materials       = {}", materials.len());

    for (i, m) in models.iter().enumerate() {
        let mesh = &m.mesh;
        println!("");
        println!("model[{}].name             = \'{}\'", i, m.name);
        println!("model[{}].mesh.material_id = {:?}", i, mesh.material_id);

        println!(
            "model[{}].face_count       = {}",
            i,
            mesh.face_arities.len()
        );

        let mut next_face = 0;
        for face in 0..mesh.face_arities.len() {
            let end = next_face + mesh.face_arities[face] as usize;

            let face_indices = &mesh.indices[next_face..end];
            println!(" face[{}].indices          = {:?}", face, face_indices);

            if !mesh.texcoord_indices.is_empty() {
                let texcoord_face_indices = &mesh.texcoord_indices[next_face..end];
                println!(
                    " face[{}].texcoord_indices = {:?}",
                    face, texcoord_face_indices
                );
            }
            if !mesh.normal_indices.is_empty() {
                let normal_face_indices = &mesh.normal_indices[next_face..end];
                println!(
                    " face[{}].normal_indices   = {:?}",
                    face, normal_face_indices
                );
            }

            next_face = end;
        }

        // Normals and texture coordinates are also loaded, but not printed in
        // this example.
        println!(
            "model[{}].positions        = {}",
            i,
            mesh.positions.len() / 3
        );
        assert!(mesh.positions.len() % 3 == 0);

        for vtx in 0..mesh.positions.len() / 3 {
            println!(
                "              position[{}] = ({}, {}, {})",
                vtx,
                mesh.positions[3 * vtx],
                mesh.positions[3 * vtx + 1],
                mesh.positions[3 * vtx + 2]
            );
        }
    }

    for (i, m) in materials.iter().enumerate() {
        println!("material[{}].name = \'{}\'", i, m.name);
        println!(
            "    material.Ka = ({}, {}, {})",
            m.ambient[0], m.ambient[1], m.ambient[2]
        );
        println!(
            "    material.Kd = ({}, {}, {})",
            m.diffuse[0], m.diffuse[1], m.diffuse[2]
        );
        println!(
            "    material.Ks = ({}, {}, {})",
            m.specular[0], m.specular[1], m.specular[2]
        );
        println!("    material.Ns = {}", m.shininess);
        println!("    material.d = {}", m.dissolve);
        println!("    material.map_Ka = {}", m.ambient_texture);
        println!("    material.map_Kd = {}", m.diffuse_texture);
        println!("    material.map_Ks = {}", m.specular_texture);
        println!("    material.map_Ns = {}", m.shininess_texture);
        println!("    material.map_Bump = {}", m.normal_texture);
        println!("    material.map_d = {}", m.dissolve_texture);

        for (k, v) in &m.unknown_param {
            println!("    material.{} = {}", k, v);
        }
    }
}
```

## Rendering Examples

For an example of integration with [glium](https://github.com/tomaka/glium) to
make a simple OBJ viewer, check out [tobj viewer](https://github.com/Twinklebear/tobj_viewer).

A sample image from the viewer is shown below. The [Rungholt](http://graphics.cs.williams.edu/data/meshes.xml)
model can be found on Morgan McGuire’s meshes page and was originally built by
kescha.

The Rungholt model is reasonably large (6.7M triangles, 12.3M vertices) and is
loaded in ~7.47s using a peak of ~1.1GB of memory on a Windows 10 machine with
an i7-4790k and 16GB of 1600Mhz DDR3 RAM with `tobj 0.1.1` on `rustc 1.6.0`.

![Rungholt](http://i.imgur.com/wImyNG4.png)

For an example of integration within a ray tracer, check out tray\_rust’s
[mesh module](https://github.com/Twinklebear/tray_rust/blob/master/src/geometry/mesh.rs).

The Stanford Buddha and Dragon from the [Stanford 3D Scanning Repository](http://graphics.stanford.edu/data/3Dscanrep/)
both load quite quickly.

The Rust logo model was made by [Nylithius on BlenderArtists](http://blenderartists.org/forum/showthread.php?362836-Rust-language-3D-logo).

The materials used are from the [MERL BRDF Database](http://www.merl.com/brdf/).

![Rust Logo with friends](http://i.imgur.com/E1ylrZW.png)

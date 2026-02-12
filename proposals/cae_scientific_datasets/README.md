# OpenUSD Schemas for CAE Simulation Datasets

A minimal schema vocabulary for representing solver native scientific
datasets in OpenUSD without format conversion.

## Summary

This proposal defines a small set of OpenUSD schema types for
describing Computer Aided Engineering (CAE) and scientific simulation
datasets.
The schemas enable cross solver/tool interoperability through
USD composition while preserving solver native files as the
source of truth.

Bulk numeric data remains in its original format and is read on demand.
OpenUSD acts as the composition and access layer, not a data container.

The proposal defines:

- **CaeDataSet** : a container prim representing one scientific dataset
- **CaeFieldArray** : a base prim for externally stored numeric arrays
- **Data model API schemas** : single apply APIs that define how to
  interpret dataset arrays
  (`CaePointCloudAPI`, `CaeMeshAPI`, `CaeDenseVolumeAPI`)
- **Format specific FieldArray subtypes** : an extension point for
  file format locator attributes, with `CaeCgnsFieldArray`
  (CGNS, AIAA R-101A-2005) provided as an example
- **Data Delegate Interface** : a minimal contract for reading arrays
  on demand from external files. Data Delegate implementations using this contract are format specific.

## Problem Statement

CAE workflows span computational fluid dynamics, structural
mechanics, electromagnetics, electronics design, climate modeling, and
more.
Each solver produces results in its own native format.

Many downstream workflows including visualization, post-processing, AI/ML
training still require exporting or converting solver outputs into
an "analysis ready" format (VTK, CSV, in-house variants) before data
can be consumed.
This creates several compounding problems:

- **Lost fidelity.**
  Format conversion frequently degrades data.
  Face centered fields may be averaged to cell or point data.
  Mixed element topologies may be simplified.
  Solver specific metadata is dropped.

- **Lost provenance.**
  Converted artifacts become the de facto "truth," disconnected from
  the solver output they derived from.

- **Increased I/O and storage.**
  Duplicate datasets, long export times, and transient intermediate
  files multiply storage and transfer costs.
  As simulation scale grows (larger meshes, more timesteps), full
  copy convert pipelines become less tractable.

- **Algorithm and workflow lock-in.**
  Every new format requires new readers, pipelines, and GPU kernels.
  Pipelines and algorithms become coupled to a specific data model,
  creating an N-by-M integration problem across solvers and tools.

What is needed is a way to represent scientific datasets, their arrays,
and a canonical interpretation contract in OpenUSD while keeping
heavy numeric data external and solver native.

## Existing OpenUSD Mechanisms Considered

OpenUSD provides strong primitives for geometry and volumes.
We evaluated whether existing schema domains could address CAE data
requirements without new types.

- **UsdGeom.**
  `UsdGeomMesh` and related types are designed for authored geometry
  where mesh topology and point positions are stored as USD attributes.
  Solver native meshes often use topologies that do not map 1:1 to
  UsdGeom conventions: mixed element types, polyhedral cells,
  face centered fields, and variable length connectivity arrays.
  Representing these through UsdGeom would require the format
  conversion step this proposal aims to avoid.

- **UsdVol / FieldAsset.**
  UsdVol defines `Volume` prims with `field:*` relationships to
  field prims that point at external OpenVDB or Field3D assets.
  This model is well suited to renderable volumetric fields (smoke,
  fire) sampled on sparse or dense grids.
  However, many CAE results include non volumetric arrays,
  unstructured connectivity, element IDs, per face or per cell
  attributes, that are not naturally representable as volume fields
  without inventing new semantics or converting the data.
  UsdVol's `FieldAsset` represents a single volume field; it does not
  define semantics for variable length topology arrays or
  mesh associated centering.

These mechanisms are excellent for their designed purposes.
CAE simulation data has requirements that fall outside their scope,
including variable length connectivity, mixed topologies,
cell/face/edge centering, and multi format provenance.

## Proposal

We propose a small, format neutral set of OpenUSD schema types.
The design is intentionally parallel to `UsdVol`: a container prim
binds named fields via `field:*` namespace relationships, and each
field references external data through asset paths.
The generalization extends this pattern beyond renderable volumes to
arbitrary scientific arrays with explicit centering metadata.

The following diagram shows how the proposed components relate to
solver native data on the left, and downstream ecosystem tooling on
the right.
Green components are standardization candidates (this proposal);
yellow components are integrator specific; gray components are
ecosystem consumers that benefit from the shared schema layer.

![Architecture diagram showing the relationship between standardization candidates (CaeDataSet, CaeFieldArray, data model API schemas, Data Delegate Interface), integrator specific pieces (FieldArray subtypes, native solver outputs, proprietary formats, vendor I/O libraries, Data Delegate Implementations), and ecosystem tooling (data importers, analysis/visualization libraries, AI/surrogate pipelines, CAE workflows).](architecture-diagram.png)

### CaeDataSet (Typed Prim)

A container prim representing one scientific dataset.
`CaeDataSet` anchors the canonical interpretation of its data (via
applied API schemas) and binds named field arrays through `field:`
namespace relationships.

It is analogous to `UsdVolVolume`, but for general scientific data.

**Schema definition:**

```usda
class CaeDataSet "CaeDataSet" (
    inherits = </Typed>
    doc = """A scientific dataset. A dataset is made up of any number
             of CaeFieldArray primitives bound together in this dataset.
             Each CaeFieldArray primitive is specified as a relationship
             with namespace prefix 'field'."""
)
{
}
```

`CaeDataSet` carries no attributes of its own.
Interpretation is provided entirely by applied API schemas
(see [Data Model API Schemas](#data-model-api-schemas) below).

**Relationships:**

| Relationship | Description |
|---|---|
| `custom rel field:<name>` | Binds a named `CaeFieldArray` prim to this dataset. Follows the same `field:` namespace pattern used by `UsdVolVolume`. |

### CaeFieldArray (Typed Prim)

Represents an N-dimensional numeric array whose bulk data is stored
outside USD.
This is the base type for all field arrays.
Format specific subtypes inherit from it and add locator attributes
specific to their file format.

**Schema definition:**

```usda
class CaeFieldArray "CaeFieldArray" (
    inherits = </Typed>
    doc = """An N-dimensional numeric array whose bulk data is stored
             outside USD and referenced by asset paths."""
)
{
    asset[] fileNames = [] (
        doc = """Specifies the assets for the files. When multiple
                 assets are specified they are treated as spatial
                 partitions of the same dataset. Temporal partitions
                 may be specified by animating this attribute using
                 time codes."""
    )

    uniform token fieldAssociation = "none" (
        allowedTokens = ["none", "vertex", "cell"]
        doc = """Specifies the dataset element this field array is
                 associated with."""
    )
}
```

**Attributes:**

| Attribute | Type | Default | Description |
|---|---|---|---|
| `fileNames` | `asset[]` | `[]` | External file references. Multiple assets represent spatial partitions. Time varying data is expressed via `fileNames.timeSamples`. |
| `fieldAssociation` | `uniform token` | `"none"` | Where data is associated: `"none"`, `"vertex"`, or `"cell"`. Extensible in future proposals to `"faceCenter"`, `"edgeCenter"`, etc. |

**Design rationale.**
`CaeFieldArray` is intentionally analogous to
`UsdVolFieldAsset.filePath` but generalized:

- Supports arrays of any rank and type, not just scalar volume fields.
- Supports variable length data (connectivity, offsets).
- The `fieldAssociation` attribute makes centering explicit, a
  critical concept in CAE that has no equivalent in UsdVol.
- The `asset[]` array type allows multiple files to represent
  spatial partitions of the same dataset.
- The `uniform` qualifier on `fieldAssociation` declares that
  centering does not vary over time.

### Format-Specific Subtypes: CaeCgnsFieldArray

Format specific subtypes of `CaeFieldArray` add the attributes needed
to locate data within a particular file format.
This proposal includes one such example:
**CGNS** (CFD General Notation System).

CGNS is chosen because:

- It is a widely adopted standard (AIAA Recommended Practice) with
  broad support across CFD solvers
  (Fluent, STAR-CCM+, OpenFOAM via converters).
- Its hierarchical database structure provides a natural fit for
  demonstrating the locator attribute pattern.

Other format subtypes (e.g., VTK, HDF5, EnSight) follow the same
extension pattern and can be created by any implementor.

**Schema definition:**

```usda
class CaeCgnsFieldArray "CaeCgnsFieldArray" (
    inherits = </CaeFieldArray>
    doc = "A CGNS data array."
)
{
    string fieldPath = "" (
        doc = """Specifies the path to the node in the CGNS
                 database."""
    )
}
```

**Additional attributes (beyond CaeFieldArray):**

| Attribute | Type | Default | Description |
|---|---|---|---|
| `fieldPath` | `string` | `""` | Path to the data node within the CGNS hierarchical database (e.g., `"/Base/Zone/GridCoordinates/CoordinateX"`). |

**Extension pattern.**
Any file format can be supported by creating a subtype of
`CaeFieldArray` and adding format specific locator attributes.
For example, a hypothetical VTK subtype might add an `arrayName`
attribute; an HDF5 subtype might add a `datasetPath` attribute.
The base `CaeFieldArray` attributes (`fileNames`,
`fieldAssociation`) are inherited by all subtypes.

### Data Model API Schemas

Data model API schemas are single apply schemas applied to a
`CaeDataSet` to define how its field arrays should be interpreted.
They are model specific, not format specific. For example, datasets
from different file formats can share the same data model API if
their underlying structure is the same.

Multiple data model APIs can compose on a single `CaeDataSet` prim.

#### CaePointCloudAPI

Defines a dataset that represents a point cloud, a set of spatial
coordinates without connectivity.

```usda
class "CaePointCloudAPI" (
    inherits = </APISchemaBase>
    doc = "Defines a dataset that represents a point cloud."
    customData = {
        token apiSchemaType = "singleApply"
    }
)
{
    rel cae:pointCloud:coordinates (
        doc = """Specifies the CaeFieldArray(s) to interpret as
                 spatial coordinates. Multiple targets may be
                 specified when individual components are split
                 among multiple field arrays."""
    )
}
```

| Relationship | Description |
|---|---|
| `cae:pointCloud:coordinates` | Target `CaeFieldArray` prim(s) providing spatial coordinate data. Multiple targets support component split arrays (e.g., separate X, Y, Z arrays as is common in CGNS). |

#### CaeMeshAPI

Defines a dataset that represents a surface or volume mesh.

```usda
class "CaeMeshAPI" (
    inherits = </APISchemaBase>
    doc = "Defines a dataset that represents a surface mesh."
    customData = {
        token apiSchemaType = "singleApply"
    }
)
{
    rel cae:mesh:points (
        doc = """Specifies the CaeFieldArray to treat as the mesh
                 points."""
    )

    rel cae:mesh:faceVertexIndices (
        doc = """Specifies the CaeFieldArray to treat as face vertex
                 indices."""
    )

    rel cae:mesh:faceVertexCounts (
        doc = """Specifies the CaeFieldArray to treat as face vertex
                 counts."""
    )
}
```

| Relationship | Description |
|---|---|
| `cae:mesh:points` | Target `CaeFieldArray` providing mesh point positions. |
| `cae:mesh:faceVertexIndices` | Target `CaeFieldArray` providing face-vertex connectivity indices. |
| `cae:mesh:faceVertexCounts` | Target `CaeFieldArray` providing the number of vertices per face. |

#### CaeDenseVolumeAPI

Defines a dataset that represents a dense (structured) volume on a
regular grid.

```usda
class "CaeDenseVolumeAPI" (
    inherits = </APISchemaBase>
    doc = "Defines a dataset that represents a dense volume."
    customData = {
        token apiSchemaType = "singleApply"
    }
)
{
    uniform int3 cae:denseVolume:minExtent (
        doc = """Specifies the minimum structured (IJK) extent for
                 the volume."""
    )

    uniform int3 cae:denseVolume:maxExtent (
        doc = """Specifies the maximum structured (IJK) extent for
                 the volume."""
    )

    uniform float3 cae:denseVolume:spacing = (1.0, 1.0, 1.0) (
        doc = "Specifies the spacing along each axis."
    )
}
```

| Attribute | Type | Default | Description |
|---|---|---|---|
| `cae:denseVolume:minExtent` | `uniform int3` | | Minimum structured (IJK) extent for the volume. |
| `cae:denseVolume:maxExtent` | `uniform int3` | | Maximum structured (IJK) extent for the volume. |
| `cae:denseVolume:spacing` | `uniform float3` | `(1.0, 1.0, 1.0)` | Grid spacing along each axis. |

### Data Delegate Interface (Informational)

A data delegate is the runtime component that reads bulk numeric data
on demand from external files.
Given a `CaeFieldArray` prim, a delegate resolves `fileNames` plus
any format specific attributes and returns a typed buffer.

Key design points:

- **Lazy, pull-based.**
  Data is read only when requested by a consuming algorithm.
  No bulk preloading is required.
- **Format specific plugins.**
  One delegate implementation per file format (e.g., a CGNS delegate,
  a VTK delegate).
  This is analogous to how UsdVol relies on external libraries
  (OpenVDB, Field3D) to read data from referenced assets.
- **Not a schema.**
  The delegate interface is a plugin contract, not a USD schema type.
  Implementations remain outside the standard.

## Examples

The following examples demonstrate the proposed schemas in practice.
Complete `.usda` files are provided in the
[`examples/`](examples/) directory.

### Example 1: Unstructured CFD Mesh (CGNS)

An unstructured CFD simulation stores its mesh and solution fields in
a CGNS file.
The `CaeDataSet` applies `CaePointCloudAPI` to define the spatial
coordinates and binds pressure, velocity, and temperature as named
fields.
All bulk data remains in the solver native `.cgns` file.

```usda
#usda 1.0
(
    defaultPrim = "World"
)

def Xform "World"
{
    def CaeDataSet "FluidDomain" (
        prepend apiSchemas = ["CaePointCloudAPI"]
    )
    {
        # Spatial coordinates via CaePointCloudAPI
        rel cae:pointCloud:coordinates = [
            </World/FluidDomain/GridCoordinatesX>,
            </World/FluidDomain/GridCoordinatesY>,
            </World/FluidDomain/GridCoordinatesZ>
        ]

        # Named field relationships
        custom rel field:pressure = </World/FluidDomain/Pressure>
        custom rel field:velocity = </World/FluidDomain/Velocity>
        custom rel field:temperature = </World/FluidDomain/Temperature>
        custom rel field:elementConnectivity = </World/FluidDomain/ElementConnectivity>
        custom rel field:elementStartOffset = </World/FluidDomain/ElementStartOffset>

        # --- CGNS field arrays (bulk data in solver native .cgns file) ---

        def CaeCgnsFieldArray "GridCoordinatesX"
        {
            asset[] fileNames = [@./simulation_results.cgns@]
            string fieldPath = "/Base/Zone/GridCoordinates/CoordinateX"
            uniform token fieldAssociation = "vertex"
        }

        def CaeCgnsFieldArray "GridCoordinatesY"
        {
            asset[] fileNames = [@./simulation_results.cgns@]
            string fieldPath = "/Base/Zone/GridCoordinates/CoordinateY"
            uniform token fieldAssociation = "vertex"
        }

        def CaeCgnsFieldArray "GridCoordinatesZ"
        {
            asset[] fileNames = [@./simulation_results.cgns@]
            string fieldPath = "/Base/Zone/GridCoordinates/CoordinateZ"
            uniform token fieldAssociation = "vertex"
        }

        def CaeCgnsFieldArray "ElementConnectivity"
        {
            asset[] fileNames = [@./simulation_results.cgns@]
            string fieldPath = "/Base/Zone/Elements/ElementConnectivity"
            uniform token fieldAssociation = "none"
        }

        def CaeCgnsFieldArray "ElementStartOffset"
        {
            asset[] fileNames = [@./simulation_results.cgns@]
            string fieldPath = "/Base/Zone/Elements/ElementStartOffset"
            uniform token fieldAssociation = "none"
        }

        def CaeCgnsFieldArray "Pressure"
        {
            asset[] fileNames = [@./simulation_results.cgns@]
            string fieldPath = "/Base/Zone/FlowSolution/Pressure"
            uniform token fieldAssociation = "cell"
        }

        def CaeCgnsFieldArray "Velocity"
        {
            asset[] fileNames = [@./simulation_results.cgns@]
            string fieldPath = "/Base/Zone/FlowSolution/Velocity"
            uniform token fieldAssociation = "cell"
        }

        def CaeCgnsFieldArray "Temperature"
        {
            asset[] fileNames = [@./simulation_results.cgns@]
            string fieldPath = "/Base/Zone/FlowSolution/Temperature"
            uniform token fieldAssociation = "cell"
        }
    }
}
```

CGNS stores coordinates as separate per-axis arrays.
The `cae:pointCloud:coordinates` relationship targets all three.

### Example 2: Time Varying Simulation

A transient analysis produces results at multiple timesteps.
USD's `timeSamples` expresses the temporal dimension using the same
pattern used throughout USD for animated data.

```usda
#usda 1.0
(
    defaultPrim = "World"
)

def Xform "World"
{
    def CaeDataSet "TransientAnalysis" (
        prepend apiSchemas = ["CaePointCloudAPI"]
    )
    {
        rel cae:pointCloud:coordinates = </World/TransientAnalysis/Points>
        custom rel field:displacement = </World/TransientAnalysis/Displacement>

        def CaeCgnsFieldArray "Points"
        {
            asset[] fileNames = [@./results_t000.cgns@]
            string fieldPath = "/Base/Zone/GridCoordinates/Coordinates"
            uniform token fieldAssociation = "vertex"
        }

        def CaeCgnsFieldArray "Displacement"
        {
            # Time varying: different files per timestep
            asset[] fileNames.timeSamples = {
                0:   [@./results_t000.cgns@],
                10:  [@./results_t010.cgns@],
                20:  [@./results_t020.cgns@],
                30:  [@./results_t030.cgns@],
            }
            string fieldPath = "/Base/Zone/FlowSolution/Displacement"
            uniform token fieldAssociation = "vertex"
        }
    }
}
```

The `fileNames` attribute is animated with `timeSamples`, producing a
different file reference at each time code.
The mesh (points) remains static while the displacement field varies
over time.
A data delegate resolves the correct file for each requested time.

### Example 3: Composition of Simulation, CAD, and AI in One Stage

USD's composition operators (sublayers, references, payloads) allow
nondestructive assembly of data from multiple sources.
This example shows a digital twin stage that composes CAD geometry,
solver results, and AI surrogate predictions, each authored
independently and retaining its own source of truth.

```usda
#usda 1.0
(
    defaultPrim = "DigitalTwin"
    subLayers = [
        @./cad_geometry.usd@,
        @./simulation_results.usd@,
        @./ai_surrogate_fields.usd@
    ]
)

def Xform "DigitalTwin"
{
    # CAD geometry from engineering (UsdGeom in the cad layer)
    def "Geometry" (
        references = </CADModel/Body>
    )
    {
    }

    # Solver results (CaeDataSet from the simulation layer)
    def "SimulationResults" (
        references = </World/FluidDomain>
    )
    {
    }

    # AI surrogate predictions (same CaeDataSet schema contract)
    def "SurrogatePrediction" (
        references = </Surrogate/PredictedFlow>
    )
    {
    }
}
```

All three data sources compose into a single stage with no format
conversion. Consumers access solver results and surrogate predictions
through the same `CaeDataSet` interface.

## Relationship to Existing USD Work

### UsdVol Analogy

`CaeFieldArray` is intentionally parallel to `UsdVolFieldAsset`.
Both point to external data files through asset paths.
Both are bound to a container prim through `field:*` namespace
relationships.
`CaeFieldArray` generalizes this pattern beyond renderable volumes
to arbitrary scientific arrays with explicit centering metadata.

### UsdGeom

`CaeDataSet` does **not** replace `UsdGeomMesh`.
When a solver's mesh can be faithfully represented as a
`UsdGeomMesh`, that remains the appropriate choice.
`CaeDataSet` serves cases where solver native topology does not map
1:1 to UsdGeom conventions (mixed elements, polyhedra,
face centered fields, variable length connectivity) or where
preserving the solver native file as source of truth is required.

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Scope creep into visualization/rendering | This proposal covers data representation and access only. Visualization is left to consuming applications. |
| Overlap with UsdVol | `CaeFieldArray` extends the `FieldAsset` pattern to scientific arrays. The two serve different domains. |

## Alternate Approaches Considered

1. **Extend UsdVol/FieldAsset.**
   Extending UsdVol for general CAE arrays would still require
   defining dataset, array, and interpretation vocabulary.
   Doing so would overload a rendering oriented schema domain with
   non rendering semantics, creating confusion about which fields are
   renderable volumes and which are scientific data.
   A separate domain makes the boundary clear.

2. **Use UsdGeom + primvars for everything.**
   This forces mesh conversion into USD attributes, losing fidelity
   for complex topologies and creating the exact data conversion
   problem this proposal aims to solve.

3. **External only approach (no USD schemas).**
   Without USD schemas, tools lose composition, provenance, and
   ecosystem benefits.
   Each tool would still need N-by-M format integrations,
   which is what we are looking to avoid.

## Out of Scope

The following topics are explicitly outside the scope of this proposal.
Each may be addressed in future work.

- **Visualization and rendering schemas.**
  How to render CAE data (colormaps, streamlines, volume rendering)
  is application specific and not part of this proposal.

- **Solver specific data models.**
  This proposal defines a neutral vocabulary.
  Format specific subtypes follow the same extension pattern
  demonstrated by `CaeCgnsFieldArray` and can be created by any
  implementor for their format of choice.

- **Data transport protocols.**
  How data moves between solver and consumer (file I/O, gRPC, shared
  memory) is an implementation concern.
  The schemas describe what data exists and how to interpret it, not
  how to move it.

- **Write-back and simulation control.**
  This proposal addresses read access and composition of results.
  Bidirectional workflows (USD to solver) are future work.

## Backward Compatibility

All proposed types are **new** schema additions.
No existing USD types, attributes, or behaviors are modified.
Stages that do not use CAE schemas are unaffected.

A stage containing `CaeDataSet` or `CaeFieldArray` prims will load
in any USD runtime as unrecognized typed prims.
Their attributes and relationships remain accessible through generic
USD APIs.
Full semantic interpretation requires the CAE schema library to be
registered.

## Reference Implementation

A public reference implementation demonstrating the proposed schemas,
multiple format delegates, and mixed dataset composition exists:

- **Kit-CAE**
  ([GitHub](https://github.com/NVIDIA-Omniverse/kit-cae)):
  Implements the proposed schema types, data delegates for CGNS, VTK,
  HDF5, and other formats, and demonstrates cross format composition
  across CFD, structural, climate, and other scientific data.

- **Digital Twins for Fluid Simulation**
  ([GitHub](https://github.com/NVIDIA-Omniverse-blueprints/digital-twins-for-fluid-simulation)):
  A reference workflow for real time digital twins using Kit-CAE
  schemas with AI surrogate models for external aerodynamic CFD.

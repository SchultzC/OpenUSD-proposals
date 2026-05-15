# OpenUSD for Scientific Data

## Contents

- [Introduction](#introduction)
- [Motivation](#motivation)
  - [Scientific data today](#scientific-data-today)
  - [OpenUSD context](#openusd-context)
- [Problem statement](#problem-statement)
  - [Two concerns to keep separate](#two-concerns-to-keep-separate)
  - [Why this matters now](#why-this-matters-now)
- [Key questions](#key-questions)
  - [What is the minimum common vocabulary?](#what-is-the-minimum-common-vocabulary)
  - [How should scientific files participate in composition?](#how-should-scientific-files-participate-in-composition)
  - [How should vendor and domain extensions mature?](#how-should-vendor-and-domain-extensions-mature)
- [Existing OpenUSD mechanisms](#existing-openusd-mechanisms)
- [Industry use cases](#industry-use-cases)
- [Design considerations](#design-considerations)
  - [Principles](#principles)
  - [Likely direction](#likely-direction)
  - [Evaluation criteria](#evaluation-criteria)
  - [Open questions for discussion](#open-questions-for-discussion)
- [Illustrative encoding](#illustrative-encoding)
- [Relationship to implementation work](#relationship-to-implementation-work)
- [Risks](#risks)
- [Alternate approaches](#alternate-approaches)
- [Out of scope](#out-of-scope)
- [Next steps](#next-steps)

## Introduction

Scientific and engineering workflows often need to assemble simulation,
measurement, geometry, and derived results in one composed scene.
OpenUSD already provides the composition model, namespace, value resolution, and
asset-resolution machinery for this kind of assembly. What is missing is a
shared way to describe scientific datasets so that field data from solver-native
formats can be discovered and queried through USD without first being converted
into an application-specific intermediate.

This proposal reframes earlier CAE schema work around a broader scientific data
problem. The goal is not to standardize a particular vendor implementation or
OpenUSD plugin architecture. The goal is to build consensus on the data-model
concepts that scientific files need when they participate in USD composition:
datasets, fields, arrays, topology association, time, provenance, and extension
points for domain-specific meaning.

**Expected outcome.** The intended result is a small, format-neutral scientific
data vocabulary for OpenUSD and examples that can be tested against
source-format adapters exposing solver-native data as native USD attributes.
The exact schema names, property names, and governance model are intentionally
left open for discussion. This proposal seeks alignment on the problem statement
and separation of concerns first.

The diagram below captures that separation. Green elements are candidate
data-model standardization areas. Yellow elements are vendor or domain
implementation details. Blue elements are existing OpenUSD mechanisms. Gray
elements are ecosystem consumers.

![Architecture diagram showing scientific source data flowing through file-format adapters and lazy value providers into OpenUSD composition, native attributes, and a candidate scientific data vocabulary consumed by applications, analysis pipelines, visualization tools, and AI workflows.](architecture-diagram.svg)

## Motivation

### Scientific data today

Computer-aided engineering (CAE), computational fluid dynamics (CFD), finite
element analysis (FEA), electronic design automation (EDA), climate modeling,
reservoir simulation, particle simulation, and laboratory measurement workflows
all produce large scientific datasets. These datasets usually originate in
specialized source formats such as CGNS, EnSight, OpenFOAM, VTK, HDF5-based
formats, reservoir formats, or proprietary solver outputs.

The source files are often the authoritative record for details that downstream
tools need:

- mesh and element topology, including mixed or variable-size elements;
- field association, such as node-, cell-, face-, or element-centered values;
- time, iteration, ensemble, or operating-point organization;
- solver-native identifiers, names, units, and boundary condition metadata;
- provenance back to the simulation or measurement system that produced the
  data.

Because the source formats are specialized, downstream applications often
convert them into VTK, CSV, USD geometry, database tables, or private cache
formats before analysis or visualization. That conversion can be useful for a
specific workflow, but it does not provide a shared USD-level contract for
scientific data.

The conversion-first pattern can cause recurring problems:

- **Loss of fidelity.** Face-centered values may be averaged, mixed topology may
  be flattened, solver names may be sanitized without a reversible mapping, and
  source metadata may be dropped.
- **Loss of provenance.** The converted artifact can become the operational
  truth even though the solver-native result is the authoritative record.
- **Increased I/O and storage.** Large simulations are copied into additional
  representations, often only to support one consumer.
- **Fragmented integrations.** Each consuming tool may need a different reader,
  cache, or export pipeline, producing an N-by-M integration problem across
  solvers and applications.
- **Limited composition.** Simulation results, CAD, sensor streams, and
  surrogate predictions may remain in parallel data systems instead of
  composing as USD assets.

### OpenUSD context

OpenUSD can address the assembly side of this problem if scientific data is
represented through USD concepts rather than through application side channels.

In a USD-native workflow, a solver-native file can be opened, sublayered,
referenced, or payloaded like any other USD asset. Its large numeric arrays are
resolved lazily, so stage construction does not imply loading every field. Tools
that understand the scientific data vocabulary can discover fields and their
associations. Tools that do not understand the domain-specific details can still
traverse the stage and preserve opinions through composition.

The proposed shift is from application-private import and delegate APIs to
USD-native composition and value resolution. A file-format adapter may be the
implementation mechanism, but the standardization target is the data model that
appears in USD.

## Problem statement

### Two concerns to keep separate

Two related problems are often conflated:

1. **Scientific data vocabulary.** USD needs a shared way to describe datasets,
   fields, arrays, associations, time, and provenance so that scientific data is
   discoverable and composable across tools.
2. **Runtime data access implementation.** Applications and vendors need ways to
   read heavy arrays from source files, defer I/O, cache values, select subsets,
   and bind proprietary libraries.

Both are necessary, but they operate at different layers. The first is a
candidate for standardization. The second is implementation territory and should
remain extensible.

This distinction matters because OpenUSD plugins are not the same thing as a
vendor extensibility model. A vendor or domain extension should be defined at
the data-model level: what concepts appear in USD, how they compose, and how
they can mature from vendor-specific practice to multi-vendor convention. A
runtime plugin is one way to deliver that behavior in a particular USD
installation. The proposal should not require every scientific-data workflow to
share one plugin architecture, one source repository, or one application stack.

### Why this matters now

Scientific USD integrations can be implemented as native USD file-format
adapters rather than application-specific delegation or import pipelines. In
that model, field arrays can become regular USD attribute values, large data can
be loaded on demand, and solver-native files can participate directly in
composition.

Without a shared data-model vocabulary, however, each adapter will expose a
different shape:

- one reader may call a field `pressure`, another may store only the source
  variable name `P`;
- one reader may represent cell-centered data as a custom field, another as a
  primvar-like property, and another as an application cache;
- one reader may expose time samples as USD time, another may model timesteps as
  children;
- one vendor may encode source-format details in custom data while another
  defines API schemas.

That would reproduce the same integration problem inside USD that exists
outside it. The proposal therefore focuses first on the separation of concerns,
then allows vendors and domains to incubate concrete mappings without blocking
on a fully centralized standard.

## Key questions

### What is the minimum common vocabulary?

Scientific formats differ substantially, but they repeatedly describe the same
conceptual roles:

| Concept | Purpose |
|---|---|
| Dataset | A logical simulation, measurement, or derived result set that can be composed as an asset. |
| Field | A named physical or computed quantity such as pressure, velocity, stress, temperature, density, or charge. |
| Array | The typed numeric storage behind coordinates, connectivity, fields, ids, masks, or other tabular data. |
| Association | The domain over which values are defined: node, cell, face, edge, element, particle, grid sample, global, etc. |
| Topology | The structure that gives arrays spatial meaning: mesh connectivity, structured grid extents, particles, volumes, or domain-specific layouts. |
| Time and ensembles | The coordinates that select snapshots, iterations, operating points, samples, or model variants. |
| Provenance | The source file, source field name, source format concept, and optional resolver information needed for traceability. |

The open question is how much of this vocabulary must be common across all
scientific data, and how much should be provided by domain extensions such as
CFD, FEA, reservoir simulation, EDA, or geoscience.

### How should scientific files participate in composition?

USD already has mechanisms for file formats, references, payloads, sublayers,
asset resolution, and value resolution. A scientific source file can
therefore appear in a stage as a layer provided by a registered file-format
adapter. The adapter can author a structural layer and provide lazy array values
through USD attribute resolution.

This proposal does not require that all implementations use the same internal
mechanism. `SdfFileFormat` and `SdfAbstractData` are one testable path for
large, read-only scientific arrays, but the data model should not depend on one
implementation detail. The composed USD view is the contract.

### How should vendor and domain extensions mature?

Scientific data is too broad for a single first proposal to standardize every
domain-specific field, topology, and source-format mapping. The mechanism needs
to support a staged path:

- vendors and projects can ship domain-specific schemas or metadata
  conventions without central approval;
- successful patterns can converge into multi-vendor conventions;
- stable, broadly exercised concepts can become candidates for OpenUSD or AOUSD
  standardization.

This proposal assumes an incubation path: independent implementation first,
promotion after demonstrated interoperability. The important point for OpenUSD
is that the extension is a data-model commitment, not merely the existence of a
runtime plugin.

## Existing OpenUSD mechanisms

### UsdGeom

`UsdGeomMesh`, `UsdGeomPointBased`, `UsdGeomPoints`, primvars, and related
schemas are appropriate when data is being represented as renderable or
scene-editable geometry. They are not a universal scientific data contract.

Many solver-native datasets include mixed element types, polyhedral cells,
variable-length connectivity arrays, face-centered fields, ghost cells,
partitioning, or solver-specific boundary-condition concepts that do not map
1:1 to `UsdGeomMesh` without conversion. Converting everything to `UsdGeom`
also makes it difficult to preserve the source file as the authoritative
record.

`UsdGeom` should remain the right target when a faithful geometric
representation is needed. Scientific data schemas should complement it rather
than replace it.

### UsdVol

`UsdVol` already provides an important precedent: a container prim binds named
fields, and field assets can refer to external data such as OpenVDB. That
pattern is directly relevant to scientific data.

However, `UsdVol` is oriented around renderable volumetric fields. Scientific
datasets also need non-volume arrays, mesh connectivity, element association,
particle or tabular data, source-format metadata, and domain-specific topology.
Overloading `UsdVol` for all of these concepts would blur the boundary between
rendering-oriented volume data and general scientific data.

### SdfFileFormat and SdfAbstractData

OpenUSD file-format plugins can present non-USD files as USD layers.
`SdfAbstractData` gives an implementation path for backing resolved USD
attribute values with custom storage and lazy reads. Together, these mechanisms
allow a source-format adapter to expose heavy scientific arrays as standard USD
attribute values without eagerly converting or copying the whole file.

That is an implementation direction that can be evaluated in prototypes. It
should be described as an implementation mechanism, not as the standardization
target itself. The standardization target is the shape and meaning of the USD
data that appears after the adapter participates in composition.

A useful prototype pattern is a lightweight structure layer plus a lazy-value
layer that contributes array values through USD value resolution. That pattern
also makes one discovery constraint visible: consumers should not assume that
every large value appears in authored-property enumeration. Field and array
descriptors need to provide the discoverable contract, while value attributes
remain retrievable through normal USD attribute APIs.

### customData and ad-hoc metadata

`customData` can carry arbitrary metadata, including source names, field
association, units, or file paths. It is useful for private workflows, but it
does not provide a discoverable or interoperable scientific data contract.
Consumers must already know each producer's keys and conventions.

The scientific data vocabulary should avoid turning `customData` into the
primary interoperability mechanism, while still allowing custom data for
implementation-specific annotations and early incubation.

## Industry use cases

### CAE, CFD, and FEA

CAE workflows need to combine solver-native meshes, field results, CAD context,
test data, and derived quantities. A CFD stage may need to expose velocity,
pressure, temperature, turbulence quantities, cell zones, boundary patches, and
time-varying snapshots. An FEA stage may need stresses, strains, displacement,
modal data, contact regions, and element sets.

The common requirement is not one fixed topology. The common requirement is a
USD-visible contract for discovering fields, understanding where values are
defined, and composing the result with the rest of the digital product context.

### Electronics, EDA, and multiphysics

Electronics and semiconductor workflows can combine thermal,
electromagnetic, mechanical, and manufacturing data. Field data may be tied to
package geometry, board layouts, component identifiers, or extracted simulation
regions. These datasets often need source-system traceability and composition
with product structure, not just visualization.

The vendor extensibility problem is visible here: specialized formats and
proprietary libraries need a way to participate without forcing their private
details into the core vocabulary.

### Energy, geoscience, and environment

Reservoir, weather, climate, and geoscience datasets often use structured,
curvilinear, corner-point, or adaptive grids. They include time series, ensemble
members, uncertainty, cell properties, masks, and domain conventions that do not
look like traditional renderable geometry.

A scientific data vocabulary should provide enough common structure for
composition and discovery while allowing domain-specific topology APIs to carry
the details.

### AI, surrogates, and digital twins

Surrogate models and engineering workflows may need to compare predictions with
source simulation or measurement data. USD composition can place CAD, ground
truth, surrogate output, deltas, and annotations in one stage. That comparison
is easier to automate when those datasets expose fields through a common
scientific data contract rather than through application-private readers.

## Design considerations

### Principles

1. **Separation of concerns.** The scientific data model and the runtime data
   access mechanism should remain distinct. Standardization should focus on the
   composed USD data contract.

2. **USD-native composition.** Scientific source files should be able to
   participate through ordinary USD mechanisms such as sublayers, references,
   payloads, asset resolution, and time samples.

3. **Native attribute access.** When a field value is visible in USD, consumers
   should access it as a USD attribute value rather than through an
   application-private side channel.

4. **Layered schema composition.** Common scientific concepts should remain
   separable from domain- and format-specific concepts. Applied API schemas are
   one candidate mechanism: a source-format adapter could expose shared
   dataset, field, and array descriptors while preserving CGNS, VTK, reservoir,
   EDA, or vendor-specific metadata on the same prims.

5. **Lazy and scalable by design.** Opening a stage should not require loading
   every numeric array. Implementations need to be free to defer reads, cache
   selectively, and use source-format libraries.

6. **Source fidelity.** The data model should preserve source names,
   associations, topology meaning, and provenance needed for traceability or
   audit workflows, even when a consumer also derives renderable geometry.

7. **Vendor and domain extensibility.** Vendors, standards bodies, and domains
   need a way to define extensions without waiting for a core standard. Those
   extensions should have a path to converge when interoperability is proven.

8. **Minimal disruption.** The design should complement `UsdGeom`, `UsdVol`,
   asset resolution, and file-format plugins. It should not require fundamental
   changes to namespace, composition, or value resolution semantics.

9. **Application neutrality.** The proposal must not depend on Kit, Omniverse,
   or any one application. A compliant scientific-data representation should be
   useful to any OpenUSD-based consumer that loads the relevant schemas and file
   format adapters.

### Likely direction

The emerging implementation direction is to expose scientific source files as
USD layers through file-format adapters. Those adapters can author a lightweight
stage structure and provide heavy arrays as lazily resolved USD attribute
values. A small set of scientific schemas or schema-like conventions can then
describe the arrays and fields in a way that downstream tools can discover.

A minimal vocabulary likely needs:

- a dataset anchor;
- a way to describe named fields;
- a way to describe named arrays and their value attributes;
- association metadata for where values live;
- relationships or properties that describe topology and source provenance;
- time and subset-selection conventions that compose cleanly;
- typed controls for source-format selection, time mapping, cache policy, or
  streaming hints where those controls affect composition or value access.

Applied API schemas, including multiple-apply APIs for repeated field and array
descriptors, are one candidate because they are discoverable, typed, and fit
USD's existing schema model. The proposal should not prematurely standardize
current incubating names or every property shape. The first decision is the
separation of concerns and the conceptual contract.

### Evaluation criteria

A proposed scientific data vocabulary should be evaluated against concrete
scenarios:

1. **Direct composition.** A source scientific file can be opened or composed as
   a USD layer, reference, or payload without first exporting all field data to a
   separate USD geometry cache.

2. **Attribute access.** A consumer can discover a field and retrieve its values
   through normal USD attribute APIs. An implementation may load the values
   lazily, and field discovery should not rely only on enumerating authored
   heavy-value attributes, but the access path remains USD-native.

3. **Association clarity.** A consumer can determine whether values are defined
   on nodes, cells, faces, elements, particles, grid samples, or another
   declared domain-specific association.

4. **Source fidelity.** A consumer can identify the source file and source field
   name for a USD-visible field, and can distinguish source-format metadata from
   format-neutral scientific concepts.

5. **Composition behavior.** Scientific datasets can compose with CAD,
   surrogate predictions, annotations, and stronger-layer opinions without
   requiring application-private state.

6. **Implementation neutrality.** The same USD-visible contract can be produced
   by more than one source-format adapter or application.

### Open questions for discussion

1. **Vocabulary boundary.** Which concepts belong in a common scientific core,
   and which should be left to domain extensions?

2. **Schema mechanism.** Should field and array descriptors be applied schemas,
   typed prims, metadata dictionaries with accessor APIs, or a hybrid?

3. **Topology representation.** How should the common vocabulary reference
   topology without forcing all domains into one mesh model?

4. **Field association.** What is the initial token set for association, and how
   should domain-specific associations extend it?

5. **Units and physical meaning.** Should units, dimensions, and quantity kinds
   be part of the first proposal, or should they be layered as a follow-up?

6. **Discovery and indexing.** How can consumers find datasets and fields in a
   large composed stage without requiring full-stage traversal in every
   workflow?

7. **File-format arguments.** Which source-format controls should be
   represented as typed USD-authored properties, and which should remain private
   file-format arguments?

8. **Layer rooting and relocation.** What conventions should source-format
   layers use for default prims, payload composition, and sublayer relocation
   into an existing stage?

9. **Subset selection.** How should source-format subset controls such as base,
   zone, part, timestep, ensemble member, or variable selection be represented
   when the source file is referenced or payloaded?

10. **Extension governance.** What naming and maturity conventions allow vendors
   to ship extensions while keeping a path toward multi-vendor convergence?

## Illustrative encoding

The examples in this section are intentionally non-normative. They show the
kind of USD view that a scientific source-format adapter could provide. Names
such as `ScientificDataset` and `ScientificFieldAPI` are placeholders for
discussion, not proposed final schema names.

### Direct composition of a source file

If a file-format adapter for CGNS is registered, a solver-native file can
participate directly as a USD layer:

```usda
#usda 1.0
(
    defaultPrim = "World"
    subLayers = [
        @./simulation_results.cgns@
    ]
)

def Xform "World"
{
}
```

The source file remains the source of truth. The adapter determines the USD
stage structure and exposes arrays through USD value resolution. This example
does not prescribe the source layer's default prim or how a host stage relocates
source data under an existing namespace; that convention needs to be decided
explicitly.

### Expanded scientific dataset view

A consumer that opens the composed stage should see a dataset and field
descriptors through USD, with heavy values available as attributes:

```usda
def ScientificDataset "FluidDomain" (
    prepend apiSchemas = [
        "ScientificFieldAPI:pressure",
        "ScientificArrayAPI:pressure",
        "ScientificFieldAPI:velocity",
        "ScientificArrayAPI:velocity"
    ]
)
{
    asset scientific:source:file = @./simulation_results.cgns@

    uniform string scientific:field:pressure:name = "Pressure"
    uniform token scientific:field:pressure:association = "cell"
    custom float[] scientific:array:pressure:value

    uniform string scientific:field:velocity:name = "Velocity"
    uniform token scientific:field:velocity:association = "cell"
    custom float3[] scientific:array:velocity:value
}
```

The important point is that `scientific:array:*:value` is a USD attribute. A
lazy implementation may provide the value from an `SdfAbstractData`-backed
layer, but the consumer still calls the normal USD attribute API.

### Composition with CAD and surrogate output

Scientific data should compose with other USD assets instead of living in a
parallel application data model:

```usda
#usda 1.0
(
    defaultPrim = "DigitalTwin"
    subLayers = [
        @./cad_geometry.usd@
    ]
)

def Xform "DigitalTwin"
{
    def "Geometry" (
        prepend references = @./cad_geometry.usd@</CADModel/Body>
    )
    {
    }

    def "MeasuredOrSolvedResult" (
        prepend payload = @./simulation_results.cgns@
    )
    {
    }

    def "SurrogatePrediction" (
        prepend payload = @./surrogate_prediction.npz@
    )
    {
    }
}
```

Ground truth, surrogate predictions, deltas, and annotations can then be
assembled using normal USD composition arcs.

Complete illustrative files are provided in the [`examples/`](examples/)
directory.

## Relationship to implementation work

NVIDIA is incubating this direction in a separate CAE USD Plugins repository.
That work contains format-agnostic scientific schemas, domain and source-format
API schemas, and `SdfFileFormat` adapters for CAE and scientific data. Kit-CAE
can depend on those plugins in a future release, while remaining an application
that provides visualization, workflows, and derived views on top of the
USD-visible contract.

The implementation direction includes:

- format-agnostic scientific dataset, field, and array schemas;
- per-format API schemas for concepts such as CGNS zones, flow solutions, and
  unstructured element sections;
- `SdfFileFormat` adapters for source formats;
- lazy array value resolution so heavy data appears as USD attributes without
  eager conversion;
- typed controls for source-format arguments where useful for payload,
  sublayer, and UI workflows.

Existing Kit-CAE delegate and importer workflows are relevant migration
context, but they are not the intended standardization target. This work is
useful as a reference implementation and proving ground. The proposal should
standardize composed USD concepts only after they have been validated across
formats, partners, and applications.

## Risks

1. **Plugin/standard conflation.** If the proposal is read as standardizing one
   plugin stack, it will be too narrow and will not serve the broader OpenUSD
   ecosystem. The proposal needs to keep implementation mechanics separate from
   the data model.

2. **Premature schema lock-in.** Standardizing names and properties before
   enough formats have exercised them may force later incompatible revisions.
   The first phase should focus on common concepts and open questions.

3. **Scope creep.** Scientific data touches visualization, units, validation,
   analytics, solver control, provenance, and AI workflows. The core proposal
   should not try to solve all of those at once.

4. **Performance expectations.** Lazy attributes make source data accessible
   through USD, but they do not guarantee every access pattern is efficient.
   Consumers still need domain-aware algorithms, caching strategies, and
   indexing.

5. **Extension fragmentation.** If vendor and domain extensions have no naming
   or maturity guidance, the ecosystem may still fragment into incompatible
   conventions.

6. **Migration ambiguity.** During transition, plugin-backed layers and legacy
   delegate or importer stages may coexist. The proposal should keep the
   USD-visible contract independent of either migration path.

## Alternate approaches

1. **Application delegate APIs.** A delegate can provide efficient data access
   inside one application, but it makes arrays invisible to standard USD
   attribute resolution and ties workflows to that application stack. This is
   the approach this revision moves away from.

2. **Importer/exporter conversion.** Importing scientific files into USD
   geometry or cache formats can be useful for specific workflows, but it
   duplicates data and can lose source fidelity. It should remain an option,
   not the interoperability contract.

3. **Use `UsdGeom` for all data.** `UsdGeom` is appropriate for renderable or
   editable geometry, but it does not cover all scientific topology and field
   semantics without conversion.

4. **Use `UsdVol` for all fields.** `UsdVol` provides an important precedent
   for external fields, but it is not intended to describe arbitrary scientific
   arrays, mesh topology, particles, or solver metadata.

5. **Standardize every source format mapping immediately.** This would be too
   broad. Format mappings should incubate as vendor or domain extensions and
   converge as real interoperability needs emerge.

## Out of scope

- Visualization controls such as colormaps, streamlines, slicing, glyphs, and
  rendering policies.
- Solver write-back, simulation control, and generating solver input from USD.
- A single mandated runtime plugin architecture or source repository.
- Complete mappings for every scientific source format.
- Standardizing NVIDIA, Kit-CAE, or Omniverse implementation details.
- Replacing `UsdGeom`, `UsdVol`, or domain-specific schemas that already solve
  narrower problems well.

## Next steps

1. Align with OpenUSD and AOUSD reviewers on the problem statement and the
   separation between scientific data vocabulary and runtime implementation.

2. Gather concrete use cases from CAE partners and adjacent scientific domains
   to validate the minimum common vocabulary.

3. Compare schema mechanisms for field and array descriptors, with special
   attention to discoverability, composition behavior, GUI presentation,
   validation, and extension maturity.

4. Use reference plugins and partner datasets to test whether source-format
   adapters can expose native USD attributes consistently across formats.

5. Draft a follow-up solution proposal that specifies concrete schema names,
   property names, composition behavior, and extension governance after the
   conceptual boundary has consensus.

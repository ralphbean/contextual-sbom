# Contextual SBOMs

## The Problem: Flat SBOMs Hide Inheritance

Current SBOM generation tools like syft scan an entire artifact (container image, AI model, etc.) and list all components found, without distinguishing between:
- Components added at the current build layer
- Components inherited from parent artifacts

This creates several issues:
1. **Duplication**: The same components appear in multiple SBOMs across the inheritance chain
2. **Lost provenance**: No way to tell which layer introduced which component
3. **Storage bloat**: Especially problematic for deep hierarchies (AI models can have 5+ levels)
4. **False positives**: Inherited content triggers redundant vulnerability alerts

### Why Not Defer to Downstream Stitching?

One alternative is to have providers reference their immediate parent SBOMs and let downstream systems like GUAC stitch together the complete view. However, this only works if providers are willing to distribute SBOMs for **all parent artifacts transitively**.

**In practice, providers often only want to distribute their final artifact** without exposing the entire build chain. Reasons include:
- **Support surface area**: Distributing parent artifacts implies support obligations
- **Proprietary bases**: Parent images/models may be internal-only or licensed differently
- **Deployment simplicity**: Consumers just want the final artifact, not the entire build toolchain

This constraint is especially acute for **deep AI model hierarchies** where distributing 5-6 levels of parent artifacts may be impractical or undesirable.

## The Solution: Hierarchical Relationships

**Contextual SBOMs** embed parent artifact content within the child SBOM using SPDX/CycloneDX relationship types:
- `DESCENDANT_OF`: Links to parent artifact
- `CONTAINS`: Lists components at each level

This approach allows:
- ✅ Single artifact distribution (just the final child)
- ✅ Complete SBOM information (all components with provenance)
- ✅ No requirement to distribute or support parent artifacts separately
- ✅ Clear component provenance (which layer introduced what)

---

## Examples

### Example 1: Simple Container Inheritance

Container image Y builds on base image X.

#### Problem: Flat SBOM

```mermaid
graph TD
    Y["Container Y SBOM<br/>━━━━━━━━━━━━━━<br/>📦 my-binary<br/>📦 rpm-1<br/>📦 rpm-2<br/>📦 rpm-3<br/>📦 rpm-4<br/>📦 rpm-5"]

    style Y fill:#ffcccc
```

**Issues**:
- No distinction between `my-binary` (added in Y) and RPMs (from X)
- If X's SBOM exists separately, components are duplicated
- Consumers can't tell provenance of each component

#### Solution: Contextual SBOM

```mermaid
graph TD
    Y["Container Y SBOM<br/>━━━━━━━━━━━━━━<br/>📦 my-binary"]
    X["Container X SBOM<br/>━━━━━━━━━━━━━━<br/>📦 rpm-1<br/>📦 rpm-2<br/>📦 rpm-3<br/>📦 rpm-4<br/>📦 rpm-5"]

    Y -->|DESCENDANT_OF| X
    Y -->|CONTAINS| my-binary["my-binary"]
    X -->|CONTAINS| rpms["rpm-1, rpm-2,<br/>rpm-3, rpm-4, rpm-5"]

    style Y fill:#ccffcc
    style X fill:#ccffcc
    style my-binary fill:#e6f3ff
    style rpms fill:#e6f3ff
```

**Benefits**:
| Metric | Flat SBOM | Contextual SBOM |
|--------|-----------|-----------------|
| Components in Y's SBOM | 6 | 1 (+ reference to X) |
| Provenance clarity | None | Clear (Y adds my-binary) |
| Storage if both distributed | ~2x duplication | No duplication |

---

### Example 2: Multi-stage Build with Builder Image

Container Z uses a builder image for compilation but only ships the runtime result, inheriting from Y.

#### Problem: Flat SBOM

```mermaid
graph TD
    Z["Container Z SBOM<br/>━━━━━━━━━━━━━━<br/>📦 my-app<br/>📦 golang-compiler<br/>📦 build-tools<br/>📦 my-binary<br/>📦 rpm-1<br/>📦 rpm-2<br/>📦 rpm-3"]

    style Z fill:#ffcccc
```

**Issues**:
- Builder tools (golang-compiler, build-tools) listed as if they're shipped
- Can't distinguish build-time from runtime dependencies
- Misleading for security scanning (build tools aren't in final image)

#### Solution: Contextual SBOM

```mermaid
graph TD
    Z["Container Z SBOM<br/>━━━━━━━━━━━━━━<br/>📦 my-app"]
    Y["Container Y SBOM<br/>━━━━━━━━━━━━━━<br/>📦 my-binary"]
    Builder["Builder Image<br/>━━━━━━━━━━━━━━<br/>📦 golang-compiler<br/>📦 build-tools"]
    X["Container X SBOM<br/>━━━━━━━━━━━━━━<br/>📦 rpm-1, rpm-2, rpm-3"]

    Z -->|DESCENDANT_OF| Y
    Z -->|BUILD_TOOL_OF| Builder
    Y -->|DESCENDANT_OF| X

    Z -->|CONTAINS| myapp["my-app"]
    Y -->|CONTAINS| mybinary["my-binary"]
    Builder -->|CONTAINS| buildtools["golang-compiler<br/>build-tools"]
    X -->|CONTAINS| rpms["rpm-1, rpm-2, rpm-3"]

    style Z fill:#ccffcc
    style Y fill:#ccffcc
    style Builder fill:#fff3cd
    style X fill:#ccffcc
    style myapp fill:#e6f3ff
    style mybinary fill:#e6f3ff
    style buildtools fill:#fff9e6
    style rpms fill:#e6f3ff
```

**Key difference**: `BUILD_TOOL_OF` relationship means builder content is **not copied forward** into Z's shipped SBOM—it's documented but marked as build-time only.

**Benefits**:
| Metric | Flat SBOM | Contextual SBOM |
|--------|-----------|-----------------|
| Components in Z's SBOM | 7 | 1 (+ references) |
| Build vs runtime clarity | None | Explicit via relationships |
| False vulnerability alerts | High (build tools included) | Low (build tools excluded) |

---

### Example 3: Deep AI Model Hierarchy

A modelcar (OCI-formatted AI model) built from a fine-tuned model, which builds on a foundation model, which was trained on datasets. This demonstrates the **depth problem** where AI artifacts commonly have 5+ levels of inheritance.

#### Problem: Flat SBOM

```mermaid
graph TD
    Modelcar["Modelcar SBOM<br/>━━━━━━━━━━━━━━━━━━━━<br/>📦 deployment-config<br/>📦 fine-tune-data-v2<br/>📦 fine-tune-code-v2<br/>📦 fine-tune-data-v1<br/>📦 fine-tune-code-v1<br/>📦 foundation-weights<br/>📦 foundation-architecture<br/>📦 pretrain-dataset-1<br/>📦 pretrain-dataset-2<br/>📦 pretrain-dataset-3<br/>📦 dataset-source-A<br/>📦 dataset-source-B<br/>📦 dataset-source-C<br/>📦 dataset-source-D<br/><br/>Total: 14 components"]

    style Modelcar fill:#ffcccc
```

**Issues at scale**:
- **Exponential duplication**: Each level re-lists all inherited components
- **Storage explosion**: 6 levels × 14 components = massive redundancy
- **Provenance lost**: Can't tell that dataset-source-A came from level 5, not level 1
- **Intractable scanning**: Same datasets trigger alerts at every level

#### Solution: Contextual SBOM

```mermaid
graph TD
    Modelcar["Modelcar SBOM<br/>━━━━━━━━━━━━━━━━<br/>📦 deployment-config"]
    FT2["Fine-tuned v2 SBOM<br/>━━━━━━━━━━━━━━━━━━<br/>📦 fine-tune-data-v2<br/>📦 fine-tune-code-v2"]
    FT1["Fine-tuned v1 SBOM<br/>━━━━━━━━━━━━━━━━━━<br/>📦 fine-tune-data-v1<br/>📦 fine-tune-code-v1"]
    Foundation["Foundation Model SBOM<br/>━━━━━━━━━━━━━━━━━━━━━━<br/>📦 foundation-weights<br/>📦 foundation-architecture"]
    Pretrain["Pretraining Datasets SBOM<br/>━━━━━━━━━━━━━━━━━━━━━━━━<br/>📦 pretrain-dataset-1<br/>📦 pretrain-dataset-2<br/>📦 pretrain-dataset-3"]
    Sources["Dataset Sources SBOM<br/>━━━━━━━━━━━━━━━━━━━━━<br/>📦 dataset-source-A<br/>📦 dataset-source-B<br/>📦 dataset-source-C<br/>📦 dataset-source-D"]

    Modelcar -->|DESCENDANT_OF| FT2
    FT2 -->|DESCENDANT_OF| FT1
    FT1 -->|DESCENDANT_OF| Foundation
    Foundation -->|DESCENDANT_OF| Pretrain
    Pretrain -->|DESCENDANT_OF| Sources

    style Modelcar fill:#ccffcc
    style FT2 fill:#ccffcc
    style FT1 fill:#ccffcc
    style Foundation fill:#ccffcc
    style Pretrain fill:#ccffcc
    style Sources fill:#ccffcc
```

**Benefits at depth**:
| Metric | Flat SBOM | Contextual SBOM |
|--------|-----------|-----------------|
| Components in Modelcar's SBOM | 14 | 1 (+ reference chain) |
| Total components stored across chain | 84 (14 × 6 levels) | 14 (each stored once) |
| Storage savings | 0% | **83%** |
| Provenance traceability | None | Full (6 levels deep) |
| Levels of depth | Hidden | Explicit (6 levels) |

**Key insight**: As hierarchy depth increases (common in AI/ML), the benefits of contextual SBOMs grow exponentially. A 6-level hierarchy creates 83% storage savings and preserves complete provenance that would otherwise be lost.

---

## Benefits Summary

| Benefit | Container Images | AI Models |
|---------|-----------------|-----------|
| **Deduplication** | Moderate (2-3 levels) | Critical (5-6+ levels) |
| **Storage savings** | 30-50% | 70-90% |
| **Provenance clarity** | Important | Essential (complex supply chain) |
| **Support surface reduction** | Nice-to-have | Business critical (avoid distributing proprietary bases) |
| **Vulnerability management** | Fewer false positives | Dramatically fewer false positives |

## References

- [KONFLUX-3966: Contextual SBOMs for container images](https://docs.google.com/document/d/1az0hrVtEZG-VKYDD2OA1_u0nqNwoRqg1ofR5On8fs74/edit?tab=t.0#heading=h.nk0gqur98cd3)
- [SPDX Relationships Specification](https://spdx.github.io/spdx-spec/v2.3/relationships-between-SPDX-elements/)
- [CycloneDX Dependency Graph](https://cyclonedx.org/use-cases/#dependency-graph)

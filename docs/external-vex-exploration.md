# External VEX Ingestion — Exploration Notes

> Working notes for the LFX project: **External VEX Ingestion for Kubescape Vulnerability Scanning**.
>
> This document records findings from codebase exploration. It intentionally separates **confirmed observations** from assumptions and open questions so the design can be discussed with the mentor before implementation.

## Project Goal

Kubescape/kubevuln currently produces vulnerability scan results and can work with VEX information, but the project is to add support for **ingesting external/vendor VEX sources** and using those statements when evaluating vulnerability findings.

The intended high-level flow we are investigating is:

```text
External VEX source
        |
        v
     VEXSource
        |
        v
 fetch / validate / parse
        |
        v
 normalize + persist VEX
        |
        v
 vulnerability scan results
        |
        v
 match VEX statements
        |
        v
 suppress / annotate applicable findings
        |
        v
 user-facing vulnerability output
```

The exact integration point is **not decided yet**.

---

## 1. Repository Structure

The repository is organized into several major areas relevant to this project:

- `core/` — core domain models, ports, and scanning services.
- `controllers/` — Kubernetes/controller logic; important for investigating `WatchHandler`, `CooldownQueue`, and the eventual `VEXSource` controller.
- `repositories/` — persistence/storage implementations.
- `api/` — Kubernetes API/CRD definitions; important when designing `VEXSource`.
- `adapters/` — integrations with scanning/external systems.
- `config/` — Kubernetes configuration and generated/deployment resources.
- `cmd/` — application entry points.
- `docs/` — project documentation and these exploration notes.

For the first phase, the most important areas are `core/`, `controllers/`, `repositories/`, and `api/`.

---

## 2. Current Vulnerability Domain Model

### `core/domain/cve.go`

The file defines severity constants:

- `Critical`
- `High`
- `Medium`
- `Low`
- `Negligible`
- `Unknown`

It also defines:

```go
type CVEExceptions []armotypes.VulnerabilityExceptionPolicy
```

and the main structure currently observed:

```go
type CVEManifest struct {
    Content            *v1beta1.GrypeDocument
    Annotations        map[string]string
    Labels             map[string]string
    Name               string
    Wlid               string
    SBOMCreatorName    string
    SBOMCreatorVersion string
    CVEScannerName     string
    CVEScannerVersion  string
    CVEDBVersion       string
}
```

### Important observation

`CVEManifest` does **not** directly contain fields such as CVE ID, package name, package version, etc. Instead, the actual Grype report is held in:

```text
CVEManifest
    |
    +-- Content: *v1beta1.GrypeDocument
                 |
                 +-- actual Grype vulnerability report
```

Therefore, to understand how VEX matching should work, we need to inspect the `GrypeDocument` structure and trace how `CVEManifest.Content` is consumed by the scan pipeline.

### Why this matters for external VEX

External VEX statements will need to be matched against information already present in Kubescape's vulnerability results. The exact matching keys (for example CVE, package, version, product, image/artifact) have **not yet been confirmed from the code**.

---

## 3. Scan Flow — To Be Traced

The next files to inspect are:

1. `core/domain/scan.go`
2. `core/domain/sbom.go`
3. `core/services/scan.go`
4. `core/ports/repositories.go`
5. `core/ports/services.go`

Questions to answer:

- Where is `ImageScanData` defined?
- How does an image enter the scan pipeline?
- How is the SBOM connected to the vulnerability scan?
- Where is the `GrypeDocument` created/populated?
- How are individual vulnerability matches represented?
- Where is `CVEManifest` created?
- Where is `VulnerabilityManifest` created/stored?
- Where is `VulnerabilityManifestSummary` created/stored?
- At what point are existing exceptions/filtering applied?
- What is the best point to introduce external VEX matching?

---

## 4. Existing VEX Support — To Be Traced

The repository already has VEX-related functionality. The next investigation should locate:

- `OpenVulnerabilityExchangeContainer`
- `StoreVEX`
- creation of VEX statements
- storage/retrieval of VEX documents
- any existing VEX matching/evaluation logic

Important question:

> Can external VEX be represented using the existing VEX storage/data model, or should it use a scoped sibling representation?

No decision has been made yet.

---

## 5. Controller / Refresh Architecture — To Be Traced

The external VEX source is expected to be represented by a Kubernetes resource (`VEXSource`). Before designing the controller, investigate:

- `WatchHandler`
- `CooldownQueue`
- existing controllers that periodically/repeatedly fetch external data
- reconciliation/update patterns
- error handling and retry behaviour
- how fetched data is persisted

Questions:

- What causes a refresh?
- How is the cooldown enforced?
- How are failed fetches handled?
- Where should parsing/validation happen?
- How should a `VEXSource` be scoped to images/workloads?

---

## 6. SecurityException Relationship — To Be Investigated

The project should compose with the existing SecurityException mechanism but should not require users to manually author exceptions for vendor VEX statements.

Investigate the existing SecurityException matching path because it may provide a useful model for:

- matching a vulnerability to a scope/selector
- filtering/suppressing findings
- preserving justification/context
- handling precedence or conflicts

The goal is to **reuse existing concepts where appropriate without coupling external VEX ingestion to user-authored SecurityExceptions**.

---

## 7. Current Understanding of the Desired Architecture

At this stage, the architecture is a hypothesis rather than a final design:

```text
             +----------------+
             |   VEXSource    |
             |      CRD       |
             +-------+--------+
                     |
                     v
             +----------------+
             | VEX controller |
             +-------+--------+
                     |
               fetch + validate
                     |
                     v
             +----------------+
             | parse/normalize |
             | OpenVEX / CSAF |
             +-------+--------+
                     |
                     v
             +----------------+
             | VEX persistence |
             +-------+--------+
                     |
                     v
+------------- vulnerability scan -------------+
|                                               |
| image -> SBOM -> Grype -> vulnerability data |
|                                               |
+----------------------+------------------------+
                       |
                       v
                VEX matching/join
                       |
          +------------+------------+
          |                         |
     applicable                 no match
     statement                     |
          |                         |
          v                         v
   suppress/annotate            unchanged
          |
          +------------+------------+
                       v
                final output
```

The exact placement of the matching/join step and the persistence model must be determined from the existing code.

---

## 8. Questions for Mentor / Community

These should only be raised after checking the existing implementation where possible:

1. Should external VEX be stored in the existing `OpenVulnerabilityExchangeContainer` representation?
2. Should the `VEXSource` be namespaced and scoped to images, workloads, or both?
3. What should happen when multiple VEX sources make different statements about the same CVE/image?
4. Which VEX statuses are authoritative for suppression (`not_affected`, `fixed`, and potentially others)?
5. What provenance should be retained in the vulnerability output (source URL, document ID, statement ID, justification)?
6. Should external VEX affect the underlying vulnerability manifest, only the summary, or the final output layer?
7. What should the refresh/caching semantics be for large vendor feeds?

---

## 9. Confirmed vs. Unconfirmed

### Confirmed from code so far

- `CVEManifest` is defined in `core/domain/cve.go`.
- `CVEManifest.Content` is a `*v1beta1.GrypeDocument`.
- `CVEManifest` contains scanner/SBOM metadata and Kubernetes-style labels/annotations.
- Vulnerability details therefore need to be traced through the Grype document rather than assumed to be fields on `CVEManifest`.

### Not yet confirmed

- Exact vulnerability-match structure.
- Exact `ImageScanData` structure and scan entry point.
- Exact `VulnerabilityManifest`/`VulnerabilityManifestSummary` flow.
- Existing VEX storage and evaluation path.
- Exact `WatchHandler`/`CooldownQueue` behaviour.
- Best integration point for external VEX matching.
- Conflict resolution between multiple external VEX sources.

---

## 10. Next Exploration Step

Inspect `core/domain/scan.go` and `core/domain/sbom.go`, then trace their usage into `core/services/scan.go`.

The objective is to produce a verified call/data flow before implementing anything.

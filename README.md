# Vortex

**Vortex is the kernel.** Not the platform, not the extensions, not the end-user product.

- **Kernel**: this repository — run lifecycle, ports/contracts, policy, state, event bus, extension supervisor.
- **Extensions**: `ext-software`, `ext-os`, `ext-hardware` live in separate repositories.
- **Product**: the family-facing product is a separate project.

Status: **design only.** No implementation code yet.

## Docs

| Doc | What |
|---|---|
| [docs/scope.md](docs/scope.md) | Kernel-only scope, in/out, definition of done |
| [docs/architecture.md](docs/architecture.md) | Runtime, state machine, security, supply chain, CVE baseline |
| [docs/implementation-plan.md](docs/implementation-plan.md) | Milestones, WBS, acceptance, risks |
| [docs/kernel-ports.md](docs/kernel-ports.md) | Ports/adapters, extension classes |
| [docs/howto.md](docs/howto.md) | Build, TLS, PQC envelope, sandbox, backup |
| [docs/robotics.md](docs/robotics.md) | Embodied extension contract |
| [docs/hardware.md](docs/hardware.md) | Motors, joints, IO, bring-up |
| [docs/world-interface.md](docs/world-interface.md) | Sensor-to-LLM world state |
| [docs/knowledge-layer.md](docs/knowledge-layer.md) | Open-world knowledge: OWL-compatible schema, RDFS/OWL RL subset |

## Naming

```text
repo:        vortex
daemon:      vortexd
cli:         vortexctl
conformance: vortex-conformance
extensions:  ext-software.* / ext-os.* / ext-hardware.*
```

## Non-goals

No device drivers, no UI, no multi-tenant platform, no Kubernetes, no package manager, no full OWL DL reasoner inside the kernel.

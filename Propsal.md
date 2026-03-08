# GSoC 2026 Proposal: HashiCorp Nomad Integration for Metaflow

**Project:** Metaflow Nomad Integration  
**Organization:** Netflix / Metaflow  
**Contributor:** Mustafa Hussain ([Mustafa11300](https://github.com/Mustafa11300))  
**Mentor:** Madhur Tandon
**Project Size:** Large (350 hours)  
**Difficulty:** Hard  

---

## 1. Abstract

This project extends Metaflow's compute backend ecosystem by adding a `@nomad` decorator that enables users to execute Metaflow steps as [HashiCorp Nomad](https://www.nomadproject.io/) batch jobs. The implementation follows the proven architecture of the existing [`@slurm` extension](https://github.com/outerbounds/metaflow-slurm), replacing SSH-based job submission with Nomad's HTTP API via the `python-nomad` library. This integration serves organizations using Nomad as their workload orchestrator, particularly those in the HashiCorp ecosystem (Vault, Consul, Terraform) and edge/hybrid deployment scenarios where Kubernetes overhead is undesirable.

**Proof of Concept:** I have already built and published a working extension that registers the `@nomad` decorator in Metaflow's plugin system:
- **Repository:** [github.com/Mustafa11300/metaflow-nomad](https://github.com/Mustafa11300/metaflow-nomad)  
- **Feature Request:** [Netflix/metaflow#2988](https://github.com/Netflix/metaflow/issues/2988)

---

## 2. Motivation & Problem Statement

### Why Nomad?

Metaflow currently supports **Kubernetes** (`@kubernetes`), **AWS Batch** (`@batch`), and **Slurm** (`@slurm`) as remote compute backends. However, a significant class of organizations rely on **HashiCorp Nomad** for workload orchestration:

- **HashiCorp ecosystem users** who already deploy infrastructure with Terraform, manage secrets with Vault, and use Consul for service discovery. For these teams, Nomad is the natural compute layer.
- **Edge/hybrid deployments** where Kubernetes' complexity and resource overhead are prohibitive. Nomad's single-binary architecture supports mixed cloud, on-prem, and edge environments.
- **Multi-scheduler environments** where different teams use different orchestrators. Adding Nomad support lets Metaflow serve as a unified ML workflow layer across all of them.

### Current Gap

Without `@nomad`, organizations on Nomad must either:
1. Deploy and maintain a parallel Kubernetes cluster solely for Metaflow workloads
2. Use local execution only, losing the benefits of remote compute, scaling, and production deployment
3. Build custom, one-off integration scripts that don't benefit from Metaflow's lifecycle management

### Impact

Adding Nomad support makes Metaflow accessible to the entire HashiCorp ecosystem community, enabling data scientists at Nomad-centric organizations to use the same `@nomad` decorator pattern they would use with `@kubernetes` or `@batch` - zero learning curve.

---

## 3. Prior Work & Preparation

### Technical Groundwork Completed

| Activity | Status | Evidence |
|----------|--------|----------|
| Local Nomad cluster setup | ✅ Done | Running `nomad agent -dev` with Docker driver |
| Docker job submission via HCL | ✅ Done | Tested batch jobs with Docker task driver |
| `python-nomad` API testing | ✅ Done | Job submit, status poll, log retrieval, termination |
| Nomad failure handling | ✅ Done | Tested non-zero exits, OOM kills, allocation failures |
| `@slurm` extension analysis | ✅ Done | Full code review of decorator, client, job, CLI patterns |
| Working PoC extension | ✅ Done | [metaflow-nomad](https://github.com/Mustafa11300/metaflow-nomad) - 17 files, installs via pip |
| Extension registered in Metaflow | ✅ Done | `@nomad` appears in `STEP_DECORATORS` alongside `@kubernetes`, `@batch` |
| Feature request filed | ✅ Done | [Netflix/metaflow#2988](https://github.com/Netflix/metaflow/issues/2988) |

### Architecture Understanding

The `@slurm` extension provided the architectural blueprint. I've mapped every component to its Nomad equivalent:

| Slurm Component | Nomad Equivalent | Key Difference |
|---|---|---|
| `SlurmClient` (SSH + asyncssh) | [NomadClient](file:///Users/mustafahussain/metaflow-nomad/metaflow_extensions/nomad_ext/plugins/nomad/nomad_client.py#8-226) (HTTP API + python-nomad) | Stateless HTTP vs stateful SSH sessions |
| `sbatch` script submission | `POST /v1/jobs` API call | JSON spec vs bash script |
| `squeue` polling | `GET /v1/job/:id/allocations` | Allocation lifecycle vs job state |
| SSH + `tail` for logs | `GET /v1/client/fs/logs/:alloc_id` | Direct API vs file tailing |
| `scancel` | `DELETE /v1/job/:id` | Both are idempotent |
| `SLURM_JOB_ID` | `NOMAD_ALLOC_ID` | Nomad exposes richer env vars |

---

## 4. Project Goals & Deliverables

### Primary Deliverables (Must-Have)

1. **`@nomad` Step Decorator** - [NomadDecorator(StepDecorator)](file:///Users/mustafahussain/metaflow-nomad/metaflow_extensions/nomad_ext/plugins/nomad/nomad_decorator.py#16-201) that hooks into Metaflow's lifecycle to redirect step execution to Nomad batch jobs

2. **Nomad HTTP Client** - [NomadClient](file:///Users/mustafahussain/metaflow-nomad/metaflow_extensions/nomad_ext/plugins/nomad/nomad_client.py#8-226) wrapping `python-nomad` for job submission, status polling, log retrieval, and job termination

3. **Job Specification Generator** - [NomadJob](file:///Users/mustafahussain/metaflow-nomad/metaflow_extensions/nomad_ext/plugins/nomad/nomad_job.py#14-239) class that generates Nomad batch job specs with Docker task driver, resource constraints (CPU/memory), environment variable propagation, and command construction

4. **CLI Integration** - Click-based CLI commands (`nomad step`) for the extension's entry point, handling parameter passing and exit code propagation

5. **Configuration System** - Environment variable-based configuration (`METAFLOW_NOMAD_ADDRESS`, `METAFLOW_NOMAD_TOKEN`, `METAFLOW_NOMAD_REGION`, `METAFLOW_NOMAD_NAMESPACE`, `METAFLOW_NOMAD_DOCKER_IMAGE`)

6. **End-to-End Testing** - Test suite covering job submission, completion, failure handling, log streaming, and retry integration using `nomad agent -dev`

7. **Documentation** - README with installation, configuration, usage examples, and architecture overview

### Stretch Goals (Nice-to-Have)

8. **Nomad `exec` driver support** - Run steps as native executables without Docker
9. **GPU resource allocation** - Leverage Nomad's device plugin system for GPU scheduling
10. **Consul Connect integration** - Service mesh support for secure inter-task communication
11. **Nomad ACL integration** - Full ACL token lifecycle management

---

## 5. Technical Design

### 5.1 Extension Architecture

```
metaflow-nomad/
├── setup.py
├── README.md
└── metaflow_extensions/
    └── nomad_ext/
        ├── config/mfextinit_nomad_ext.py      # Configuration (env vars)
        ├── plugins/mfextinit_nomad_ext.py      # Plugin registration
        ├── plugins/nomad/
        │   ├── nomad_decorator.py              # StepDecorator lifecycle hooks
        │   ├── nomad_client.py                 # HTTP API wrapper
        │   ├── nomad_job.py                    # Job spec & lifecycle
        │   ├── nomad.py                        # Orchestration layer
        │   ├── nomad_cli.py                    # Click CLI commands
        │   └── nomad_exceptions.py             # Custom exceptions
        └── toplevel/
            ├── mfextinit_nomad_ext.py
            └── toplevel.py                     # Extension metadata
```

### 5.2 Decorator Lifecycle Flow

```
User writes @nomad on a step
         │
         ▼
step_init() ─────── Validate: no @parallel, store references
         │
package_init() ──── Verify python-nomad is installed
         │
runtime_init() ──── Store flow, graph, package, run_id
         │
runtime_task_created() ── Upload code package to datastore
         │
runtime_step_cli() ── Redirect CLI to "nomad step" command
         │
         ▼
   nomad_cli.py "nomad step" executes:
   1. Build Nomad job spec (Docker, resources, env vars)
   2. Submit job via POST /v1/jobs
   3. Wait for allocation → running → complete
   4. Retrieve logs and exit code
   5. Propagate exit code to Metaflow
         │
         ▼
task_pre_step() ─── (Inside Nomad container) Register metadata
         │
task_finished() ─── Sync local metadata to datastore
```

### 5.3 Key Technical Decisions

**1. HTTP API vs CLI wrapping**  
Using `python-nomad` library for HTTP API calls instead of shelling out to [nomad](file:///Users/mustafahussain/metaflow-nomad/metaflow_extensions/nomad_ext/plugins/nomad) CLI. This provides better error handling, structured responses, and doesn't require the Nomad binary on the client machine.

**2. Docker task driver as default**  
Nomad supports multiple task drivers (Docker, exec, raw_exec, Java). Docker is chosen as the default because it provides environment isolation consistent with `@kubernetes` and `@batch`, and is the most commonly used driver.

**3. Lazy class inheritance for circular import resolution**  
The decorator module is loaded during Metaflow's plugin resolution phase, before `metaflow.__init__` is complete. To avoid circular imports, [NomadDecorator](file:///Users/mustafahussain/metaflow-nomad/metaflow_extensions/nomad_ext/plugins/nomad/nomad_decorator.py#16-201) is initially defined as inheriting from `object`, with the base class patched to `StepDecorator` at first instantiation.

**4. Allocation-based lifecycle tracking**  
Unlike Kubernetes pods or Slurm jobs, Nomad uses an evaluation → allocation → task lifecycle. The client tracks the allocation ID (not just job ID) for accurate status monitoring and log retrieval.

---

## 6. Timeline (12 Weeks)

### Phase 1: Foundation (Weeks 1-2)

| Week | Tasks |
|------|-------|
| **Week 1** | Refine PoC based on mentor feedback. Fix any issues discovered during initial review. Set up CI with GitHub Actions (lint, basic import tests). |
| **Week 2** | Ensure end-to-end flow execution works: `python flow.py run` → Nomad batch job → completion. Fix command construction and mflog integration. |

**Milestone 1:** A simple `@nomad` flow runs end-to-end on `nomad agent -dev`.

### Phase 2: Core Robustness (Weeks 3-5)

| Week | Tasks |
|------|-------|
| **Week 3** | Implement proper error handling: allocation failures, OOM kills, non-zero exits. Map Nomad error states to Metaflow exceptions. |
| **Week 4** | Integrate with `@retry` decorator. Implement timeout handling (`run_time_limit`). Add job cleanup on task cancellation. |
| **Week 5** | Log streaming: implement real-time log tailing from Nomad allocations using the FS logs API. Integrate with mflog for structured log capture. |

**Milestone 2:** Robust job lifecycle management with proper error handling, retry, and log streaming.

### Phase 3: Testing & Integration (Weeks 6-8)

| Week | Tasks |
|------|-------|
| **Week 6** | Write integration test suite using `nomad agent -dev`. Test: basic flow, foreach steps, error handling, retry, log capture. |
| **Week 7** | Test with multiple datastores (local, S3). Test metadata sync. Validate with Metaflow's existing QA test patterns. |
| **Week 8** | Test multi-step flows, branching, joins. Ensure artifact passing works correctly through the datastore. |

**Milestone 3:** Comprehensive test suite passing on local Nomad dev cluster.

### Phase 4: Polish & Documentation (Weeks 9-10)

| Week | Tasks |
|------|-------|
| **Week 9** | Stretch goal: `exec` driver support. Configuration validation and user-friendly error messages. |
| **Week 10** | Complete documentation: README, architecture guide, configuration reference. Create example flows for common use cases. |

**Milestone 4:** Production-ready extension with full documentation.

### Phase 5: Release & Stretch (Weeks 11-12)

| Week | Tasks |
|------|-------|
| **Week 11** | Stretch goal: GPU support via Nomad device plugins. Publish to PyPI. Submit PR to Netflix/metaflow for official listing. |
| **Week 12** | Final testing, code review incorporation, write blog post / demo video. Buffer for any remaining work. |

**Final Milestone:** Published PyPI package, merged PR, and documentation.

---

## 7. About Me

**Name:** Mustafa Hussain  
**GitHub:** [Mustafa11300](https://github.com/Mustafa11300)  
**Email:** mustafa05121@gmail.com 
**Timezone:** IST (UTC+5:30)  
**University:** Ajay Kumar Garg Engineering College 
**Degree:** B.Tech CSE-DS(3rd Year) 

### Relevant Experience

- **Open Source Contributions:**
  - [mofa-org/mofa PR #992](https://github.com/mofa-org/mofa/pull/992) - Unified GlobalPromptRegistry error handling (Rust/Go)
  - [Netflix/metaflow#2988](https://github.com/Netflix/metaflow/issues/2988) - Nomad integration feature request with working PoC

- **Technical Skills:**
  - **Languages:** Python, Go, Rust, JavaScript
  - **Infrastructure:** Docker, HashiCorp Nomad/Vault/Consul/Terraform, Kubernetes basics
  - **Tools:** Git, GitHub Actions, pytest, Click CLI framework

- **Metaflow-Specific Knowledge:**
  - Deep understanding of Metaflow's extension system (namespace packages, plugin resolution, StepDecorator lifecycle)
  - Studied and implemented the `@slurm` extension architecture pattern
  - Built a working `@nomad` extension that registers in Metaflow's plugin system
  - Familiar with Metaflow's mflog, datastore, and metadata provider subsystems

### Availability

- **Hours per week:** 15-20 hours (medium project)
- **Other commitments:** [mention any exams, jobs, etc.]
- **Communication:** Available on Metaflow Slack, GitHub, and email. Can adapt to mentor's preferred timezone for sync meetings.

---

## 8. Why Me?

1. **Working proof of concept** - I haven't just researched this project; I've built it. The [metaflow-nomad](https://github.com/Mustafa11300/metaflow-nomad) extension is installable via pip and successfully registers the `@nomad` decorator in Metaflow's plugin system.

2. **Deep architectural understanding** - I've read and understood the `@slurm` extension source code line by line, mapped every component to its Nomad equivalent, and solved non-trivial integration challenges (circular import resolution during plugin loading).

3. **Practical Nomad experience** - I've set up local Nomad clusters, submitted Docker batch jobs via both HCL files and the Python API, and tested failure scenarios (non-zero exits, OOM kills, allocation failures).

4. **Track record of shipping** - My [mofa PR #992](https://github.com/mofa-org/mofa/pull/992) demonstrates I can contribute to production open-source codebases with proper error handling patterns.

---

## 9. References

- [Metaflow Documentation](https://docs.metaflow.org/)
- [Metaflow Extension Template](https://github.com/Netflix/metaflow-extensions-template)
- [Metaflow Slurm Extension](https://github.com/outerbounds/metaflow-slurm) (architectural reference)
- [HashiCorp Nomad API Documentation](https://developer.hashicorp.com/nomad/api-docs)
- [python-nomad Library](https://pypi.org/project/python-nomad/)
- [My PoC: metaflow-nomad](https://github.com/Mustafa11300/metaflow-nomad)
- [Feature Request: Netflix/metaflow#2988](https://github.com/Netflix/metaflow/issues/2988)

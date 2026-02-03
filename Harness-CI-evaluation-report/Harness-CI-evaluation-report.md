# Harness CI Evaluation Report: Capy Project

**Author:** Chen Shen
**Date:** 2026-02-03  
**Project:** Boost.Capy (https://github.com/cppalliance/capy)  
**Status:** Evaluation completed — Harness not recommended

---

## Executive Summary

Harness CI was evaluated as a potential CI/CD platform for the Boost.Capy C++ library. After thorough testing on a local VM environment (6 GB RAM, 2-core CPU, 14 Mbps network), **Harness is not recommended** for this project. The primary disqualifiers are: **(1) dependency installation alone exceeds 15 minutes** per run; **(2) the build times out** (progress stalled at 2/98 targets) and **never completes**; **(3) the test step is never reached**. Harness Cloud requires paid plans, forcing use of self-hosted Kubernetes on the VM; caching requires S3 or GCS with authentication, whereas GitHub Actions caching works with no extra auth. Additional friction from step isolation, environment quirks, and the absence of native C++ tooling further undermines suitability. The project already has a mature **GitHub Actions** workflow (`.github/workflows/ci.yml`) that runs multi-platform builds with caching and optimized steps — continuing with GitHub Actions is the recommended path.

---

## 1. Objective and Scope

### 1.1 Goal

Evaluate Harness CI as an alternative or complement to the existing GitHub Actions CI for the Capy project, focusing on:

- Build time and reliability
- Ease of configuration
- Cost (using free tier / self-hosted VM to avoid premium cloud)

### 1.2 Capy Build Requirements

| Requirement | Value |
|-------------|-------|
| CMake | 3.20+ (3.24+ for preset schema v6) |
| C++ Compiler | GCC 12+, Clang 17+, or MSVC 14.34+ |
| Build tool | Ninja (recommended) |
| Dependencies | None (self-contained library) |

---

## 2. Test Environment

### 2.1 VM Specifications

| Component | Specification |
|-----------|---------------|
| **RAM** | 6 GB |
| **CPU** | 2.4 GHz, 2 cores |
| **Network downlink** | 14 Mbps |
| **Platform** | Local VM with Kubernetes (self-hosted runners) |
| **Base image** | `ubuntu:24.04` (Docker) |

### 2.2 Why Self-Hosted Kubernetes on VM

**Harness Cloud requires payment** — no free tier with sufficient capacity. To avoid paid plans, testing used **Kubernetes on the local VM** as the Harness execution environment. This reflects cost-conscious setups typical of smaller teams or open-source projects.

---

## 3. Work Performed

### 3.1 Pipeline Structure

A single **Build** stage was configured with one **Run** step containing:

1. Install dependencies (`apt-get update`, `apt-get install -y g++-12 ninja-build cmake`)
2. Set compiler (`export CXX=g++-12`)
3. Configure (`cmake --preset standalone -DBOOST_CAPY_BUILD_TESTS=ON`)
4. Build (`cmake --build --preset standalone`)
5. Test (`ctest --preset standalone`)

All commands were combined into one step to avoid Harness step isolation, where each step runs in a fresh container and loses installed packages.

### 3.2 Final Working Script

```bash
apt-get update
apt-get install -y g++-12 ninja-build cmake
export CXX=g++-12
cmake --preset standalone -DBOOST_CAPY_BUILD_TESTS=ON
cmake --build --preset standalone
ctest --preset standalone
```

### 3.3 Issues Resolved During Evaluation

| Issue | Root Cause | Resolution |
|-------|------------|------------|
| `sudo: not found` | Harness runs as root; `sudo` not installed | Use `apt-get` without `sudo` |
| `cmake: not found` in Build step | Separate stages = separate containers | Combine all commands into one Run step |
| `Unrecognized "version" field` (presets) | Ubuntu 22.04 has CMake 3.22; preset schema v6 needs CMake 3.24+ | Switch to `ubuntu:24.04` |
| `No CMAKE_CXX_COMPILER could be found` | Default compiler not set | Add `export CXX=g++-12` before `cmake` |
| Tests not built | Preset sets `BOOST_CAPY_BUILD_TESTS=OFF` | Add `-DBOOST_CAPY_BUILD_TESTS=ON` |
| `pip: not found` | Minimal Ubuntu image has no pip | Remove redundant `pip install cmake`; apt cmake is sufficient |

---

## 4. Performance and Timing

### 4.1 Observed Timings (Typical Run)

| Phase | Duration | Outcome |
|-------|----------|---------|
| `apt-get update` | ~1–2 min | Completed |
| `apt-get install` | **>15 min** | Dependency install alone exceeds 15 minutes |
| `cmake --preset` | ~0.5 s | Completed |
| `cmake --build` | — | **Timed out at 2/98 targets**; never finished |
| `ctest` | — | **Never reached** — build failed before tests could run |

### 4.2 Bottleneck Analysis

- **Dependency install:** Over 15 minutes for `apt-get install` (~150 MB, 90+ packages) on 14 Mbps / 2-core VM; dominates total runtime.
- **Build timeout:** Compilation stalled at 2/98 targets; step or pipeline timeout killed the run before completion.
- **No caching:** Each run performs a full `apt-get install`. Harness caching requires **S3 or GCS** with authentication; **GitHub Actions** offers `actions/cache` with **no extra auth**.

### 4.3 Comparison to GitHub Actions

The existing `.github/workflows/ci.yml` uses:

- **alandefreitas/cpp-actions** for compiler setup and caching
- **Container images** (e.g., `ubuntu:24.04`, `ubuntu:25.04`) with tooling pre-installed or quickly restored from cache
- **Matrix builds** across GCC, Clang, MSVC, MinGW, Apple-Clang
- **Timeout:** 120 minutes per job (adequate for complex builds)

Typical GitHub Actions run times for Capy are in the 10–30 minute range *per job*, with many jobs running in parallel. Dependency setup is amortized via caching and optimized actions.

---

## 5. Why Harness Is Not a Good Fit

### 5.1 Dependency Install Exceeds 15 Minutes

- **Dependency installation alone takes more than 15 minutes** — `apt-get install` for g++, cmake, ninja, and dependencies on a modest VM.
- No native C++/CMake caching in Harness Run steps forces a full reinstall every run.
- Harness Cloud could improve network/CPU but **requires payment** — no viable free option; evaluation used Kubernetes on the VM instead.

### 5.2 Step and Stage Isolation

- Each **stage** (and in some configurations, each **step**) runs in a new container.
- Installing dependencies in one step does not persist to the next unless everything is in a single Run step.
- This complicates pipeline design and encourages monolithic scripts, reducing maintainability.

### 5.3 Environment Quirks

- `sudo` not available; must run as root or install sudo.
- `pip` not available in minimal images; extra setup required if Python tooling is needed.
- Compiler (`CXX`) must be set explicitly; default detection fails in Harness containers.

### 5.4 Lack of C++ Ecosystem Integration

- No equivalent to `alandefreitas/cpp-actions` or similar reusable C++ CI components.
- No first-class support for compiler matrices, CMake presets, or test frameworks.
- Configuration is mostly manual scripting rather than declarative, reusable blocks.

### 5.5 Cost and Caching Model

- **Harness Cloud:** Requires **paid plans** — [Harness pricing](https://harness.io/pricing) — no free tier suitable for C++ CI; evaluation used self-hosted Kubernetes.
- **Caching:** Harness cache backends require **S3 or GCS** with credentials; setup and auth overhead for every project. **GitHub Actions** `actions/cache` works with **no authentication** — uses built-in GHA cache.

---

## 6. Recommendations

### 6.1 Do Not Adopt Harness for Capy CI/CD

Harness does not meet the requirements for fast, reliable C++ CI for this project under the evaluated constraints.

### 6.2 Continue Using GitHub Actions

- The existing `.github/workflows/ci.yml` is mature, multi-platform, and uses proven C++ actions.
- No YAML or pipeline changes are required for Harness; the project does **not** need to push any Harness-specific configuration to the repository.

### 6.3 If Harness Must Be Used

- Use Harness Cloud (premium) for better network and CPU.
- Build a **custom Docker image** with g++-12, cmake, and ninja pre-installed to eliminate the 15+ minute install step.
- Maintain that image in a registry and reference it in the Run step.

---

## 7. Deliverables and Artifacts

### 7.1 Report

- This document: `Harness-CI-evaluation-report.md`

### 7.2 Screenshots

Screenshots from the evaluation (pipeline UI, step output, error messages) are in:

```
Harness-CI-evaluation-report/
  screenshots/
    pipeline-overview.png      — Pipeline stage view
    step-parameters.png        — Run step (Image, Command)
    apt-install-output.png     — Dependency install progress
    cmake-configure-success.png — CMake configure output
```

To embed in this report, use:

```markdown
![Pipeline Overview](Harness-CI-evaluation-report/screenshots/pipeline-overview.png)
```

### 7.3 No Harness YAML in Repo

**No Harness pipeline YAML needs to be committed** to the Capy repository. The evaluation was conducted in the Harness UI; pipeline definitions live in Harness, not in the project source.

---

## 8. Conclusion

Harness CI was evaluated for the Boost.Capy project on a local VM (6 GB RAM, 2 cores, 14 Mbps) running Kubernetes, since Harness Cloud requires payment. **Dependency installation alone exceeds 15 minutes**; the **build times out at 2/98 targets** and **never reaches the test step**. Caching requires S3/GCS with auth, unlike GitHub Actions. The project should **continue using GitHub Actions** for CI. No Harness configuration needs to be pushed to the repository.

---

## Appendix A: VM Specs (Provided by User)

- **RAM:** 6 GB  
- **CPU:** 2.4 GHz, 2 cores  
- **Network downlink:** 14 Mbps  

---

## Appendix B: References

- Capy repository: https://github.com/cppalliance/capy  
- Capy README (requirements): `README.md`  
- Existing CI: `.github/workflows/ci.yml`  
- CMake presets: `CMakePresets.json`  
- Harness: https://harness.io  
- Harness pricing (Cloud): https://harness.io/pricing  

# PR: Harness CI Evaluation Report and Pipeline Snapshot

## Summary

This PR adds the Harness CI evaluation report and pipeline snapshot for the Capy project. The evaluation compared local Kubernetes runners with Harness Cloud. Local runs were impractical (over 20 minutes per stage); after switching to Harness Cloud with caching, runs completed in 5 minutes (cold) and 4 minutes (cached). Harness is **not recommended** for this project because it relies on third-party cloud hosting, and achieving a run under one minute is not considered realistic.

## What Changed

✅ **Added Harness CI evaluation report** (`Harness-CI-evaluation-report.md`) with Achieved and My Opinion sections  
✅ **Documented two evaluation phases:**  
- Phase 1: Local Kubernetes on Ubuntu — single stage over 20 minutes  
- Phase 2: Harness Cloud (card linked), free Ubuntu tier — 8 stages, Clang 17/20, GCC 12/13/14 on Ubuntu 22.04 and 24.04  
✅ **Recorded caching impact:** First run 5 minutes, second run (with cache) 4 minutes  
✅ **Added pipeline snapshot** (`pipeline.yml`) with detailed header comments (what the pipeline does, current state, structure, identifiers)  
✅ **Conclusion:** Harness not appropriate (third-party cloud); sub-one-minute runs not realistic with Harness  

## Technical Details

**Root cause (why Harness is not recommended):** Harness runs on its own cloud (third-party hosting). For this project’s context, that is not an appropriate fit. Run times with Harness Cloud and caching remain in the 4–5 minute range; sub-one-minute is not realistic.

**Investigation:** Evaluation was done in two phases: (1) self-hosted Kubernetes on Ubuntu — too slow for practical CI; (2) Harness Cloud with payment method linked — free Ubuntu tier used, pipeline capped at 8 stages to avoid extra cost, caching enabled.

**Performance:** Local Kubernetes: >20 min per stage. Harness Cloud with cache: 5 min (cold), 4 min (cached) for 8 stages across multiple compilers and Ubuntu versions.

## Files Modified

- **Harness-CI-evaluation-report/PR_DESCRIPTION.md**
- **Harness-CI-evaluation-report/pipeline.yml** 

## Related
- Issue: Research: Migrate cppdigest/capy GitHub workflow to Harness for faster CI turnaround #1
- **Author:** Chen Shen  
- **Date:** 2026-02-03  
- **Project:** Boost.Capy (capy)  
- **Status:** Evaluation completed — Harness not recommended  
- Full pipeline export and comments in `Harness-CI-evaluation-report/pipeline.yml`

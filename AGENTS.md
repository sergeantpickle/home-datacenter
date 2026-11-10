# Agent Instructions

## Repository organization reference

This project uses [repo-template](https://github.com/sergeantpickle/repo-template) as its living reference for repository organization and growth.

Before introducing permanent files, directories, documentation categories, agent instructions, skills, or workflows, consult the structural references in its [AGENTS.md](https://github.com/sergeantpickle/repo-template/blob/main/AGENTS.md). Choose the closest project shape, then read the linked annotated example under [examples/](https://github.com/sergeantpickle/repo-template/tree/main/examples). Use the trees as patterns to adapt, not as structures to copy wholesale.

Check organizational fit during structural changes and meaningful milestones. Correct clear, low-risk drift when it is within the task's authority; preserve useful information and report what changed. Ask before destructive changes or broad restructuring.

When asked to **"audit this project using repo-template,"** follow its [audit guide](https://github.com/sergeantpickle/repo-template/blob/main/docs/audit.md).

## Project direction

home-datacenter is dependency-first.

The project should compose maintained upstream software before implementing new infrastructure. Its innovation belongs in integration, capability discovery, heterogeneous-topology planning, backend selection, lifecycle management, and a simple home-cluster experience.

Before implementing any substantial subsystem:

1. Search for maintained upstream dependencies that already solve the requirement.
2. Verify current capabilities against source code and current documentation.
3. Test the candidate on target hardware where practical.
4. Prefer direct dependency use.
5. Prefer a thin adapter when interfaces differ.
6. Fork or replace only when a measured requirement cannot be satisfied upstream.

Do not recreate distributed inference engines, tensor/expert parallelism, collective communication, model formats, KV-cache algorithms, cluster runtimes, or service-discovery systems without evidence that existing maintained dependencies cannot meet the requirement.

Treat the dependency list in README.md as candidates, not mandatory architecture. Replace or remove dependencies when research and measurements support a better choice.

## Working boundaries

- Keep the repository small until real responsibilities earn additional structure.
- Do not create speculative folders or placeholder architecture.
- Put current project purpose, normal use, and stable entry points in README.md.
- Put project-wide non-derivable agent instructions here.
- Keep task progress in issues, pull requests, or another bounded work record rather than accumulating session logs in the repository.
- Add durable technical documentation only when validated project knowledge becomes expensive or risky to reconstruct.
- Use 2 spaces for indentation where the format supports it.

---
title: "Containerazing applications"
teaching: 30
exercises: 30
questions:
- "How can one build applications in a container?"
objectives:
- "learning objective 1, 2, 3."
keypoints:
- "(FIXME)"
---

Build applications in containers for later use on Blue Waters or other supercomputers.

1. MPI (MPI bandwidth / benchmark)
    - not optimized for the system whete it will be used
2. GPU (1 node GPU app)
    -  CUDA version in the container vs. that on Blue Waters
3. MPI + GPU
    - Singularity: `--nv` + some trickery (bind-mount folders) for MPI (kinda "just works")
    - Shifter: tricks for GPU + the same trickery for MPI
4. (Python / R / Haskell)-based software stacks
    - `apt-get` / `yum` / `pip` / `haskell`

{% include links.md %}

# SLSA Playground

This repository contains a playground for simulating how SLSA can be integrated
into a package registry. It simulates the Python/PyPI ecosystem, but the ideas
translate to any package ecosystem.

## Setup

```bash
git submodule update
python3 -m venv venv
venv/bin/pip install -r requirements.txt
```

## Components

Source repository:

-   Lives at `https/<path>/`, corresponding to URI `https://<path>/`.
-   Expected to be built using the command `python3 setup.py sdist` from within
    that directory. This must produce a single file named `dist/*.tar.gz`.

Build platform (CI/CD):

-   Implemented in `bin/builder`.
-   To simulate a CI/CD build, use `bin/builder https/<path>`. This:
    -   fetches `https://<path>` (using the local copy),
    -   builds (using `python3 setup.py dist`, which again is expected to
        produce output at `dist/*.tar.gz`), and
    -   generates SLSA provenance (at `dist/*.tar.gz.intoto.jsonl`).

Package registry:

-   Implemented in `bin/registry`.
-   This is a SLSA-aware registry that verifies a SLSA policy before
    publication.
-   To simulate a package publication event, use
    `bin/registry https/<path>/dist/<name>-<version>.tar.gz [<...>.intoto.jsonl]`. This:
    -   loads the `tar.gz`
    -   loads the policy for `<name>` from `policies.cfg`,
    -   if a policy is found:
        -   loads the provenance
        -   verifies that the provenance is valid
        -   verifies that the provenance contains the `tar.gz` sha256
        -   verifies that the provenance conforms to the policy
    -   copies the artifact and provenance to `out/`, which is a valid PyPI
        index
-   To serve the index, use `python3 -m http.server -d out`.
-   To use from the index, browse http://localhost:8000 or use `pip -i
    http://localhost:8000`.

## Things to try

-   Publishing a "bad" version of `requests`:
    -   Without provenance
    -   With tampered provenance
    -   With tampered artifact (post-build)
    -   Built from a different source repo
    -   Built from a different builder

## TODO

-   Use `python3 -m build --sdist` rather than `python3 setup.py sdist`;
    requires venv with dependencies.
-   Use gVisor to sandbox the build.
-   Simulate a deployment environment to show the difference between "software"
    and "service" policies.

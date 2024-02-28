# SLSA Playground

This repository contains a playground for simulating how SLSA can be integrated
into a package registry. It simulates the Python/PyPI ecosystem, but the ideas
translate to any package ecosystem.

## How it works

To simulate a build:

-   Place the source code for `https://<path>/` into the local directory
    `https/<path>/`. You can fetch many pre-chosen examples using:

    ```bash
    git submodule update
    ```

    It is expected that each repository is built using the command `python3
    setup.py sdist` from within that directory and produces a file named
    `dist/*.tar.gz`.

-   Run a "CI/CD build" using `bin/build https/<path>`. This runs the `python3
    setup.py sdist` command and generates SLSA provenance at
    `dist/*.tar.gz.intoto.jsonl`.

-   Publish the package using `bin/publish https/<path>/*.tar.gz`.
    **TODO**

    -   Policy for PyPI package named `<package_name>` lives at
        `policy/pypi/<package_name>.policy.json`.

## TODO

- [ ] Package by simulating upload
- [ ] Sign provenance using keys checked into directory; requires venv with
  dependencies
- [ ] Use `python3 -m build --sdist` rather than `python3 setup.py sdist`;
  requires venv with dependencies
- [ ] Use gVisor to sandbox the build.

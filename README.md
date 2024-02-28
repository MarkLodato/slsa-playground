# SLSA Playground

This repository contains a playground for simulating how SLSA can be integrated
into a package registry. It simulates the Python/PyPI ecosystem, but the ideas
translate to any package ecosystem.

## One-time setup

```bash
python3 -m venv venv
venv/bin/pip install -r requirements.txt
```

## How to simulate a build and publish

### Step 1: source

First, find a git repository that can be built using `python3 setup.py sdist`.
It is expected that this command produces a single file named `dist/*.tar.gz`.
Examples:

-   https://github.com/psf/requests (tag: v2.31.0)
-   https://github.com/boto/boto3 (tag: 1.34.49)

### Step 2: build

Next, simulate a build on a SLSA-compliant build platform using `bin/builder`.
Example:

```
bin/builder git+https://github.com/psf/requests@v2.31.0
```

This:

-   clones the git repository `https://github.com/psf/requests` and checks out
    the branch/tag `v2.31.0`
    -   for efficiency, this is stored in `cache/github.com/psf/requests`, but
        you should pretend is completely ephemeral
-   runs `python3 setup.py dist` (which again is expected to produce output at
    `dist/*.tar.gz`)
-   generates SLSA provenance at `dist/*.tar.gz.intoto.jsonl`, using fake
    cryptography
-   writes the output `.tar.gz` and `.intoto.jsonl` to `built/`

### Step 3: publish

Finally, simulate a package publication (aka upload) to the PyPI package
registry using `bin/registry`. Pretend that this is a server, and that the CLI
is actually an RPC interface. This server checks a SLSA policy before allowing
the publication.

```
bin/registry built/requests-2.31.0.tar.gz{,.intoto.jsonl}
```

This:

-   loads the `tar.gz`
-   loads the policy for `requests` from `policies.cfg`
-   if a policy is found:
    -   loads the provenance `.intoto.jsonl`
    -   verifies that the provenance is valid
    -   verifies that the provenance contains the sha256 of the `tar.gz`
    -   verifies that the provenance conforms to the policy
-   copies the artifact and provenance to `published/pool/`
-   builds a valid PyPI index from `published/pool/`

You can now serve the index via any HTTP server, e.g.:

```
python3 -m http.server -d published
```

To browse the index, go to http://localhost:8000 or use `pip -i
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

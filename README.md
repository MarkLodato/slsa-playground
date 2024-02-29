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

First, find a git repository that can be built using `python3 -m build --sdist`.
It is expected that this command produces a single file named `dist/*.tar.gz`.
For simplicity we only support sdist, i.e. `tar.gz`; extending to wheels should
be trivial if desired.

Example source repositories of popular Python packages:

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
-   runs `python3 setup.py dist`
-   copies the output `dist/*.tar.gz` to `built/*.tar.gz`
-   generates SLSA provenance at `built/*.tar.gz.intoto.jsonl`, using fake
    cryptography

The resulting artifact and its provenance are then found in `built/`:

```
$ ls built
requests-2.31.0.tar.gz  requests-2.31.0.tar.gz.intoto.jsonl
```

### Step 3: publish

Finally, simulate a package publication (aka upload) to the PyPI package
registry using `bin/registry`. Pretend that this is a server, and that the CLI
is actually an RPC interface. This server checks a SLSA policy before allowing
the publication.

NOTE: This doesn't simulate authentication and authorization of the uploader. In
real life that would still happen, and the policy check is a defense in depth.

```
bin/registry built/requests-2.31.0.tar.gz{,.intoto.jsonl}
```

This:

-   loads the `tar.gz`
-   loads the policy at `policy/pypi/requests.policy.json`, if present
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

First, run through the steps above to publish a "good" version of a package. It
should work, which you can verify by seeing that the "built" version and
"published" version are the same. (You'll get a different hash than `ac8b6...`
since the build is not reproducible, but the two files should agree in your
terminal.)

```bash
$ bin/builder git+https://github.com/psf/requests@v2.31.0
...
Writing built/requests-2.31.0.tar.gz
...
Writing built/requests-2.31.0.tar.gz.intoto.jsonl
$ bin/registry built/requests-2.31.0.tar.gz*
...
Passed the policy check for package 'requests'
Publishing artifact requests-2.31.0.tar.gz
...
$ jq '.releases["2.31.0"][0].url' published/pypi/requests/2.31.0/json
"/pool/requests-2.31.0.tar.gz"
$ sha256sum {built,published/pool}/requests-2.31.0.tar.gz
ac8b68783b2038dd99ebff21c52e97d8ce315be04ab4d445f340c727dc7711df  built/requests-2.31.0.tar.gz
ac8b68783b2038dd99ebff21c52e97d8ce315be04ab4d445f340c727dc7711df  published/pool/requests-2.31.0.tar.gz
```

Now try to get a "bad" version of `requests` to be published, meaning a `tar.gz`
that contains anything other than what is in the official `psf/requests`
repository. It should not be possible. See https://slsa.dev/threats for
inspiration. In particular:

-   Built from a different source repo (i.e. provenance doesn't match policy)

    See
    https://github.com/MarkLodato/requests/commit/4f951ea3597859aaf6ca978b62afd37be5957f6f,
    which is an unofficial commit on an unofficial fork.

    ```bash
    $ bin/builder git+https://github.com/MarkLodato/requests@v2.31.666
    ...
    $ tar xaf built/requests-2.31.666.tar.gz --to-command 'egrep -Hn --label="$TAR_FILENAME" tampered || true'
    requests-2.31.666/requests/__init__.py:47:print("I'm in danger! This library has been tampered with!")
    $ bin/registry built/requests-2.31.666.tar.gz*
    ...
    Exception: buildDefinition.externalParameters.sourceRepo: expected 'git+https://github.com/psf/requests@*', got 'git+https://github.com/MarkLodato/requests@*'
    ```

-   Without provenance

    ```bash
    $ bin/registry built/requests-2.31.666.tar.gz
    ...
    Exception: Package 'requests' has a policy set, but no provenance given
    ```

-   With "good" provenance that doesn't apply to the artifact in question

    ```bash
    $ bin/registry built/requests-2.31.666.tar.gz built/requests-2.31.0.tar.gz.intoto.jsonl
    ...
    Exception: no subject found with digest.sha256 '0924469d3c9335e672188e66e77975f591ad34de52e409e1bde626c62587f629'
    ```

    NOTE: This is equivalent to tampering with a "good" artifact by transforming
    `requests-2.31.0.tar.gz` into `requests-2.31.666.tar.gz`.

-   With tampered provenance (good predicate but invalid signature)

    ```bash
    # Create "good" looking provenance payload
    $ PAYLOAD=$(jq -r .payload built/requests-2.31.666.tar.gz.intoto.jsonl | base64 -d | sed s/MarkLodato/psf/ | base64 -w0)
    # Stuff it in an existing envelope.
    $ sed 's/"payload": "[^"]\+"/"payload": "'$PAYLOAD'"/' built/requests-2.31.666.tar.gz.intoto.jsonl > built/fake.intoto.jsonl
    $ bin/registry built/requests-2.31.666.tar.gz built/fake.intoto.jsonl
    ...
    Exception: invalid signature
    ```

-   Built from a different builder that allows faking the provenance or
    otherwise tampering during the build

    ```bash
    $ bin/builder git+https://github.com/MarkLodato/requests@v2.31.666 \
        --override-source-repo git+https://github.com/psf/requests@v2.31.666 \
        -b BadBuilder -f
    $ bin/registry built/requests-2.31.666.tar.gz*
    ...
    Exception: runDetails.builder.id: expected 'https://example.com/MyBuilder', got 'https://example.com/BadBuilder'
    ```

Other package names should work because they don't have a policy and thus don't
have SLSA protections.

## TODO

-   Use gVisor to sandbox the build.
-   Simulate a deployment environment to show the difference between "software"
    and "service" policies.

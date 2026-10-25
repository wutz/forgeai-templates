# forgeai templates

Public job and sandbox templates for [forgeai](https://github.com/wutz/forgeai).
The forgeai GUI reads this repository as a template source: its New job and New
sandbox dialogs list the templates here, and picking one prefills the form. A
template never touches a cluster by itself. You review the prefilled form and
create the job or sandbox yourself.

The layout follows [kubeyard-catalog](https://github.com/wutz/kubeyard-catalog):
a small index that summarizes the source, one file per template, and a validator
that runs before anything is published.

## Layout

```
index.yaml                          the index: every template and its summary
jobs/<name>/template.yaml           a job template (kind Job)
sandboxes/<name>/template.yaml      a sandbox template (kind Sandbox)
```

Only `index.yaml` and the files it lists are read.

## Index

```yaml
apiVersion: templates.forgeai.wutz.dev/v1alpha1
kind: TemplateIndex
metadata:
  name: forgeai-default
spec:
  displayName: forgeai default templates
  homepage: https://github.com/wutz/forgeai
  templates:
    - name: nccl-tests
      kind: Job                      # Job or Sandbox
      displayName: NCCL tests
      description: All-reduce bandwidth and correctness of every GPU on each worker (single node, no MPI). Passes when the log has no NCCL error and every row shows 0 wrong values.
      version: 1.1.0
      path: jobs/nccl-tests/template.yaml
```


Rules, checked when the source is loaded:

- `apiVersion` and `kind` are exactly `templates.forgeai.wutz.dev/v1alpha1` and `TemplateIndex`.
- At least one template is listed. Unknown fields are an error.
- `kind` is `Job` or `Sandbox`. `name` is unique per kind.
- `path` is a clean relative path inside the source: no leading `/`, no `..`, no `//`.

An index that cannot be read or parsed fails the whole source. The GUI then
shows the error in the dialog and offers none of its templates.

## Template files

A job template:

```yaml
apiVersion: templates.forgeai.wutz.dev/v1alpha1
kind: JobTemplate
metadata:
  name: nccl-tests
spec:
  displayName: NCCL tests
  description: All-reduce bandwidth and correctness of every GPU on each worker (single node, no MPI). Passes when the log has no NCCL error and every row shows 0 wrong values.
  version: 1.1.0
  # The public CoreWeave nccl-tests image, as in scclabs/k8s-in-action
  # (ai/nccl-tests): CUDA 12.9 runs on the RTX 5090 nodes and the nodes pull it
  # from ghcr.io. Use the CUDA 13.1 tag (13.1.0-devel-ubuntu22.04-nccl2.29.2-1-2276a5e)
  # on B200/B300.
  image: ghcr.io/coreweave/nccl-tests:12.9.1-devel-ubuntu22.04-nccl2.27.6-1-7c12c62
  # Each worker runs its own single-node test over the GPUs the pod holds, with
  # the sweep of the reference: 8 B to 8 GB, doubling.
  command: |
    set -euo pipefail
    gpus="$(nvidia-smi -L | wc -l)"
    echo "nccl-tests on $(hostname): ${gpus} GPUs"
    nvidia-smi -L
    exec /opt/nccl_tests/build/all_reduce_perf -b 8 -e 8G -f 2 -g "${gpus}"
  workers: 1
  env:
    # Ring keeps results comparable between runs; WARN keeps the log short.
    - name: NCCL_ALGO
      value: Ring
    - name: NCCL_DEBUG
      value: WARN
```

Optional job fields are `maxRetries` (0 to 20) and `retryOn` (`Infra` or `Any`).
A job template prefills the New job form, and its `workers` is the default count.

A sandbox template:

```yaml
apiVersion: templates.forgeai.wutz.dev/v1alpha1
kind: SandboxTemplate
metadata:
  name: python-runtime
spec:
  displayName: Python runtime
  description: The agent-sandbox Python runtime image (the platform default) for code execution and tooling.
  version: 1.0.0
  image: registry.k8s.io/agent-sandbox/python-runtime-sandbox:v0.1.0
```

Optional sandbox fields are `command`, `env` and `runtimeClass` (a RuntimeClass the
platform offers). `cuda-devbox` sets a `command` that ends in `exec sleep infinity`,
so its container stays up until a shell is opened from the GUI.

Both kinds decode strictly: an unknown field is an error, so a typo does not
silently drop a setting. The file's `metadata.name` must equal the index name,
and its `displayName`, `description` and `version` must equal the index entry,
so the list the GUI shows is the list the files say.

A template is a prefill, so it does not choose the job name, the instance type
or the sandbox name, and it does not choose volumes. Those stay with the user.

A template with a broken file is still listed, with its error, so one bad file
does not hide the others. The GUI does not offer a template that has an error.

## Validation

The rules above are enforced by `pkg/catalog` in forgeai. A template is checked
with the same validation the GUI applies to the form it submits: a job template
with `training.Form.Validate` and a placeholder instance type, a sandbox template
with `sandbox.RenderTemplate`. To check a checkout of this repository, run this
from a forgeai checkout:

```bash
FORGEAI_TEMPLATE_SOURCE=file:///path/to/forgeai-templates GOWORK=off go test ./pkg/catalog/ -run TestTemplateSource -v
```

It fails when a template does not load, when the index and a file disagree, or
when a template file exists without an index entry.

## Where the GUI reads it

The platform config names the source:

```yaml
templates:
  source: https://raw.githubusercontent.com/wutz/forgeai-templates/main
```

- `https://` (or `http://`) is a base URL. The GUI fetches `index.yaml` and the listed files from it.
- `file://` is an absolute directory, for a local checkout.
- Empty uses the forgeai default, `https://raw.githubusercontent.com/wutz/forgeai-templates/main`.

Users can add other published sources in the GUI, under **Template sources** in
the sidebar, by the same two forms. A source is read before it is added, and its
templates are marked as coming from a source the user added.

The GUI reads the sources each time a create dialog opens, so a fixed template
shows up without restarting the app.

## Images

A job or sandbox template names an image, and the image must be pullable from
the cluster. The templates use public images, and forgeai builds none:

- `nccl-tests` runs the public CoreWeave image `ghcr.io/coreweave/nccl-tests`.
  Its CUDA 12.9 tag runs on RTX 5090 nodes. Use the CUDA 13.1 tag
  (`13.1.0-devel-ubuntu22.04-nccl2.29.2-1-2276a5e`) on B200 and B300. The
  binaries are in `/opt/nccl_tests/build`.
- `nccl-tests` is single node. forgeai workers expose no SSH, so no MPI launcher
  is used. Each worker runs `all_reduce_perf` over its own GPUs, so a pool is
  checked with one worker per node. A run passes when there is no NCCL error,
  every row shows 0 wrong values, and the peak `busbw` is near the hardware limit.

A cluster that cannot reach `ghcr.io` mirrors the image into its project
registry and overrides the image in the form.

## Adding a template

1. Add the template file under `jobs/<name>/` or `sandboxes/<name>/`.
2. Add its entry to `index.yaml`, with the same `displayName`, `description` and `version`.
3. Run the validation above.
4. Bump `version` when the template changes in a way a user would notice.

## Not yet

- Signing: the source is not signed. Signing the index with cosign is a
  follow-up. Until then a source is trusted by its URL, so `templates.source`
  should point to a location the platform team controls.
- OCI: publishing a source as an OCI artifact, so it can be pushed and pulled
  by digest (`oci://`), is planned as a follow-up to the file and HTTP sources.

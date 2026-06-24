---
title: "Scanning Helm Charts at Scale with helmsniff"
date: 2026-06-24T23:32:05+01:00
type: "post"
tags: ["kubernetes", "security"]
---

# Scanning Helm Charts at Scale with helmsniff

*A practical guide for developers and security researchers.*

---

## Why I Built This

Kubernetes is everywhere now. Most teams do not write raw Kubernetes YAML by hand. They use **Helm charts**. A Helm chart is a template. You give it some values, and Helm turns the template into real Kubernetes manifests.

This is great for productivity. But it is risky for security.

A single chart can create a `Deployment`, a `Service`, a `Secret`, a `ConfigMap`, and more. Each of those objects has many security settings. It is very easy to leave one wrong. Maybe a container runs as root. Maybe it mounts the Docker socket. Maybe it shares the host network. Most of these mistakes are invisible until someone abuses them.

Now imagine you do not have one chart. Imagine you have **ten thousand** charts. Maybe you study a public chart registry. Maybe you audit every chart inside a big company. You cannot open ten thousand YAML files by hand.

This is the problem `helmsniff` solves. It is a small, fast command line tool. It reads rendered Kubernetes manifests and writes a simple report. One row per Kubernetes object. One column per security check. The report is plain CSV or JSON, so you can load it into pandas, a spreadsheet, or a database and study trends across thousands of charts.

This post explains what `helmsniff` does, how to use it, how each check works, where it fits among other tools, and where its limits are. The audience is developers and security researchers. 

The tool is open source. You can find the code, build it, and open issues here: **[github.com/VahidR/helmsniff](https://github.com/VahidR/helmsniff)**.

---

## What helmsniff Is (and Is Not)

Let me be clear about the scope first, because scope matters in security tools.

`helmsniff` **is**:

- A **static analyzer**. It reads YAML text. It never talks to a cluster.
- A tool for **already-rendered** manifests. You render the chart first with `helm template`, then you scan the output.
- A **batch** tool. It is built to run thousands of times in parallel, one run per chart.
- A **data producer**. Its job is to turn messy YAML into a clean table of `0`s and `1`s.

`helmsniff` **is not**:

- A live cluster scanner. It will not connect to your API server.
- A policy engine. It does not block deployments or fail your CI by itself.
- A Helm renderer. It does not run templates. You do that step yourself.
- A fixer. It finds problems. It does not change your files.

If you keep this scope in mind, everything else makes sense.

---

## A Little Background: Why These Misconfigurations Matter

Before the commands, let me explain *why* the checks exist. If you are a developer, this is the "so what". If you are a security researcher, this is the attack surface.

A Kubernetes Pod is just a group of containers on a node. By default, the container is isolated from the host. Many misconfigurations break that isolation. When isolation breaks, a container escape becomes possible. A container escape means an attacker who controls one Pod can take over the whole node, and often the whole cluster.

Here are the classic ways isolation breaks. `helmsniff` looks for all of them:

- **`hostPID`, `hostIPC`, `hostNetwork`**: these put the container into the host's process, IPC, or network namespace. With `hostPID`, a container can see and signal host processes. With `hostNetwork`, it can sniff node traffic and reach services bound to localhost.
- **Mounting `/var/run/docker.sock`**: the Docker socket is full control over the container runtime. A container that can talk to it can start a new privileged container on the host. This is a one-step escape.
- **`SYS_ADMIN` capability**: people call it "the new root". It unlocks many escape paths, including mounting filesystems and abusing cgroups.
- **`SYS_MODULE` capability**: it lets a container load kernel modules. A malicious kernel module owns the kernel.
- **`allowPrivilegeEscalation: true`**: it lets a process gain more privileges than its parent (for example through setuid binaries).
- **`seccompProfile: Unconfined`**: seccomp filters which system calls a container may use. `Unconfined` turns this protection off, so the container can call every syscall the kernel offers.
- **`hostAliases`**: custom `/etc/hosts` entries inside the Pod. An attacker can use them to point a trusted hostname at a server they control.
- **Plaintext `http://`**: data and credentials travel without encryption. Easy to read, easy to change in transit.
- **In-manifest `Secret` data**: secrets stored directly in the chart often end up in Git history and CI logs. Base64 is **not** encryption.
- **No `securityContext`**: without one, the container often runs as root with default, loose settings.
- **No resource limits**: a single container can eat all CPU and memory on a node. This is a denial-of-service risk, not just a tidiness problem.
- **No `NetworkPolicy`**: by default, every Pod can talk to every other Pod. One compromised Pod can scan and attack the whole namespace.
- **No rolling update**: a `Recreate` strategy causes downtime during deploys. This is a reliability concern, not a direct security hole, but it is still a signal of a rushed chart.

Each of these maps to one column in the report. Now let us produce one.

---

## Getting Started

### Prerequisites

You need two things:

- **Go 1.22 or newer** (to build the binary).
- **GNU Make** (the project ships a small `Makefile`).

That is all. The tool has a single external dependency (`gopkg.in/yaml.v3`). Nothing else. This keeps the binary tiny, the build fast, and the supply chain almost empty — which matters when *you* are the security person.

### Build

Clone the repo, then run:

```bash
make build
```

The binary lands in `bin/helmsniff`. Run it with no flags to see the usage line:

```bash
$ ./bin/helmsniff
usage: ./bin/helmsniff --root <rendered_dir|-> --out <path|-> [--format csv|json]
```

There are only three flags. That is the whole interface.

| Flag | Meaning |
|------|---------|
| `--root` | A directory of rendered manifests, **or** `-` to read YAML from stdin. |
| `--out` | A path to write the report to, **or** `-` to write to stdout. |
| `--format` | `csv` (default) or `json`. |

Exit codes are predictable, which helps in scripts:

- `0` — success.
- `1` — a runtime error (for example, stdin could not be read).
- `2` — bad usage (a flag is missing or `--format` is wrong).

---

## Your First Scan

`helmsniff` reads *rendered* YAML, not chart templates. So step one is always to render. Helm does that for you:

```bash
helm template my-release ./charts/payment-api > rendered/payment-api/manifest.yaml
```

To make this post concrete, here is a small but deliberately dangerous manifest. It is a `Deployment` plus a `Secret`. Almost everything is wrong on purpose:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  labels:
    app.kubernetes.io/managed-by: Helm
    helm.sh/chart: payment-api-1.2.0
spec:
  strategy:
    type: Recreate            # no rolling update
  template:
    spec:
      hostPID: true           # host PID namespace
      hostNetwork: true       # host network namespace
      hostAliases:            # custom /etc/hosts
        - ip: "10.0.0.1"
          hostnames: ["internal.db"]
      containers:
        - name: api
          image: registry.example.com/payment-api:latest
          env:
            - name: WEBHOOK_URL
              value: "http://webhook.internal/notify"   # plaintext http
          securityContext:
            allowPrivilegeEscalation: true              # privilege escalation
            seccompProfile:
              type: Unconfined                          # seccomp off
            capabilities:
              add: ["SYS_ADMIN", "SYS_MODULE"]          # dangerous caps
      volumes:
        - name: docker
          hostPath:
            path: /var/run/docker.sock                  # docker socket mount
---
apiVersion: v1
kind: Secret
metadata:
  name: payment-api-secrets
  namespace: payments
type: Opaque
stringData:
  DB_PASSWORD: "super-secret-123"                       # secret in the manifest
```

Now scan the directory and print the result to the terminal:

```bash
./bin/helmsniff --root rendered/payment-api --out -
```

Here is the real output:

```
DIR,YAML_FULL_PATH,WITHIN_MANIFEST_SECRET,VALID_TAINT_SECRET,SEC_CONT_OVER_PRIVIL,INSECURE_HTTP,NO_SECU_CONTEXT,NO_DEFAULT_NSPACE,NO_RESO,NO_ROLLING_UPDATE,NO_NETWORK_POLICY,TRUE_HOST_PID,TRUE_HOST_IPC,DOCKERSOCK_PATH,TRUE_HOST_NET,CAP_SYS_ADMIN,HOST_ALIAS,ALLOW_PRIVI,SECCOMP_UNCONFINED,CAP_SYS_MODULE,K8S_STATUS,HELM_STATUS
rendered/payment-api,rendered/payment-api/manifest.yaml,0,0,0,1,0,1,1,1,1,1,0,1,1,1,1,1,1,1,true,true
rendered/payment-api,rendered/payment-api/manifest.yaml,1,0,0,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,true,false
```

Two YAML documents went in, so two rows came out. The first row is the `Deployment`. Read the `1`s: insecure HTTP, no namespace, no resource limits, no rolling update, no network policy, host PID, docker socket, host network, `SYS_ADMIN`, host alias, allow privilege escalation, seccomp unconfined, `SYS_MODULE`. That is a long list. This chart is a playground for an attacker.

The second row is the `Secret`. Only two `1`s: `WITHIN_MANIFEST_SECRET` (it carries data) and `NO_NETWORK_POLICY` (a chart-level flag, more on that soon). Notice `HELM_STATUS` is `false` for the Secret because it has no Helm labels, while the Deployment has them, so it is `true`.

That is the core loop: render, scan, read the table. Everything else is detail.

---

## Reading the Report: All 22 Columns

Every row has the same shape. Two text columns for context, then the checks. Each check column is a binary integer: `1` means "found / violation", `0` means "not found / ok". The last two columns are booleans (`true`/`false`) that describe the document itself.

Here is the full schema, in column order.

| # | Column | Type | `1` (or `true`) means |
|---|--------|------|-----------------------|
| 1 | `DIR` | text | The chart directory scanned (or `stdin`). |
| 2 | `YAML_FULL_PATH` | text | The exact file the row came from (or `stdin`). |
| 3 | `WITHIN_MANIFEST_SECRET` | 0/1 | A `Secret` object carries `data` or `stringData`. |
| 4 | `VALID_TAINT_SECRET` | 0/1 | Reserved. Always `0` today (see *Limitations*). |
| 5 | `SEC_CONT_OVER_PRIVIL` | 0/1 | Reserved placeholder. Always `0` today (see *Limitations*). |
| 6 | `INSECURE_HTTP` | 0/1 | An `http://` string appears anywhere in the document. |
| 7 | `NO_SECU_CONTEXT` | 0/1 | No `securityContext` at the Pod level **and** none on any container. |
| 8 | `NO_DEFAULT_NSPACE` | 0/1 | No `metadata.namespace` set. |
| 9 | `NO_RESO` | 0/1 | A container has neither resource `limits` nor `requests`. |
| 10 | `NO_ROLLING_UPDATE` | 0/1 | A `Deployment` uses `Recreate` or has no `rollingUpdate`. |
| 11 | `NO_NETWORK_POLICY` | 0/1 | The whole chart has no `NetworkPolicy` object. |
| 12 | `TRUE_HOST_PID` | 0/1 | `hostPID: true`. |
| 13 | `TRUE_HOST_IPC` | 0/1 | `hostIPC: true`. |
| 14 | `DOCKERSOCK_PATH` | 0/1 | `/var/run/docker.sock` mounted via `hostPath`. |
| 15 | `TRUE_HOST_NET` | 0/1 | `hostNetwork: true`. |
| 16 | `CAP_SYS_ADMIN` | 0/1 | A container adds the `SYS_ADMIN` capability. |
| 17 | `HOST_ALIAS` | 0/1 | The Pod spec defines `hostAliases`. |
| 18 | `ALLOW_PRIVI` | 0/1 | A container sets `allowPrivilegeEscalation: true`. |
| 19 | `SECCOMP_UNCONFINED` | 0/1 | A container sets `seccompProfile.type: Unconfined`. |
| 20 | `CAP_SYS_MODULE` | 0/1 | A container adds the `SYS_MODULE` capability. |
| 21 | `K8S_STATUS` | bool | The document has both `apiVersion` and `kind`. |
| 22 | `HELM_STATUS` | bool | The document has Helm labels. |

### Why `0` and `1` instead of `true`/`false`?

This is a deliberate design choice, and it is a good one for researchers. Because the check columns are integers, you can **sum** them. The sum of a column across ten thousand rows is the count of charts with that problem. The sum of a row is a rough "danger score" for one object. You can do this in one line of pandas or even in a spreadsheet. Booleans would force you to convert first.

---

## How Each Check Actually Works

This is the part security researchers care about most. A scanner is only useful if you know exactly what triggers it. False confidence is dangerous. So here is the real logic behind every check, taken straight from the source code. I also point out the edge cases, because edge cases are where bugs hide.

### Context and status checks

**`K8S_STATUS`** is `true` when the document has a non-empty `apiVersion` **and** a non-empty `kind`. This is a quick "is this even a Kubernetes object" filter. A stray values file or a README fragment will be `false`.

**`HELM_STATUS`** is `true` when `metadata.labels` contains `app.kubernetes.io/managed-by: Helm`, **or** when it has a `helm.sh/chart` label. Helm adds these labels automatically. This column lets you tell apart objects that Helm manages from objects that slipped in by other means.

### Secrets

**`WITHIN_MANIFEST_SECRET`** is `1` only when `kind` is `Secret` **and** the object has at least one entry in `data` or `stringData`. An empty Secret shell does not trigger it. The point is to find real credentials baked into the chart. Remember: Kubernetes `Secret` data is base64, not encrypted. Anyone who reads the manifest reads the secret.

### Insecure transport

**`INSECURE_HTTP`** walks the **entire** document — every nested map, every list, every string, and even the map keys — and returns `1` if any of them contains the substring `http://` (case-insensitive). It stops at the first match.

This is a broad, simple check. Know its two sides:

- It is **thorough**: it will catch an `http://` URL hidden deep in an annotation or an env var.
- It is **literal**: it is a substring match. A value like `"http://"` inside a longer harmless string still triggers it, and a value of `https://` never does. It does not understand context. Treat a `1` as "look here", not "definitely exploitable".

### Pod-level escapes

These checks read the Pod spec. For a bare `Pod`, that is `spec`. For a workload like a `Deployment`, `StatefulSet`, `DaemonSet`, `Job`, or `CronJob`, that is `spec.template.spec`. Non-workload kinds (a `Service`, a `ConfigMap`) have no Pod spec, so all of these return `0`.

**`TRUE_HOST_PID`**, **`TRUE_HOST_IPC`**, **`TRUE_HOST_NET`** each return `1` when the matching field (`hostPID`, `hostIPC`, `hostNetwork`) is the boolean `true` in the Pod spec.

**`HOST_ALIAS`** returns `1` when the Pod spec has a non-empty `hostAliases` list.

**`DOCKERSOCK_PATH`** loops over `spec.volumes` and returns `1` if any volume is a `hostPath` whose `path` contains `/var/run/docker.sock`.

### Container-level dangers

These checks look at **every** container, both normal `containers` and `initContainers`. If any one container is dangerous, the whole row gets a `1`.

**`CAP_SYS_ADMIN`** and **`CAP_SYS_MODULE`** read each container's `securityContext.capabilities.add` list. The comparison is case-insensitive (the value is upper-cased before checking), so `SYS_ADMIN` and `sys_admin` both match.

**`ALLOW_PRIVI`** returns `1` if any container sets `securityContext.allowPrivilegeEscalation: true`.

**`SECCOMP_UNCONFINED`** returns `1` if any container sets `securityContext.seccompProfile.type` to `Unconfined` (case-insensitive).

### Hygiene and reliability

**`NO_SECU_CONTEXT`** returns `1` only when the Pod has **no** Pod-level `securityContext` **and none** of its containers define one either. This is important and a little surprising: if even one container has a `securityContext` — even a *bad* one — this column is `0`. In our example above, `NO_SECU_CONTEXT` is `0`, because the container has a (very dangerous) `securityContext`. So a `0` here does not mean "safe". It means "a securityContext exists somewhere". Read it together with the other columns.

**`NO_RESO`** returns `1` if **any** container has neither `resources.limits` nor `resources.requests`. One container without limits is enough to flag the row. This is your noisy-neighbor and denial-of-service signal.

**`NO_ROLLING_UPDATE`** only applies to `Deployment` objects. It returns `1` when the strategy `type` is `Recreate`, or when there is no `rollingUpdate` block at all. Other kinds always get `0`.

**`NO_DEFAULT_NSPACE`** returns `1` when `metadata.namespace` is missing or empty. One honest caveat: cluster-scoped objects (like a `ClusterRole` or the `NetworkPolicy` namespace edge cases) and objects that rely on the install-time namespace will also be flagged. So a `1` here is a hint to check, not always a real problem.

### The one chart-level check

**`NO_NETWORK_POLICY`** is special. It is not computed per document. It is computed once **per chart**. Before scanning any file, `helmsniff` does a quick first pass over every YAML file in the directory and asks: "does any object have `kind: NetworkPolicy`?" If yes, every row from that chart gets `NO_NETWORK_POLICY = 0`. If no, every row gets `1`.

That is why, in our first example, even the `Secret` row showed `NO_NETWORK_POLICY = 1`. The flag describes the chart, and it is copied onto each row of that chart. This is handy: you can filter on any single row to learn whether the whole chart lacks network isolation.

---

## A Clean Chart, for Contrast

To trust a scanner, you must also see it stay quiet when things are fine. Here is a safe chart. It has a rolling update, a security context, resource limits, a namespace, and a `NetworkPolicy`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: web
  labels:
    app.kubernetes.io/managed-by: Helm
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
  template:
    spec:
      securityContext:
        runAsNonRoot: true
      containers:
        - name: web
          image: web:1.0.0
          resources:
            requests: { cpu: "100m" }
            limits: { cpu: "200m" }
          securityContext:
            allowPrivilegeEscalation: false
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-default-deny
  namespace: web
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
```

Scan it:

```
DIR,YAML_FULL_PATH,WITHIN_MANIFEST_SECRET,...,NO_NETWORK_POLICY,...,K8S_STATUS,HELM_STATUS
rendered/safechart,rendered/safechart/manifest.yaml,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,true,true
rendered/safechart,rendered/safechart/manifest.yaml,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,true,false
```

All checks are `0`. Notice `NO_NETWORK_POLICY` is `0` on **both** rows, including the `NetworkPolicy` object itself, because the chart-level pre-scan found a `NetworkPolicy`. This is exactly what we want.

---

## The Three Ways to Feed It Data

`helmsniff` is flexible about where input comes from and where output goes.

### 1. Directory mode

Point `--root` at a folder. The tool walks it recursively and reads every `.yaml` and `.yml` file:

```bash
./bin/helmsniff --root rendered/payment-api --out report.csv
```

When you give a real file path to `--out`, the tool creates any missing parent directories for you.

### 2. Stdin pipe mode

Use `--root -` to read YAML straight from a pipe. This skips the intermediate file. It pairs perfectly with `helm template`:

```bash
helm template my-release ./charts/payment-api \
  | ./bin/helmsniff --root - --out - --format json
```

In this mode the `DIR` and `YAML_FULL_PATH` columns are both set to `stdin`, since there is no file on disk. The chart-level `NetworkPolicy` pre-scan still works: it runs over the documents that arrive in the stream.

### 3. Output to stdout or to a file

`--out -` writes to stdout (good for piping into `jq`, `grep`, or another script). A real path writes to a file. Both work with either format.

---

## JSON Output

CSV is best for big data work. JSON is best when you want to read one object closely or feed another program. Add `--format json`:

```bash
./bin/helmsniff --root rendered/payment-api --out - --format json
```

You get a pretty-printed array, one element per document:

```json
[
  {
    "dir": "rendered/payment-api",
    "yaml_full_path": "rendered/payment-api/manifest.yaml",
    "within_manifest_secret": 0,
    "insecure_http": 1,
    "no_default_nspace": 1,
    "no_reso": 1,
    "no_rolling_update": 1,
    "no_network_policy": 1,
    "true_host_pid": 1,
    "dockersock_path": 1,
    "true_host_net": 1,
    "cap_sys_admin": 1,
    "host_alias": 1,
    "allow_privi": 1,
    "seccomp_unconfined": 1,
    "cap_sys_module": 1,
    "k8s_status": true,
    "helm_status": true
  }
]
```

Because it is standard JSON, you can post-process with `jq`. For example, list only the documents that mount the Docker socket:

```bash
./bin/helmsniff --root rendered/payment-api --out - --format json \
  | jq '.[] | select(.dockersock_path == 1) | .yaml_full_path'
```

---

## Scanning Thousands of Charts in Parallel

This is the use case `helmsniff` was really built for. One run handles one chart. To scan many charts, run many copies at once. GNU `parallel` makes this a one-liner:

```bash
parallel ./bin/helmsniff --root {} --out reports/{/}.csv ::: rendered/*/
```

Here `{}` is each chart directory and `{/}` is just its base name. You end up with one CSV per chart in `reports/`. On a normal laptop, each small chart scans in a few milliseconds, so even tens of thousands of charts finish quickly.

To get one big table instead of many small ones, keep the header from the first file and append the data rows of the rest:

```bash
# header once
head -n 1 "$(ls reports/*.csv | head -n 1)" > all.csv
# data rows from every file
for f in reports/*.csv; do tail -n +2 "$f" >> all.csv; done
```

Now `all.csv` holds every object from every chart in one place. This is your dataset.

---

## Turning the Report into Findings with pandas

The whole design — binary columns, one row per object — exists to make this step easy. Here is a short analysis you can run on `all.csv`:

```python
import pandas as pd

df = pd.read_csv("all.csv")

# The security check columns (skip the two text columns and the two booleans).
checks = [
    "WITHIN_MANIFEST_SECRET", "INSECURE_HTTP", "NO_SECU_CONTEXT",
    "NO_DEFAULT_NSPACE", "NO_RESO", "NO_ROLLING_UPDATE", "NO_NETWORK_POLICY",
    "TRUE_HOST_PID", "TRUE_HOST_IPC", "DOCKERSOCK_PATH", "TRUE_HOST_NET",
    "CAP_SYS_ADMIN", "HOST_ALIAS", "ALLOW_PRIVI", "SECCOMP_UNCONFINED",
    "CAP_SYS_MODULE",
]

# 1. How common is each problem across all objects?
print(df[checks].sum().sort_values(ascending=False))

# 2. A simple danger score per object, then the worst offenders.
df["score"] = df[checks].sum(axis=1)
print(df.sort_values("score", ascending=False)[["YAML_FULL_PATH", "score"]].head(10))

# 3. Which charts mount the Docker socket? (a near-instant escape)
print(df[df["DOCKERSOCK_PATH"] == 1]["DIR"].unique())

# 4. How many charts ship with no NetworkPolicy at all?
no_np_charts = df[df["NO_NETWORK_POLICY"] == 1]["DIR"].nunique()
print(f"Charts without any NetworkPolicy: {no_np_charts}")
```

Because every check is a `0` or `1`, `sum()` gives you counts directly. You can build prevalence charts, rank the most dangerous charts, and slice by any column. This is the kind of analysis that turns "I have a feeling charts are insecure" into "37% of the charts in this registry mount the host network".

---

## How helmsniff Works Inside

You do not need this to use the tool. But if you want to trust it, extend it, or debug a surprising result, here is the design in plain terms.

**Two passes.** For a directory, `helmsniff` walks the files twice. The first pass only looks for a `NetworkPolicy` and stops at the first one it finds. This is cheap because of the early exit. The second pass does the real per-document analysis. The two passes exist because `NO_NETWORK_POLICY` is a property of the whole chart, not of a single file, so it must be known before per-file work begins.

**Untyped YAML.** The tool does **not** use the official Kubernetes Go structs. It parses each document into a generic `map[string]interface{}`. This sounds lazy but it is smart: Kubernetes has hundreds of resource kinds, and the tool only reads a handful of fields. Generic maps mean unknown and custom resources (CRDs) never crash the parser. The trade-off is that the checks must walk the map carefully with small helper functions (`asMap`, `asSlice`, `str`, `boolVal`) that fail safe when a field is missing or has the wrong type.

**Pod spec normalization.** One helper, `podSpecFromDoc`, hides the difference between a bare `Pod` (`spec`) and a workload (`spec.template.spec`). All the Pod-level checks call it, so they do not each have to know about every workload kind.

**Error tolerance by design.** If a file is not valid YAML, the tool **skips it silently** and moves on. For a single file this might surprise you. For a batch of ten thousand charts, it is essential — one broken file should never stop the other 9,999 from being scanned. Keep this in mind: a file that produces no rows may simply have failed to parse.

**Tiny footprint.** The whole thing is a few hundred lines of Go and one dependency. You can read all of it in an afternoon. For a security tool, being auditable is a feature.

---

## Honest Limitations

A good security tool tells you what it does **not** do. Here are the sharp edges, so you do not trust a `0` more than you should.

- **`SEC_CONT_OVER_PRIVIL` and `VALID_TAINT_SECRET` are placeholders.** They are in the schema, but the current code never sets them. They are always `0`. So `helmsniff` does **not** yet detect `privileged: true` containers through this column. This is the single most important caveat: do not read a `0` in `SEC_CONT_OVER_PRIVIL` as "no privileged container". The check is simply not implemented yet. (The columns are kept for parity with an older toolchain and for future work.)
- **`NO_SECU_CONTEXT` only checks presence, not quality.** As shown earlier, a container with a *terrible* `securityContext` still makes this column `0`. Always read it next to the capability and privilege columns.
- **`INSECURE_HTTP` is a substring match.** It is thorough but literal. It does not parse URLs or understand context. Expect some noise.
- **`NO_DEFAULT_NSPACE` flags cluster-scoped objects too.** Many resources are namespaced at install time and legitimately omit `metadata.namespace`. Treat a `1` as "verify", not "wrong".
- **It scans what Helm rendered with your values.** If a dangerous setting only appears under a different `values.yaml`, render that combination too. The tool sees only the YAML you give it.
- **One chart per process; parallelism is external.** A single very large chart is scanned in one thread. Scale comes from running many processes with `parallel`, not from internal threading.
- **It finds, it does not fix.** There is no auto-remediation and no built-in CI gate. You wire that around it.

None of these make the tool less useful. They just tell you how to read the output correctly. In security, knowing the limits of your tool is half the job.

---

## Adding Your Own Check

Because the codebase is small, adding a check is a five-step pattern. Say you want to detect privileged containers properly (and finally make a real check out of that idea).

1. **Add a field to the `Row` struct** in `internal/scanner/row.go`:

   ```go
   SecContOverPrivil int `json:"sec_cont_over_privil"`
   ```
   (This one already exists — you would just wire it up. For a brand new check, add a new field.)

2. **Write the check function** in `internal/scanner/security_checks.go`. Follow the house style: accept a map, guard against `nil`, return `1` or `0`.

   ```go
   func hasPrivileged(spec map[string]interface{}) int {
       if spec == nil {
           return 0
       }
       for _, c := range containersFromSpec(spec) {
           sc := asMap(asMap(c)["securityContext"])
           if boolVal(sc["privileged"]) {
               return 1
           }
       }
       return 0
   }
   ```

3. **Call it** in `AnalyzeFile` (and `AnalyzeReader`) inside `internal/scanner/scanner.go`:

   ```go
   row.SecContOverPrivil = hasPrivileged(ps)
   ```

4. **Add the column header** in `internal/config/constants.go` (only if it is a brand new column; the order must match).

5. **Add it to the CSV record slice** in `cmd/main.go` so the value is written in the right position.

Then run the tests with `make test` and you are done. The column order is the one rule you must respect: the header list, the record slice, and the struct must stay aligned, or the CSV will be shifted.

---

## Where helmsniff Fits Among Other Tools

There are many Kubernetes security tools: `kubesec`, `kube-score`, `checkov`, `kube-linter`, `trivy`, and others. Most of them are richer than `helmsniff`. They have severities, remediation advice, policy languages, and pretty terminal output. If your goal is "block a bad Deployment in CI" or "give a developer a fix-it list", reach for one of those.

So why use `helmsniff`? Its niche is **research at scale**. It does one thing the big tools do not do well: it turns thousands of charts into a single clean numeric table that you can load into pandas and study. The output is boring on purpose. Boring `0`/`1` columns are exactly what you want when you ask questions like:

- "Across this whole public registry, how common is each misconfiguration?"
- "Did host-network usage go up or down between two snapshots of a chart repo?"
- "Which 1% of charts are the most dangerous, by raw count of red flags?"

For those questions, a small, fast, single-purpose, auditable tool beats a heavy one. Use the big scanners to gate and to fix. Use `helmsniff` to measure and to research.

---

## A Suggested Research Workflow

Putting it all together, here is how I use it end to end:

```bash
# 1. Render every chart you collected.
for chart in charts/*/; do
  name=$(basename "$chart")
  mkdir -p "rendered/$name"
  helm template "$name" "$chart" > "rendered/$name/manifest.yaml" 2>/dev/null
done

# 2. Scan them all in parallel, one CSV per chart.
parallel ./bin/helmsniff --root {} --out reports/{/}.csv ::: rendered/*/

# 3. Merge into one table.
head -n 1 "$(ls reports/*.csv | head -n 1)" > all.csv
for f in reports/*.csv; do tail -n +2 "$f" >> all.csv; done

# 4. Analyze in pandas (see the script above), find the worst charts,
#    then open those by hand and confirm the findings.
```

Step 4 is the important one. `helmsniff` does not give you proof of exploitation. It gives you a map of where to look. The substring `http://` match might be harmless. The missing namespace might be fine. But the chart that lights up with `hostPID`, the Docker socket, `SYS_ADMIN`, and an unconfined seccomp profile all at once? That one is worth your full attention. The tool's job is to point you there fast, across a haystack too big to search by hand.

---

## The Research Behind helmsniff

`helmsniff` is not just a weekend script. It grew out of my **master's thesis**, where I studied Kubernetes and Helm security misconfigurations at scale. The thesis explains *why* these checks were chosen, how common each problem is across real charts, and what the measurements say about the state of Helm chart security in the wild. The tool is the practical engine behind that research: it is what turned thousands of charts into the numbers the thesis analyzes.

If you want the full background — the methodology, the dataset, and the detailed results — you can read and download the complete thesis PDF here:

**[Master's thesis on Google Scholar](https://scholar.google.com/citations?view_op=view_citation&hl=en&user=3DL59s0AAAAJ&citation_for_view=3DL59s0AAAAJ:u5HHmVD_uO8C)**

I recommend reading it if you plan to use `helmsniff` for serious research. It gives the reasoning behind each check and the limits of the approach in much more depth than a blog post can.

---

## Wrap Up

`helmsniff` is small on purpose. It renders nothing, fixes nothing, and connects to nothing. It reads Kubernetes YAML and writes a clean table of security signals. That narrow focus is its strength.

If you are a developer, run it on your own chart before you ship. The first row of `1`s is a fast wake-up call. If you are a security researcher, point it at a whole registry, load the CSV into pandas, and let the data tell you which charts deserve a deeper look.

The same pattern shows up again and again in security: the danger is rarely one exotic bug. It is a pile of small, boring defaults left wrong, repeated across thousands of charts. A simple scanner that counts those defaults — honestly, and at scale — is more useful than it looks.

Render. Scan. Read the table. Then go look at the charts that scared the tool.

---

*Links: the code is at [github.com/VahidR/helmsniff](https://github.com/VahidR/helmsniff), and the full research is in my master's thesis, available [here on Google Scholar](https://scholar.google.com/citations?view_op=view_citation&hl=en&user=3DL59s0AAAAJ&citation_for_view=3DL59s0AAAAJ:u5HHmVD_uO8C).*

---

For comments, please send me 📧 [*<span style="border-bottom: 1px dashed #666;">an email</span>*](mailto:vahid.rafiei@gmail.com).

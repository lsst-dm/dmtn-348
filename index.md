# Nightly Ephemerides Cache Generation and Service at the USDF

```{abstract}
This technote documents the design and implementation of the Rubin Observatory
solar system small body ephemerides generation system. The system comprises two
main components, mpsky and ephemcache. mpsky answers which small bodies lie
within a given field of view at a given time, using nightly ephemerides caches
precomputed from the Minor Planet Center orbit catalog. This note describes the
architecture, configuration, measured performance, and known operational gaps of
this system.
```

## Scope and status

This note is written for Rubin Operations staff who will operate, maintain, fix
and upgrade this system. It concentrates on how to keep it running rather than on
the algorithms it uses; {ref}`the software inventory <algorithms>` separates what
we own from what is upstream.

The system is **still under development**. It runs at `usdfdev` only, several
references are not frozen, and no one has yet been assigned to own it. Wherever
the deployed reality differs from where the system is heading, this note
describes reality first and marks the difference:

:::{important}
**Current status** boxes like this one flag something that is provisional,
missing, or known to be wrong. Believe the box over the surrounding text.
:::

Two facts shape everything else in this note:

1. **AP pipelines at USDF already query the service.** It is not a sandbox. An
   outage has downstream effect.
2. **A night's cache must exist before that night's observing starts.** Cache
   generation is on the critical path for the night, not a background chore.

:::{warning}
**There is no failure alerting.** A failed or missing nightly run is currently
silent: nothing pages, emails, or posts to Slack. Combined with the two facts
above, this is the single largest operational risk in the system, and closing it
is the first thing a new owner should do. See {ref}`gaps`.
:::

The reasoning behind non-obvious settings is given inline rather than left
implicit, particularly where an operator could reasonably "tidy up" something
load-bearing and break the system. Those places are called out as they arise.

## What the system does

Alert Production needs to know which known solar system small bodies fall within
a given field of view at a given time, so that alerts can be associated with
already-catalogued objects rather than reported as new discoveries.

Answering that from orbital elements on demand would be far too slow. Instead the
work is done once per night, ahead of time:

- **[`ephemcache`](https://github.com/lsst-sqre/phalanx/tree/main/applications/ephemcache)**
  runs nightly and precomputes ephemerides for every known small body across the
  coming night, writing a compact per-night **cache** file.
- **[`mpsky`](https://github.com/lsst-sqre/phalanx/tree/main/applications/mpsky)**
  is a long-running HTTP service that loads that cache and answers positional
  queries from it in milliseconds.

```{mermaid}
flowchart LR
  MPC[("MPC orbit replica<br/>PostgreSQL at USDF")]
  subgraph CJ["ephemcache CronJob — usdfdev, every 3 h"]
    direction TB
    S1["1 · fetch MPCORB"] --> S2["2 · chunk orbits"]
    S2 --> S3["3 · sorcha ephemerides<br/>N chunks in parallel"]
    S3 --> S4["4 · mpsky build"]
  end
  TREE["mpsky-data/ on /sdf/group/rubin/web_data<br/>caches/ · catalogs/"]
  MPSKY["mpsky service"]
  AP["AP pipelines"]
  MPC --> CJ
  CJ --> TREE
  TREE -- "HTTPS" --> MPSKY
  MPSKY --> AP
```

**A normal night.** The CronJob wakes every three hours, works out which
observing night it is, and checks whether that night's cache already exists. Most
of the time it does, and the job exits in under a second — doing nothing is the
common case. When the cache is missing, the job queries the MPC replica for the
current orbit catalogue (~9 min), splits it into 100 chunks, runs `sorcha` across
48 cores to compute ephemerides for every known object (~15 min), packs the
results into a single ~213 MB cache file and renames it into place. On the other
side, `mpsky` wakes once a minute; when the night label changes it fetches that
night's cache, retrying each minute until the cache exists. So a cache that lands
late is picked up on its own and nobody has to restart anything. AP pipelines then
query `mpsky` over HTTP until the night rolls over and the cycle repeats.

End to end a real build takes about 26 minutes against a three-hourly schedule,
so there is a wide margin before the night starts. Note that this margin is
currently the only thing protecting the deadline, because a run that fails or
never starts raises no alarm.

The two components are coupled only through files in a directory that is served
over HTTP. `ephemcache` never talks to `mpsky`, and `mpsky` never talks to the
database. That is deliberate: either can be restarted, redeployed or rolled back
independently.

(algorithms)=
### The two services we own, and the upstream software they use

Everything in the first table is ours: we deploy it, and we are responsible for
fixing it. Everything in the second is upstream, listed so that a reader knows
where a given behaviour comes from — none of it is maintained here.

**Ours.** Two services, one repository each:

| Service | Repository | What it is |
|---|---|---|
| `ephemcache` | [`mjuric/lsst-gen-ephemcache`](https://github.com/mjuric/lsst-gen-ephemcache) | The **cache generator**. The whole service: pipeline scripts, container entrypoint and selftest, `Dockerfile`, and the GitHub Actions workflow that publishes the image. |
| `mpsky` | [`mjuric/mpsky`](https://github.com/mjuric/mpsky) | The **ephemerides server**. Also provides `mpsky build`, which `ephemcache` runs as its final stage to write the cache. |

Note that `mpsky` appears on both sides of the system: it is the server, and it
is also the tool that writes the file the server reads. That coupling is the
reason {ref}`upgrading it <upgrading-mpsky>` needs care.

**Upstream, for reference.** Pulled in as dependencies; algorithms are out of
scope for this note:

| Concern | Package |
|---|---|
| Ephemeris generation | [`sorcha`](https://github.com/dirac-institute/sorcha) — invoked per chunk in stage 3 |
| N-body integration | [`assist`](https://github.com/matthewholman/assist), used by sorcha |
| Integrator underlying `assist` | [`rebound`](https://github.com/hannorein/rebound) |
| Planetary and lunar positions | JPL/NAIF SPICE kernels, baked into the image |
| SPICE bindings | `spiceypy` |

## Architecture and data products

### The served tree

Everything the two components share lives under
`/sdf/group/rubin/web_data/mpsky-data`, which is published at
<https://s3df.slac.stanford.edu/data/rubin/mpsky-data>.

| Path | Written by | Read by | Notes |
|---|---|---|---|
| `caches/eph.<mjd>.<date>.bin` | stage 4 | `mpsky` | ~213 MB per night; the cache proper |
| `catalogs/mpcorb-orbits.<date>.csv` | stage 1 | `mpsky`, stage 3 | ~190 MB |
| `catalogs/mpc_orbits.<date>.sqlite.zst` | stage 1 | `mpsky` | ~1.6 GB; needed for extended element queries |
| `catalogs/mpcorb-colors.<date>.csv` | stage 1 | stage 3 | not read by `mpsky` |
| `_workdir/` | stages 2–3 | stage 4 | ~41 GB of transient intermediates |
| `logs/` | entrypoint | humans | mode 700, deliberately **not** served |

**`mpsky` needs all three of the cache, the orbits CSV and the sqlite database
for a night.** A night with only some of them present will fail at query time,
not at load time.

**`mpsky` discovers which nights are available by scraping the HTML directory
index of `caches/`.** It requests the file for the night it needs and only then
discovers whether it can actually fetch it. Two consequences that matter when
diagnosing problems: a file that is listed but unreadable produces a download
failure rather than a clean "no such night", and anything that suppresses the
directory index — such as adding an `index.html` to `caches/` — would make every
night invisible at once.

:::{important}
**Current status.** `_workdir` sits inside the served tree, so ~41 GB of
intermediates are web-visible between runs. It is cleared at the *start* of each
run, not the end, so it persists until the next run. Harmless but untidy; see
{ref}`gaps`.
:::

### The four stages

An operator rarely needs more detail than this. Each stage is skipped if its
output already exists, which is what makes re-running safe.

1. **Fetch MPCORB.** Queries the USDF-internal MPC orbit replica and writes the
   three `catalogs/` files for the night. Roughly 9 minutes. Skipped if that
   night's catalogs are already present.
2. **Chunk.** Splits the orbit catalogue into N partitions (default 100) under
   `_workdir`. Seconds.
3. **Compute ephemerides.** Runs `sorcha` over the N chunks in parallel under GNU
   `parallel`, one process per chunk, converting each result to HDF5. This
   dominates the runtime.
4. **Build the cache.** `mpsky build` packs the per-chunk HDF5 files into the
   single `.bin` cache, writing to a temporary name and then renaming it into
   place. Because the rename happens within one filesystem it is atomic, so
   `mpsky` never observes a partially written cache.

### Observing nights and the query-time mapping

Cache filenames are keyed by the **observing night**, computed with a 17:00
`America/Santiago` rollover, not by wall-clock date. This is why re-running is
cheap: `(MJD, date)` is a pure function of the night, so a second run for the
same night finds its output already present and exits in under a second.

When diagnosing a query failure it matters that **`mpsky` derives the night from
the query time as `floor(t) - 1`**. A query at `t = 61259.5` is served from the
cache for night `61258`. An operator chasing an HTTP 400 should convert the
client's `t` before concluding a cache is missing.

## How it's deployed

Both components are Phalanx applications deployed by Argo CD at `usdfdev`, with
**manual** sync policy — a push does not deploy by itself.

| | `ephemcache` | `mpsky` |
|---|---|---|
| Kind | `CronJob` | `Deployment` + `Service` |
| Namespace | `ephemcache` | `mpsky` |
| Chart | `applications/ephemcache` | `applications/mpsky` |
| Image | `ghcr.io/mjuric/lsst-gen-ephemcache` | `ghcr.io/mjuric/mpsky-daily` |
| Exposure | none | LoadBalancer, `172.24.10.34:80` |

### Settings that matter

From `applications/ephemcache/values.yaml`, overridden in
`values-usdfdev.yaml`. The full generated reference is in the chart's
`README.md`; these are the ones with consequences.

| Value | usdfdev | Why it matters |
|---|---|---|
| `schedule` | `0 */3 * * *` | Frequent retries are free because runs skip when the cache exists |
| `suspend` | `false` | Chart default is `true`, so a new environment cannot start computing unnoticed |
| `command` | `run` | Chart default is `selftest`, which validates the image and needs no database |
| `ncores` | `48` | **Must match `resources.limits.cpu`** — see below |
| `resources` | 48 CPU / 64 Gi | Both requests and limits |
| `activeDeadlineSeconds` | `5400` | 90 min; a wedged run cannot occupy the pool indefinitely |
| `outputDirMount.subPath` | `web_data/mpsky-data` | The security boundary — see below |
| `logToFile` | `true` | Durable log copy under `logs/` |
| `nodeSelector`, `tolerations` | RSP pool | **Both** are required — see below |

:::{warning}
**`ncores` must equal `resources.limits.cpu`.** The scripts otherwise default
their parallelism to `nproc`, which reports the *node's* core count (128 on these
nodes), not the pod's CPU limit. At roughly 4 GB per parallel chunk that
oversubscribes badly and can OOM-kill the pod. `selftest` warns when `ncores` is
unset, but nothing prevents the two drifting apart. Change them together.
:::

### Node placement

The job runs on the RSP node pool: four nodes, `sdfk8so001`–`004`, each 128 CPU
and roughly 503 GiB.

Those nodes carry the taint
`edu.stanford.slac.sdf.project/rsp=true:NoSchedule`, so **both** keys are needed
and neither is redundant. The toleration grants *permission* to land on tainted
nodes but does not attract the pod anywhere; without the `nodeSelector` the
scheduler could place it on any general node. The `nodeSelector` matches a
*label* of the same name and confines the pod to those four.

This is not merely preference. Bare probe pods requesting 48 CPU / 64 Gi,
32 CPU / 64 Gi, 48 CPU / 240 Gi **and 16 CPU / 32 Gi** all sat `Pending` on the
general pool; a representative message was `151 node(s) had untolerated
taint(s), 22 Too many pods, 33 Insufficient memory, 35 Insufficient cpu`. Small
`selftest` pods at 1 CPU / 2 Gi do schedule there, so the ceiling is somewhere
well below 16 cores.

`nublado` is the only other application using this pool, and its values file is
effectively the only documentation of the convention — there is nothing about the
pool in the Phalanx docs or in `slaclab/sdf-docs`.

### Storage and the write path

Output is written directly into the published directory. There is no staging area
and no publish step: a finished cache is renamed into its final location.

The mount uses the `sdf-group-rubin` claim with
`subPath: web_data/mpsky-data`. **The `subPath` is the security boundary** — the
rest of `/sdf/group/rubin`, a shared and publicly served filesystem, is not
visible inside the container at all. That is what bounds the `rm -rf
outputs/_workdir` at the start of every run.

:::{warning}
**Never delete the `sdf-group-rubin` PVC without first patching its PV to
`persistentVolumeReclaimPolicy: Retain`.** That storage class provisions PVs
whose `source.path` is **`/sdf/group/rubin` itself**, with
`reclaimPolicy: Delete` — so deleting the claim points a reclaim routine at the
entire Rubin group filesystem rather than at a per-claim subdirectory. Whether it
would actually delete has deliberately not been tested.

The chart closes the two automated paths to that with
`argocd.argoproj.io/sync-options: Prune=false` and
`helm.sh/resource-policy: keep` on the PVC. **Neither annotation is decoration;
do not remove them.** Neither protects against a manual `kubectl delete pvc`.
:::

:::{warning}
**Write access comes from a named-user ACL, not from group membership.** `ls -l`
reports `drwxrwsr-x+` on `mpsky-data`, but with ACLs present that group `rwx` is
the *mask*; the real entry is `group::r-x`, with no write bit. Writes succeed
because of an explicit `user:mjuric:rwx` ACL and because the pod runs as
`runAsUser: 18728`.

Changing `runAsUser` therefore breaks writing, and `fsGroup` will not compensate:
kubelet does not apply `fsGroup` to a statically mounted host filesystem. If the
service account or uid ever changes, the ACL must be updated to match.
:::

### Database credentials

Stage 1 reads the USDF-internal MPC replica. Credentials are provisioned by
environment administrators in Vault and reach the pod as `PGUSER` and
`PGPASSWORD` via `VaultSecret`.

They are deliberately **not** placed in the connection string. The DSN is passed
to `get-mpcorb.py` as a command-line argument, so anything embedded in it would
appear in the process arguments and in any traceback that printed them. `libpq`,
which `psycopg2` uses, reads `PGUSER`/`PGPASSWORD` from the environment, so no
credential needs to appear in the chart at all. Both are marked optional at the
pod level so that `selftest` still runs before the secret exists.

### Where the image comes from

`lsst-gen-ephemcache` builds and publishes to ghcr on every push to its
integration branch via GitHub Actions. The chart consumes the resulting tag.

:::{important}
**Current status — references are not frozen.**

- `image.tag` is the **mutable branch tag** `u-mjuric-ephemcache`, with
  `imagePullPolicy: Always`. A rebuild changes what runs without any chart
  change. Each build also publishes an immutable `sha-<commit>` tag; switching
  `image.tag` to that form is a promotion prerequisite.
- `install.sh` inside the image clones **`mpsky` at the tip of its `auto-load`
  branch**, not a commit. A new commit there silently lands in the next image
  build.
- The image records what it actually used in `/app/build-manifest.mpsky-sha.txt`
  along with conda and pip manifests, so a given image can be audited after the
  fact even though the inputs float.
:::

## Operating the system

### Is it working?

In rough order of directness:

1. **Did tonight's cache appear?**
   ```
   ls -l /sdf/group/rubin/web_data/mpsky-data/caches/ | tail
   ```
   One `eph.<mjd>.<date>.bin` per night, ~213 MB, mode `-rw-r--r--`.

2. **Query the service.** Convert the time you care about with
   `night = floor(t) - 1`:
   ```
   curl -s -o /dev/null -w '%{http_code}\n' \
     'http://172.24.10.34/ephemerides/?t=61259.5&ra=180&dec=0&radius=1'
   ```
   `200` with a couple of hundred kB of Apache Arrow payload is healthy.
   `http://172.24.10.34/version` reports the running `mpsky` commit.

3. **Read the logs in Loki.** Pod logs are shipped off the cluster and are
   queryable in Grafana Explore with `{app="ephemcache"}`. **This is the only
   complete record**: the vcluster reaps completed pods within about half an
   hour, and the CronJob's history limits retain Job *objects*, which carry
   status but no output. `kubectl logs` therefore works only while a job is
   running or shortly after.

4. **Read the durable log copy.** With `logToFile` enabled, each run also writes
   `logs/<UTC-start>.log` in the output tree, timestamped per line and with
   progress bars collapsed. Mode 700, so not web-readable.

5. **Ask Argo.** `argocd app get ephemcache` for sync and health.

### Running it by hand

Derive a one-off Job from the CronJob rather than editing the CronJob:

```
kubectl -n ephemcache create job --from=cronjob/ephemcache manual-$(date -u +%Y%m%dT%H%M%SZ)
```

To force a rebuild of a night that already has a cache, move that cache aside
first — the run will otherwise skip in under a second. Note that stage 1 will
re-fetch MPCORB if the catalogs are also absent, adding roughly 9 minutes.

Chunk count is overridable per run with the `NCHUNKS` environment variable
without rebuilding the image.

### What to expect

Measured at `usdfdev`, 100 partitions, 48 CPU / 64 Gi:

| | |
|---|---|
| Wall time, stages 2–4 | **~17 min** (1031 s and 1061 s in two runs) |
| Wall time including stage 1 | **~26 min** |
| Run-to-run variance | 2.8% |
| Peak memory | **~21 GiB** anonymous, against a 64 Gi limit |
| `_workdir` peak | 41 GB |
| Output size | ~213 MB per night |
| A run that skips | under 1 second |

Two results worth recording because they contradict plausible intuitions.
**Increasing the partition count makes it slower**: 300 partitions took 1270 s
against 1061 s for 100, about 20% worse, because each `sorcha` process must load
~780 MB of SPICE kernels at startup and tripling the process count triples that
fixed cost. And **output is bit-for-bit reproducible** at a fixed partition
count — four independent runs produced md5
`be922ce37232562008ec50affea0c5f9` — which makes it easy to confirm that a
change was behaviour-neutral.

## Maintaining and upgrading

### Changing the cache builder

Push to `lsst-gen-ephemcache`; Actions builds and publishes; the CronJob picks
the new image up on its next run because the tag is mutable and the pull policy
is `Always`. No chart change is needed today, which is convenient and is also
exactly why the tag must be pinned before production.

Note that GitHub can take several minutes to create the push-triggered workflow
run. Do not conclude the trigger is broken and dispatch manually — the workflow
sets `concurrency: cancel-in-progress: true`, so a manual dispatch races the real
run and one of them is cancelled.

### Changing the deployment

Open a PR against `lsst-sqre/phalanx`, then sync in Argo. Before pushing, run the
repository's own lint gate rather than any hand-rolled equivalent:

```
uvx prek run --all-files
```

`helm-docs` alone runs three times there, once per chart root
(`applications`, `environments`, `charts`), and running only one of them leaves a
generated `README.md` stale and fails CI.

(upgrading-mpsky)=
### Upgrading mpsky

`mpsky` appears **twice**: as the deployed service, and inside the cache-builder
image, where `mpsky build` writes the cache. Those two must agree on file
formats.

They have already been observed diverging. A builder image whose `mpsky` came
from `main` could not even parse the deployed service's query response, failing
in `ipc_read`. Both are currently built from the `auto-load` branch, and the
compatibility of the current pair has been verified end to end, including the
extended-elements path that reads the 1.6 GB sqlite database.

**When upgrading either side, upgrade both, and re-verify.** The check is cheap:
point `mpsky query --source http://172.24.10.34 <t> <ra> <dec> --radius 1
--return-elements extended` at the service from inside the builder image and
confirm it decodes.

### Upgrading the SPICE kernels

The ~780 MB of JPL/NAIF kernels are baked into the image so that the image tag
answers "which physics produced this cache?". Updating them therefore requires an
image rebuild. The build must `chmod -R a+rX` them afterwards: `pooch` writes
each download through a mode-600 temporary file and the build runs as root, so
without the chmod every kernel is unreadable to the non-root pod.

## Troubleshooting

Symptoms actually observed, with causes and fixes.

| Symptom | Cause | Fix |
|---|---|---|
| `night=N not in available in <url>` and HTTP 400 | That night has no cache in the datastore. Remember `night = floor(t) - 1` before concluding it is missing | Build that night, or accept the gap |
| HTTP 403 fetching a cache that is listed in the index | File is not world-readable. The web server is not a member of `rubin_users`, so mode 660 is not enough | `chmod o+r` the file; new output is already 644 |
| Job stays `Pending` indefinitely | Missing toleration or `nodeSelector`, or a request too large for the general pool | Ensure both RSP keys are present |
| `The JPL planet ephemeris file has not been found` | Kernels present but unreadable — a permission denial wearing a missing-file message | `chmod -R a+rX` at image build time |
| `ValueError: output array is read-only` in `mpsky build` | pandas 3 makes Copy-on-Write mandatory, so `.values` arrays are read-only | The image pins `pandas<3`; do not relax it without fixing mpsky |
| Pod OOM-killed during stage 3 | `ncores` does not match `resources.limits.cpu`, so parallelism was taken from `nproc` | Set them together |
| Job reports success but no cache appeared | A pipeline whose last stage is `tee` masks the real exit status | The entrypoint exits with `${PIPESTATUS[0]}`; preserve that if editing |
| No logs available for a run that finished | Pods are reaped within ~30 minutes | Use Loki, or the `logs/` copy |
| `furnsh_c --> FURNSH --> ZZLDKER` | `meta_kernel.txt` records the **absolute path** of the directory the kernels were bootstrapped into; if the cache is moved or staged, SPICE points at a directory that no longer exists | Ensure `PATH_VALUES` matches the kernels' actual location |
| Whole pipeline silently capped at 100 chunks | A glob of the form `orbits-000*.csv` matches only three-digit chunk numbers | Fixed; the glob is `orbits-*.csv` |

A general lesson from several of these: **a check that passes is not the same as
a check that verified something.** A kernel-readability check once reported
`all 0 kernels readable` for months because `find` does not descend into a
symlinked directory. When diagnosing, confirm that the check you are trusting
actually enumerated anything.

(gaps)=
## Known gaps

Ordered by how much they should worry a new owner.

1. **No failure alerting.** Nothing reports a failed or missing run. With real
   consumers and a hard nightly deadline, this is the top priority. A meaningful
   alert needs both a failure signal and a lateness signal — "no cache for
   tonight by *T*" — since a run that never starts produces no failure at all.
2. **References are not frozen.** The image tag is mutable and the bundled
   `mpsky` tracks a branch tip. Two builds of the same Dockerfile can differ.
3. **No retention policy.** Nothing prunes old caches or catalogs. At roughly
   2 GB per night against ~1.4 TB free, that is on the order of two years of
   headroom — not urgent, but unbounded.
4. **`usdfdev` only.** There is no production deployment.
5. **`_workdir` in the served tree.** ~41 GB of intermediates are web-visible
   between runs. Cleaning at the end of a run, or moving it to a scratch volume,
   would fix it. Note that a scratch volume was tried and reverted: it produced
   no measurable wall-time benefit and made `_workdir` a mountpoint, which broke
   code that creates and removes that directory. If it is revisited, do it for
   containment, not for speed.
6. **Permissions are fragile.** Writes depend on a single named-user ACL and a
   hardcoded uid, as described above.
7. **Night 61257 (2026-08-05) is missing** from production, because the legacy
   cron was disabled before producing it. Queries in that window return HTTP 400.
   Judged not worth backfilling.
8. **Efficiency headroom, unclaimed.** The pod is CPU-throttled 10–17% of
   scheduling periods because each `sorcha` process sizes its BLAS/numba thread
   pool from the node's 128 cores rather than the pod's 48-core quota. Setting
   `OMP_NUM_THREADS` and friends to 1 is untested but promising. Memory is also
   over-provisioned — a ~21 GiB observed peak against a 64 Gi limit — though 1/min
   sampling may have missed shorter spikes, so reduce it with measurement rather
   than arithmetic.
9. **Minor `mpsky` bugs.** In no-datastore mode a cache miss reaches
   `next(caches.values())`, which raises `TypeError` because `dict_values` is not
   an iterator. And `--return-elements basic` reports `Vmag` as `nan`; only
   `extended` populates it.

## What promotion to production requires

Not yet done, and listed here as a handover checklist rather than a plan of
record. The first three are prerequisites rather than improvements.

1. **Wire up alerting**, including a lateness check, per gap 1.
2. **Freeze the references.** Pin `image.tag` to a `sha-<commit>` tag, and pin
   the bundled `mpsky` to a commit rather than a branch tip.
3. **Assign an owner**, and record the owning team and Slack channels — both in
   this note and in the [df-ops service
   page](https://df-ops.lsst.io/usdf-applications/ap/ephemcache/index.html),
   whose contact fields are currently blank.
4. **Decide a retention policy** for caches and catalogs.
5. **Re-examine the resource request** with measurement: memory looks reducible,
   and the throttling finding suggests the CPU request may be too, once thread
   oversubscription is fixed.
6. **Reconsider the node pool.** The RSP pool is shared with interactive
   `nublado` users, and this is a batch job. The nodes also carry
   `edu.stanford.slac.sdf.storage/sdf-group`, which expresses the requirement
   this job actually has — filesystem access plus capacity — more precisely than
   "the RSP project" does. Worth asking SDF whether a batch workload belongs
   here.
7. **Move `_workdir` out of the served tree**, per gap 5.
8. **Retire the legacy epyc builder**, below.

## The legacy epyc system

The original implementation is a **crontab on `epyc`**, a University of
Washington host outside Rubin infrastructure, which has been generating these
caches hourly for years and publishing them at
`https://epyc.astro.washington.edu/~mjuric/mpsky-data`. It is still running.

The USDF deployment described in this note is intended to replace it, and the
`mpsky` service at `usdfdev` has already been repointed to the USDF tree. Until
epyc is retired, remember that it is a live production service on
non-Rubin-operated hardware, and that its output remains the historical record:
of the 155 nights now in the USDF tree, the great majority were produced by epyc
and copied, not regenerated.

Retiring it requires confirming that nothing still reads its URL, and accepting
the USDF tree as the sole source. That is a decision for the incoming owner.

## References

| What | Where |
|---|---|
| Cache builder | <https://github.com/mjuric/lsst-gen-ephemcache> |
| `mpsky` | <https://github.com/mjuric/mpsky> |
| Deployment charts | `applications/ephemcache` and `applications/mpsky` in <https://github.com/lsst-sqre/phalanx> |
| Service pages | <https://df-ops.lsst.io/usdf-applications/ap/mpsky/index.html> |
| Logs | Grafana Explore, Loki, `{app="ephemcache"}` |
| Served output | <https://s3df.slac.stanford.edu/data/rubin/mpsky-data> |

Every measurement quoted in this note was taken on the deployed system at
`usdfdev` during 2026-08-06/07. Where a design choice was made and later
reversed, the reasoning and the measurement that settled it are given at the
point where they matter rather than in a separate history.

---
name: harvester-robot-tests
description: Set up, run, debug and extend the Robot Framework tests in harvester-tests/harvester_robot_tests against a local Harvester cluster. Covers which tests run in the CI (pr-baseline), venv, S3 backup target, LHv2 prerequisites, run.sh usage for agents, reading results, and reusing existing keywords/libs before adding new ones.
---

# harvester-robot-tests

Operational know-how for `harvester-tests/harvester_robot_tests`. Architecture, layer rules, naming, suite setup/teardown and tagging live in the repo (`AGENTS.md`, `harvester_robot_tests/README.md`). Read those, do not repeat them here.

## Before you write any test code: reuse first

The keyword API must stay small and tidy. Before adding anything, search what exists, in this order:

```
cd harvester_robot_tests
ls keywords/*.resource                                   # layer 2, one file per component
grep -rin "<word>" keywords/ libs/keywords/              # keyword names and Python wrappers
grep -rn "def <word>" libs/                              # components (layer 4)
source venv/bin/activate && export PYTHONPATH=$PWD/libs:$PWD/../apiclient
python -m robot.libdoc keywords/<component>.resource list    # all keywords of a resource file
python -m robot.libdoc keywords/<component>.resource show "<keyword>"   # arguments and docs
```

Also grep `tests/` for a similar scenario: copy its setup/teardown pattern.

Rules:
- Use an existing keyword if it does the job, even if you need to pass a new optional argument.
- Extend an existing keyword/lib (new optional argument, small helper next to its siblings) before creating a new one. Add to the existing `keywords/<component>.resource`, `libs/keywords/<component>_keywords.py` and `libs/<component>/` instead of making new files.
- A new file or new component only when no existing component fits (and then follow the same shape as a sibling component).
- Do not add a keyword that wraps one existing keyword with different defaults. Do not add a REST/CRD variant nobody needs: if a suite is CRD only, say so (`Skip If` on the strategy, like `test_vm_backup_longhorn_v2.robot`).
- Before finishing, re-grep for the names you added: no near-duplicates of existing keywords.
- Pure API features (no `crd.py` equivalent, e.g. list queries or websocket watch) may be REST only. See `libs/vm_list/` as a small example.

### New code is shared code

Whatever you add becomes part of the keyword API that other suites call. Write it so the next suite can use it without changing it:
- Describe the resource, not your test: `VM is deleted    ${name}    ${namespace}=default`, not `Delete VM Of List Test`. Suite-specific names, fixtures and constants stay in the suite or in suite-level keywords of the resource file, not in `libs/`.
- Pass values in, do not hardcode them: namespace, labels, sizes, filters, timeouts are parameters. Use the defaults of the sibling keywords (`namespace=default`, `${WAIT_TIMEOUT}`), so existing calls keep working.
- New parameters are optional with a default that keeps the old behaviour. Forward them through every layer (`.resource` -> `*_keywords.py` -> component -> CRD and REST implementation), not only in the strategy you run.
- Prefer generic over specific: one keyword with a parameter (`filter`, `ignore_order`, `&{params}`) instead of several variants that differ in one value. Return plain data (dicts, lists, ids) that callers can check themselves, instead of asserting a single case inside the lib.
- Keep the signature of a sibling: same argument names and order as the neighbouring keywords of the component (`vm_name`, `namespace`, `timeout`).
- Easy to extend: when the next suite needs more, it should add an optional parameter or a branch in one place, not rewrite or copy your code. Build specs and requests in one function per resource (e.g. one `create` that takes `**kwargs`), keep each function doing one thing, and avoid logic spread over several layers or duplicated between CRD and REST.
- Do not build for imagined callers: generic means usable by the next suite as is and easy to grow, not extra options nobody passes.

## CI or not: tell the developer

Only some Robot tests run in the CI of harvester PRs: those tagged `pr-baseline` (CRD, parallel, no special hardware). Everything else runs only when someone starts it.

When you create or change a test, say explicitly which case it is:
- It carries `pr-baseline`: it will run on every PR. It must be CRD-capable and parallel-safe (own resources, unique names). Hardware/host dependent suites (LHv2, backup target, HA) do not belong there.
- It does not: it will not run in the CI. Point that out, and mention that a PR can still pull it in through lines in its description, e.g. `E2E-Tests-Files: tests/regression/vm/<suite>.robot` (also `E2E-Tests-Branch`, `E2E-Tests-Tags`; see `.github/workflows/e2e-tests.yml` in harvester/harvester).

Do not add `pr-baseline` to a suite that has not proven stable on a real cluster. Ask the developer. "E2E tests" of a harvester PR means these Robot tests, not `make test-integration` (Go).

Reproduce the CI run locally: `./run.sh -i pr-baseline -p 8`.

## Environment

### Cluster access

- Local test cluster: mostly the Vagrant PXE cluster, VIP `192.168.0.131`, nodes `192.168.0.3[0-2]`. The endpoint is in `.env` (`HARVESTER_ENDPOINT`).
- `.env` holds the admin password. Never `cat` it or print it into logs/answers. `grep -E '^HARVESTER_ENDPOINT' .env` is fine.
- Check reachability first, with a timeout (a plain `kubectl get nodes` hangs for minutes if the cluster is down):

```
kubectl get nodes --request-timeout=5s
curl -sk -m 5 -o /dev/null -w '%{http_code}\n' "$(grep -E '^HARVESTER_ENDPOINT=' .env | cut -d= -f2)/ping"   # 200 = API up
```

If the cluster is unreachable: stop and tell the user. Do not start tests, they only time out. Offline you can still run `--dryrun` (see below).

### venv

```
cd harvester_robot_tests
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

`venv/`, `.env` and `results/` are git-ignored. If `venv/` already exists, just activate it.

### .env

`.env` exists and is configured; do not create or edit it. `run.sh` sources it automatically.

### S3 backup target (backup suites)

Backup suites call `Backup target is configured`: they reuse the cluster's `backup-target` setting, else configure NFS from `BACKUP_TARGET_NFS_ENDPOINT`, else **skip**. A skipped backup suite is not a pass. For S3 use MinIO (credentials `minioadmin`/`minioadmin`):

```
docker run -d -p 9000:9000 -p 9001:9001 cgr.dev/chainguard/minio server /data --console-address ":9001"
docker ps --format '{{.ID}} {{.Names}}' | head -3     # find the container
docker inspect -f '{{.NetworkSettings.IPAddress}}' <container>   # usually 172.17.0.2
```

- MinIO does not create buckets: create bucket `test` in the console (http://localhost:9001) or with `mc mb`.
- The endpoint must be reachable **from the Harvester nodes**, not only from your machine. Check from a node: `ssh rancher@192.168.0.30 curl -s -m 5 http://172.17.0.2:9000/minio/health/live`.

Configure in Harvester (Advanced > Settings > `backup-target`):

```
Type: S3
Refresh Interval: 0
Endpoint: http://172.17.0.2:9000
Bucket Name: test
Bucket Region: foo
User: minioadmin
Password: minioadmin
Virtual Hosted-Style: False
```

Same via kubectl:

```
kubectl patch settings.harvesterhci.io backup-target --type merge -p \
 '{"value":"{\"type\":\"s3\",\"endpoint\":\"http://172.17.0.2:9000\",\"bucketName\":\"test\",\"bucketRegion\":\"foo\",\"accessKeyId\":\"minioadmin\",\"secretAccessKey\":\"minioadmin\",\"virtualHostedStyle\":false,\"refreshIntervalInSeconds\":0}"}'
kubectl get settings.harvesterhci.io backup-target -o jsonpath='{.status.conditions}'   # configured=True
```

Ask the user before changing the cluster's `backup-target` if it already has a value.

### LHv2 prerequisites

`test_vm_backup_longhorn_v2.robot` and `tests/regression/host/` need:
- the `longhorn-v2-data-engine-enabled` setting enabled (Advanced > Settings). The suite enables it when needed.
- an additional LHv2 disk per node: a free NVMe/extra disk, at least 40 GiB. In the Vagrant cluster add the disk to the VMs first. Check with `kubectl get blockdevices.harvesterhci.io -A`.
- a backup target (for the backup suites).
- CRD strategy only.

If a prerequisite is missing, tell the user what to provision, do not fake it.

## Running tests

Always run from `harvester_robot_tests/`.

### Check without a cluster (do this after every change)

```
source venv/bin/activate && export PYTHONPATH=$PWD/libs:$PWD/../apiclient
robot --dryrun --output NONE --log NONE --report NONE tests/regression/<area>/<suite>.robot
```

Dry run catches syntax errors, unknown keywords and wrong argument counts. It does not catch wrong Python behaviour, `Evaluate` mistakes at runtime or timing problems.
Lint Python changes: `flake8 --max-line-length 99 libs/` (CI runs `tox -e pep8` on the whole repo).

### Run on the cluster

```
source venv/bin/activate
./run.sh -f tests/regression/<area>/<suite>.robot
```

Agent specifics:
- Activate the venv first or pass `-W`, otherwise `run.sh` asks `Continue? (y/N)` on stdin and blocks.
- Suites take minutes to an hour (image import, VM boot). Run them in the background (`background: true` on the shell tool) with a log file, and do not poll. You get notified when it ends:

```
./run.sh -f tests/regression/backup/test_vm_backup_longhorn_v2.robot -d /tmp/opencode/robot-results > /tmp/opencode/robot-run.log 2>&1
```

- Narrow runs: `-t "Test case name"` (one case), `-s vm` (folder/suite name), `-i p0 -e backup` (tags). `-v NAME:value` overrides a variable (e.g. `-v WAIT_TIMEOUT:1200`). `-L DEBUG` for more log.
- Strategy: `-S rest` runs the same suite against the REST API; default is `crd`. CRD needs a working kubeconfig (`kubectl get nodes` must work).
- Parallel: `-p N` (pabot, suite level). Only for suites that are parallel safe. For debugging a failure always run serially.
- LVM suites are excluded from broad runs; run them explicitly (`-f tests/regression/addon/lvm`).
- Keep a failing run's resources for inspection: skip the teardowns with the direct `robot` call below. Clean up manually afterwards (`kubectl get vm,pvc,vmimage -A | grep <suffix>`).

Direct `robot` call (needed for options `run.sh` does not offer):

```
source venv/bin/activate
set -a; source .env; set +a      # as run.sh does, do not echo it
export PYTHONPATH=$PWD/libs:$PWD/../apiclient
robot -d /tmp/opencode/robot-results --prerunmodifier skip_teardown.SkipSuiteTeardown <suite>
```

### Results

`run.sh` prints the exit code (0 = all passed). Output goes to `$ROBOT_OUTPUT_DIR` (.env: `./results`) or `-d`, with timestamped `output-*.xml`, `log-*.html`, `report-*.html`. Read failures without opening HTML:

```
python3 - <<'PY'
import glob, xml.etree.ElementTree as ET
for f in sorted(glob.glob("<outdir>/output*.xml")):
    for t in ET.parse(f).iter("test"):
        st = t.find("status")
        if st.get("status") != "PASS":
            print(st.get("status"), t.get("name"), "-", (st.text or "").strip()[:300])
PY
```

A suite that reports `SKIP` (missing backup target, wrong strategy, cluster too old) did not test anything. Report it as skipped, not passed.

## Workflow for a test change

1. Search for existing keywords/libs (see top).
2. Implement in the right layer, smallest change.
3. `--dryrun` the suite, plus every other suite that imports a resource file you changed (`grep -rl "<file>.resource" tests/`).
4. flake8 on touched Python.
5. If the cluster is up, run the suite (or at least the changed test cases with `-t`) and report the real result. If it is not up, say that only the dry run was done.
6. Clean up: no leftover namespaces/VMs/images from your runs, no `results/` or `.env` in commits.

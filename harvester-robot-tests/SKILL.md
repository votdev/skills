---
name: harvester-robot-tests
description: Set up, run, debug and extend the Robot Framework tests in harvester-tests/harvester_robot_tests against a local Harvester cluster. Covers which tests run in the CI (pr-baseline), S3 backup target, LHv2 prerequisites, run.sh usage for agents, reading results, and reusing existing keywords/libs before adding new ones.
metadata:
  author: Volker Theile
  email: vtheile@suse.com
---

# harvester-robot-tests

Operational know-how for `harvester-tests/harvester_robot_tests`. Architecture, layer rules, naming, suite setup/teardown, BDD test steps (Given/When/Then) and tagging live in the repo (`AGENTS.md`, `harvester_robot_tests/README.md`, `.github/COPILOT_INSTRUCTIONS.md`, which the reviews check). Read those, do not repeat them here.

## Before you write any test code: reuse first

The keyword API must stay small and tidy. Before adding anything, search what exists, in this order:

```
cd harvester_robot_tests
ls keywords/*.resource                                   # layer 2, one file per component
grep -rin "<word>" keywords/ libs/keywords/              # keyword names and Python wrappers
grep -rn "def <word>" libs/                              # components (layer 4)
source venv/bin/activate
python -m robot.libdoc keywords/<component>.resource list    # all keywords of a resource file
python -m robot.libdoc keywords/<component>.resource show "<keyword>"   # arguments and docs
```

Also grep `tests/` for a similar scenario: copy its setup/teardown pattern.

Rules:
- Use an existing keyword if it does the job, even if you need to pass a new optional argument.
- Extend an existing keyword/lib (new optional argument, small helper next to its siblings) before creating a new one. Add to the existing `keywords/<component>.resource`, `libs/keywords/<component>_keywords.py` and `libs/<component>/` instead of making new files.
- A new file or new component only when no existing component fits (and then follow the same shape as a sibling component). A new kind of resource gets a component of its own; do not put it into the component of the suite you are writing, the next suite will not find it there.
- Existing code that breaks the layer rules is no precedent. Do not copy it, and do not rewrite it as a side effect of your change.
- Do not add a keyword that wraps one existing keyword with different defaults. Do not add a REST/CRD variant nobody needs: if a suite is CRD only, say so (`Skip If` on the strategy, like `test_vm_backup_longhorn_v2.robot`).
- Before finishing, re-grep for the names you added: no near-duplicates of existing keywords.
- Give every new keyword a `[Documentation]` that says what it does and what its arguments mean: `libdoc show` (above) prints it, that is how the next one finds the keyword.
- Pure API features (no `crd.py` equivalent, e.g. list queries or websocket watch) may be REST only. See `libs/list_api/` as a small example.

### New code is shared code

Whatever you add becomes part of the keyword API that other suites call. Write it so the next suite can use it without changing it:
- Describe the resource, not your test: `VM is deleted    ${vm_name}    ${namespace}=default`, not `Delete VM Of List Test`. Suite-specific names, fixtures and constants stay in the suite or in suite-level keywords of the resource file, not in `libs/`.
- Pass values in, do not hardcode them: namespace, labels, sizes, filters, timeouts are parameters. Use the defaults of the sibling keywords (`namespace=default`, `${WAIT_TIMEOUT}`), so existing calls keep working.
- New parameters are optional with a default that keeps the old behaviour. Forward them through every layer (`.resource` -> `*_keywords.py` -> component -> CRD and REST implementation), not only in the strategy you run.
- Prefer generic over specific: one keyword with a parameter (`filter`, `ignore_order`, `&{params}`) instead of several variants that differ in one value. Return plain data (dicts, lists, ids) that callers can check themselves, instead of asserting a single case inside the lib.
- Keep the signature of a sibling: same argument names and order as the neighbouring keywords of the component (`vm_name`, `namespace`, `timeout`).
- Easy to extend: when the next suite needs more, it should add an optional parameter or a branch in one place, not rewrite or copy your code. Build specs and requests in one function per resource (e.g. one `create` that takes `**kwargs`), keep each function doing one thing, and avoid logic spread over several layers or duplicated between CRD and REST.
- Do not build for imagined callers: generic means usable by the next suite as is and easy to grow, not extra options nobody passes.

## CI or not: tell the developer

Only some Robot tests run in the CI of harvester PRs: those tagged `pr-baseline` (CRD, parallel, no special hardware). Everything else runs only when someone starts it.

When you create or change a test, say explicitly which case it is:
- It carries `pr-baseline`: it will run on every PR (rules for the tag: README, "PR baseline").
- It does not: it will not run in the CI. Point that out, and mention that a PR can still pull it in through lines in its description, e.g. `E2E-Tests-Files: tests/regression/vm/<suite>.robot` (also `E2E-Tests-Branch`, `E2E-Tests-Tags`; see `.github/workflows/e2e-tests.yml` in harvester/harvester).

The CI runs the suites in parallel (`-p 8`) on one shared cluster, in a container (`docker-run.sh`, so a new Python dependency must be in `requirements.txt`). Before you propose `pr-baseline` for a test, check that nothing it compares is changed by other suites (e.g. every image creates a storage class, so a whole-cluster list of storage classes is not stable), that it needs no internet besides the artifact server (do not ask the developer, look: the CI overrides `OPENSUSE_IMAGE_URL` with its artifact server and only mirrors docker.io, and no `pr-baseline` suite uses `UBUNTU_IMAGE_URL` or a public registry, so assume a public image is not reachable and exclude the test), and that it has no fixed sleep for a product race. Exclude single tests with `[Tags]    -pr-baseline` when the suite carries `Test Tags    pr-baseline`.

A test that needs a product fix that is not merged yet stays out with `-pr-baseline` until the fix is merged: the CI builds the code of each PR, so every PR without the fix would fail.

Do not add `pr-baseline` to a suite that has not proven stable on a real cluster. Ask the developer. "E2E tests" of a harvester PR means these Robot tests, not `make test-integration` (Go).

## Environment

### Cluster access

- Local test cluster: mostly the Vagrant PXE cluster, VIP `192.168.0.131`, nodes `192.168.0.3[0-2]`. The endpoint is in `.env` (`HARVESTER_ENDPOINT`).
- `.env` exists and is configured; do not create or edit it. It holds the admin password: never `cat` it or print it into logs/answers. `grep -E '^HARVESTER_ENDPOINT' .env` is fine.
- Check reachability first, with a timeout (a plain `kubectl get nodes` hangs for minutes if the cluster is down):

```
kubectl get nodes --request-timeout=5s
curl -sk -m 5 -o /dev/null -w '%{http_code}\n' "$(grep -E '^HARVESTER_ENDPOINT=' .env | cut -d= -f2)/"   # 000 = no answer, any other code = the VIP answers
```

If the cluster is unreachable: stop and tell the user. Do not start tests, they only time out. Offline you can still run `--dryrun` (see below).

### S3 backup target (backup suites)

Without a backup target the backup suites skip (README, "Optional Requirements"). For S3 use MinIO (credentials `minioadmin`/`minioadmin`):

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
source venv/bin/activate
robot --dryrun --output NONE --log NONE --report NONE tests/regression/<area>/<suite>.robot
```

Dry run catches syntax errors, unknown keywords and wrong argument counts. It does not catch wrong Python behaviour, `Evaluate` mistakes at runtime or timing problems.
The venv pins the Robot Framework version in `requirements.txt` (check with `python -c "import robot; print(robot.__version__)"`). Syntax from newer releases fails there, so do not use it, e.g. `GROUP`, `VAR ... scope=SUITES` or typed arguments (`[Arguments]    ${count: int}`). The first two already fail in the dry run, typed arguments pass it and fail at run time.
Typical runtime-only bug: `Evaluate    [$x + n for n in $names]` fails ("variable cannot be seen") because `$x` is not visible inside a comprehension. Write `['${x}' + n for n in $names]` or avoid the comprehension.

### Run on the cluster

```
source venv/bin/activate
./run.sh -f tests/regression/<area>/<suite>.robot
```

Agent specifics:
- Activate the venv first or pass `-W`, otherwise `run.sh` asks `Continue? (y/N)` on stdin and blocks. Every shell call is a new shell: activate again in each call, otherwise `run.sh` stops with "Robot Framework not installed".
- Suites take minutes to an hour (image import, VM boot). Run them in the background (`background: true` on the shell tool) with a log file, and do not poll. You get notified when it ends:

```
./run.sh -f tests/regression/backup/test_vm_backup_longhorn_v2.robot -d /tmp/opencode/robot-results > /tmp/opencode/robot-run.log 2>&1
```

- Do not `pkill -f robot`, it kills your own shell too.
- The console log is full of `DEBUG` lines and colour codes: `sed 's/\x1b\[[0-9;]*m//g' run.log | grep -av DEBUG | grep -a "tests,"`. The result summary is the last `N tests, N passed, N failed` line.
- A hung test is usually a wait loop that logs nothing (e.g. `wait_for_deleted`, default 600 s). For debug runs shorten it with `export WAIT_TIMEOUT=60`.
- After an aborted or failed run, and after runs with `-t`, delete its namespaces (label `harvesterhci.io/test=robot-framework`; `kubectl get ns -o name | grep <prefix> | xargs kubectl delete --wait=false`), also leftover VMs/PVCs of your own experiments.
- Tag expressions: Robot needs the operators in upper case (`AND`, `OR`, `NOT`; `p0 not backup` fails with "contains no tests matching tag"). `run.sh` passes `-i` through `eval` unquoted, so a space splits the expression and Robot fails with "Parsing '.../NOT' ... failed". Write it without spaces, as the harvester CI does (`pr-baselineOR<tag>`): `-i p0NOTbackup`. Spaces are fine in a direct `robot --include "p0 NOT backup"` call.
- Run `-S crd` and `-S rest` when you touched both implementations: REST goes through the API stores and finds other problems than CRD.
- Debug a failure serially, not with `-p`.
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
import glob
from robot.api import ExecutionResult, ResultVisitor

class Failed(ResultVisitor):
    def visit_test(self, test):
        if test.status != "PASS":
            print(test.status, test.full_name, "-", test.message[:300])

for f in sorted(glob.glob("<outdir>/output*.xml")):
    ExecutionResult(f).visit(Failed())
PY
```

### What the first cluster run finds (the dry run cannot)

Expect failures in a new suite and fix them one by one, rerun only the failed test with `-t` (run.sh takes one `-t`; use a glob like `-t "VM Schema*"` to be specific). Typical findings:
- Python signature clashes between layers (a kwarg that is passed twice, e.g. `namespace` positional and in `**kwargs`).
- Leftovers of earlier or parallel runs: a failed run keeps its resources, so a filter on a cluster-scoped list (namespaces, labels) also needs the unique prefix of the run.
- Objects the cluster creates itself (every namespace has a `kube-root-ca.crt` config map): filter, do not expect only your own objects.
- The real format of an API answer differs from the assumption (look at the real answer, do not guess).
- `Evaluate`: `\d` loses its backslash, write `[0-9]`; a continuation line (`...    if x`) becomes the next argument (`modules`), keep the expression on one line.
- Races of the product (an operation right after the creation of the object fails or leaves resources behind). Reproduce with a small python script (`init_harvester_api_client` from `libs/utility/utility.py`, `set -a; . ./.env; set +a`), then wait for the settled state in the test with a comment why, instead of retrying a non-idempotent call. Check that the state you wait for really is the one that matters (reproduce 10 times). If the product shows no such state, a short fixed wait in a named keyword with `[Documentation]` of the cause is acceptable; report the product race to the developer.
- To find the cause of such a race, read the server code (the harvester repo, `vendor/` for Steve and Wrangler) and measure it: the same call after a delay of 0, 1, 2 and 5 s, five runs each, and note per run the answer, whether the object is gone and what it left behind. Clean up with the same call you test, otherwise your own cleanup causes leftovers.
- A client retry after a 500 hides the real first answer (a later 404). Use `set_retries(1, total=0)` for calls whose first answer matters.
- When the product answers with an error that guarantees nothing happened (e.g. 409 Conflict), retry that call in the test with `Wait Until Keyword Succeeds` instead of a fixed `Sleep`. A retry is only safe when the failed call changed nothing.
- A successful delete can answer 204 instead of 200 (the object is gone at once). A lib that asserts `== 200` reports a failure and its retry then gets a 404. Accept both.
- Filter and scope options (query parameters, subscribe options) may not mean what their name suggests. Look up in the server code what an option is compared with, and check the filter from both sides: the matching value yields results, a different value yields none.

A suite that reports `SKIP` (missing backup target, wrong strategy, cluster too old) did not test anything. Report it as skipped, not passed.

## Workflow for a test change

1. Search for existing keywords/libs (see top).
2. Implement in the right layer, smallest change. Write the test steps in the BDD style of the README ("Test Steps (BDD Style)"); a step that computes something (`Evaluate`, `${{ }}` with logic) or digs into a response (`${result}[body][count]`) is a sign of a missing keyword, literal test data is fine.
3. `--dryrun` the suite, plus every other suite that imports a resource file you changed (`grep -rl "<file>.resource" tests/`).
4. flake8 on touched Python (`AGENTS.md`).
5. If the cluster is up, run the suite (or at least the changed test cases with `-t`) and report the real result. If it is not up, say that only the dry run was done. When a test verifies a fix of the product, run the reproduction against the old image first (so you see it fail), then against the fixed one.
6. Clean up: no leftover namespaces/VMs/images from your runs.

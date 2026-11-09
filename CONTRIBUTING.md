# Contributing

Thank you for your interest in improving Full System Monitoring. This guide
covers the license terms, the sign-off requirement, the branching workflow,
the review process, and the checks every change should pass.

We welcome pull requests of these kinds:

- **Dashboards**: new Grafana dashboards, or fixes to the existing GPU, host,
  and vLLM inference views — wrong units, mis-scaled panels, missing series.
- **Collection and scrape configuration**: Prometheus scrape jobs and
  recording rules, and `xpumd` exporter settings.
- **Deployment**: `docker-compose.yml` improvements, including secure
  defaults, resource limits, and image pinning.
- **Docs and CI**: the README, the agent guide, and the SBOM workflow.

## License

Full System Monitoring is licensed under the terms in [LICENSE](LICENSE).
By contributing to the project, you agree to the license and copyright terms
therein and release your contribution under these terms.

Third-party components — Intel XPU Manager, Prometheus, Grafana, vLLM — are
referenced, not redistributed, by this repository. See [NOTICE](NOTICE).

## Sign your work

Please use the sign-off line at the end of each patch. Your signature
certifies that you wrote the patch or otherwise have the right to pass it on
as an open-source patch. The rules are simple: if you can certify the below
(from [developercertificate.org](http://developercertificate.org/)):

```
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.
660 York Street, Suite 102,
San Francisco, CA 94110 USA

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.

Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

then add a line to every git commit message:

    Signed-off-by: Joe Smith <joe.smith@email.com>

Use your real name. Pseudonyms and anonymous contributions are not accepted.

If your `user.name` and `user.email` git configs are set, `git commit -s`
adds the sign-off automatically.

## Branching and pull requests

All changes land through pull requests against `main`. Do not commit to
`main` directly.

1. Fork the repository, or clone it directly if you have write access.
2. Create a topic branch from the tip of `main`, named as described in
   [Branch naming](#branch-naming) below.
3. Make your change. Keep each commit to one logical change and sign it off.
4. Run the checks described below and make sure they pass.
5. Push the branch and open a pull request against `main`. Explain what
   changed and why, and link any related issue.
6. Address review feedback with follow-up commits or a rebase. Rebase on
   `main` rather than merging `main` into your branch.

### Branch naming

Use `<prefix>/<short-topic>`. Either prefix form is accepted:

| Form | Use it for | Examples |
| --- | --- | --- |
| `<type>/<topic>` | Changes described by what they do | `fix/quickview-gpu-units`, `feat/dcgm-style-panel`, `docs/branch-naming` |
| `<username>/<topic>` | Your own in-progress work | `yourname/fix-kv-cache-scale`, `yourname/add-power-panel` |

Recognized types are `feat` (new capability), `fix`, `docs`, `test`,
`chore` (tooling, CI, maintenance), and `refactor`.

Rules:

- Lowercase, with hyphens between words. No spaces, and no `#` or `~`.
- One slash. `feat/new-dashboard`, not `feat/dash/new`.
- Keep it short and specific. Name the thing you changed, so the branch is
  recognizable in a list of many: `fix/quickview-gpu-units`, not `fix/bug`.
- Nothing in CI parses the prefix, so it is a readability convention rather
  than a hard gate. Follow it anyway so branch lists stay scannable.

Topic branches are short-lived: merge them within days and delete them after
merge. Releases are tagged directly from `main`, so the project keeps no
long-lived release, `develop`, or version branches. Do not open a pull
request from your fork's `main`.

## Review process

- Request a review from any maintainer listed at the end of this guide.
- Open work in progress as a draft pull request. Mark it ready for review
  once it is complete and the checks pass.
- Simple changes need at least one maintainer approval before merging.
- Complex changes, such as a new dashboard or a change to the shipped
  deployment defaults, need at least two maintainer approvals.
- All CI checks must pass.
- When changing a dashboard panel, say in the pull request which metric and
  unit you verified it against. A panel that renders is not the same as a
  panel that is correct: a unit mismatch between the query and the panel can
  silently scale a value by 100.

## Checks before you push

Everything this repository ships is configuration, so the checks are about
parsing and schema rather than compilation.

Validate every YAML and JSON file:

```sh
python3 - <<'PY'
import glob, json, sys, yaml
ok = True
for f in ["docker-compose.yml", "xpumd/config.yaml",
          "prometheus/prometheus.yml", "prometheus/gpu_labels.yml",
          "grafana/provisioning/dashboards/dashboards.yml",
          "grafana/provisioning/datasources/prometheus.yml"]:
    try:
        yaml.safe_load(open(f)); print("YAML OK  ", f)
    except Exception as e:
        ok = False; print("YAML FAIL", f, e)
for f in sorted(glob.glob("grafana/dashboards/*.json")):
    try:
        json.load(open(f)); print("JSON OK  ", f)
    except Exception as e:
        ok = False; print("JSON FAIL", f, e)
sys.exit(0 if ok else 1)
PY
```

Validate the Compose file against the Compose schema. This is offline and
needs no running Docker daemon:

```sh
docker compose -f docker-compose.yml config --quiet
```

Check the Prometheus configuration and recording rules, if you have
`promtool`:

```sh
promtool check config prometheus/prometheus.yml
promtool check rules prometheus/gpu_labels.yml
```

Bring the stack up on a host with an Intel GPU and confirm every scrape
target is `up` before opening the pull request:

```sh
docker compose up -d
curl -s localhost:9090/api/v1/targets | python3 -m json.tool | grep -c '"health": "up"'
```

CI additionally runs Trivy over every image named in `docker-compose.yml`
and publishes a merged SPDX SBOM. That workflow scans the images a user
would pull; it does not scan this repository, which contains no code.

## Changing a dashboard

- Edit the JSON in `grafana/dashboards/`. Grafana provisions this directory
  read-only, so export from the Grafana UI and commit the result rather than
  hand-editing where you can.
- Keep the `uid` stable. Changing it breaks every existing bookmark, link,
  and annotation that points at the dashboard.
- Match the panel `unit` to what the query actually returns. `percent`
  expects a value already in percent; `percentunit` expects a 0–1 fraction
  and multiplies by 100 for display. Using `percentunit` on a query that
  already multiplies by 100 scales the displayed value by 100.
- Say in the panel description what the metric means and what a healthy
  value looks like, especially where a metric is easy to misread.
- Check the dashboard renders with no data as well as with data. A panel
  that only makes sense under load should say so rather than showing a
  misleading number while idle.

## Maintainers and contributors

Maintainers:

- Yogini Dandekar ([@yogit2020](https://github.com/yogit2020),
  <yogini.dandekar@intel.com>)

Contributors:

- Kilari, Manasa (<manasa.kilari@intel.com>)
- Robert Dower ([@rdower](https://github.com/rdower),
  <robert.dower@intel.com>)

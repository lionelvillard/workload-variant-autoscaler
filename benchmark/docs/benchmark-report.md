# Benchmark reports and Grafana snapshots

`benchmark/hack/benchmark_report.py`'s `configure` / `snapshot` / `report` /
`all` subcommands turn a single finished session into a standalone
`report.html`, with links to the relevant Grafana panels (vLLM KV-cache
utilization and queue size) for the exact time window of that session's
benchmark run.

For the live, multi-session dashboard (`serve`), see
[`interactive-dashboard.md`](interactive-dashboard.md) instead -- this doc
covers per-session reports and the Grafana setup they (and `serve`'s Live
dashboard links) depend on.

Grafana is expected to run locally (see "One-time setup" below). Prometheus
is always reached remotely, through OpenShift's Thanos Querier Route --
auto-discovered against the current kube context, no port-forwarding needed
for it. Pass `--prometheus-url` yourself to skip discovery (e.g. for a
plain, non-OpenShift Prometheus you still want to port-forward by hand).

## Grafana snapshots

Prometheus doesn't retain data forever, so a plain dashboard link goes stale.
To keep the link useful after retention expires, the tool captures a Grafana
**snapshot** — a frozen, self-contained copy of the panel data — immediately
after the run, while Prometheus still has it.

This assumes a local Grafana at `http://localhost:3000` that you install;
the tool points it at the benchmark cluster's Prometheus for you
(auto-discovered via OpenShift's Thanos Querier), provisions the dashboard
inside it, and drives its snapshot API.

## One-time setup

Run everything through `benchmark/hack/benchmark_report.sh` (not
`benchmark_report.py` directly) — it creates and maintains its own venv at
`benchmark/hack/.venv/` with `requirements.txt` installed, so it works even
on systems (e.g. Homebrew Python) where plain `pip install` is blocked. No
manual venv/pip step needed.

Install Grafana locally — either Homebrew:

```bash
brew install grafana
brew services start grafana
```

or Docker:

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana-oss
```

Either way it comes up at `http://localhost:3000` (default login `admin`/`admin`).

Provision the Prometheus datasource and the benchmark dashboard into your
local Grafana:

```bash
benchmark/hack/benchmark_report.sh configure
```

No Prometheus port-forward needed: Prometheus is always reached remotely,
through OpenShift's Thanos Querier. `configure` automatically:
1. discovers the `thanos-querier` Route's host in the `openshift-monitoring`
   namespace and uses `https://<host>` as the Prometheus datasource URL,
2. finds a ServiceAccount in `openshift-monitoring` already bound to the
   `cluster-monitoring-view` ClusterRole (skipping stale bindings whose
   ServiceAccount no longer exists) and mints a token from it,
3. sets TLS skip-verify on the datasource (Grafana doesn't have the
   cluster's internal CA bundle to validate the Route's cert).

It fails with an actionable error if no such ServiceAccount binding exists
yet — bind one first, e.g. `oc adm policy add-cluster-role-to-user
cluster-monitoring-view -z default -n openshift-monitoring`. Use
`--prometheus-service-account-namespace` to look in a different namespace,
and `--prometheus-token-duration` (default `8760h`) to control how long the
minted token lasts — re-run `configure` with a fresh one before it expires.
`--context` selects which kube context to discover against (default: the
current one).

This is idempotent — re-run it any time you're unsure Grafana is set up
correctly, or after editing `benchmark/config/grafana/dashboard.json`.

### Skipping Prometheus auto-discovery

If the cluster's Prometheus isn't reachable through OpenShift's Thanos
Querier (a non-OpenShift cluster, or one without in-cluster monitoring
enabled), pass `--prometheus-url` yourself and discovery is skipped
entirely -- e.g. a plain Prometheus you've port-forwarded by hand:

```bash
# kube-prometheus-stack (the Helm install used by benchmark/README.md's
# "Prepare a Kubernetes cluster" section)
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090

benchmark/hack/benchmark_report.sh configure --prometheus-url http://localhost:9090
```

### Why Thanos Querier specifically

On OpenShift, vLLM's `PodMonitor`-scraped metrics live in the
**user-workload** Prometheus while `kube-state-metrics` (needed for the
`kube_pod_labels` join -- see "Identifying a session's metrics" below) is
scraped by the **platform** Prometheus. Those are two separate TSDBs; a
query can't join across them directly. Thanos Querier merges both, so it's
the only endpoint where the join panels actually return data. This was
confirmed by querying each Prometheus directly: `prometheus-operated` in
`openshift-user-workload-monitoring` has `vllm:*` but zero
`kube_pod_labels` series, and `prometheus-operated` in
`openshift-monitoring` has `kube_pod_labels` but zero `vllm:*` series.

### Alternative: Grafana already running in-cluster (no port-forwarding)

If your cluster already runs its own Grafana (e.g. in a `grafana`
namespace, reachable via a Route) instead of a local `brew`/`docker`
install, point `configure` at it directly instead of port-forwarding
anything. Prometheus resolves the same way either way (auto-discovered via
Thanos Querier, unless you pass `--prometheus-url` yourself):

```bash
benchmark/hack/benchmark_report.sh configure \
  --grafana-url https://<grafana-route-host> \
  --grafana-user admin --grafana-password <...>
```

or, to reach Thanos Querier over the in-cluster Service DNS instead of its
Route (avoids a hop through the router, but only resolves from inside the
cluster -- fine since Grafana is in-cluster here) and pin the token to a
specific ServiceAccount, do it by hand:

```bash
benchmark/hack/benchmark_report.sh configure \
  --grafana-url https://<grafana-route-host> \
  --grafana-user admin --grafana-password <...> \
  --prometheus-url https://thanos-querier.openshift-monitoring.svc.cluster.local:9091 \
  --prometheus-bearer-token "$(oc create token <grafana-serviceaccount> -n <grafana-namespace> --duration=8760h)" \
  --prometheus-tls-skip-verify
```

This requires:
- The `<grafana-serviceaccount>` (the ServiceAccount your Grafana pod runs
  as) bound to the `cluster-monitoring-view` ClusterRole, so its token can
  read Thanos Querier. `oc get clusterrolebinding -o wide | grep
  cluster-monitoring-view` shows whether this already exists; if not:
  `oc adm policy add-cluster-role-to-user cluster-monitoring-view
  -z <grafana-serviceaccount> -n <grafana-namespace>`.
- `--prometheus-url` as the **in-cluster Service DNS name**
  (`thanos-querier.<namespace>.svc.cluster.local:9091`, the `web` port),
  not the external Route -- Grafana's pod resolves this directly over the
  cluster network, which is the whole point of eliminating
  port-forwarding. The token above is long-lived (`--duration=8760h`);
  re-run `configure` with a fresh one before it expires.
- `--prometheus-tls-skip-verify` because Grafana's pod doesn't have the
  cluster's internal CA bundle mounted by default, so it can't validate
  the Service's TLS certificate. This trades a small amount of transport
  trust for not having to inject the `openshift-service-ca.crt` ConfigMap
  into the Grafana pod -- if you'd rather not skip verification, mount
  that bundle and point Grafana at it (`jsonData.tlsAuthWithCACert`)
  instead of using this flag.

With this in place, nothing runs a `kubectl port-forward`: Grafana reaches
Prometheus over the cluster network, and your browser/script reaches Grafana
via its Route. `snapshot`/`report`/`all`/`serve` all take the same
`--grafana-url` (and, since they proxy queries through Grafana rather than
querying Prometheus directly, don't need the Prometheus flags at all --
only `configure` talks to Prometheus's URL, and only to write it into the
datasource). `serve`'s system-status strip does add `--prometheus-url`
`--prometheus-bearer-token`/`--prometheus-tls-skip-verify` support for its
own Prometheus reachability check, but that's cosmetic (a status badge),
not required for the dashboard panels to work.

## After a benchmark run

While the cluster's Prometheus still has the run's data (retention permitting --
no port-forward to keep alive, since it's reached remotely), run:

```bash
benchmark/hack/benchmark_report.sh all <session_dir>
```

where `<session_dir>` is the top-level `llmdbenchmark` session directory, e.g.
`benchmark/results/<user>-<timestamp>/` (the `--workspace benchmark/results`
passed to `standup`/`run`/`teardown`). This:

1. re-checks Grafana is configured (skip with `--skip-configure`),
2. captures a Grafana snapshot for each experiment under `<session_dir>/results/`,
   writing `grafana_snapshot.yaml` next to that experiment's other result
   files,
3. renders `<session_dir>/report.html` with a section per experiment: run
   metadata, the lifecycle metrics from `summary_lifecycle_metrics.json`,
   the workload config, and both Grafana links (permanent snapshot + live
   time-boxed dashboard).

Each step can also be run on its own — `configure`, `snapshot <experiment_dir>`,
`report <session_dir>` — see `benchmark/hack/benchmark_report.sh --help`.

If Grafana isn't running, `snapshot` fails for that experiment but `report`
still succeeds; the affected experiment's section just says no snapshot was
captured, with the command to capture one later (as long as Prometheus still
has the data).

## Identifying a session's metrics

The vLLM metrics this dashboard queries (`vllm:kv_cache_usage_perc`,
`vllm:num_requests_waiting`, `vllm:num_requests_running` -- see
`benchmark/config/grafana/dashboard.json`) carry only standard labels:
`namespace` and `pod`. There's no `session_id` on them directly. Every
"which metrics belong to this session" link -- the session-level
Observability line and Live dashboard link in the
[interactive dashboard](interactive-dashboard.md), and the Grafana snapshots
covered here -- **default** to working around that by scoping the Grafana
dashboard's `namespace` variable to the session's namespace and time-boxing
the query to that session's (or that experiment's run's) window. That's
sufficient as long as **one namespace is used by only one session at a
time**, which is how the "Start a session" form and the `run-benchmark`
skill both work in practice (a fresh namespace, or an intentionally reused
one you tear down before reusing).

It breaks down if two sessions ever overlap in the same namespace, or if a
namespace is reused enough that retention no longer cleanly separates their
time windows. For that case, the sibling `llm-d-benchmark` clone's harness
and serving pod templates now stamp a `llmdbench.ai/session-id: <session-id>`
label on the pods that produce these metrics (`config/templates/jinja/13_ms-values.yaml.j2`,
`14_standalone-deployment_yaml.j2`, `20_harness_pod.yaml.j2`), where
`<session-id>` is the llmdbenchmark workspace/session directory name (e.g.
`<user>-<timestamp>`) -- the same value used as the session ID everywhere
else in this dashboard. `session_id` is also a dashboard template variable
here (`benchmark/config/grafana/dashboard.json`) and is threaded through
`_live_dashboard_url`/`_capture_snapshot` in `benchmark/hack/benchmark_report.py`
the same way `namespace` is, so every generated link already carries
`var-session_id=<id>` and every snapshot's queries have `$session_id`
substituted.

Both shipped panels join onto the vLLM metrics via `kube_state_metrics`'s
`kube_pod_labels`, e.g.:

```
vllm:kv_cache_usage_perc{namespace=~"$namespace"}
  * on (pod, namespace) group_left(label_llmdbench_ai_session_id)
    kube_pod_labels{namespace=~"$namespace", label_llmdbench_ai_session_id=~"$session_id"}
```

`kube_pod_labels` carries one series per pod regardless of whether
`label_llmdbench_ai_session_id` is populated -- kube-state-metrics emits the
base metric (`pod`, `namespace`, `uid`) unconditionally, and only attaches
the `label_llmdbench_ai_session_id` dimension if that pod label is in its
`--metric-labels-allowlist`. **OpenShift's built-in cluster-monitoring
kube-state-metrics ships with `--metric-labels-allowlist=pods=[*]`** (every
pod label, confirmed on a live OCP cluster), so this works with zero extra
config there. A manually-installed `kube-prometheus-stack` (e.g. the "local
Kind" path earlier in this doc) doesn't allowlist custom labels by default --
set `--metric-labels-allowlist=pods=[llmdbench.ai/session-id]`, or the
chart's `kube-state-metrics.metricLabelsAllowlist` equivalent, to enable it
there. Either way, with the default `session_id=.*` the join is a no-op
multiply-by-1: panels render identically whether or not the allowlist is
configured. Setting `session_id` to one session's actual ID only filters
correctly once the allowlist is enabled -- without it, the label is simply
absent from every series, so a specific (non-`.*`) value matches nothing
and the panels go blank. That's the tell that the allowlist isn't set, not
a bug.

**Both sides of the join are scoped to `$namespace`.** `kube_pod_labels` is
cluster-wide (every namespace, every pod, on a shared Prometheus this can be
thousands of series), and Prometheus's vector matching requires the match
group (`pod`, `namespace`) to be unique on the `kube_pod_labels` side across
*everything the query returns* -- not just the pods that actually match the
left-hand side. On a busy shared cluster it's common for some unrelated
pod, anywhere in any namespace, to transiently have two label sets in
`kube_pod_labels` at once (e.g. mid-rollout, momentarily overlapping
old/new label values before the stale series drops out); if the right-hand
side isn't namespace-scoped, that unrelated duplicate makes Prometheus
reject the *entire* query with `"many-to-many matching not allowed"` --
observed live against a real OpenShift cluster's shared Prometheus while
validating this. Filtering `kube_pod_labels` by the same `$namespace` as the
left-hand side keeps the match group small enough that this essentially
never happens for a benchmark's own (few-pod) namespace.

Once the allowlist is configured, scope any panel (or an ad-hoc Explore
query) to one session by setting the dashboard's `session_id` variable, or
by appending the same join to a new PromQL expression -- always scoped to
`namespace=~"$namespace"` on the `kube_pod_labels` side for the reason
above.

## Extending the dashboard

`benchmark/config/grafana/dashboard.json` starts minimal — KV-cache
utilization and queue size — scoped by a `namespace` dashboard variable so
the same dashboard works across benchmark namespaces. Add panels the same
way (replica counts, scaling activity, latency) and re-run `configure` to
push the update; `snapshot` picks up whatever panels/targets exist on the
dashboard at capture time.

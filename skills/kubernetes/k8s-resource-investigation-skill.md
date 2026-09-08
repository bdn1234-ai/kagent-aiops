---
name: k8s-resource-investigation
description: >
  Investigate Kubernetes resource state, configuration, and events during
  incident RCA — pod/deployment/service/configmap status, cross-resource
  conflicts, scheduling failures, and connectivity issues. Use when
  diagnosing CrashLoopBackOff, ImagePullBackOff, Pending pods, Evicted pods,
  stuck rollouts, service connectivity failures, or configuration conflicts
  between resources. This skill owns resource STATE and CONFIGURATION, not
  time-series metrics (hand off to the Prometheus investigator) or log
  content analysis at scale (hand off to the Loki investigator when the
  observability stack is available; use k8s_get_pod_logs here only as a
  fallback when Loki isn't deployed or the pod is too new to have shipped
  logs yet).
---

# Kubernetes Resource Investigation

Diagnose Kubernetes issues by inspecting resource state, spec, and events
directly via the Kubernetes API — no telemetry backend required. This is the
layer that answers "what does the cluster currently believe is true" as
opposed to "what happened over time" (metrics) or "what did the application
log" (logs at scale).

## Guidelines

**Absence of evidence is not evidence.** Zero matching events, an empty
`describe` output section, or a resource that "looks fine" does not confirm
health — it may mean you queried the wrong namespace, the wrong label, or a
too-narrow time window. Report what you actually found, not what the
absence implies.

**A resource being `Running`/`Ready` does not mean it is correctly
configured.** A Service with zero matching endpoints still shows as a
normal Service object; a ConfigMap with conflicting values from two
unrelated Helm releases mounts and starts without error. State health and
configuration correctness are different questions — check both.

**Never conclude a root cause from a single resource in isolation when the
symptom suggests interaction between resources.** If a message contains
words like "conflict", "duplicate", "already exists", "only one ... can
be", "invalid config", or "provisioning error", the cause is almost always
a SECOND resource you have not looked at yet — usually created by a
different Helm release, operator, or team, sharing a label, selector, or
name with the one that's failing. Widen the search across the whole
cluster (not just the failing resource's namespace) before concluding.

**A resource created recently is a stronger lead than one that has existed
unchanged for a long time.** When multiple resources are candidates,
`creationTimestamp` is often the fastest way to spot which one caused a
regression.

**Restart count and phase are current-state snapshots, not history.**
`k8s_describe_resource` shows the current restart count and the most
recent events, but a pod may have already cycled through several distinct
failure reasons. Read the full event timeline (`k8s_get_events`), not just
the latest state, before committing to a single failure mode.

## Tool sequencing strategy

The following tools are available. Each answers a different kind of
question — do not default to `k8s_describe_resource` on the failing
resource alone when the question calls for a wider view.

1. **`k8s_get_resources`** — start here to enumerate what exists. Critical
   use case: **searching across the WHOLE cluster by label**, not just the
   namespace of the failing resource — this is how a second, unrelated
   resource causing a conflict is found. Prefer this over guessing a
   specific resource name when the cause could be any of several
   candidates.
2. **`k8s_describe_resource`** — use once you have a specific resource
   identified. Returns spec, status, conditions, AND recent events for
   that one object in a single call — usually the most information-dense
   single query available.
3. **`k8s_get_resource_yaml`** — use when you need to compare exact field
   values between two resources (e.g. does this Service's `selector` match
   that Pod's `labels`? do two ConfigMaps both set the same field to
   conflicting values?). `describe` output is summarized and can omit the
   exact field you need to diff.
4. **`k8s_get_events`** — use for the event TIMELINE across a namespace
   (or cluster), not just the tail attached to one resource's `describe`
   output. Necessary when a resource has cycled through multiple failure
   reasons, or when investigating a symptom whose cause resource isn't
   known yet (e.g. "something evicted several pods around 14:00").
5. **`k8s_check_service_connectivity`** — use specifically when a Service
   is suspected of routing incorrectly (zero endpoints, wrong selector) —
   confirms the symptom (unreachable) before you go looking for the
   config mismatch that causes it.
6. **`k8s_get_available_api_resources`** — use when investigating a
   Custom Resource and you are not sure of its exact kind/API group/version.
7. **`k8s_get_cluster_configuration`** — use for cluster-level context
   (versions, general config) when a symptom might be an infrastructure
   compatibility issue rather than a workload misconfiguration.
8. **`k8s_get_pod_logs`** — FALLBACK ONLY. If the Loki investigator is
   available, prefer it for anything beyond "show me the last N lines of
   this one pod" — it supports structured querying across many pods and
   time ranges that this tool cannot. Use this tool directly only when no
   observability backend is available, or when a pod is too new/short-lived
   for its logs to have reached Loki yet.

## What NOT to use (excluded by design, not by oversight)

`k8s_patch_resource`, `k8s_annotate_resource`, `k8s_remove_annotation`,
`k8s_label_resource`, `k8s_remove_label`, `k8s_create_resource`,
`k8s_create_resource_from_url`, `k8s_delete_resource`, `k8s_apply_manifest`
— all mutate cluster state. `k8s_execute_command` is also excluded: even
though it does not directly mutate a Kubernetes object, it runs an
arbitrary command inside a live container, which can write files, kill
processes, or otherwise change runtime state — out of scope for a
read-only investigator under the least-privilege principle. If an
investigation seems to require exec-ing into a container, report that as a
recommended next step for a human, not something to do here.

## Failure-mode taxonomy

| Mode | Pivotal signal | Investigate |
|---|---|---|
| **CrashLoopBackOff** | `describe` shows `Back-off restarting failed container`, restart count > 0 | Check `last_terminated_reason`-equivalent (`State.Last State.Reason` in describe/YAML): `OOMKilled` → hand off to Prometheus investigator for memory trend; `Error` → check logs; `ContainerCannotRun` → image/exec path below |
| **ImagePullBackOff / ErrImagePull** | Events show `Failed to pull image`, `manifest not found`, `429 Too Many Requests`, or `unauthorized` | Wrong tag/typo (manifest not found) vs. missing `imagePullSecrets` (unauthorized) vs. registry rate limit (429) — the exact event message distinguishes these; do not guess |
| **Pending — unschedulable** | Pod stuck in `Pending`, events show `FailedScheduling` | Read the event message fully — it names the exact constraint (insufficient CPU/memory, node affinity/taint mismatch, no nodes match selector) |
| **Pending — image/init stuck** | `Pending` but scheduled to a node, no `FailedScheduling` | Check init container status separately — a stuck or crashing init container blocks the main container without a scheduling-layer event |
| **Evicted** | Pod `status.reason == "Evicted"`, event reason `Evicted` | Check the node's condition (disk/memory pressure) via `k8s_describe_resource` on the Node — an evicted pod is a symptom of node-level resource exhaustion, not a pod-level bug |
| **Config conflict** | Log/error message mentions "conflict", "duplicate", "already exists", "only one ... can be", "invalid config" | Search cluster-wide by label/selector for a SECOND resource contributing to the same logical config (see Guidelines above) — this is the highest-value, most commonly missed investigation step |
| **Service unreachable, pod healthy** | Pod is `Running`/`Ready`, but callers get timeout/connection refused | Compare `Service.spec.selector` (via `k8s_get_resource_yaml`) against the Pod's actual `labels` — a mismatch produces zero endpoints silently; confirm with `k8s_check_service_connectivity` |
| **PVC Pending** | PVC stuck `Pending`, event mentions storage class | Check `k8s_get_resources` for available StorageClasses and compare the exact name requested — a typo or removed StorageClass is the most common cause |
| **RBAC / Forbidden** | Application logs show `is forbidden`, `cannot get/list/watch resource` | Identify the ServiceAccount the pod runs as (from `k8s_get_resource_yaml` on the Pod/Deployment) — the fix is a Role/RoleBinding, not something this read-only agent can apply, but naming the exact missing verb+resource is the useful diagnostic output |
| **Stuck rollout** | New ReplicaSet pods not becoming Ready; old pods still serving | `k8s_get_events` on the new ReplicaSet for `FailedCreate` (admission rejection) or `FailedScheduling` (no node fits); compare `Deployment.status.availableReplicas` to `.replicas` |

## Investigation flow

An investigation is not a fixed checklist — compress or skip steps once you
have enough evidence. Terminate as soon as you can state a hypothesis at a
known confidence; do not keep querying past the point of diminishing
returns.

### 1. Orient
Resolve the target resource: kind, name, namespace. If the symptom is
vague ("something is broken in namespace X"), start with `k8s_get_events`
on that namespace rather than guessing a specific resource.

### 2. Characterize
`k8s_describe_resource` on the target — this single call usually gives
phase/status, restart count, and recent events together.

### 3. Classify
Match against the failure-mode taxonomy above. If the message contains
any of the conflict/duplicate keywords, treat the cross-resource search as
mandatory, not optional — this is not a "nice to have" step.

### 4. Corroborate
- If a second resource is implicated (config conflict, selector mismatch):
  `k8s_get_resource_yaml` on both candidates, diff the specific field.
- If timing matters (what changed recently): `k8s_get_events`, sorted by
  time, cluster- or namespace-wide, not just the one resource's tail.
- If a metric would settle the question (memory trend, CPU throttling):
  name the specific metric and hand off to the Prometheus investigator
  rather than guessing from resource state alone.
- If log content would settle the question beyond what
  `describe`/events show: hand off to the Loki investigator (or use
  `k8s_get_pod_logs` as the documented fallback).

### 5. Synthesize and stop
State the hypothesis once evidence supports it at a known confidence.
Two resources both plausibly contributing is a valid output — say so
rather than forcing a single cause.

## Output format

Return a structured evidence item:
```json
{
  "type": "k8s_resource",
  "resource_checked": "<kind>/<name> in namespace <ns> (or 'cluster-wide' for a label search)",
  "query_performed": "<which tool, with which exact selector/name/namespace>",
  "finding": "<verbatim or near-verbatim relevant field/event, not a paraphrase>",
  "interpretation": "<one or two sentences, strictly grounded in the finding above>",
  "cross_resource_check_performed": true | false
}
```
If a cross-resource conflict pattern was matched (keywords above) but you
have NOT yet performed the cluster-wide search, do not finalize — that
search is a required step, not optional corroboration.

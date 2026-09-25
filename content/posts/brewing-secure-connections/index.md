---
title: "Brewing Secure Connections: A TLS Termination Benchmark"
description: "A deep-dive benchmark comparing TLS termination, passthrough, and pod/VM topologies across .NET, Java, Go, and Rust under heavy fresh-connection churn."
summary: "Measuring the latency, saturation, and runtime trade-offs of TLS termination points across Direct VMs, Kubernetes Pods, Traefik Passthrough, and Traefik Termination."
date: 2026-09-25T08:00:00+02:00
lastmod: 2026-09-25T08:00:00+02:00
featureimage: "brewing-secure-connections.svg"
categories:
  - "Performance Engineering"
  - "Cloud Infrastructure"
tags:
  - "tls"
  - "kubernetes"
  - "traefik"
  - "gateway-api"
  - "benchmarking"
  - "networking"
  - "golang"
  - "dotnet"
  - "java"
  - "rust"
showReadingTime: true
showWordCount: false
showTableOfContents: true
---

Freshly brewed HTTPS connections across VMs, Kubernetes, and Traefik's Gateway API implementation.

## Motivation

In my last project, we migrated a gateway/proxy to a self-hosted Kubernetes
cluster. Later, while preparing for an exam that covered the Kubernetes Gateway API, I got
interested in TLS passthrough. One question kept bugging me: what does moving
TLS termination cost, and when does connection setup start limiting the work
we can complete?

A platform team, cloud provider, or regulatory requirement such as FIPS or
eIDAS may constrain where and how TLS terminates. Serverless platforms can limit
control over ingress and databases or message brokers often handle TLS in the
process itself. Kubernetes gives us several places to terminate a connection,
so I wanted to measure the trade-offs instead of guessing.

This experiment measures the cost of equivalent paths on the same Azure cloud network
and identifies where the connection-establishment path starts to limit completed work.

## Experiment Design

Since I tried to keep the experiment in a reasonable budget and quota (<150$), therefore
the environment is somewhat recycled (deallocated) between scenarios. But technically, it would
be possible to run multiple scenarios at the same time via spining up another instance of the infra stack.

Speaking of infra, there isn't too much abstraction or vendor lock-in in terms of resource types. Besides VMSS, LBs and a VNet
you wont find too much interfering with the HTTP requets. The K8S cluster is self-hosted, keeping it bare-minimum:
CNI (Cilium), CoreDNS and the default kube-system pods. In the repository (links in the end), there are other infra related
charts for observability, such as `kube-prom-stack` or `pyroscope`, but these are deployed only ad-hoc for debugging and are not
part of the tested scenarios. Oh, and there is a small jumpbox container, also provisioned ad-hoc, its main purpose is basically
to act as a ssh bridge into the VMSS, since the VMSS dont have a public IP (except the K8s Control Plane).

The experiment compares equivalent paths and looking at their throughput, saturation,
and trade-offs. Traefik (Ingress) and Cilium (CNI) were purely personal choices,
Traefik was choosen because I already run it in my homelab and Cilium was choosen because of
its mature eBPF and observability implementation. k6, well, let me just say I didn't want to generate JMX configs neither.
For the analysis, I couldn't really choose between Postgre(TimescaleDb) or InfluxDB,
therefore I didn't, at least not in the begining. So I ended up outputing to JSONL on the client
and pushing it to the ACR (OCI), because who needs an additional blob store, if you already
have one backing your ACR, duh. In the end, I did spin up an InfluxDB (v2) on local dev
and reimported the raw timeseries from the ACR, mainly because I was already familiar with Postgre,
so I took Influx for a ride just to verify all the hearsay about its controversial query language <insert hide the pain harold>.

### Overview

{{< figure src="brewing-secure-connections.svg" alt="Tested TLS termination paths across VMs, Kubernetes, and Traefik" >}}

Every test starts from the same `client-loadgen-vmss` and follows one of these
paths:

- **Direct VM:** TLS terminates in the workload's container on a dedicated VM, managed by [systemd and Podman](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_atomic_host/7/html/managing_containers/running_containers_as_systemd_services_with_podman).
- **Kubernetes pod:** TLS terminates in the application pod, routed through service [(Azure) Load Balancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer).
- **Traefik passthrough:** with Gateway API
  [TLS passthrough](https://gateway-api.sigs.k8s.io/guides/tls/#downstream-tls),
  Traefik forwards the original encrypted connection. TLS terminates at the
  application pod.
- **Traefik termination:** with Gateway API
  [TLS termination](https://gateway-api.sigs.k8s.io/guides/tls/#downstream-tls),
  TLS terminates at Traefik, which opens a separate validated TLS connection
  to the backend pod.

Traefik ran as a fixed three-replica deployment on its own `Standard_D4s_v7`
worker. Each replica requested 1.2 vCPU and 1.5 GiB of memory, so the three pods
reserved 3.6 of the node's four vCPUs while leaving some room for Kubernetes
and node networking. Three replicas also meant three separate proxy processes,
pod IPs, network namespaces, and ephemeral-port spaces. For a fresh-connection
benchmark, this spreads proxy processing and backend connection tuples across
three processes and three source-port spaces. The benchmark therefore compares
the complete deployed paths. The Kubernetes Service distributes connections
across the replicas, with some natural imbalance between pods.

For Traefik termination, the Gateway API
[`BackendTLSPolicy`](https://gateway-api.sigs.k8s.io/guides/tls/#upstream-tls)
configures backend certificate validation and SNI. This is encrypted upstream
TLS, **not mTLS**: the pod does not authenticate Traefik with a client certificate.

The Gateway API also defines an
[upstream client-certificate setting, `Gateway.spec.tls.backend.clientCertificateRef`](https://gateway-api.sigs.k8s.io/guides/tls/#gateways-client-certificate-configuration)
for mTLS. At the time of writing, Traefik **v3.7.13**
[did not support `GatewayBackendClientCertificate`](https://github.com/traefik/traefik/blob/v3.7.13/integration/gateway-api-conformance-reports/v1.6.1/experimental-v3.7-default-report.yaml#L57-L62).
I found out late, after getting rather attached to testing Traefik under
load. That's on me, but the results here measure validated upstream TLS, not
upstream mTLS. But I'd like to compare Gateway implementations such as Cilium,
Envoy, and Istio in a follow-up too.

### SKU, Configurations & Versions

{{< accordion mode="open" separated=true >}}
  {{< accordionItem title="Infrastructure" icon="server" open=false >}}

| Resource | Value |
| --- | --- |
| Azure region | West US 2 |
| Network path | Azure `Standard` Load Balancer |
| VM Client SKU | `Standard_D8s_v7` Xeon® 6 6973PC 8c 16G max 25,000Mbps |
| VM Server SKU | `Standard_D4s_v7` Xeon® 6 6973PC 4c 32G max 25,000Mbps |
| VM OS | Ubuntu `26.04` LTS - Linux Kernel `7.0.0-1012-azure` |

  {{< /accordionItem >}}

  {{< accordionItem title="Connection" icon="network" >}}

| Setting | Value |
| --- | --- |
| TLS version | TLS `1.2`; TLS `1.3` |
| Server certificate | ECDSA `P-256` certificate |
| Key-exchange groups | `P-256`; `X25519` |
| TLS 1.2 cipher suite | `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256` |
| TLS 1.3 cipher suite | `TLS_AES_128_GCM_SHA256` |
| TLS 1.2 handshake | 2-RTT |
| TLS 1.3 handshake | 1-RTT |

  {{< /accordionItem >}}

  {{< accordionItem title="Stack" icon="code" >}}

| App | Version |
| --- | --- |
| Kubernetes | `1.36.4` |
| Traefik | `3.7.13` |
| Cilium | `1.20.1` |
| Gateway API | `1.6.1` |
| K6 client | `2.2.0`; golang `1.26.5` |
| .NET workload | `10.0.12` based on `26.04.1 resolute-raccoon`; Kestrel; OpenSSL `3.5.5` |
| Java workload | Eclipse Temurin `25.0.4+7` based on `26.04.1 resolute-raccoon`; Netty; BoringSSL `2.0.84` |
| golang workload | `1.27.1` based on `distroless/debian trixie`; crypto/tls `1.27.1` |
| Rust workload | `1.98.1` based on `debian trixie`; rustls `0.23.45` |

*I chose the base images deliberately. One specific C library is part of that story,
but it deserves its own post too.*

  {{< /accordionItem >}}
{{< /accordion >}}

## Workloads and Metrics

Every request opens a new TCP connection, completes a full TLS handshake with
certificate validation, exchanges one HTTP request and a 1,024-byte response,
and then closes the connection. This deliberately removes connection pooling
and TLS resumption from the measurement.

Why do it this way you ask? Because we're measuring connection setup cost, not
just how fast the app responds. Keep-alive traffic skips most of that work.Production systems usually see both patterns, bursts of new connections after scale-outs or restarts, and quieter
periods with long-lived connections. This benchmark deliberately stresses the
fresh-connections, a worst case scenario under heavy connection churn.

I defined three load-pattern families across all four workloads. A variaton of steady request rates, a linear-pressure ramp, and a fixed-work race. Each profile
runs three times without a separate warm-up. I skipped warm-up, not **only**
because I want to see Java suffer, but because cold starts do exist
and different workloads might perform differently during ramp-up. I tested TLS 1.2 and TLS 1.3, with P-256 and X25519 key exchange in separate runs.

### Baseline load shapes

`blog-tls-base.yaml` uses arrival-rate executors to apply three load shapes:

| Profile | Scheduled load | Duration | Purpose |
| --- | --- | --- | --- |
| Linear | 100 to 3,000 requests/s | 60 seconds | Show how throughput, latency, and dropped work change as offered load rises |
| Flat | 1,000 requests/s | 60 seconds | Show the same metrics but at a constant offered rate |
| Sine | Nominally 0 to 2,500 requests/s over two cycles | 60 seconds | Test the response to repeated rises and falls in offered load |

The profiles begin with 100 pre-allocated virtual users and may use up to
1,000. k6 schedules iterations at the configured rate independently of how
quickly previous iterations finish.

{{< figure src="charts/workload/baseline-workload-timeline-filled.svg" alt="Baseline workload profiles showing three linear, flat, and sine request-rate runs" >}}

{{< figure src="charts/workload/baseline-vu-workload-timeline-filled.svg" alt="Active virtual users for three linear, flat, and sine baseline runs" >}}

### High-pressure linear ramp

`blog-tls-linear-peak.yaml` schedules the same linear increase from 100 to
6,000 requests/s twice over 90 seconds. Both profiles start with 100
pre-allocated VUs, but one is capped at 100 VUs while the other may expand to
1,000.

The two profiles apply the same open-model arrival-rate schedule with different
VU limits. The first keeps the client pool fixed at 100 VUs. The second starts
with 100 pre-allocated VUs and allows k6 to create up to 1,000 when more
concurrent workers are needed to follow the schedule. The two profiles measure
how each server performs as client load increases. If k6 runs out of VUs and
drops iterations, the server sees less than the scheduled load, so I check
the achieved rate before comparing latency.

{{< figure src="charts/workload/linear-workload-timeline-filled.svg" alt="Achieved request rate for fixed and expandable high-pressure linear-ramp profiles" >}}

{{< figure src="charts/workload/linear-vu-workload-timeline-filled.svg" alt="Active virtual users for fixed and expandable high-pressure linear-ramp profiles" >}}

### Fixed-work race

`blog-tls-race.yaml` uses k6's shared-iterations executor. It distributes
100,000 total iterations across fixed pools of 10, 100, and 1,000 VUs. Each
profile has a 90-second safety limit. The VUs work through a shared pool,
with each VU starting its next request as soon as the previous one finishes.

{{< figure src="charts/workload/race-workload-timeline-filled.svg" alt="Achieved request rate for fixed-work race profiles using 10, 100, and 1,000 virtual users" >}}

{{< figure src="charts/workload/race-vu-workload-timeline-filled.svg" alt="Allocated virtual users for fixed-work race profiles using 10, 100, and 1,000 virtual users" >}}

## Results at a glance

Here's how moving the TLS termination point played out in these runs:

- **At the 1,000 req/s baseline,** direct VM HTTP P50 sat around `0.20–0.30 ms` across stacks. Routing through a Kubernetes Service or Traefik added sub-millisecond HTTP time.
- **On the 6,000 req/s ramp,** the topology and stack curves spread out as offered load rose. I look at client VU limits alongside those latency curves.
- **In the 100,000-request race,** direct VM led most stacks at 10 VUs, while Traefik termination finished in a tight `21.7–23.5 s` band at 100 VUs. The concurrency level changes the comparison.

> **Reading the charts:** HTTP and TLS percentiles pool underlying samples
> from three Profile Runs. TLS timings exclude zero non-handshake observations.
> Percentile axes stop at `1,000 ms`, with exact overflow values in the columns.
> Race charts keep the three completion times separate.

### TLS 1.2 with P-256

I use TLS 1.2 / P-256 as the starting point for the four termination paths.
Separate charts show HTTP response time and the client-facing TLS handshake
so we can compare each measurement on its own.

{{< accordion mode="open" separated=true >}}

  {{< accordionItem title="Linear baseline: 100 to 3,000 requests/s" icon="chart-line" >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-http-baseline-linear.svg" alt="Pooled HTTP P50, P90, and P99 across topologies and stacks for the 100 to 3,000 requests per second baseline with TLS 1.2 and P-256" >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-tls-baseline-linear.svg" alt="Pooled client-facing TLS handshake P50, P90, and P99 across topologies and stacks for the linear baseline with TLS 1.2 and P-256" >}}
  {{< /accordionItem >}}

  {{< accordionItem title="Flat baseline: 1,000 requests/s" icon="chart-line" >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-http-baseline-flat.svg" alt="Pooled HTTP P50, P90, and P99 across topologies and stacks at a steady 1,000 requests per second with TLS 1.2 and P-256" >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-tls-baseline-flat.svg" alt="Pooled client-facing TLS handshake P50, P90, and P99 across topologies and stacks at a steady 1,000 requests per second with TLS 1.2 and P-256" >}}
  {{< /accordionItem >}}

  {{< accordionItem title="Sine baseline: 0 to 2,500 requests/s, two cycles" icon="chart-line" open=true >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-http-baseline-sine.svg" alt="Pooled HTTP P50, P90, and P99 across topologies and stacks for the sine baseline using TLS 1.2 and P-256" >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-tls-baseline-sine.svg" alt="Pooled client-facing TLS handshake P50, P90, and P99 across topologies and stacks for the sine baseline using TLS 1.2 and P-256" >}}
  {{< /accordionItem >}}

{{< /accordion >}}

Given the **direct VM** as our starting point, it's the shortest path
through the Azure Load Balancer to the container. On that path, **HTTP response time**
sat around `0.20–0.30 ms` and the **TLS handshake** around `0.80–1.30 ms` at
P50 depending on the stack. Java/Netty with its `BoringSSL` usually clocked
the lowest HTTP response times, while `rustls` delivered the fastest handshakes.

As we introduce Kubernetes and ingress routing, we can notice an added latency cost to our baselines:

- **Kubernetes Pod (Service LB):** Compared with the Direct VM, the path through the Kubernetes Service to the pod came out about **`+0.06 ms`** HTTP (`+25%`) and **`+0.80 ms`** TLS (`+80%`) at P50.
- **Traefik Passthrough:** With Traefik forwarding the encrypted TCP connection to the pod, I measured about **`+0.20 ms`** HTTP (`+85%`) and **`+1.30 ms`** for the TLS handshake (`+137%`) compared with the Direct VM. That's more than double the direct VM handshake time.
- **Traefik Termination:** Here Traefik terminates client TLS and opens another TLS connection to the pod. Compared with the Direct VM, HTTP response time rose by ~0.42 ms (`+170%`) and the client-facing TLS handshake by **`0.80 ms`** (`+90%`). That sounds like a massive HTTP increase, but the direct VM started around `0.25 ms`. The client-facing handshake sat around `1.80 ms` across backends, since the client shakes hands with the same Traefik configuration.

The Flat, Linear, and Sine baselines had similar patterns. TLS P99 stayed
around `7–11 ms` for almost every row. The sole outlier is the **.NET passthrough tail**, which should get its own dedicated breakdown.

At 3,000 req/s on our 4-core CPU, all topologies stayed mostly stable. Doubling that to 6,000 req/s is where CPU saturation kicks in, and the architectural differences, and especially the TLS library implementations between the stacks really start to show:

{{< accordion mode="open" separated=true >}}

  {{< accordionItem title="Fixed ramp: up to 100 VUs" icon="chart-line" >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-http-linear-fixed-100-vus.svg" alt="Pooled HTTP percentile ranges across topologies and stacks for the 100-VU-capped 6,000 requests per second ramp with TLS 1.2 and P-256" >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-tls-linear-fixed-100-vus.svg" alt="Pooled client-facing TLS handshake percentile ranges across topologies and stacks for the 100-VU-capped ramp with TLS 1.2 and P-256" >}}
  {{< /accordionItem >}}

  {{< accordionItem title="Expandable ramp: up to 1,000 VUs" icon="chart-line" open=true >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-http-linear-expandable-1000-vus.svg" alt="Pooled HTTP percentile ranges across topologies and stacks for the expandable 6,000 requests per second ramp with TLS 1.2 and P-256" >}}
{{< figure src="charts/scenario-summary-tls12-p-256/scenario-percentiles-tls-linear-expandable-1000-vus.svg" alt="Pooled client-facing TLS handshake percentile ranges across topologies and stacks for the expandable ramp with TLS 1.2 and P-256" >}}
  {{< /accordionItem >}}

{{< /accordion >}}

### 100K requests with different concurrency

In the fixed-work race, each VU starts another request as soon as its previous
one finishes. At 10 VUs, the Direct VM generally gets through all 100,000
requests first. At 100 VUs, Traefik termination takes the lead.

#### 10 VUs: Short paths lead

With `10 VUs`, each VU works through its requests sequentially while the ten
VUs run concurrently. The short direct path led in this profile:

{{< figure src="charts/scenario-summary-tls12-p-256/scenario-completion-race-10-vus.svg" alt="Three completion times per topology and stack for the 100,000-request race with 10 VUs using TLS 1.2 and P-256" >}}

- **Direct VM:** Took roughly **`33–48 s`** across stacks.
- **Kubernetes Pod:** Took **`45–52 s`**, roughly **`5–11 s`** longer than the matched Direct VM runs.
- **Traefik Passthrough:** Most stacks landed at **`48–52 s`**. .NET took **`65–73 s`**.
- **Traefik Termination:** The four backends clustered in a narrow **`48–51 s`** band.

For golang, the Kubernetes Pod (s4) path took **`45.3–45.4 s`**, while Traefik
termination (s6) took **`48.1–49.0 s`**. In s4, the golang application pod handles
the client-facing TLS handshake, meanwhile in s6, the handshake ends at Traefik (proxy
written in golang). The complete request cycle in the
first run, from TCP setup through TLS and the HTTP response, averaged
**`4.52 ms`** on the Pod path and **`4.89 ms`** through Traefik.
The extra **`0.37 ms`** repeats roughly 10,000 times per VU, adding about
**`3.7 s`** to the first run's completion time. Keep that small per-request
cost in mind for the 100- and 1,000-VU races, where the same paths handle
many more connections at once.

> **Disclaimer:** Traefik is deployed with three replicas, which provide three separate source-port
> ranges for connections to the upstream pod. With just one replica, every
> new backend connection would draw from the one finite range. If too many
> ports would be still occupied by active or recently closed connections, new
> upstream dials could fail even while the application pod was healthy.

#### 100 VUs: The ordering changes

At `100 VUs`, Traefik termination recorded the shortest completion times
across all four stacks:

{{< figure src="charts/scenario-summary-tls12-p-256/scenario-completion-race-100-vus.svg" alt="Three completion times per topology and stack for the 100,000-request race with 100 VUs using TLS 1.2 and P-256" >}}

- **Direct VM:** Most stacks needed **`31–38 s`**.
- **Kubernetes Pod:** Took **`24–30 s`**.
- **Traefik Termination:** All four stacks fell within **`21.7–23.5 s`**.

At 100 VUs, the golang comparison already flips. The Kubernetes Pod (s4)
path took **`28.8–29.3 s`** across three runs, compared with
**`21.9–23.5 s`** behind Traefik termination (s6). In the raw k6 samples,
TCP connecting averaged **`23.9–24.2 ms`** per request on the Pod path
and **`12.8–13.8 ms`** through Traefik. About **`2.0–2.1%`** of Pod-path
connects took over `500 ms`, compared with **`1.0–1.1%`** through Traefik.

HTTP and client-facing TLS were quicker on the Pod path, but more time went
into opening the client TCP connections. It's one golang pod accepting those
connections versus, well, three smaller Traefik replicas on a
separate worker.

A cleaner one-to-one test would mean picking a VU level where a single Traefik
replica stays safely below its backend source-port limit, then comparing
it directly against the golang Pod path before bringing the three-replica
numbers back in.

#### 1,000 VUs: Heavy connection churn

At `1,000 VUs`, the same 100,000 fresh connections are spread across a
larger concurrent client pool:

{{< figure src="charts/scenario-summary-tls12-p-256/scenario-completion-race-1000-vus.svg" alt="Three completion times per topology and stack for the 100,000-request race with 1,000 VUs using TLS 1.2 and P-256" >}}

- **Direct VM:** Times ranged from **`21–32 s`** across stacks.
- **Kubernetes Pod:** The four stacks stayed within **`23–28 s`**.
- **Traefik Passthrough:** golang, Rust, and Java took **`21–26 s`**. The .NET outlier remained at **`48–54 s`**.
- **Traefik Termination:** Almost all runs took **`20–22 s`**.

### Does changing the TLS configuration change the story?

#### Steady baseline (1,000 req/s)

At steady load, the measurements showed consistently across all four topologies:

- **TLS 1.2 (P-256 vs X25519):** Swapping the curve changed handshake P50 by less than **`0.05 ms`** in every topology and stack pair. Direct VM stayed at **`0.85–1.26 ms`**, Kubernetes Pods at **`1.65–2.05 ms`**, and Traefik hovered at **`~1.80 ms`** on termination and **`2.16–2.81 ms`** on passthrough.
- **TLS 1.3 with P-256:** Handshake P50 increased compared to TLS 1.2 / P-256 across all topologies, adding roughly **`0.08–0.48 ms`**. Direct VM rose to **`0.97–1.39 ms`**, Kubernetes Pods to **`2.08–2.45 ms`**, and Traefik termination to **`~2.16 ms`**.
- **TLS 1.3 with X25519:** Handshake P50 dropped cleanly compared to TLS 1.2 / P-256 in every single pair. It fell by about **`43–50%`** on Kubernetes Pods (**`0.82–1.12 ms`**), Traefik passthrough (**`1.18–1.57 ms`**), and Traefik termination (**`0.97–0.98 ms`**). The Direct VM reduction was smaller, about **`7–14%`**, landing at **`0.77–1.08 ms`**.

**An extra round trip for TLS 1.3 / P-256?**

The P-256 slowdown almost certainly comes down to an "unexpected" TLS
[`HelloRetryRequest`](https://www.rfc-editor.org/rfc/rfc8446.html#section-2.1).
k6 v2.2.0 relies on golang's
[default curve preferences](https://github.com/golang/go/blob/go1.26.5/src/crypto/tls/defaults.go#L19-L35),
which prioritize `X25519MLKEM768`. In the
[initial key-share selection](https://github.com/golang/go/blob/go1.26.5/src/crypto/tls/handshake_client.go#L144-L159),
the client sends that hybrid share alongside an `X25519` fallback share. But P-256 is
only advertised in the supported groups extension. Its share only comes later if the server explicitly requests it.

That triggers a round-trip exchange, the server responds with a `HelloRetryRequest`, then the client generates the P-256 share, and only then does the handshake finish. That extra round trip wipes out the 1-RTT advantage of TLS 1.3 over TLS 1.2, whereas X25519 finishes in one. In TLS 1.2, both curves take two round trips anyway because the server chooses the group before the client sends its key exchange.

The transferred wire bytes line up with this cleanly. Rust on Direct VM received 126 more bytes per request with `P-256` than with `X25519`, matching a 93-byte `HelloRetryRequest` plus a 33-byte larger server key share. The exact same byte delta showed up across all four stacks.

{{< accordion mode="open" separated=true >}}
  {{< accordionItem title="Steady baseline: TLS handshakes by topology" icon="chart-line" open=true >}}
{{< figure src="charts/tls-comparison/tls-percentiles-tls-baseline-flat-s2-vm.svg" alt="Client-facing TLS handshake percentiles at 1,000 requests per second for Direct VM and four application stacks, comparing TLS 1.2 and 1.3 with P-256 and X25519" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-tls-baseline-flat-s4-k8s-pod.svg" alt="Client-facing TLS handshake percentiles at 1,000 requests per second for Kubernetes Pod and four application stacks, comparing TLS 1.2 and 1.3 with P-256 and X25519" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-tls-baseline-flat-s5-traefik-passthrough.svg" alt="Client-facing TLS handshake percentiles at 1,000 requests per second for Traefik Passthrough and four application stacks, comparing TLS 1.2 and 1.3 with P-256 and X25519" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-tls-baseline-flat-s6-traefik-terminate.svg" alt="Client-facing TLS handshake percentiles at 1,000 requests per second for Traefik Termination and four application stacks, comparing TLS 1.2 and 1.3 with P-256 and X25519" >}}
  {{< /accordionItem >}}
  {{< accordionItem title="Steady baseline: HTTP response times by topology" icon="chart-line" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-http-baseline-flat-s2-vm.svg" alt="HTTP response-time percentiles at a steady 1,000 requests per second for Direct VM and four application stacks, comparing TLS 1.2 and 1.3 with P-256 and X25519" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-http-baseline-flat-s4-k8s-pod.svg" alt="HTTP response-time percentiles at a steady 1,000 requests per second for Kubernetes Pod and four application stacks, comparing TLS 1.2 and 1.3 with P-256 and X25519" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-http-baseline-flat-s5-traefik-passthrough.svg" alt="HTTP response-time percentiles at a steady 1,000 requests per second for Traefik Passthrough and four application stacks, comparing TLS 1.2 and 1.3 with P-256 and X25519" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-http-baseline-flat-s6-traefik-terminate.svg" alt="HTTP response-time percentiles at a steady 1,000 requests per second for Traefik Termination and four application stacks, comparing TLS 1.2 and 1.3 with P-256 and X25519" >}}
  {{< /accordionItem >}}
  {{< accordionItem title="Steady baseline: client bytes sent per request by topology" icon="chart-line" >}}
{{< figure src="charts/tls-comparison/tls-bytes-sent-baseline-flat-s2-vm.svg" alt="Client bytes sent per request at 1,000 requests per second for Direct VM across four stacks and four TLS configurations, using total bytes divided by recorded requests across three Profile Runs" >}}
{{< figure src="charts/tls-comparison/tls-bytes-sent-baseline-flat-s4-k8s-pod.svg" alt="Client bytes sent per request at 1,000 requests per second for Kubernetes Pod across four stacks and four TLS configurations, using total bytes divided by recorded requests across three Profile Runs" >}}
{{< figure src="charts/tls-comparison/tls-bytes-sent-baseline-flat-s5-traefik-passthrough.svg" alt="Client bytes sent per request at 1,000 requests per second for Traefik Passthrough across four stacks and four TLS configurations, using total bytes divided by recorded requests across three Profile Runs" >}}
{{< figure src="charts/tls-comparison/tls-bytes-sent-baseline-flat-s6-traefik-terminate.svg" alt="Client bytes sent per request at 1,000 requests per second for Traefik Termination across four stacks and four TLS configurations, using total bytes divided by recorded requests across three Profile Runs" >}}
  {{< /accordionItem >}}
  {{< accordionItem title="Steady baseline: client bytes received per request by topology" icon="chart-line" >}}
{{< figure src="charts/tls-comparison/tls-bytes-received-baseline-flat-s2-vm.svg" alt="Client bytes received per request at 1,000 requests per second for Direct VM across four stacks and four TLS configurations, using total bytes divided by recorded requests across three Profile Runs" >}}
{{< figure src="charts/tls-comparison/tls-bytes-received-baseline-flat-s4-k8s-pod.svg" alt="Client bytes received per request at 1,000 requests per second for Kubernetes Pod across four stacks and four TLS configurations, using total bytes divided by recorded requests across three Profile Runs" >}}
{{< figure src="charts/tls-comparison/tls-bytes-received-baseline-flat-s5-traefik-passthrough.svg" alt="Client bytes received per request at 1,000 requests per second for Traefik Passthrough across four stacks and four TLS configurations, using total bytes divided by recorded requests across three Profile Runs" >}}
{{< figure src="charts/tls-comparison/tls-bytes-received-baseline-flat-s6-traefik-terminate.svg" alt="Client bytes received per request at 1,000 requests per second for Traefik Termination across four stacks and four TLS configurations, using total bytes divided by recorded requests across three Profile Runs" >}}
  {{< /accordionItem >}}
{{< /accordion >}}

#### Linear ramp (up to 6,000 req/s)

Given the extra round trip baked into TLS 1.3 with P-256, the cleanest
comparison is TLS 1.2 directly against TLS 1.3 using **X25519**
across each topology and stack. But no wories, the charts still show 
all four configurations for the pesky readers
who want to inspect the P-256 results as well.

- **Handshake P50 was generally quicker:** Most matched pairs saw a modest **`5–10%`** drop. golang on Direct VM was a minor exception (**`1.06 → 1.09 ms`**), while .NET passthrough saw a larger bump (**`6.04 → 7.85 ms`**).
- **Handshake P90 showed no clean winner:** I expected P90 to follow P50 down, but it dropped in eight pairs and climbed in the other eight. The median absolute swing was about **`10%`**, with no uniform direction. For instance, the .NET Pod behind the Azure LB improved from **`58.01 → 43.72 ms`**, while golang on Direct VM drifted up from **`6.97 → 8.04 ms`**.
- **HTTP response-time P90 rose in all 16 pairs:** The median increase across pairs was **`63%`**, ranging from 28% up to 125%. Since [k6's HTTP duration](https://grafana.com/docs/k6/latest/using-k6/metrics/reference/#http-specific-built-in-metrics) only starts ticking once the client wraps up its handshake, any trailing server-side TLS work or socket queueing bleeds straight into the HTTP timer. That wasn't just a ramp effect either. Even at the steady 1,000 req/s baseline, HTTP P90 rose across all 16 pairs by a median of **`24%`**.

{{< accordion mode="open" separated=true >}}
  {{< accordionItem title="Linear ramp: TLS handshakes by topology" icon="chart-line" open=true >}}
{{< figure src="charts/tls-comparison/tls-percentiles-tls-linear-expandable-1000-vus-s2-vm.svg" alt="Pooled client-facing TLS handshake percentiles for the expandable 6,000 requests per second ramp on Direct VM across four stacks and four TLS configurations" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-tls-linear-expandable-1000-vus-s4-k8s-pod.svg" alt="Pooled client-facing TLS handshake percentiles for the expandable 6,000 requests per second ramp on Kubernetes Pod across four stacks and four TLS configurations" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-tls-linear-expandable-1000-vus-s5-traefik-passthrough.svg" alt="Pooled client-facing TLS handshake percentiles for the expandable 6,000 requests per second ramp through Traefik Passthrough across four stacks and four TLS configurations" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-tls-linear-expandable-1000-vus-s6-traefik-terminate.svg" alt="Pooled client-facing TLS handshake percentiles for the expandable 6,000 requests per second ramp with Traefik Termination across four stacks and four TLS configurations" >}}
  {{< /accordionItem >}}
  {{< accordionItem title="Linear ramp: HTTP response times by topology" icon="chart-line" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-http-linear-expandable-1000-vus-s2-vm.svg" alt="Pooled HTTP response-time percentiles for the expandable 6,000 requests per second ramp on Direct VM across four stacks and four TLS configurations" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-http-linear-expandable-1000-vus-s4-k8s-pod.svg" alt="Pooled HTTP response-time percentiles for the expandable 6,000 requests per second ramp on Kubernetes Pod across four stacks and four TLS configurations" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-http-linear-expandable-1000-vus-s5-traefik-passthrough.svg" alt="Pooled HTTP response-time percentiles for the expandable 6,000 requests per second ramp through Traefik Passthrough across four stacks and four TLS configurations" >}}
{{< figure src="charts/tls-comparison/tls-percentiles-http-linear-expandable-1000-vus-s6-traefik-terminate.svg" alt="Pooled HTTP response-time percentiles for the expandable 6,000 requests per second ramp with Traefik Termination across four stacks and four TLS configurations" >}}
  {{< /accordionItem >}}
{{< /accordion >}}

#### 100K Race (10, 100 and 1,000 VUs)

As in the ramp, only TLS 1.2 vs  TLS 1.3 will be compared with **X25519** key group
The race still pushes the same 100,000 fresh requests at each VU level.
All three runs are plotted for every configuration, while the percentiles combine the underlying samples

- **At 10 VUs, passthrough showed the clearest TLS 1.3 completion-time win.** Even the slowest TLS 1.3 run beat the fastest TLS 1.2 run for each of the four stacks. Java finished in **`37.1–45.1 s`** with TLS 1.3 versus **`49.1–50.5 s`** with TLS 1.2. Pods and Traefik termination also saw a substantial handshake-tail improvement: TLS P99 fell by **`50–61%`** across their eight topology and stack pairs.
- **At 100 VUs, modest completion-time changes hid a much larger HTTP shift.** The median absolute change in completion time was about **`3%`**. Meanwhile, HTTP P50 rose in all 16 pairs, with a median increase of **`67%`**. The HTTP increase now covered both P50 and P90 in every pair.
- **At 1,000 VUs, HTTP P50 rose across all 16 pairs again**, with a median increase of **`86%`**. For overall completion, .NET behind Traefik termination stood out: TLS 1.2 finished in **`20.7–20.9 s`**, while TLS 1.3 took **`23.1–24.6 s`**. Every TLS 1.3 run took longer than every TLS 1.2 run, with a **`16%`** increase in the median completion time.
- **Traefik termination's handshake median reversed direction.** In the baseline and ramp, TLS 1.3 lowered handshake P50 across all four backends. At both 100 and 1,000 VUs, it rose across all four. For Java at 1,000 VUs, it went from **`8.81 → 11.64 ms`**, about **`32%`** higher.

I originally expected TLS 1.3 to easily win here across the board. But looking closer at the client runtime, two subtle mechanics complicate things.

First, under Go's default TLS configuration, the client
[generates a hybrid ML-KEM key share](https://github.com/golang/go/blob/go1.26.5/src/crypto/tls/key_schedule.go)
for every fresh TLS 1.3 connection, even when the server negotiates plain X25519.
Generating that unused post-quantum share burns client CPU cycles and inflates the ClientHello payload.
When hundreds of virtual users open connections in a tight loop, that extra computational and serialization tax
can matter more than the single round trip saved on a low-latency cloud subnet.

There is also an interesting metric-boundary quirk between protocol versions. With
[TLS 1.3](https://github.com/golang/go/blob/go1.26.5/src/crypto/tls/handshake_client_tls13.go),
the client considers its handshake complete the moment it sends its final handshake message. But the server still has to receive and process that flight.
[TLS 1.2](https://github.com/golang/go/blob/go1.26.5/src/crypto/tls/handshake_client.go)
waits for the server's final Finished message first. In k6,
[`Tracer.TLSHandshakeDone()`](https://github.com/grafana/k6/blob/v2.2.0/lib/netext/httpext/tracer.go#L242-L247)
timestamps the client-side completion, and
[`Tracer.Done()`](https://github.com/grafana/k6/blob/v2.2.0/lib/netext/httpext/tracer.go#L346-L381)
measures HTTP sending time from that timestamp, adding waiting and receiving time. Under TLS 1.3, any trailing server-side handshake processing or queueing bleeds directly into the HTTP duration measurement. That boundary shift likely accounts for a chunk of the apparent HTTP response-time increase.

A solid follow-up experiment would rerun these high-churn races with **`GODEBUG=tlsmlkem=0` on the k6 process** to remove the unused hybrid share, paired with Pyroscope CPU profiles on client and server alongside packet captures.

{{< accordion mode="open" separated=true >}}
  {{< accordionItem title="100K race: three completion times per TLS configuration at 10 VUs" icon="chart-line" open=true >}}
{{< figure src="charts/tls-comparison/tls-completion-race-10-vus.svg" alt="Three separate 100,000-request completion bars per topology and stack for each of TLS 1.2 and 1.3 with P-256 and X25519 at 10 virtual users" >}}
  {{< /accordionItem >}}
  {{< accordionItem title="100K race: three completion times per TLS configuration at 100 VUs" icon="chart-line" >}}
{{< figure src="charts/tls-comparison/tls-completion-race-100-vus.svg" alt="Three separate 100,000-request completion bars per topology and stack for each of TLS 1.2 and 1.3 with P-256 and X25519 at 100 virtual users" >}}
  {{< /accordionItem >}}
  {{< accordionItem title="100K race: three completion times per TLS configuration at 1,000 VUs" icon="chart-line" >}}
{{< figure src="charts/tls-comparison/tls-completion-race-1000-vus.svg" alt="Three separate 100,000-request completion bars per topology and stack for each of TLS 1.2 and 1.3 with P-256 and X25519 at 1,000 virtual users" >}}
  {{< /accordionItem >}}
{{< /accordion >}}

## Results

### Direct VM

In this topology, the workload runs directly inside a dedicated VM container behind the Azure Load Balancer. TLS terminates within the application runtime.

{{< accordion mode="open" separated=true >}}
  {{< accordionItem title="Steady baseline (1,000 req/s): Direct VM" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s2-vm-tls12-p-256/baseline-rps-timeline.svg" alt="Direct VM achieved request rate timeline at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/baseline-http-flat-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/baseline-tls-flat-percentiles.svg" alt="Direct VM client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s2-vm-tls12-x25519/baseline-rps-timeline.svg" alt="Direct VM achieved request rate timeline at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/baseline-http-flat-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/baseline-tls-flat-percentiles.svg" alt="Direct VM client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s2-vm-tls13-p-256/baseline-rps-timeline.svg" alt="Direct VM achieved request rate timeline at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/baseline-http-flat-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/baseline-tls-flat-percentiles.svg" alt="Direct VM client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s2-vm-tls13-x25519/baseline-rps-timeline.svg" alt="Direct VM achieved request rate timeline at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/baseline-http-flat-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/baseline-tls-flat-percentiles.svg" alt="Direct VM client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
  {{< accordionItem title="High-pressure linear ramp (up to 6,000 req/s): Direct VM" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s2-vm-tls12-p-256/linear-rps-timeline.svg" alt="Direct VM achieved request rate timeline for linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/linear-active-vus-timeline.svg" alt="Direct VM active VUs timeline for linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/linear-http-expandable-1000-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles for expandable linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/linear-tls-expandable-1000-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles for expandable linear ramp (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s2-vm-tls12-x25519/linear-rps-timeline.svg" alt="Direct VM achieved request rate timeline for linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/linear-active-vus-timeline.svg" alt="Direct VM active VUs timeline for linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/linear-http-expandable-1000-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles for expandable linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/linear-tls-expandable-1000-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles for expandable linear ramp (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s2-vm-tls13-p-256/linear-rps-timeline.svg" alt="Direct VM achieved request rate timeline for linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/linear-active-vus-timeline.svg" alt="Direct VM active VUs timeline for linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/linear-http-expandable-1000-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles for expandable linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/linear-tls-expandable-1000-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles for expandable linear ramp (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s2-vm-tls13-x25519/linear-rps-timeline.svg" alt="Direct VM achieved request rate timeline for linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/linear-active-vus-timeline.svg" alt="Direct VM active VUs timeline for linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/linear-http-expandable-1000-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles for expandable linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/linear-tls-expandable-1000-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles for expandable linear ramp (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
  {{< accordionItem title="Fixed-work race (10, 100, and 1,000 VUs): Direct VM" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s2-vm-tls12-p-256/race-rps-10-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/race-http-10-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/race-tls-10-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/race-rps-100-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/race-http-100-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/race-tls-100-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/race-rps-1000-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/race-http-1000-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s2-vm-tls12-p-256/race-tls-1000-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s2-vm-tls12-x25519/race-rps-10-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/race-http-10-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/race-tls-10-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/race-rps-100-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/race-http-100-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/race-tls-100-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/race-rps-1000-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/race-http-1000-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s2-vm-tls12-x25519/race-tls-1000-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s2-vm-tls13-p-256/race-rps-10-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/race-http-10-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/race-tls-10-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/race-rps-100-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/race-http-100-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/race-tls-100-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/race-rps-1000-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/race-http-1000-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s2-vm-tls13-p-256/race-tls-1000-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s2-vm-tls13-x25519/race-rps-10-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/race-http-10-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/race-tls-10-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/race-rps-100-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/race-http-100-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/race-tls-100-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/race-rps-1000-vus-timeline.svg" alt="Direct VM achieved request rate timeline at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/race-http-1000-vus-percentiles.svg" alt="Direct VM HTTP response-time percentiles at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s2-vm-tls13-x25519/race-tls-1000-vus-percentiles.svg" alt="Direct VM TLS handshake percentiles at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
{{< /accordion >}}

### Direct Kubernetes Pod

Here, client connections route through the Azure Load Balancer directly to the Kubernetes Service and application Pod, where TLS terminates in the container runtime.

{{< accordion mode="open" separated=true >}}
  {{< accordionItem title="Steady baseline (1,000 req/s): Direct Kubernetes Pod" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/baseline-rps-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/baseline-http-flat-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/baseline-tls-flat-percentiles.svg" alt="Direct Kubernetes Pod client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/baseline-rps-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/baseline-http-flat-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/baseline-tls-flat-percentiles.svg" alt="Direct Kubernetes Pod client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/baseline-rps-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/baseline-http-flat-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/baseline-tls-flat-percentiles.svg" alt="Direct Kubernetes Pod client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/baseline-rps-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/baseline-http-flat-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/baseline-tls-flat-percentiles.svg" alt="Direct Kubernetes Pod client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
  {{< accordionItem title="High-pressure linear ramp (up to 6,000 req/s): Direct Kubernetes Pod" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/linear-rps-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline for linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/linear-active-vus-timeline.svg" alt="Direct Kubernetes Pod active VUs timeline for linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/linear-http-expandable-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles for expandable linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/linear-tls-expandable-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles for expandable linear ramp (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/linear-rps-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline for linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/linear-active-vus-timeline.svg" alt="Direct Kubernetes Pod active VUs timeline for linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/linear-http-expandable-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles for expandable linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/linear-tls-expandable-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles for expandable linear ramp (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/linear-rps-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline for linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/linear-active-vus-timeline.svg" alt="Direct Kubernetes Pod active VUs timeline for linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/linear-http-expandable-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles for expandable linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/linear-tls-expandable-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles for expandable linear ramp (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/linear-rps-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline for linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/linear-active-vus-timeline.svg" alt="Direct Kubernetes Pod active VUs timeline for linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/linear-http-expandable-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles for expandable linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/linear-tls-expandable-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles for expandable linear ramp (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
  {{< accordionItem title="Fixed-work race (10, 100, and 1,000 VUs): Direct Kubernetes Pod" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/race-rps-10-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/race-http-10-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/race-tls-10-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/race-rps-100-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/race-http-100-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/race-tls-100-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/race-rps-1000-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/race-http-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-p-256/race-tls-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/race-rps-10-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/race-http-10-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/race-tls-10-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/race-rps-100-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/race-http-100-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/race-tls-100-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/race-rps-1000-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/race-http-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls12-x25519/race-tls-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/race-rps-10-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/race-http-10-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/race-tls-10-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/race-rps-100-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/race-http-100-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/race-tls-100-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/race-rps-1000-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/race-http-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-p-256/race-tls-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/race-rps-10-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/race-http-10-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/race-tls-10-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/race-rps-100-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/race-http-100-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/race-tls-100-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/race-rps-1000-vus-timeline.svg" alt="Direct Kubernetes Pod achieved request rate timeline at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/race-http-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod HTTP response-time percentiles at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s4-k8s-pod-tls13-x25519/race-tls-1000-vus-percentiles.svg" alt="Direct Kubernetes Pod TLS handshake percentiles at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
{{< /accordion >}}

### Traefik Passthrough

In this Gateway API setup, Traefik forwards the encrypted client TLS connection directly to the application pod without decrypting it. TLS terminates inside the application container.

{{< accordion mode="open" separated=true >}}
  {{< accordionItem title="Steady baseline (1,000 req/s): Traefik Passthrough" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/baseline-rps-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/baseline-http-flat-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/baseline-tls-flat-percentiles.svg" alt="Traefik Passthrough client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/baseline-rps-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/baseline-http-flat-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/baseline-tls-flat-percentiles.svg" alt="Traefik Passthrough client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/baseline-rps-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/baseline-http-flat-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/baseline-tls-flat-percentiles.svg" alt="Traefik Passthrough client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/baseline-rps-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/baseline-http-flat-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/baseline-tls-flat-percentiles.svg" alt="Traefik Passthrough client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
  {{< accordionItem title="High-pressure linear ramp (up to 6,000 req/s): Traefik Passthrough" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/linear-rps-timeline.svg" alt="Traefik Passthrough achieved request rate timeline for linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/linear-active-vus-timeline.svg" alt="Traefik Passthrough active VUs timeline for linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/linear-http-expandable-1000-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles for expandable linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/linear-tls-expandable-1000-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles for expandable linear ramp (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/linear-rps-timeline.svg" alt="Traefik Passthrough achieved request rate timeline for linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/linear-active-vus-timeline.svg" alt="Traefik Passthrough active VUs timeline for linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/linear-http-expandable-1000-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles for expandable linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/linear-tls-expandable-1000-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles for expandable linear ramp (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/linear-rps-timeline.svg" alt="Traefik Passthrough achieved request rate timeline for linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/linear-active-vus-timeline.svg" alt="Traefik Passthrough active VUs timeline for linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/linear-http-expandable-1000-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles for expandable linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/linear-tls-expandable-1000-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles for expandable linear ramp (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/linear-rps-timeline.svg" alt="Traefik Passthrough achieved request rate timeline for linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/linear-active-vus-timeline.svg" alt="Traefik Passthrough active VUs timeline for linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/linear-http-expandable-1000-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles for expandable linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/linear-tls-expandable-1000-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles for expandable linear ramp (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
  {{< accordionItem title="Fixed-work race (10, 100, and 1,000 VUs): Traefik Passthrough" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/race-rps-10-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/race-http-10-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/race-tls-10-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/race-rps-100-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/race-http-100-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/race-tls-100-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/race-rps-1000-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/race-http-1000-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-p-256/race-tls-1000-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/race-rps-10-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/race-http-10-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/race-tls-10-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/race-rps-100-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/race-http-100-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/race-tls-100-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/race-rps-1000-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/race-http-1000-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls12-x25519/race-tls-1000-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/race-rps-10-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/race-http-10-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/race-tls-10-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/race-rps-100-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/race-http-100-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/race-tls-100-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/race-rps-1000-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/race-http-1000-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-p-256/race-tls-1000-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/race-rps-10-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/race-http-10-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/race-tls-10-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/race-rps-100-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/race-http-100-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/race-tls-100-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/race-rps-1000-vus-timeline.svg" alt="Traefik Passthrough achieved request rate timeline at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/race-http-1000-vus-percentiles.svg" alt="Traefik Passthrough HTTP response-time percentiles at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s5-traefik-passthrough-tls13-x25519/race-tls-1000-vus-percentiles.svg" alt="Traefik Passthrough TLS handshake percentiles at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
{{< /accordion >}}

### Traefik Termination

In this topology, Traefik terminates the client-facing TLS connection, validates the backend certificate, and opens a separate TLS connection to the application pod.

{{< accordion mode="open" separated=true >}}
  {{< accordionItem title="Steady baseline (1,000 req/s): Traefik Termination" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/baseline-rps-timeline.svg" alt="Traefik Termination achieved request rate timeline at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/baseline-http-flat-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/baseline-tls-flat-percentiles.svg" alt="Traefik Termination client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/baseline-rps-timeline.svg" alt="Traefik Termination achieved request rate timeline at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/baseline-http-flat-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/baseline-tls-flat-percentiles.svg" alt="Traefik Termination client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/baseline-rps-timeline.svg" alt="Traefik Termination achieved request rate timeline at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/baseline-http-flat-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/baseline-tls-flat-percentiles.svg" alt="Traefik Termination client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/baseline-rps-timeline.svg" alt="Traefik Termination achieved request rate timeline at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/baseline-http-flat-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/baseline-tls-flat-percentiles.svg" alt="Traefik Termination client-facing TLS handshake percentiles at 1,000 req/s (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
  {{< accordionItem title="High-pressure linear ramp (up to 6,000 req/s): Traefik Termination" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/linear-rps-timeline.svg" alt="Traefik Termination achieved request rate timeline for linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/linear-active-vus-timeline.svg" alt="Traefik Termination active VUs timeline for linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/linear-http-expandable-1000-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles for expandable linear ramp (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/linear-tls-expandable-1000-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles for expandable linear ramp (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/linear-rps-timeline.svg" alt="Traefik Termination achieved request rate timeline for linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/linear-active-vus-timeline.svg" alt="Traefik Termination active VUs timeline for linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/linear-http-expandable-1000-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles for expandable linear ramp (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/linear-tls-expandable-1000-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles for expandable linear ramp (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/linear-rps-timeline.svg" alt="Traefik Termination achieved request rate timeline for linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/linear-active-vus-timeline.svg" alt="Traefik Termination active VUs timeline for linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/linear-http-expandable-1000-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles for expandable linear ramp (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/linear-tls-expandable-1000-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles for expandable linear ramp (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/linear-rps-timeline.svg" alt="Traefik Termination achieved request rate timeline for linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/linear-active-vus-timeline.svg" alt="Traefik Termination active VUs timeline for linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/linear-http-expandable-1000-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles for expandable linear ramp (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/linear-tls-expandable-1000-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles for expandable linear ramp (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
  {{< accordionItem title="Fixed-work race (10, 100, and 1,000 VUs): Traefik Termination" icon="chart-line" open=false md=false >}}
{{< tabs >}}
{{< tab md=false label="TLS 1.2 / P-256" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/race-rps-10-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/race-http-10-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/race-tls-10-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 10 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/race-rps-100-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/race-http-100-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/race-tls-100-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 100 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/race-rps-1000-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/race-http-1000-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-p-256/race-tls-1000-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 1,000 VUs (TLS 1.2 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.2 / X25519" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/race-rps-10-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/race-http-10-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/race-tls-10-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 10 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/race-rps-100-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/race-http-100-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/race-tls-100-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 100 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/race-rps-1000-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/race-http-1000-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls12-x25519/race-tls-1000-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 1,000 VUs (TLS 1.2 / X25519)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / P-256" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/race-rps-10-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/race-http-10-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/race-tls-10-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 10 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/race-rps-100-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/race-http-100-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/race-tls-100-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 100 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/race-rps-1000-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/race-http-1000-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-p-256/race-tls-1000-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 1,000 VUs (TLS 1.3 / P-256)" >}}
{{< /tab >}}
{{< tab md=false label="TLS 1.3 / X25519" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/race-rps-10-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/race-http-10-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/race-tls-10-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 10 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/race-rps-100-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/race-http-100-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/race-tls-100-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 100 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/race-rps-1000-vus-timeline.svg" alt="Traefik Termination achieved request rate timeline at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/race-http-1000-vus-percentiles.svg" alt="Traefik Termination HTTP response-time percentiles at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< figure src="charts/s6-traefik-terminate-tls13-x25519/race-tls-1000-vus-percentiles.svg" alt="Traefik Termination TLS handshake percentiles at 1,000 VUs (TLS 1.3 / X25519)" >}}
{{< /tab >}}
{{< /tabs >}}
  {{< /accordionItem >}}
{{< /accordion >}}

## A Few Bitter Grounds

Every performance test leaves behind a few mysterious anomalies that force you down the rabbit hole. Digging through decrypted packet captures, CPU flamegraphs, and kernel traces surfaced three architectural quirks that explained some of the wilder swings in my initial testing.

### .NET !close_notify

One of the biggest headaches during initial high-churn testing was client-side port exhaustion on .NET workloads, where k6 suddenly threw `cannot assign requested address` (`EADDRNOTAVAIL`). Go and Java never hit this under identical loads. That was partly on me for running the load generator with only one outbound IP in the beginning. But would I have ever caught this teardown difference if I had started with multiple IPs? Probably not.

- **.NET (Kestrel) relies on quiet shutdown.** In Kestrel's [`HttpsConnectionMiddleware`](https://github.com/dotnet/aspnetcore/blob/v10.0.0/src/Servers/Kestrel/Core/src/Middleware/HttpsConnectionMiddleware.cs#L231-L249), connection teardown disposes the stream without an explicit `SslStream.ShutdownAsync()`. Under the hood, [.NET's OpenSSL interop](https://github.com/dotnet/runtime/blob/v10.0.0/src/libraries/Common/src/Interop/Unix/System.Security.Cryptography.Native/Interop.OpenSsl.cs#L246-L255) enables quiet shutdown via [`pal_ssl.c`](https://github.com/dotnet/runtime/blob/v10.0.0/src/native/libs/System.Security.Cryptography.Native/pal_ssl.c#L639-L643). The server skips sending a reciprocal TLS [`close_notify`](https://www.rfc-editor.org/rfc/rfc8446.html#section-6.1) alert. The client sends a TCP `FIN`, the server acknowledges it, and the client socket lands in Linux's [`TIME-WAIT`](https://github.com/torvalds/linux/blob/v6.12/net/ipv4/tcp_minisocks.c#L122-L167) state. At 1,000 VUs opening fresh connections, the client accumulated 28,232 sockets in `TIME-WAIT`, completely exhausting its local ephemeral port pool (`32768–60999`).
- **golang and Java showed a different shutdown sequence.** In the packet captures, [golang](https://github.com/golang/go/blob/go1.26.5/src/crypto/tls/conn.go#L1423-L1483) and Netty using the JDK provider sent a reciprocal `close_notify`. Those bytes arrived after the client had closed its socket, and Linux responded with an RST, releasing connection state sooner.

### libc penalty

My early test iterations used Alpine base images for .NET containers to keep images slim. That turned out to be a major trap too.

Comparing identical .NET workloads on Ubuntu (glibc + OpenSSL 3.5.5) versus Alpine (musl libc + OpenSSL 3.5.8) under a 100-VU race produced "some" differences:

- **glibc:** Completed all 100,000 scheduled requests in `~48.7 s` with zero dropped iterations.
- **musl:** Hit the scenario's 60-second test deadline, dropped nearly 4,000 iterations, and completed only 96,035 requests.

Pyroscope profiling showed more sampled CPU time under OpenSSL allocation calls
on Alpine, along with more time waiting in allocation paths containing
`futex_wait`. The captured profiles recorded **`1.26 s`** of sampled CPU time
under `CRYPTO_malloc` on Ubuntu versus **`5.32 s`** on Alpine. That makes
allocation behavior worth investigating, especially with a
[fresh SSL context being created for every connection](https://github.com/dotnet/runtime/blob/v10.0.12/src/libraries/Common/src/Interop/Unix/System.Security.Cryptography.Native/Interop.OpenSsl.cs#L181-L211). I couldn't pin down how much of the difference came from libc and how much came from the other native components that changed between the images yet, but I definitelly want to dig down deeper and come up with a follow-up

### BoringSSL

If you terminate TLS in your Java web server and your platform or organizational policies allow switching to BoringSSL, do yourself a favor and give it a shot.

In the JDK-provider run, the CPU profiles showed much more sampled time under
SunEC's elliptic-curve code, including
[`ECDHKeyAgreement`](https://github.com/openjdk/jdk/blob/jdk-25%2B7/src/java.base/share/classes/sun/security/ec/ECDHKeyAgreement.java)
and ECDSA signing.

With Netty using
[`netty-tcnative-boringssl-static`](https://github.com/netty/netty-tcnative),
the separate 100,000-request comparison finished in **`24.24 s`**, versus
**`52.19 s`** with the JDK's SunJSSE provider. Both runs completed all the
requests, but changing its TLS provider made a substantial difference.

## Conclusions

My main takeaway is that the TLS implementation and connection handling deserve
more attention than the web framework name alone.

Most benchmarks measure keep-alive connections, which basically tests how fast an HTTP framework can serialize bytes over an open socket. But when every request opens a fresh connection, the framework matters a lot less. In Java scenarios, swapping SunJSSE for BoringSSL dropped the 100,000-request race from **`52.19 s`** to **`24.24 s`** without touching any application code. When connection churn is high, the crypto library implementation,  the base image libraries, and how your runtime handles socket teardown can decide how much throughput you will actually get.

The topologies come down to where you want that connection work to happen. Direct VM and Pod paths give you the lowest latency under light load because there is no proxy in the middle, but your application container has to handle every incoming handshake and socket cleanup directly. Ingress termination adds a small hop delay of around 0.4 ms, but offloads client handshakes so your backend doesn't have to deal with thousands of new TLS connections. Passthrough makes sense when you need to route non-HTTP TLS streams like databases that an L7 proxy can't parse or when you want SNI routing without giving your private keys to the ingress, or when the backend needs to authenticate client certificates directly. You won't get better performance out of it though, since you still pay for the proxy hop while the application still runs the full handshake.

In the end, choosing where to terminate TLS is about deciding which layer of your setup absorbs that connection overhead, and making sure the runtime sitting there is actually configured to handle it.

## Reproducibility

Everything used to run these benchmarks is available in the [where-to-tls repository](https://github.com/slabbancz/where-to-tls): Terraform
and cloud-init definitions, Kubernetes manifests, application code, k6 load scripts, and exact TLS configurations.

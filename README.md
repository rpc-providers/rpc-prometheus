# Monitoring setup with federated prometheus

## Federated prometheus
To be able to show and measure individual provider data we need to be able to access the prometheus statistics of that provider. This could be done by adding all worker nodes of a provider to a central prometheus but this would be cumbersome since nodes can change for maintenance, replacement or other reasons. 

More logical is to use a federated prometheus setup, were the individual providers maintain their own prometheus instance which then federates its data to a central monitoring instance:

```mermaid
flowchart BT
    subgraph "Monitoring"
        direction BT
        A[(Central Prometheus)]
        A --> Y[Grafana] 
        A --> Z[Reporting] 
    end

    subgraph "Provider 1"
    direction BT
    B[(Provider Prometheus)] -- "/federate" --> A
    D[Node01] -- "/metrics" --> B
    E[Node02] -- "/metrics" --> B

    end
    subgraph "Provider 2"
    direction BT
    C[(Provider Prometheus)] -- "/federate" --> A
    G[Node01] -- "/metrics" --> C
    H[Node02] -- "/metrics" --> C
    end
```

## Central Prometheus

The central prometheus would have a job per provider to ingest the federate information from the provider's prometheus:
 
```
  - job_name: "stakeworld"
    scrape_interval: 15s
    honor_labels: true
    metrics_path: "/federate"
    params:
      "match[]":
        - '{job="stakeworld"}'
    static_configs:
      - targets:
        - monitor.stakeworld.io:9091
  - job_name: "provider2"
    scrape_interval: 15s
    honor_labels: true
    metrics_path: "/federate"
    params:
      "match[]":
        - '{job="provider2"}'
    static_configs:
      - targets:
        - mon.provider2.io:9090
  - job_name: "provider3"
    scrape_interval: 15s
    (...)
```

If you want to add your job to the monitoring config please add it via a pull request to the [prometheus-master.yml](https://github.com/rpc-providers/rpc-prometheus/blob/master/prometheus-master.yml) configuration file in this repository. 

## Provider prometheus

The provider's prometheus would scrape it's own rpc node's and enable access to this information to the central prometheus. 

Best is to use a dedicated Prometheus instance that only knows the relevant endpoints. For example a separate VPS just for monitoring. Alternatively, you can add a new job to an existing Prometheus instance, but keep in mind that:
- Other data may become visible to the monitoring instance.
- Make sure you don't measure the same data twice. A good practice is to separate it using a unique identifier, like {job=...}, so your panels stay accurate.

A typical job would be: 

```
scrape_configs:
  - job_name: "stakeworld"
    scrape_interval: 5s
    static_configs:
      - targets: ['node01:9615']
      - targets: ['node02:9615']
```

A full example can be found in [prometheus-provider.yml](https://github.com/rpc-providers/rpc-prometheus/blob/master/prometheus-provider.yml) 

## Access 

Access should be granted to the central monitor to access the provider prometheus, this can be done by opening a port via the firewall, for example:

``` 
ufw allow from 18.156.32.189 to any port 9090
ufw allow from 2a05:d014:14e6:7900:f077:59ec:99f3:9bd7 to any port 9090
```

If there is a wish in the future this could also be done through a vpn connection, so the connection would be encoded.

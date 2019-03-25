# Debugging Istio Telemetry

This repo contains a demo/tutorial for debugging Istio telemetry (specifically Mixer), part of [Next 2019: HYB303 Debugging Istio](). 

## Cluster Setup

First, create a cluster that we'll use to install [Istio 1.0.6](https://github.com/istio/istio/releases/tag/1.0.6):
```
gcloud container clusters create debugging-telemetry \
    --cluster-version=latest --machine-type=n1-standard-2 \
    --num-nodes=4
```

Next, grab the cluster credentials and create a `cluster-admin-binding`:
- `gcloud container clusters get-credentials debugging-telemetry`
- `kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)`

Then download the appropriate Istio 1.0.6 for your platform (`linux`, `mac`, or `win`) from the [Istio 1.0.6 release](https://github.com/istio/istio/releases/tag/1.0.6).

Now install Istio 1.0.6:
- `kubectl apply -f install/kubernetes/istio-demo.yaml`

Wait a few minutes for the Pods and Services to spin up, then active automatic sidecar injection:
- `kubectl label ns default istio-injection=enabled`

## Deploy the Weather App

*Note:* We'll use the sample [weatherinfo](https://github.com/crcsmnky/weatherinfo) app as our example. If you don't have the weatherinfo images in your project, you'll need to clone that repo, build, and push those images before proceeding.

First, apply `weather-rules.yaml` to setup the appropriate `VirtualService`, `ServiceEntry`, and `Gateway` configurations:
- `kubectl apply -f weather-rules.yaml`

Next, apply `weather-deployment.yaml` to deploy the sample app:
- `kubectl apply -f weather-deployment.yaml`

## Monitoring with Prometheus and Grafana

First, let's open up the built-in Prometheus and Grafana:
- `kubectl port-forward -n istio-system deployment/prometheus 9090:9090`
- `kubectl port-forward -n istio-system deployment/grafana 3000:3000`

Looking around Prometheus we can see all of the Istio-specific metrics we would expect, like `istio_requests_total`. But what about our custom metrics? In `weatherinfo` there are custom Prometheus `Gauge` metrics that don't appear here. 

Turns out you need to add specific `annotations` to your Kubernetes deployment manifest to have those metrics picked up. The following example should be placed inside of `spec.template.metadata`:

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "5000"
  prometheus.io/path: "/metrics"
```

That's it! If we update our deployment with those `annotations` for `weather-backend` we'll start seeing our custom Prometheus metrics:
- `kubectl apply -f weather-deployment-annotated.yaml`

## Troubleshooting Telemetry

But what if you're still experiencing missing metrics? Let's run through some diagnostic steps you can use to troubleshoot Istio's telemetry components.

### Confirming Mixer `Report` calls

Start by confirming that Mixer's metrics are configured and providing data:
- `kubectl port-forward -n istio-system deployment/istio-telemetry 9093 &`
- `curl localhost:9093/metrics | grep 'grpc'`

You should see something like the following:

```
# HELP grpc_server_msg_received_total Total number of RPC stream messages received on the server.
# TYPE grpc_server_msg_received_total counter
grpc_server_msg_received_total{grpc_method="Report",grpc_service="istio.mixer.v1.Mixer",grpc_type="unary"} 171829
# HELP grpc_server_msg_sent_total Total number of gRPC stream messages sent by the server.
# TYPE grpc_server_msg_sent_total counter
grpc_server_msg_sent_total{grpc_method="Report",grpc_service="istio.mixer.v1.Mixer",grpc_type="unary"} 171829
```

If you don't see anything like `grpc_method="Report"`, ensure that your services have the `istio-proxy` sidecar in their Pods.

### Identifying Mixer configuration problems

Next, confirm that there are no errors in the configuration supplied to Mixer:
- `curl localhost:9093/metrics | grep 'mixer_'`

```
# HELP mixer_config_adapter_info_config_count The number of known adapters in the current config.
# TYPE mixer_config_adapter_info_config_count counter
mixer_config_adapter_info_config_count{configID="-1"} 0
```

The following Mixer metrics should always be 0. If not, something is wrong in the supplied Mixer configuration.
- `mixer_config_adapter_info_config_error_count`
- `mixer_config_handler_validation_error_count`
- `mixer_config_instance_config_error_count`
- `mixer_config_rule_config_error_count`
- `mixer_config_rule_config_match_error_count`
- `mixer_config_unsatisfied_action_handler_count`
- `mixer_handler_handler_build_failure_count`

### Examining Mixer logs

Then let's check if Mixer is receiving information from the `istio-proxy` containers in your Pods:
- `kubectl logs -n istio-system -l app=telemetry -c mixer`

Looking through there, you should see several log events that you can read through to determine if Mixer is receiving telemetry data.

```json
{"level":"info","time":"2019-03-23T21:28:50.295944Z","instance":"accesslog.logentry.istio-system","apiClaims":"","apiKey":"","clientTraceId":"","connection_security_policy":"none","destinationApp":"telemetry","destinationIp":"10.36.3.6","destinationName":"istio-telemetry-78f76f9d6-r9z2m","destinationNamespace":"istio-system","destinationOwner":"kubernetes://apis/apps/v1/namespaces/istio-system/deployments/istio-telemetry","destinationPrincipal":"","destinationServiceHost":"istio-telemetry.istio-system.svc.cluster.local","destinationWorkload":"istio-telemetry","httpAuthority":"mixer","latency":"2.135929ms","method":"POST","protocol":"http","receivedBytes":1506,"referer":"","reporter":"destination","requestId":"4fce5cfb-c6f6-4b21-944e-33cd69f32671","requestSize":1123,"requestedServerName":"","responseCode":200,"responseSize":5,"responseTimestamp":"2019-03-23T21:28:50.297931Z","sentBytes":141,"sourceApp":"loadgenerator","sourceIp":"10.36.0.18","sourceName":"loadgenerator-58d68bddcc-72jt5","sourceNamespace":"default","sourceOwner":"kubernetes://apis/apps/v1/namespaces/default/deployments/loadgenerator","sourcePrincipal":"","sourceWorkload":"loadgenerator","url":"/istio.mixer.v1.Mixer/Report","userAgent":"","xForwardedFor":"10.36.0.18"}
```

### Reviewing handler and metrics configuration

Next let's check to make sure that rules for Mixer exist to send data to Prometheus:
- `kubectl get rules --all-namespaces | grep prom`

You should see something similar to the output below:
```
istio-system   promhttp                 1d
istio-system   promtcp                  1d
```

Now check that the Prometheus handler is correctly setup:
- `kubectl get prometheus.config.istio.io -n istio-system -o yaml`

The output yaml should contain a list of metrics at `.items[0].spec.metrics` including (but not limited to):
- `requestcount.metric.istio-system` and
- `requestduration.metric.istio-system`

You can also check that the Istio metrics are also configured:
- `kubectl get metrics.config.istio.io -n istio-system`

```
NAME              AGE
requestcount      2d
requestduration   2d
requestsize       2d
responsesize      2d
tcpbytereceived   2d
tcpbytesent       2d
```

### Updating Mixer configuration

If any of these configurations are incorrect, you will need to supply an updated Mixer configuration: 
- `wget https://github.com/istio/istio/archive/1.0.6.tar.gz && tar zxf 1.0.6.tar.gz && cd istio-1.0.6` 
- `for FLAG in true false; do helm template --set mixer.enabled=$FLAG --namespace istio-system install/kubernetes/helm/istio > mixer-$FLAG.yaml; done`
- `diff --line-format=%L mixer-true.yaml mixer-false.yaml > mixer-config.yaml`

Then you can apply `mixer-config.yaml` to update your configuration and address any configuration issues.



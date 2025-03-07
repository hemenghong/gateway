# Customize EnvoyProxy

Envoy Gateway provides a [EnvoyProxy][] CRD that can be linked to the ParametersRef
in GatewayClass y cluster admins to customize the managed EnvoyProxy Deployment and
Service. To learn more about GatewayClass and ParametersRef, please refer to [Gateway API documentation][].

## Installation

Follow the steps from the [Quickstart Guide](quickstart.md) to install Envoy Gateway and the example manifest.
Before proceeding, you should be able to query the example backend using HTTP.

## Add GatewayClass ParametersRef

First, you need to add ParametersRef in GatewayClass, and refer to EnvoyProxy Config:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: config.gateway.envoyproxy.io
    kind: EnvoyProxy
    name: custom-proxy-config
    namespace: envoy-gateway-system
EOF
```

## Customize EnvoyProxy Deployment Replicas

You can customize the EnvoyProxy Deployment Replicas via EnvoyProxy Config like:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: config.gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        replicas: 2
EOF
```

After you apply the config, you should see the replicas of envoyproxy changes to 2.
And also you can dynamically change the value.

``` shell
kubectl get deployment envoy-gateway
```

## Customize EnvoyProxy Image

You can customize the EnvoyProxy Image via EnvoyProxy Config like:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: config.gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        container:
          image: envoyproxy/envoy:v1.25-latest
EOF
```

After applying the config, you can get the deployment image, and see it has changed.

## Customize EnvoyProxy Pod Annotations

You can customize the EnvoyProxy Pod Annotations via EnvoyProxy Config like:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: config.gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        pod:
          annotations:
            custom1: deploy-annotation1
            custom2: deploy-annotation2
EOF
```

After applying the config, you can get the envoyproxy pods, and see new annotations has been added.

## Customize EnvoyProxy Deployment Resources

You can customize the EnvoyProxy Deployment Resources via EnvoyProxy Config like:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: config.gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        container:
          resources:
            requests:
              cpu: 150m
              memory: 640Mi
            limits:
              cpu: 500m
              memory: 1Gi
EOF
```

## Customize EnvoyProxy Deployment Env

You can customize the EnvoyProxy Deployment Env via EnvoyProxy Config like: 

```shell
cat <<EOF | kubectl apply -f -
apiVersion: config.gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        container:
          env:
          - name: env_a
            value: env_a_value
          - name: env_b
            value: env_b_value
EOF
```

> Envoy Gateway has provided two initial `env` `ENVOY_GATEWAY_NAMESPACE` and `ENVOY_POD_NAME` for envoyproxy container.

After applying the config, you can get the envoyproxy deployment, and see resources has been changed.

## Customize EnvoyProxy Deployment Volumes or VolumeMounts

You can customize the EnvoyProxy Deployment Volumes or VolumeMounts via EnvoyProxy Config like:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: config.gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        container:
          volumeMounts:
          - mountPath: /certs
            name: certs
            readOnly: true
        pod:
          volumes:
          - name: certs
            secret:
              secretName: envoy-cert   
EOF
```

After applying the config, you can get the envoyproxy deployment, and see resources has been changed.

## Customize EnvoyProxy Service Annotations

You can customize the EnvoyProxy Service Annotations via EnvoyProxy Config like:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: config.gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        annotations:
          custom1: svc-annotation1
          custom2: svc-annotation2

EOF
```

After applying the config, you can get the envoyproxy service, and see annotations has been added.

## Customize EnvoyProxy Bootstrap Config

You can customize the EnvoyProxy Bootstrap Config via EnvoyProxy Config like:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: config.gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-proxy-config
  namespace: envoy-gateway-system
spec:
  bootstrap: |
    admin:
      access_log:
      - name: envoy.access_loggers.file
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
          path: /dev/null
      address:
        socket_address:
          address: 127.0.0.1
          port_value: 20000
    dynamic_resources:
      ads_config:
        api_type: DELTA_GRPC
        transport_api_version: V3
        grpc_services:
        - envoy_grpc:
            cluster_name: xds_cluster
        set_node_on_first_message_only: true
      lds_config:
        ads: {}
      cds_config:
        ads: {}
    static_resources:
      clusters:
      - connect_timeout: 10s
        load_assignment:
          cluster_name: xds_cluster
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: envoy-gateway
                    port_value: 18000
        typed_extension_protocol_options:
          "envoy.extensions.upstreams.http.v3.HttpProtocolOptions":
             "@type": "type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions"
             "explicit_http_config":
               "http2_protocol_options": {}
        name: xds_cluster
        type: STRICT_DNS
        transport_socket:
          name: envoy.transport_sockets.tls
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
            common_tls_context:
              tls_params:
                tls_maximum_protocol_version: TLSv1_3
              tls_certificate_sds_secret_configs:
              - name: xds_certificate
                sds_config:
                  path_config_source:
                    path: "/sds/xds-certificate.json"
                  resource_api_version: V3
              validation_context_sds_secret_config:
                name: xds_trusted_ca
                sds_config:
                  path_config_source:
                    path: "/sds/xds-trusted-ca.json"
                  resource_api_version: V3
    layered_runtime:
      layers:
      - name: runtime-0
        rtds_layer:
          rtds_config:
            ads: {}
          name: runtime-0
EOF
```

You can use [egctl translate](https://gateway.envoyproxy.io/latest/user/egctl.html#validating-gateway-api-configuration)
to get the default xDS Bootstrap configuration used by Envoy Gateway.

After applying the config, the bootstrap config will be overridden by the new config you provided.
Any errors in the configuration will be surfaced as status within the `GatewayClass` resource.
You can also validate this configuration using [egctl translate](https://gateway.envoyproxy.io/latest/user/egctl.html#validating-gateway-api-configuration).

[Gateway API documentation]: https://gateway-api.sigs.k8s.io/
[EnvoyProxy]: https://gateway.envoyproxy.io/latest/api/config_types.html#envoyproxy 

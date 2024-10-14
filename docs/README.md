# Home Assistant core chart

Chart for deploying and configuring [Home Assistant](https://www.home-assistant.io/installation) core, also provides ability  to deploy some useful features and addons, such as:

- [Storage provisioner].
- [Cloudflare](https://www.cloudflare.com/) tunnel.
- [Mosquitto] MQTT broker.
- [Ring MQTT] bridge.

## Configuration

The following table lists the main configurable parameters of the chart and their default values.

```yaml

## Home Assistant
ha: 
  connectivity:
    hostNetwork: true # Preferable, since many integrations require host network
    service:
      type: LoadBalancer
      externalTrafficPolicy: Cluster
    annotations:
      metallb.universe.tf/loadBalancerIPs: 192.168.2.30 # Preferable, since without it, metallb will assign random IP every time. If you have traefik or any other load balancer, use it is own annotation.
    ports:
      - name: http
        protocol: TCP
        containerPort: 8123
        servicePort: 8123
    probes:
      readiness:
        port: http
  volumes:
      - name: config
        persistentVolumeClaim:
          claimName: config # Claim must be created beforehand
      - name: bluetooth # Required for bluetooth integrations
        hostPath:
          path: /run/dbus
  volumeMounts:
      - mountPath: /config
        name: config
      - mountPath: /run/dbus
        name: bluetooth
  resources:
    limits:
      cpu: 1000m
      memory: 2Gi
    requests:
      cpu: 500m
      memory: 512Mi
  deployment:
      strategy:
        type:
          Recreate # Since HA currently do not support scalability, it is better to use Recreate strategy
```

## Optional configuration

```yaml

# Home Assistant storage provisioning (optional)
persistence:
  enabled: true # Default is false
  volumes: [] # list of PV and PVC to be provisioned, see how to configure here :https://github.com/kostiantyn-matsebora/helm-storage-provisioner?tab=readme-ov-file#configuration

# Cloudflare argo tunnel (optional)
# Use it if you want to expose HA to the internet without exposing it directly using public IP and port
# More information about configuration can be found here: 
tunnel:
  enabled: true # Default is false
  cloudflare:
    account: 
    tunnelName: 
    tunnelId:
    secret: 
    ingress:
```

## Addons

### Mosquitto MQTT broker

[Mosquitto] MQTT broker with authentication enabled. You can configure users and ACLs using `mosquitto.auth.users` and `mosquitto.auth.users.acl` respectively.  

Full configuration can be found [here](https://github.com/SINTEF/mosquitto-helm-chart/blob/main/charts/mosquitto/values.yaml).

Example:

```yaml
mosquitto:
  enabled: true # Default is false
  auth:
    enabled: true
    users:
      - username: user
        password: password
        acl:
          - topic: "#"
            access: readwrite
  service:
    type: LoadBalancer
    externalTrafficPolicy: Cluster
    annotations:
      metallb.universe.tf/loadBalancerIPs: 192.168.2.20
  ingress:
    enabled: false
```

 Take into account that you need to configure `mqtt` integration in HA to use this broker.
 Use either IP address or kubernetes service of the broker as a server address in the integration configuration.

### Ring MQTT bridge

[`Ring MQTT`](https://github.com/tsightler/ring-mqtt) bridge. You can configure MQTT broker URL and options using `ring-mqtt.mqtt_url` and `ring-mqtt.mqtt_options` respectively.  

Full configuration can be found [here](https://github.com/truecharts/public/blob/master/charts/stable/ring-mqtt/values.yaml).

Example:

```yaml
ring-mqtt:
  enabled: true # Default is false
  mqtt_url: mqtt://admin:encodedpassword@192.168.2.20:1883
  mqtt_options: --rejectUnauthorized=false
  persistence:
    data:
      enabled: true
      mountPath: /data
      size: 2Gi
      retain: true
  service:
    rtsp:
      clusterIP: 10.43.13.193 # It is important to use ClusterIP, since it is used for communication between HA and the bridge, and it is value you going to use in HA generic camera integration configuration.
```

Take into account that after deployment you need to:

- Configure `ring-mqtt` bridge, see pod logs for further instructions.
- Configure `generic camera` integration in HA to use this bridge.
  Use IP address of the bridge service as a server address in the integration configuration using following format: `rtsp://[service IP]:8554/[id_of_device]_live`


## Usage

Add helm [repository](https://kostiantyn-matsebora.github.io/helm-charts/) first:

```bash
helm repo add kostiantyn-matsebora https://kostiantyn-matsebora.github.io/helm-charts/
```

Install/upgrade helm chart using your custom values:

```bash

helm upgrade ha kostiantyn-matsebora/ha --install --values ./your-values.yaml

```

## Contributing

If you experience any issues, have a question or a suggestion, or if you wish
to contribute, feel free to [open an issue][issues] or
[start a discussion][discussions].

[issues]: https://github.com/kostiantyn-matsebora/helm-ha/issues
[discussions]: https://github.com/kostiantyn-matsebora/helm-ha/discussions

## License

[`MIT License`](../LICENSE)

[Ring MQTT]:https://github.com/tsightler/ring-mqtt
[Mosquitto]:https://mosquitto.org/
[Cloudflare tunnel]:https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
[Storage provisioner]:https://github.com/kostiantyn-matsebora/helm-storage-provisioner

# Immich Charts

Installs [Immich](https://github.com/immich-app/immich), a self-hosted photo and video backup solution directly 
from your mobile phone. 

> [!WARNING]
> The HTTP-based helm repo at https://immich-app.github.io/immich-charts/ is deprecated and will stop receiving updates soon.

# Goal

This repo contains helm charts the immich community developed to help deploy Immich on Kubernetes cluster.

It leverages the bjw-s [common-library chart](https://github.com/bjw-s-labs/helm-charts/tree/common-4.3.0/charts/library/common) to make configuration as easy as possible. 

# Installation

```
$ helm install --create-namespace --namespace immich immich oci://ghcr.io/immich-app/immich-charts/immich -f values.yaml
```

You should not copy the full values.yaml from this repository. Only set the values that you want to override.

There are a few things that you are required to configure in your values.yaml before installing the chart:
* You need to separately create a PVC for your library volume and configure `immich.persistence.library.existingClaim` to reference that PVC
* You need to make sure that Immich has access to a redis and postgresql instance. 
  * Redis (via Valkey) can be enabled directly in the `values.yaml` by setting `valkey.enabled: true`.
  * Postgres can now be optionally deployed by the chart by setting `postgres.enabled: true`. This uses the official Immich-optimized Postgres image with vector support.
  * Alternatively, you can point to an existing Postgres instance by setting the environment variables under `controllers.main.containers.main.env`.
* You need to set `image.tag` to the version you want to use, as this chart does not update with every Immich release.

# Configuration

The immich chart is highly customizable. You can see a detailed documentation
of all possible changes within the `charts/immich/values.yaml` file. Anything not covered there can be done by making direct use of the underlying common library chart (see below).

## Chart architecture 

This chart uses the [common library](https://github.com/bjw-s-labs/helm-charts/tree/common-4.3.0/charts/library/common). 
- **Global Settings:** Top level keys like `controllers` are applied to every component of the Immich stack (server, machine-learning, microservices, valkey, postgres).
- **Component-Specific Settings:** Entries under keys like `server`, `microservices`, `machine-learning`, `valkey`, and `postgres` define specific values for each component. These will override or merge with the global settings.

### Microservices Separation

By default, recent Immich versions combine the server and microservices logic. However, for better resource management and process priority (e.g., preventing the web server from hanging during heavy transcoding), you can split them:

```yaml
microservices:
  enabled: true
  controllers:
    main:
      pod:
        priorityClassName: <your-priority-class>
      containers:
        main:
          resources:
            requests:
              cpu: 200m
              memory: 1Gi
            limits:
              memory: 4Gi
```

> [!TIP]
> Setting a lower pod priority for `machine-learning` and `microservices` components can prevent them from "choking" the web server or other critical pods when the system is under heavy upload or transcoding load.

### Component Resources and Priorities

Best practice is to define resources and priorities per component to ensure they are correctly applied to the specific deployment:

```yaml
server:
  controllers:
    main:
      pod:
        priorityClassName: <high-priority-class>
      containers:
        main:
          resources:
            requests:
              cpu: 200m
              memory: 512Mi

machine-learning:
  controllers:
    main:
      pod:
        priorityClassName: <low-priority-class>
      containers:
        main:
          resources:
            limits:
              memory: 4Gi
```

## Uninstalling the Chart

To see the currently installed Immich chart:

```console
helm ls --namespace immich
```

To uninstall/delete the `immich` chart:

```console
helm delete --namespace immich immich
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

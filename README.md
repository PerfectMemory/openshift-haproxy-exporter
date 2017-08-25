# openshift haproxy-exporter

This project patches the `prom/haproxy-exporter:v0.8.0` docker image so that
the image can be used natively with the Openshift router addon without having
to set the docker image entrypoint or command arguments.

## How to use

As the openshift cluster administrator, run:

```sh
oc adm router -n default --expose-metrics --metrics-image=jperville/openshift-haproxy-exporter:v0.8.0
```

Then, once the Openshift router service is available, run:

```sh
oc annotate svc router -n default prometheus.io/scrape=true prometheus.io/port=9101 --overwrite
```

## Why?

Exposing haproxy metrics with Openshift is documented [here](https://docs.openshift.com/container-platform/3.5/install_config/router/default_haproxy_router.html#exposing-the-router-metrics);
however, following the above instructions result does not result in the exporter scraping any haproxy metrics.

The reason is that the generated sidecar container is started without argument (using the default entrypoint),
having no way to know how to access the haproxy statistics (which are password protected). The information to
access the haproxy instance is exposed through instance variables in the container, but the `/bin/haproxy_exporter` entrypoint
does not know about those instance variables.

To use `prom/haproxy-exporter` image with the openshift router, the `router` deploymentconfig has to be edited
to provide the `metrics-exporter` container with an explicit `command` which looks like this:

```yaml
      - name: metrics-exporter
        command:
        - /bin/sh
        - -c
        - exec /bin/haproxy_exporter --haproxy.scrape-uri "http://${STATS_USERNAME}:${STATS_PASSWORD}@localhost:${STATS_PORT}/;csv"
```

This project simply wraps the `prom/haproxy-exporter` docker image to provide an openshift-compatible entrypoint, removing
the need to edit the deploymentconfig by hand.

# KYacht
Kustom Yacht

KYacht is a kubernetes (k8s) templating library. It is very similar to kustomize/helm. 

## Purpose
The purpose of KYacht is to help ease the pain of verbose and many k8s files. 
KYacht will take a single service definition file, and apply all the values to a template. 

The templating engine uses jinja, the same template engine used in ansible/salt-stack. 

KYacht uses a single file to create many other components, allowing all components to share values if needed.

## FAQ

* Why not just use Helm/Kustomize/Other?
This library was built with frustration of using helm/kustomize. Helm has a pretty high barrier to entry, and personal preference felt that it was overly verbose. 
Kustomize is decent, but requires a *lot* of boilerplate and files to do simple things. 
Both of these two tools are designed to be "generic kubernetes templates" for anything in kubernetes. KYacht however, is designed specifically as a templating framework for services to be deployed. 

* Why is this library not written in Go?
Jinja is a python library. 


## Example

The following is an example values.yaml

```yaml
service_name: dataset-service

migration_command: ['alembic', 'upgrade', 'head']

components:
    - name: service
      template: web
      values:
        default:
            heartbeat_url: /healthz
            readiness_url: /readyz
            port: 80
            env:
                TIER: web
        dev:
            min: 1
            max: 2
        prod:
            min: 3
            max: 4
    - name: worker
      template: worker
      command: ['tini', 'python', '/app/worker.py']
      values:
        default:
            env:
                TIER: worker
        dev:
            min: 1
            max: 2
        prod:
            min: 4
            max: 10
    - name: cron
      template: worker
      command: ['tini', 'python', '/app/cron.py']
      values: 
        default:
            min: 1
            max: 1
            env:
                RABBIT_EXCHANGE: ds-cron

values: # defaults
    default:
       env: 
           SOME_ENV: somevalue
           OTHER_ENV: somevalue
    dev:
        env:
            TOKEN: abc
    prod:
        env:
            TOKEN: cba

```

will create 6 different k8s files, `service-dev.yaml`, `service-prod.yaml`, `worker-dev.yaml`, `worker-prod.yaml`, `cron-dev.yaml`, and `cron-prod.yaml`. The output of these files is based on the input template. You can see the template being named in the components section. An example `web` template could be:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: {{ _name }}
    tier: web
    service: {{ service_name }}
  name: {{ _name }}
spec:
  replicas: {{ scale | default(1) }}
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: {{ _name }}
      tier: web
      service: {{ service_name }}
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: {{ _name }}
        tier: web
        service: {{ service_name }}
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - {{ _name }}
                topologyKey: kubernetes.io/hostname
              weight: 100
      containers:
      - name: {{ _name }}
        image: gcr.io/hivewire/{{ service_name }}:{{ _tag }}
        env:
        {%- for k, v in _env.items() %}
          - name: {{ k }}
            value: {{ v }}
        {%- endfor %}
        imagePullPolicy: Always
        lifecycle:
          preStop:
            exec:
              command:
              - sleep
              - "10"
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: {{ liveness_url | default(heartbeat_url, '/heartbeat') }}
            port: {{ port | default(8080) }}
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 10
        {%- for line in container_custom | yaml_encode %}
        {{ line }}
        {%- endfor %}
        ports:
        - containerPort: {{ port | default(8080) }}
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: {{ readiness_url | default(heartbeat_url, '/heartbeat') }}
            port: {{ port | default(8080) }}
            scheme: HTTP
          initialDelaySeconds: 20
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 10
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      {%- for line in pod_custom | yaml_encode %}
      {{ line }}
      {%- endfor %}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30

---

apiVersion: v1
kind: Service
metadata:
  name: {{ service_name }}
spec:
  type: NodePort
  selector:
    app: {{ service_name }}
  ports:
    - protocol: TCP
      port: 80
      targetPort: {{ port }}

```

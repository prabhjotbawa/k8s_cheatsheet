# k8s_cheatsheet

### Adding a customer prometheus rule to fire an alert when PVC is at 30% capacity
Source: https://developers.redhat.com/articles/2023/10/03/how-configure-openshift-application-monitoring-and-alerts?source=sso#basic_concepts

Sample PVC resource
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  annotations:
    pv.kubernetes.io/bind-completed: 'yes'
    pv.kubernetes.io/bound-by-controller: 'yes'
    volume.beta.kubernetes.io/storage-provisioner: rook-ceph.rbd.csi.ceph.com
    volume.kubernetes.io/storage-provisioner: rook-ceph.rbd.csi.ceph.com
  name: postgres-13-awx-postgres-13-0
  namespace: awx
  labels:
    app.kubernetes.io/component: database
    app.kubernetes.io/instance: postgres-13-awx
    app.kubernetes.io/managed-by: awx-operator
    app.kubernetes.io/name: postgres-13
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 16Gi
  storageClassName: rook-ceph-block
  volumeMode: Filesystem
status:
  phase: Bound
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 16Gi

```

1. Enable user workload monitoring, configurations, and alert manager

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    prometheusK8s:
      retention: 48h                             -  Additional configurations for retention size
      retentionSize: 10GB
    enableUserWorkload: true                     -  Enabling user workload monitoring
    alertmanagerMain:
      enableUserAlertmanagerConfig: true         -  Enable deployed alert manager for user workloads
```

During the execution of the above ConfigMap, the Prometheus, Thanos ruler, and other pods will be deployed in the openshift-user-workload-monitoring project automatically.

Deploying a service monitor is a good practice to tell prometheus to scrape project metrics but is not needed to fire alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  creationTimestamp: '2024-02-04T02:41:22Z'
  generation: 1
  labels:
    app: awx
  name: awx-example-monitor
  namespace: awx
spec:
  endpoints:
    - interval: 30s
      port: web
      scheme: http
  selector:
    matchLabels:
      app: awx
```

2. Create a Rule, An example below:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
  name: example-alert-pvc
  namespace: awx # namespace of the app
spec:
  groups:
    - name: awx
      rules:
        - alert: pvclimithit
          annotations:
            description: AWX PVC - Expand
            message: 'PVC is at {{ $value }}' # This corresponds to the value of the expr.
            summary: PVC has reached half its capacity
          expr: >-
            (kubelet_volume_stats_used_bytes{job="kubelet", namespace="awx",
            persistentvolumeclaim="postgres-13-awx-postgres-13-0"} /
            kubelet_volume_stats_capacity_bytes)>0.3
          labels:
            app: awx # Must match lables of the service
            severity: critical
```

3. Alerts can be seen as below:
<img width="1589" alt="Screenshot 2024-02-03 at 10 31 54 PM" src="https://github.com/prabhjotbawa/k8s_cheatsheet/assets/42355705/13ab69dd-15c5-451a-86a3-16a3f3f88b24">

<img width="1621" alt="Screenshot 2024-02-03 at 10 50 45 PM" src="https://github.com/prabhjotbawa/k8s_cheatsheet/assets/42355705/68214919-45fe-4a5f-933b-85fd2c846376">

4. Create the Alertmanager configuration to define the external notification system for receiving the alerts created in OpenShift and visible in the OpenShift UI.

In the following example, the email notification system has been configured. We used the Gmail SMTP server to send notifications to certain email IDs.

Note: This resource can be created by cluster-admin or cluster administrator and assigned alert-routing-edit role to the user or group responsible for monitoringdemo project.
```yaml
apiVersion: monitoring.coreos.com/v1beta1
kind: AlertmanagerConfig
metadata:
  name: alert-notifications
  namespace: awx
  labels:
    alertmanagerConfig: main
spec:
  route:
    receiver: mail
    groupby: [job]
    groupInterval: 5m # Interval between the notifications
    groupWait: 30s # How long to wait on sending updated notification
    repeatInterval: 2h # When to send a reperat notification
  receivers:
  - name: mail
    emailConfigs:
    - to: xxxxxxxxx				              - This can be any email address or group email ID
      from: xxxxxxxx
      smarthost: smtp.gmail.com:587		      - Gmail SMTP server details
      hello: smtp.gmail.com
      authUsername: xxxxxxxx 			      - Gmail ID as the authentication username
      authPassword:
        name: mail-password
        key: password
---
apiVersion: v1
kind: Secret
metadata:
  name: mail-password
  namespace: awx
stringData:
  password: xxxxxxx 				          -  Need to create application password in gmail
```


An example

```yaml
kind: AlertmanagerConfig
metadata:
  creationTimestamp: '2024-02-05T00:16:47Z'
  generation: 4
  labels:
    alertmanagerConfig: main
  managedFields:
    - apiVersion: monitoring.coreos.com/v1beta1
      fieldsType: FieldsV1
      fieldsV1:
        'f:metadata':
          'f:labels':
            .: {}
            'f:alertmanagerConfig': {}
        'f:spec':
          .: {}
          'f:route':
            .: {}
            'f:receiver': {}
      manager: Mozilla
      operation: Update
      time: '2024-02-05T00:16:47Z'
    - apiVersion: monitoring.coreos.com/v1alpha1
      fieldsType: FieldsV1
      fieldsV1:
        'f:spec':
          'f:receivers': {}
          'f:route':
            'f:groupInterval': {}
            'f:groupWait': {}
            'f:repeatInterval': {}
      manager: Mozilla
      operation: Update
      time: '2024-02-05T00:51:46Z'
  name: alert-notifications
  namespace: awx
  resourceVersion: '396111984'
  uid: 298f0409-8f43-4edc-b11c-badd22d2c483
spec:
  receivers:
    - emailConfigs:
        - authPassword:
            key: password
            name: mail-password
          authUsername: prabhjotbawa@gmail.com
          from: prabhjotbawa@gmail.com
          hello: smtp.gmail.com
          smarthost: 'smtp.gmail.com:587'
          to: prabhjotbawa@gmail.com
      name: mail
  route:
    groupInterval: 1m
    groupWait: 30s
    receiver: mail
    repeatInterval: 30s
```
If you are using gmail, an app paaword can be set up to configure the email server but it's not secure. Instructions: https://support.google.com/mail/answer/185833?hl=en#zippy=%2Cwhy-you-may-need-an-app-password%2Cremove-app-passwords



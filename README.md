# k8s-wait-for

A simple script that allows to wait for a k8s service, job or pods to enter desired state.

## Using

Please consult `wait_for.sh -h` for detailed documentation.

## Example

A complex Kubernetes deployment manifest (generated by [helm](https://github.com/kubernetes/helm)), which uses old json syntax for init container declaration. This deployment wait for one job to finish and 2 pods to enter ready state.

~~~~
---
# Source: cross-support-job-3p/charts/onedata-3p/charts/oneprovider-krakow/templates/deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: develop-oneprovider-krakow
  labels:
    app: develop-oneprovider-krakow
    chart: oneprovider-krakow
    release: develop
    heritage: Tiller
  annotations:
    version: "0.2.7"
spec:
  replicas: 
  template:
    metadata:
      labels:
        app: develop-oneprovider-krakow
        chart: "oneprovider-krakow"
        release: "develop"
        heritage: "Tiller"
      annotations:
        version: "0.2.7"
      annotations:
        pod.beta.kubernetes.io/init-containers: '[
          {
              "name": "wait-for-volume-s3-init",
              "image": "groundnuty/k8s-wait-for:0.1",
              "imagePullPolicy": "Always",
              "imagePullSecrets": [],
              "args": [
                  "job", "develop-volume-s3-krakow-init"
              ]
          },
          {
              "name": "wait-for-volume-ceph",
              "image": "groundnuty/k8s-wait-for:0.1",
              "imagePullPolicy": "Always",
              "imagePullSecrets": [],
              "args": [
                  "pod", "-lapp=develop-volume-ceph-krakow"
              ]
          },
          {
              "name": "wait-for-volume-gluster",
              "image": "groundnuty/k8s-wait-for:0.1",
              "imagePullPolicy": "Always",
              "imagePullSecrets": [],
              "args": [
                  "pod", "-lapp=develop-volume-gluster-krakow"
              ]
          }]'
    spec:
      hostname: node1
      subdomain: develop-oneprovider-krakow
      imagePullSecrets: []
      volumes:
      - name: nfs
        persistentVolumeClaim:
          claimName: develop-oneprovider-krakow-nfs-pvc
      containers:
      - name: oneprovider-krakow
        image: docker.onedata.org/oneprovider:ID-75d8645bec
        imagePullPolicy: Always
        resources:
          requests:
            cpu: 2
            memory: 4Gi
        ports:
          - containerPort: 53
          - containerPort: 80
          - containerPort: 443
          - containerPort: 5555
          - containerPort: 5556
          - containerPort: 6665
          - containerPort: 6666
          - containerPort: 7443
          - containerPort: 8443
          - containerPort: 8876
          - containerPort: 8877
          - containerPort: 9443
        lifecycle:
          preStop:
            exec:
              command:
                - "sh"
                - "-c"
                - >
                  op_panel stop ;
                  op_worker stop ;
                  cluster_manager stop ;
                  /etc/init.d/couchbase-server stop ;
                  pkill -f oneprovider.py ;
        readinessProbe:
          exec:
            command:
              - "/bin/bash"
              - "-c"
              - >
                service=provider ;
                user="$(curl -k -u a:b -sS --tlsv1.2 -X GET 'https://localhost:9443/api/v3/onepanel/users/user')";
                if [[ "$user" == "" ]] ; then exit 1; else  exit 0;  fi ;
        env:
          - name: ONEPANEL_LOG_LEVEL
            value: "info"
          - name: ONEPANEL_BATCH_MODE
            value: "true"
          - name: ONEPROVIDER_CONFIG
            value: |
              cluster:
                domainName: "develop-oneprovider-krakow.develop.svc.dev.onedata.uk.to"
                nodes:
                  n1:
                    hostname: node1
                managers:
                  mainNode: n1
                  nodes:
                    - n1
                workers:
                  nodes:
                    - n1
                databases:
                  nodes:
                    - n1
                storages:
                  posix:
                    type: posix
                    mountPoint: /volumes/storage
                  nfs:
                    type: posix
                    mountPoint: /volumes/nfs
                  s3:
                    type: s3
                    hostname: develop-volume-s3-krakow:8000
                    bucketName: test
                    accessKey: accessKey
                    secretKey: verySecretKey
                    insecure: true
                  ceph:
                    type: ceph
                    username: client.k8s
                    key: A
                    monitorHostname: develop-volume-ceph-krakow
                    clusterName: ceph
                    poolName: test
                  gluster:
                    type: glusterfs
                    hostname: develop-volume-gluster-krakow
                    volume: test
                    transport: tcp
              oneprovider:
                register: true
                name: develop-oneprovider-krakow
                redirectionPoint: https://develop-oneprovider-krakow.develop.svc.dev.onedata.uk.to
                geoLatitude: 50.0647
                geoLongitude: 19.945
              # TODO: make it possible for onedata services to communicate using 
              # system configured DNS. this will allow to put here just service name
              # instead of FQDN
              onezone:
                domainName: develop-onezone.develop.svc.dev.onedata.uk.to
              onepanel:
                users:
                  admin:
                    password: a
                    userRole: a
                  user:
                    password: a
                    userRole: a
        volumeMounts:
          - mountPath: /volumes/nfs
            name: nfs
~~~~

## Complex deployment use case

This container is used extensively in deployments of Onedata system [onedata/charts](https://github.com/onedata/charts) for the the purpose of specifying dependencies. It leverages Kubernetes [init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/), thus providing:

- a detailed event log in `kubectl describe <pod>`, on what init container is pod hanging at the moment.
- a comprehensive view in `kubectl get pods` output where init containers are shown in a form `Init:<ready>/<total>`

Example output from the deployment run of ~16 pod with dependencies just after deployment:

~~~bash
NAME                                                   READY     STATUS              RESTARTS   AGE
develop-cross-support-job-3p-krk-3-lis-c-b4nv1         0/1       Init:0/1            0          11s
develop-cross-support-job-3p-krk-3-par-c-lis-n-z7x6w   0/1       Init:0/1            0          11s
develop-cross-support-job-3p-krk-3-x9719               0/1       Init:0/1            0          11s
develop-cross-support-job-3p-krk-g-par-3-ztvz0         0/1       Init:0/1            0          11s
develop-cross-support-job-3p-krk-g-v5lf2               0/1       Init:0/1            0          11s
develop-cross-support-job-3p-krk-n-par-3-pnbcm         0/1       Init:0/1            0          11s
develop-cross-support-job-3p-lis-3-cpj3f               0/1       Init:0/1            0          11s
develop-cross-support-job-3p-par-n-8zdt2               0/1       Init:0/1            0          11s
develop-cross-support-job-3p-par-n-lis-c-kqdf0         0/1       Init:0/1            0          11s
develop-oneclient-krakow-2773392814-wc1dv              0/1       Init:0/3            0          11s
develop-oneclient-lisbon-3267879054-2v6cg              0/1       Init:0/3            0          9s
develop-oneclient-paris-2076479302-f6hh9               0/1       Init:0/3            0          9s
develop-onedata-cli-krakow-1801798075-b5wpj            0/1       Init:0/1            0          11s
develop-onedata-cli-lisbon-139116355-fwtjv             0/1       Init:0/1            0          10s
develop-onedata-cli-paris-2662312307-9z9l1             0/1       Init:0/1            0          11s
develop-oneprovider-krakow-3634465102-tftc6            0/1       Pending             0          10s
develop-oneprovider-lisbon-3034775369-8n31x            0/1       Init:0/3            0          8s
develop-oneprovider-paris-3034358951-19mhf             0/1       Init:0/3            0          10s
develop-onezone-304145816-dmxn1                        0/1       ContainerCreating   0          11s
develop-volume-ceph-krakow-479580114-mkd1d             0/1       ContainerCreating   0          11s
develop-volume-ceph-lisbon-1249181958-1f0mt            0/1       ContainerCreating   0          9s
develop-volume-ceph-paris-400443052-dc347              0/1       ContainerCreating   0          9s
develop-volume-gluster-krakow-761992225-sj06m          0/1       Running             0          11s
develop-volume-gluster-lisbon-3947152141-jlmvb         0/1       Running             0          8s
develop-volume-gluster-paris-3588749681-9bnw8          0/1       ContainerCreating   0          11s
develop-volume-nfs-krakow-2528947555-6mxzt             1/1       Running             0          10s
develop-volume-nfs-lisbon-3473018547-7nljf             0/1       ContainerCreating   0          11s
develop-volume-nfs-paris-2956540513-4bdzt              0/1       ContainerCreating   0          11s
develop-volume-s3-krakow-23786741-pdxtj                0/1       Running             0          9s
develop-volume-s3-krakow-init-gqmmp                    0/1       Init:0/1            0          11s
develop-volume-s3-lisbon-3912793669-d4xh5              0/1       Running             0          10s
develop-volume-s3-lisbon-init-mq9nk                    0/1       Init:0/1            0          11s
develop-volume-s3-paris-124394749-qwt18                0/1       Running             0          8s
develop-volume-s3-paris-init-jb4k3                     0/1       Init:0/1            0          11s
~~~

1 min after, you can see the changes in the *Status* column:

~~~bash
develop-cross-support-job-3p-krk-3-lis-c-b4nv1         0/1       Init:0/1          0          1m
develop-cross-support-job-3p-krk-3-par-c-lis-n-z7x6w   0/1       Init:0/1          0          1m
develop-cross-support-job-3p-krk-3-x9719               0/1       Init:0/1          0          1m
develop-cross-support-job-3p-krk-g-par-3-ztvz0         0/1       Init:0/1          0          1m
develop-cross-support-job-3p-krk-g-v5lf2               0/1       Init:0/1          0          1m
develop-cross-support-job-3p-krk-n-par-3-pnbcm         0/1       Init:0/1          0          1m
develop-cross-support-job-3p-lis-3-cpj3f               0/1       Init:0/1          0          1m
develop-cross-support-job-3p-par-n-8zdt2               0/1       Init:0/1          0          1m
develop-cross-support-job-3p-par-n-lis-c-kqdf0         0/1       Init:0/1          0          1m
develop-oneclient-krakow-2773392814-wc1dv              0/1       Init:0/3          0          1m
develop-oneclient-lisbon-3267879054-2v6cg              0/1       Init:0/3          0          58s
develop-oneclient-paris-2076479302-f6hh9               0/1       Init:0/3          0          58s
develop-onedata-cli-krakow-1801798075-b5wpj            0/1       Init:0/1          0          1m
develop-onedata-cli-lisbon-139116355-fwtjv             0/1       Init:0/1          0          59s
develop-onedata-cli-paris-2662312307-9z9l1             0/1       Init:0/1          0          1m
develop-oneprovider-krakow-3634465102-tftc6            0/1       Init:1/3          0          59s
develop-oneprovider-lisbon-3034775369-8n31x            0/1       Init:2/3          0          57s
develop-oneprovider-paris-3034358951-19mhf             0/1       PodInitializing   0          59s
develop-onezone-304145816-dmxn1                        0/1       Running           0          1m
develop-volume-ceph-krakow-479580114-mkd1d             1/1       Running           0          1m
develop-volume-ceph-lisbon-1249181958-1f0mt            1/1       Running           0          58s
develop-volume-ceph-paris-400443052-dc347              1/1       Running           0          58s
develop-volume-gluster-krakow-761992225-sj06m          1/1       Running           0          1m
develop-volume-gluster-lisbon-3947152141-jlmvb         1/1       Running           0          57s
develop-volume-gluster-paris-3588749681-9bnw8          1/1       Running           0          1m
develop-volume-nfs-krakow-2528947555-6mxzt             1/1       Running           0          59s
develop-volume-nfs-lisbon-3473018547-7nljf             1/1       Running           0          1m
develop-volume-nfs-paris-2956540513-4bdzt              1/1       Running           0          1m
develop-volume-s3-krakow-23786741-pdxtj                1/1       Running           0          58s
develop-volume-s3-lisbon-3912793669-d4xh5              1/1       Running           0          59s
develop-volume-s3-paris-124394749-qwt18                1/1       Running           0          57s
~~~

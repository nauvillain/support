## Spark Logs Cleanup (CP4D)

Public Support Page: https://wwwpoc.ibm.com/support/pages/node/6980928

We have seen a lot of Spark logs are generated from Spark Jobs/Kernels which can fill up PVC. Not only this, since the number of log files are also increased in that case K8s takes long time to create the container and can easily timeout by giving the message "Context deadline exceeded". The Spark pod will then get stuck in `CreateContainerErr` state.

We can manually clean the logs or make use of cronjobs to do so. Here are the cronjob for most of the usecases -

## Get Spark Nginx Image

Run this command and get the spark-hb-nginx image copied as it is required in the next steps -
```
oc get deploy spark-hb-nginx -o=jsonpath='{$.spec.template.spec.containers[:1].image}'
```

### Manual cleanup using Pod with root and SeLinuxRelabelling disabled

```
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-1
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: <spark-hb-nginx-image-here>
    imagePullPolicy: IfNotPresent
    securityContext:
      allowPrivilegeEscalation: false
      runAsUser: 0
      seLinuxOptions:
        type: "spc_t"
    volumeMounts:
      - mountPath: /mnt/asset_file_api
        name: file-api-pv
    args:
      - /bin/sh
      - -c
      - sleep infinity
  volumes:
      - name: file-api-pv
        persistentVolumeClaim:
          claimName: file-api-claim
EOF
```


### Cronjob to cleanup Spark jobs/kernels from Watson Studio Project

This cronjob runs every 48 hours and cleans the spark logs that are older than 2 days.
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: spark-project-logs-spark-events-cleanup
spec:
  schedule: "0 0 */2 * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: demo-clean
            image: <spark-nginx-image-here>
            args:
            - /bin/sh
            - -c
            - "find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type d -name 'logs' -mtime +2 -exec rm -rf {} + && find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type d -name 'spark-events' -mtime +2 -exec rm -rf {} + && find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type d -name '.cache' -mtime +2 -exec rm -rf {} + && find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type f -name 'javacore.*' -mtime +2 -exec rm -f {} + && find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type f -name 'heapdump.*' -mtime +2 -exec rm -f {} + && find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type f -name 'Snap.*' -mtime +2 -exec rm -f {} +"
            volumeMounts:
            - name: file-api-pv
              mountPath: /mnt/asset_file_api
          resources:
            limits:
              cpu: 400m
              memory: 512Mi
            requests:
              cpu: 200m
              memory: 256Mi
          restartPolicy: OnFailure
          volumes:
          - name: file-api-pv
            persistentVolumeClaim:
              claimName: file-api-claim

```

### Cronjob to cleanup Spark jobs/kernels from Watson Studio Git Project

This cronjob runs every 48 hours and cleans the spark logs that are older than 2 days.
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: spark-project-logs-spark-events-cleanup
spec:
  schedule: "0 0 */2 * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: demo-clean
            image: <spark-nginx-image-here>
            args:
            - /bin/sh
            - -c
            - "find /mnt/asset_file_api/projects/*/*/spark-runtimes/ -maxdepth 1 -mindepth 1 -type d -mtime +2"
            volumeMounts:
            - name: file-api-pv
              mountPath: /mnt/asset_file_api
          resources:
            limits:
              cpu: 400m
              memory: 512Mi
            requests:
              cpu: 200m
              memory: 256Mi
          restartPolicy: OnFailure
          volumes:
          - name: file-api-pv
            persistentVolumeClaim:
              claimName: file-api-claim

```

### Cronjob to cleanup Spark jobs created by Service Instances with Spaces enabled

This cronjob runs every 48 hours and cleans the spark logs that are older than 2 days.
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: spark-spaces-logs-cleanup
spec:
  schedule: "0 0 */2 * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: demo-clean
            image: <spark-nginx-image-here>
            args:
            - /bin/sh
            - -c
            - "find /mnt/asset_file_api/spaces/*/assets/runtimes/spark -maxdepth 1 -mindepth 1 -type d -mtime +2 -exec rm -rf {} +"
            volumeMounts:
            - name: file-api-pv
              mountPath: /mnt/asset_file_api
          resources:
            limits:
              cpu: 400m
              memory: 512Mi
            requests:
              cpu: 200m
              memory: 256Mi
          restartPolicy: OnFailure
          volumes:
          - name: file-api-pv
            persistentVolumeClaim:
              claimName: file-api-claim
```

### Cronjob to cleanup Spark jobs created by WKC (Profiling) Catalogs.

This cronjob runs every 48 hours and cleans the spark logs that are older than 2 days.
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: spark-wkc-logs-cleanup
spec:
  schedule: "0 0 */2 * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: demo-clean
            image: <spark-nginx-image-here>
            args:
            - /bin/sh
            - -c
            - "find /mnt/wkc_volume/<instance-id> -maxdepth 1 -mindepth 1 -type d -mtime +2 -exec rm -rf {} +"
            volumeMounts:
            - name: wkc_volume
              mountPath: /mnt/wkc_volume
          resources:
            limits:
              cpu: 400m
              memory: 512Mi
            requests:
              cpu: 200m
              memory: 256Mi
          restartPolicy: OnFailure
          volumes:
          - name: wkc_volume
            persistentVolumeClaim:
              claimName: volumes-profstgintrnl-pvc
```

### [CP4D 4.0] Cronjob to cleanup Spark jobs/kernels created by Watson Studio

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: spark-project-logs-spark-events-cleanup
spec:
  schedule: "0 0 */2 * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: demo-clean
            image: <spark-nginx-image-here>
            args:
            - /bin/sh
            - -c
            - "find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type d -name 'logs' -mtime +2 -exec rm -rf {} + && find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type d -name 'spark-events' -mtime +2 -exec rm -rf {} + && find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type d -name 'conda' -mtime +2 -exec rm -rf {} + && find /mnt/asset_file_api/projects/*/* -maxdepth 1 -mindepth 1 -type d -name 'user-libs' -mtime +2 -exec rm -rf {} +" 
            volumeMounts:
            - name: file-api-pv
              mountPath: /mnt/asset_file_api
          resources:
            limits:
              cpu: 400m
              memory: 512Mi
            requests:
              cpu: 200m
              memory: 256Mi
          restartPolicy: OnFailure
          volumes:
          - name: file-api-pv
            persistentVolumeClaim:
              claimName: file-api-claim

```

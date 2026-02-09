# Goal 

Create a kubernetes cron job that capture the kasten log every 3 hours, and store them with a timestamp

# Detailed implementation 

With a cluster admin role you can capture the kasten logs using this command 
```
VERSION=8.5.1
curl -s https://docs.kasten.io/downloads/$VERSION/tools/k10_debug.sh | bash;
```

By default, the debug script will generate a compressed archive file k10_debug_logs.tar.gz which will have separate logs files for Veeam Kasten services.

The cron will rename the file by adding a date prefix with the format `date +"%Y-%m-%d-%H-%M"`

We throw an error if the process do not execute under 30 minutes.

All the logs are stored in a 20Gb PVC which is enough to hold more than 500 tar file (each tar file is usually around 3Mb to 10Mb), but you can change the size if you need more retention.

The cron will run every 3 hours so that even on very busy cluster logs will be captured before they roll over. 

When the size on the PVC reach the 90% the cron will always delete the oldest tar file before creating the new one.

# Deployment 

create the namespace and a config map that contains the script adapt `VERSION` to your kasten version and `NAMESPACE` if you want to use another namespace name.
```
VERSION=8.5.1
NAMESPACE=kasten-cron-logs

kubectl create ns $NAMESPACE
wget https://docs.kasten.io/downloads/$VERSION/tools/k10_debug.sh
kubectl create configmap k10-debug-sh -n $NAMESPACE --from-file k10_debug.sh=k10_debug.sh
```

Create a pvc, adapt `SIZE` and `STORAGE_CLASS` to your needs 
```
SIZE=20Gi
STORAGE_CLASS=ocs-storagecluster-ceph-rbd

cat<<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kasten-logs
  namespace: $NAMESPACE
spec: 
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: $SIZE
  storageClassName: $STORAGE_CLASS
EOF
```


Deploy the rest with:

```
kubectl create -n $NAMESPACE -f kasten-cron-logs.yaml
```

The kasten-cron-logs.yaml will create 
- the service account with the cluster-admin role, 
- the cronjob itself which is based on an image that has tar, bash and kubectl and that is publically available on docker hub.

# Test it 

```
kubectl create job -n $NAMESPACE kasten-cron-logs-manual --from=cronjob/kasten-cron-logs
```

# Retrieve the logs 

Once the job is finished you need to create a pod that will attach to the PVC so you can list and copy the tar files locally.

Create a debug pod with the PVC mounted:
```
cat<<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  namespace: $NAMESPACE
spec:
  containers:
  - name: debug
    image: bitnami/kubectl:latest
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: kasten-logs
      mountPath: /kasten-logs
  volumes:
  - name: kasten-logs
    persistentVolumeClaim:
      claimName: kasten-logs
EOF

kubectl wait -n $NAMESPACE --for=condition=ready pod/debug-pod --timeout=60s
```

List the collected log files:
```
kubectl exec -n $NAMESPACE debug-pod -- ls -lh /kasten-logs
```

Copy a specific log file to your local machine:
```
kubectl cp $NAMESPACE/debug-pod:/kasten-logs/2026-02-09-15-43-k10_debug_logs.tar.gz ./2026-02-09-15-43-k10_debug_logs.tar.gz
```

Or copy all log files:
```
for file in $(kubectl exec -n $NAMESPACE debug-pod -- sh -c 'ls /kasten-logs/*.tar.gz 2>/dev/null'); do
  filename=$(basename $file)
  kubectl cp $NAMESPACE/debug-pod:/kasten-logs/$filename ./$filename
done
```

Delete the debug pod when done:
```
kubectl delete pod -n $NAMESPACE debug-pod
```





# pvc-inspector-k8s

## Introduction
PVC-Inspector is a deployment that can be used in a Kubernetes environment to access PVCs that have already been registered for or by some other artifact.

This allows us, in case the artifact using this PVC has errors and we need to access the PVC data for any reason, to do so using this small and simple deployment that we have in this repository.

## Example
Let's suppose we have an active namespace in a Kubernetes environment where there already exists a PVC named svc-claim0-pvc. The artifact using this volume is encountering some error or has been deleted. However, we still need to access the data that is already stored in this volume.

### PVC example

<pre><code>
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: svc-claim0-pvc
    namespace: namespace # this is a false name
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 300Mi
</code></pre>

### Pvc-Inspector example

In order to access this previously defined PVC, we would need to modify and launch the pvc-inspector deployment located in this repository as follows:

<pre><code>
    kind: Deployment
apiVersion: apps/v1
metadata:
  name: pvc-inspector
  namespace: namespace # Change to a valid namespace 
spec:
  replicas: 0
  selector:
    matchLabels:
      app: pvc-inspector
  template:
    metadata:
      labels:
        app: pvc-inspector
    spec:
      volumes:
        - name: svc-claim0
          persistentVolumeClaim:
            claimName: svc-claim0-pvc
      containers:
        - name: pvc-inspector
          image: busybox
          command:
            - tail
          args:
            - '-f'
            - /dev/null
          resources:
            limits:
              cpu: 200m
              memory: 500Mi
            requests:
              cpu: 100m
              memory: 250Mi
          volumeMounts:
            - name: svc-claim0
              mountPath: /path/to/data/dir # Enter the mountPath yo want to access
          imagePullPolicy: Always
      restartPolicy: Always
</code></pre>

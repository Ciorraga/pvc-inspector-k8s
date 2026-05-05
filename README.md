# pvc-inspector-k8s

## Introduction

`pvc-inspector-k8s` is a small Kubernetes helper manifest intended to mount an existing `PersistentVolumeClaim` in a disposable container so its contents can be inspected when the original workload is unavailable, broken, or already removed.

The current approach uses a simple `Deployment` with `replicas: 0` by default. You customize the namespace, PVC name, and mount path, apply the manifest, and scale it up only when you need access.

## When This Is Useful

Typical cases:

- the application using the PVC is failing and you need to inspect files directly
- the original workload has been deleted but the PVC still exists
- you need a quick operational helper without rebuilding the original application image

## Important Notes

- The current manifest mounts the PVC as read-only.
- The current manifest uses a plain `busybox` container that stays alive with `tail -f /dev/null`.
- The current manifest is an operational helper, not a production workload.

This is the safest default for an inspection helper because it prevents accidental modifications to the mounted data.

## Example PVC

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: svc-claim0-pvc
  namespace: namespace
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 300Mi
```

## Inspector Manifest

The repository includes a base manifest in [pvc-inspector.yml](./pvc-inspector.yml).

You need to customize at least:

- `metadata.namespace`
- `spec.template.spec.volumes[].persistentVolumeClaim.claimName`
- `spec.template.spec.volumes[].name`
- `spec.template.spec.containers[].volumeMounts[].name`
- `spec.template.spec.containers[].volumeMounts[].mountPath`

## Usage Flow

### 1. Edit the manifest

Update [pvc-inspector.yml](./pvc-inspector.yml) with the target namespace, PVC name, and mount path.

### 2. Apply the manifest

```bash
kubectl apply -f pvc-inspector.yml
```

### 3. Start the inspector pod

The manifest is intentionally created with `replicas: 0`. Scale it up only when you need it:

```bash
kubectl scale deployment pvc-inspector -n <namespace> --replicas=1
```

### 4. Access the container

```bash
kubectl exec -it deployment/pvc-inspector -n <namespace> -- sh
```

Once inside the container, inspect the mounted files under the configured `mountPath`.

### 5. Stop it when finished

```bash
kubectl scale deployment pvc-inspector -n <namespace> --replicas=0
```

### 6. Delete the helper when you no longer need it

```bash
kubectl delete deployment pvc-inspector -n <namespace>
```

If you prefer deleting it from the original file:

```bash
kubectl delete -f pvc-inspector.yml
```

## Quick Publish And Removal Guide

### Publish it

```bash
kubectl apply -f pvc-inspector.yml
kubectl scale deployment pvc-inspector -n <namespace> --replicas=1
```

### Use it

```bash
kubectl exec -it deployment/pvc-inspector -n <namespace> -- sh
```

### Remove it afterwards

```bash
kubectl scale deployment pvc-inspector -n <namespace> --replicas=0
kubectl delete deployment pvc-inspector -n <namespace>
```

## Example Manifest

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: pvc-inspector
  namespace: namespace
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
            - "-f"
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
              mountPath: /path/to/data/dir
              readOnly: true
          imagePullPolicy: Always
      restartPolicy: Always
```

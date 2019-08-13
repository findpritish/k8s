# Persistent Volume Exception Troubleshooting

This chapter describes troubleshooting methods for persistent storage exceptions (PV, PVC, StorageClass, etc.).

In general, you can execute the `kubectl describe pv/pvc <pod-name>` command to see the current PV event, regardless of the abnormal state of the PV. These events often help to troubleshoot problems with PV or PVC.

```sh
kubectl get pv
kubectl get pvc
kubectl get sc

kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name>
kubectl describe sc <storage-class-name>
```

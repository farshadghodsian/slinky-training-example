# Example Slinky Training Workload on Kubernetes

## Clone this repo

```bash
git clone https://github.com/farshadghodsian/slinky-training-example.git
```

## Installing Slinky Prerequisites

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
	--namespace cert-manager --create-namespace --set crds.enabled=true
helm install prometheus prometheus-community/kube-prometheus-stack \
	--namespace prometheus --create-namespace --set installCRDs=true
```

## Installing Slinky Operator

```bash
helm install slurm-operator oci://ghcr.io/slinkyproject/charts/slurm-operator \
  --values=values-operator.yaml --version=0.1.0 --namespace=slinky --create-namespace
```

Make sure the operator deployed successfully with:

```sh
kubectl --namespace=slinky get pods
```

Output should be similar to:

```sh
NAME                                      READY   STATUS    RESTARTS   AGE
slurm-operator-7444c844d5-dpr5h           1/1     Running   0          5m00s
slurm-operator-webhook-6fd8d7857d-zcvqh   1/1     Running   0          5m00s
```

## Installing Slurm Cluster

```bash
helm install slurm oci://ghcr.io/slinkyproject/charts/slurm \
  --values=values-slurm.yaml --version=0.1.0 --namespace=slurm --create-namespace
```

Make sure the Slurm cluster deployed successfully with:

```sh
kubectl --namespace=slurm get pods
```

Output should be similar to:

```sh
NAME                              READY   STATUS    RESTARTS       AGE
slurm-accounting-0                1/1     Running   0              5m00s
slurm-compute-gpu-node            1/1     Running   0              5m00s
slurm-controller-0                2/2     Running   0              5m00s
slurm-exporter-7b44b6d856-d86q5   1/1     Running   0              5m00s
slurm-mariadb-0                   1/1     Running   0              5m00s
slurm-restapi-5f75db85d9-67gpl    1/1     Running   0              5m00s
```

## Prepping Compute Node

1. Get SLURM Compute Node Name

    ```bash
    SLURM_COMPUTE_POD=$(kubectl get pods -n slurm | grep ^slurm-compute-gpu-node | awk '{print $1}');echo $SLURM_COMPUTE_POD
    ```

2. Add Slurm user to video and render group and create Slurm user home directory to Slrum Compute node

    ```bash
    kubectl exec -it -n slurm $SLURM_COMPUTE_POD -- bash -c "
        usermod -aG video,render slurm
        mkdir -p /home/slurm
        chown slurm:slurm /home/slurm"
    ```

3. Copy PyTorch test script to Slurm compute node

    ```bash
    kubectl cp test.py slurm/$SLURM_COMPUTE_POD:/tmp/test.py 
    ```

4. Copy Fashion MNIST Image Classification Model Training script to Slurm compute node

    ```bash
    kubectl cp train_fashion_mnist.py slurm/$SLURM_COMPUTE_POD:/tmp/train_fashion_mnist.py 
    ```

5. Run test.py script on compute node to confirm GPUs are accessible

    ```bash
    kubectl exec -it slurm-controller-0 -n slurm --  srun python3 test.py
    ```

6. Run single-GPU training script on compute node

    ```bash
    kubectl exec -it slurm-controller-0 -n slurm --  srun python3 train_fashion_mnist.py
    ```

7. Run multi-GPU training script on compute node

    ```bash
    kubectl exec -it slurm-controller-0 -n slurm --  srun apptainer exec --rocm --bind /tmp:/tmp torch_rocm.sif torchrun --standalone --nnodes=1 --nproc_per_node=8 --master-addr localhost train_mnist_distributed.py
    ```

## Other Useful Slurm Commands

### Check Slurm Node Info

```bash
kubectl exec -it slurm-controller-0 -n slurm --  sinfo
```

### Check Job Queue

```bash
kubectl exec -it slurm-controller-0 -n slurm --  squeue
```

### Check Node Resources

```bash
kubectl exec -it slurm-controller-0 -n slurm -- sinfo -N -o "%N %G"
```

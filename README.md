
# ACR Pull Performance Investigation (`acr-pull-perf`)

## Objective

This report looks into the performance of pulling container images from Azure Container Registry (ACR) under different configurations and tools using a controlled Azure environment. The focus is on measuring image pull duration, evaluating system disk vs RAM disk performance during extraction, and identifying bottlenecks in real-world usage scenarios.

For all of these, I will be using Virtual Machine SKUs with Accelerated Networking enabled.

## Tool and Environment Versions

| Component        | Version / Info                                                                 |
|------------------|---------------------------------------------------------------------------------|
| **OS**           | Ubuntu 22.04.5 LTS (Jammy)                                                      |
| **Kernel/Driver**| Mellanox ConnectX-4 Lx VF (`MT27710`, rev 80)                                   |
| **Podman**       | 3.4.4 (Go 1.18.1)                                                               |
| **Docker (Client)** | 28.0.4 (API 1.48, Go 1.23.7, Git b8034c0)                                  |
| **Docker (Server)** | 28.0.4 (containerd 1.7.27, runc 1.2.5, init 0.19.0)                         |
| **Buildah**      | 1.23.1 (Go 1.17, Image Spec 1.0.1, CNI 0.4.0)                                   |

## Environment Setup

### Azure Resources

```bash
BASENAME="acr-pull-perf"
RESOURCE_GROUP="rg-${BASENAME}"
LOCATION="eastus2"
ACR_NAME="acrpullperf$RANDOM"
VM_NAME="ubuntuVM-${BASENAME}"
VNET_NAME="vnet-${BASENAME}"
SUBNET_NAME="sn-${BASENAME}"
IP_NAME="pip-${BASENAME}"
NIC_NAME="nic-${BASENAME}"
USERNAME="azureuser"
SSH_KEY="$HOME/.ssh/id_rsa.pub"
```

### Resource Provisioning

```bash
az group create --name $RESOURCE_GROUP --location $LOCATION

az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Premium --location $LOCATION

az network vnet create \
  --resource-group $RESOURCE_GROUP \
  --name $VNET_NAME \
  --subnet-name $SUBNET_NAME \
  --location $LOCATION

az network public-ip create --resource-group $RESOURCE_GROUP --name $IP_NAME

az network nic create \
  --resource-group $RESOURCE_GROUP \
  --name $NIC_NAME \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --public-ip-address $IP_NAME \
  --accelerated-networking true \
  --location $LOCATION

az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --nics $NIC_NAME \
  --image Ubuntu2204 \
  --admin-username $USERNAME \
  --size Standard_D2s_v3 \
  --generate-ssh-keys \
  --location $LOCATION
```

### Connect to the VM

```bash
IP_ADDRESS=$(az vm show --name $VM_NAME --resource-group $RESOURCE_GROUP -d --query publicIps -o tsv)
ssh $USERNAME@$IP_ADDRESS
```

## Tool Installation

### Docker

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
sudo apt install docker-ce
```

Login using:

```bash
docker login acrpullperfXXXX.azurecr.io
```

### Podman

```bash
sudo apt-get -y install podman
```

## Image Import to ACR

```bash
az acr import --name $ACR_NAME \
  --source docker.io/tensorflow/tensorflow:nightly-jupyter \
  --image tensorflow:nightly-jupyter \
  --resource-group $RESOURCE_GROUP
```

## Image Pull Tests

```bash
time docker pull acrpullperf10464.azurecr.io/tensorflow:nightly-jupyter
time buildah pull acrpullperf10464.azurecr.io/tensorflow:nightly-jupyter
time podman pull acrpullperf10464.azurecr.io/tensorflow:nightly-jupyter
```

Image size: `tensorflow/tensorflow:nightly-jupyter` — **689.8 MB (compressed)**

| VM SKU                                   | Tool            | Source | Concurrent Downloads | Time (real) |
|------------------------------------------|------------------|--------|-----------------------|--------------|
| Standard_D2s_v3                          | Docker           | ACR    | 3                     | 0m53.650s    |
|                                          | Podman + tempfs  | ACR    | 3                     | 0m24.658s    |
|                                          | Podman           | ACR    | 3                     | 0m47.282s    |
| Standard_D16s_v3 (16 vCPU, 64 GiB RAM)   | Docker           | ACR    | 3                     | 0m53.040s    |
|                                          | Podman           | ACR    | 23                    | 0m25.415s    |
| Standard_D64s_v3 (64 vCPU, 256 GiB RAM)  | Podman           | ACR    | 27                    | 0m19.385s    |
|                                          | Docker (default) | ACR    | 10                    | 0m53.307s    |
|                                          | Buildah + tmpfs  | ACR    | 23                    | 0m18.353s    |
|                                          | Podman + tmpfs   | ACR    | 27                    | 0m18.377s    |
|                                          | Docker + tmpfs   | ACR    | 12                    | 0m26.726s    |

## Using `tmpfs` for Data Root

```bash
mkdir /mnt/ramdisk
mount -t tmpfs -o size=50G tmpfs /mnt/ramdisk
```

edit the `/etc/docker/daemon.json` to include the following:

```json
{
  "max-concurrent-uploads": 30,
  "max-concurrent-downloads": 30,
  "data-root": "/mnt/ramdisk"
}
```

then restart the docker daemon:

```bash
sudo service docker restart
```

## Private Endpoint Test

```bash
nslookup acrpullperf10464.azurecr.io
```

Sample output:

```
acrpullperf10464.azurecr.io     canonical name = acrpullperf10464.privatelink.azurecr.io.
acrpullperf10464.privatelink.azurecr.io canonical name = eus2.fe.azcr.io.
Address: 20.49.102.151
```

## Disk Performance Testing

### Docker Disk (Default)

```bash
fio --name=docker-disk --directory=/var/lib/docker --size=512M \
    --bs=4k --rw=randwrite --iodepth=16 --numjobs=4 \
    --time_based --runtime=10s --group_reporting > docker-disk.fio.txt
```

### RAM Disk

```bash
fio --name=ramdisk --directory=/mnt/ramdisk --size=512M \
    --bs=4k --rw=randwrite --iodepth=16 --numjobs=4 \
    --time_based --runtime=10s --group_reporting > ramdisk.fio.txt
```

Compare:
```
grep -E 'IOPS=|bw=' docker-disk.fio.txt
grep -E 'IOPS=|bw=' ramdisk.fio.txt
```

### I/O Performance Comparison

| Metric                  | Docker Disk              | RAM Disk                   |
|-------------------------|--------------------------|-----------------------------|
| Write IOPS              | 37.0k                    | 1.95 million (1953k)        |
| Write Bandwidth         | 144 MiB/s (151 MB/s)     | 7630 MiB/s (8001 MB/s)      |
| Total Data Written      | 2048 MiB (2 GiB)         | 74.5 GiB (80 GB)            |
| Test Duration           | ~14.2 seconds            | ~10.0 seconds               |

## Conclusions

1. **Podman and Buildah outperformed Docker in every scenario**. Across all VM sizes and configurations, Podman and Buildah consistently delivered faster pull times compared to Docker. The advantage became even more pronounced when concurrency was increased and temporary storage was optimized.

2. **Using a RAM disk for extraction dramatically improved performance**. When configured with `tmpfs`, both Podman and Docker showed significant improvements in pull time. The reduction in extraction time was due to the high IOPS and bandwidth provided by memory-backed storage, which outperformed the standard disk by an order of magnitude.

3. **Using a private endpoint made little to no measurable impact**. Tests run with and without the private ACR endpoint showed similar pull durations, indicating that network latency or routing over the private link was not a performance bottleneck in this case.

4. **Using a larger VM SKU improved performance**, but not to the same extent as the RAM disk. Upgrading to higher-tier VM sizes (e.g., Standard_D16s_v3 and Standard_D64s_v3) provided increased CPU and memory resources, which enabled higher concurrency during image pulls and improved overall throughput. However, while larger VMs reduced some bottlenecks related to compute and parallelism, they did not significantly reduce the time spent in the extraction phase when writing to the standard OS disk. The underlying disk I/O limitations persisted regardless of available CPU or RAM, making the improvement marginal in comparison to the dramatic gains observed when using a `tmpfs`-backed RAM disk. The RAM disk effectively eliminated the storage bottleneck, resulting in much faster decompression and layer extraction across all tools.

## Resource Cleanup

```bash
az group delete -n $RESOURCE_GROUP
```
# vLLM Ray Cluster Node Docker for DGX Spark

This repository contains the Docker configuration and startup scripts to run a multi-node vLLM inference cluster using Ray. It supports InfiniBand/RDMA (NCCL) and custom environment configuration for high-performance setups.

## 1\. Building the Docker Image

The Dockerfile includes specific **Build Arguments** to allow you to selectively rebuild layers (e.g., update the vLLM source code without re-downloading PyTorch).

### Option A: Standard Build (First Time)

```bash
docker build -t vllm-node .
```

### Option B: Fast Rebuild (Update vLLM Source Only)

Use this if you want to pull the latest code from GitHub but keep the heavy dependencies (Torch, FlashInfer, system deps) cached.

```bash
docker build \
  --build-arg CACHEBUST_VLLM=$(date +%s) \
  -t vllm-node .
```

### Option C: Full Rebuild (Update All Dependencies)

Use this to force a re-download of PyTorch, FlashInfer, and system packages.

```bash
docker build \
  --build-arg CACHEBUST_DEPS=$(date +%s) \
  -t vllm-node .
```

### Copying the container to another Spark node

To avoid extra network overhead, you can copy the image directly to your second Spark node via ConnectX 7 interface by using the following command:

```bash
docker save vllm-node | ssh your_username@another_spark_hostname_or_ip "docker load"
```

-----

## 2\. Running the Container

Ray and NCCL require specific Docker flags to function correctly across multiple nodes (Shared memory, Network namespace, and Hardware access).

```bash
docker run -it --rm \
  --gpus all \
  --net=host \
  --ipc=host \
  --privileged \
  --name vllm_node \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm-node bash
```

Or if you want to start the cluster node (head or regular), you can launch with the run-cluster.sh script (see details below):

**On head node:**

```bash
docker run --privileged --gpus all -it --rm \
  --ipc=host --shm-size 10.24g \
  --network host \
  --name vllm_node \ 
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm-node ./run-cluster-node.sh \
    --role head 
    --host-ip 192.168.177.11 
    --eth-if enp1s0f1np1 
    --ib-if rocep1s0f1 
```

**On worker node**

```bash
docker run --privileged --gpus all -it --rm \
  --ipc=host --shm-size 10.24g \
  --network host \
  --name vllm_node \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm-node ./run-cluster-node.sh \
    --role node
    --host-ip 192.168.177.12
    --eth-if enp1s0f1np1
    --ib-if rocep1s0f1
    --head-ip 192.168.177.11
```


**Flags Explained:**

  * `--net=host`: **Required.** Ray needs full control over network ports; port mapping is insufficient for multi-node clusters.
  * `--ipc=host`: **Required.** Allows shared memory access for PyTorch/NCCL.
  * `--privileged`: **Required for InfiniBand.** Grants the container access to RDMA devices (`/dev/infiniband`).

-----

## 3\. Using `run-cluster-node.sh`

Once inside the container, use the included script to configure the environment and launch Ray.

### Syntax

```bash
./run-cluster-node.sh [OPTIONS]
```

| Flag | Long Flag | Description | Required? |
| :--- | :--- | :--- | :--- |
| `-r` | `--role` | Role of the machine: `head` or `node`. | **Yes** |
| `-h` | `--host-ip` | The IP address of **this** specific machine (IB or Eth IP). | **Yes** |
| `-e` | `--eth-if` | Ethernet interface name (e.g., `eth0`, `enp3s0`). | **Yes** |
| `-i` | `--ib-if` | InfiniBand interface name (e.g., `ib0`, `rocep1s0f1`). | **Yes** |
| `-m` | `--head-ip` | The IP address of the **Head Node**. | Only if role is `node` |

### Example: Starting the Head Node

```bash
./run-cluster-node.sh \
  --role head \
  --host-ip 192.168.177.11 \
  --eth-if enp1s0f1np1 \
  --ib-if rocep1s0f1
```

### Example: Starting a Worker Node

```bash
./run-cluster-node.sh \
  --role node \
  --host-ip 192.168.177.12 \
  --eth-if enp1s0f1np1 \
  --ib-if rocep1s0f1 \
  --head-ip 192.168.177.11
```

-----

## 4\. Configuration Details

### Environment Persistence

The script automatically appends exported variables to `~/.bashrc`. If you need to open a second terminal into the running container for debugging, simply run:

```bash
docker exec -it vllm_node bash
```

All environment variables (NCCL, Ray, vLLM config) set by the startup script will be loaded automatically in this new session.

### Hardware Architecture

**Note:** The Dockerfile defaults to `TORCH_CUDA_ARCH_LIST=12.1a` (NVIDIA GB10). If you are using different hardware, update the `ENV` variable in the Dockerfile before building:

  * **H100:** `9.0`
  * **A100:** `8.0`
  * **L40S:** `8.9`

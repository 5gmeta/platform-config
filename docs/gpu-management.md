# Kubernetes GPU Management

## GPU Operator

The [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html#install-nvidia-gpu-operator) automatically setups and manages the NVIDIA software components on the worker nodes. For installing the operator the official documentacion has been followed: [https://docs.nvidia.com/datacenter/cloud-native/kubernetes/install-k8s.html#step-4-setup-nvidia-software](https://docs.nvidia.com/datacenter/cloud-native/kubernetes/install-k8s.html#step-4-setup-nvidia-software)

The GPU worker nodes in the Kubernetes cluster need to be enabled with the following components:

1.  NVIDIA drivers
2.  NVIDIA Container Toolkit
3.  NVIDIA Kubernetes Device Plugin (and optionally GPU Feature Discovery plugin)
4.  (Optional) DCGM-Exporter to gather GPU telemetry and integrate into a monitoring stack such as Prometheus

**NVIDIA drivers**
````
sudo  apt-get  install  linux-headers-$(uname  -r)
distribution=$(. /etc/os-release;echo $ID$VERSION_ID | sed -e 's/\.//g') \
   && wget https://developer.download.nvidia.com/compute/cuda/repos/$distribution/x86_64/cuda-$distribution.pin \
   && sudo mv cuda-$distribution.pin /etc/apt/preferences.d/cuda-repository-pin-600
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/$distribution/x86_64/7fa2af80.pub \
   && echo "deb http://developer.download.nvidia.com/compute/cuda/repos/$distribution/x86_64 /" | sudo tee /etc/apt/sources.list.d/cuda.list
sudo apt-get update && sudo apt-get -y install cuda-drivers
````

**NVIDIA Container Toolkit**
````
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
   && curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
   && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update \
   && sudo apt-get install -y nvidia-docker2
sudo nano /etc/docker/daemon.json
	# add the following:
	{
	   "default-runtime": "nvidia",
	   "runtimes": {
	      "nvidia": {
	            "path": "/usr/bin/nvidia-container-runtime",
	            "runtimeArgs": []
	      }
	   }
	}
sudo  systemctl  restart  docker
````

**NVIDIA Kubernetes Device Plugin**
````
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 \
   && chmod 700 get_helm.sh \
   && ./get_helm.sh
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin \
   && helm repo update
helm install --generate-name nvdp/nvidia-device-plugin
````

## GPU Monitoring

**DCGM-Exporter**

[DCGM-Exporter](https://github.com/NVIDIA/dcgm-exporter)  is a tool based on the Go APIs to  [NVIDIA DCGM](https://developer.nvidia.com/dcgm)  that allows users to gather GPU metrics and understand workload behavior or monitor GPUs in clusters.  dcgm-exporter  is written in Go and exposes GPU metrics at an HTTP endpoint (`/metrics`) for monitoring solutions such as Prometheus.

For information on the profiling metrics available from DCGM, refer to [this section](https://docs.nvidia.com/datacenter/dcgm/latest/dcgm-user-guide/feature-overview.html#profiling)  in the official documentation.

````
helm  repo  add  gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update
helm  install --generate-name gpu-helm-charts/dcgm-exporter
````

**GPU Metrics in Grafana**
Dashboard to be imported in Grafana, [https://grafana.com/grafana/dashboards/12239](https://grafana.com/grafana/dashboards/12239)

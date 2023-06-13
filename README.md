# Slurm cluster for parallel computing with CUDA on Google Cloud Platform. 

The setup has three components:

1) A custom slurm cluster built using Terraform
2) An NVIDIA docker image for the OS distribution and the chosen CUDA toolkit.
3) [Singularity Container Platform](https://sylabs.io/singularity/) for packaging and executing workloads in a container format. 

    We will follow the GCP tutorial [Deploying containerized workloads to Slurm on Compute Engine](https://cloud.google.com/architecture/deploying-containerized-workloads-slurm-cluster-compute-engine).



## Step 1

Slurm Cluster set up with [Terraform](https://developer.hashicorp.com/terraform). 

Please refer to [my branch](https://github.com/mfmotta/slurm-gcp) based on [SchedMD/slurm-gcp](https://github.com/SchedMD/slurm-gcp)


## Step 2


A Dockerfile to build an image for CUDA development environment for and your OS ditribution (we will use 11.6.0 and Ubuntu 20.04).

We will use this image: `nvidia/cuda:11.6.0-devel-ubuntu20.04`

**Requirements:**

**nvidia images:** https://hub.docker.com/r/nvidia/cuda

**nvidia container runtime:** https://github.com/NVIDIA/nvidia-container-runtime

2) a) Installation of container runtime for OS distribution --we use Ubuntu 20.04:

 add the NVIDIA Container Runtime repository and its GPG key to the package manager's configuration on your distribution's system
 with [the instructions](https://nvidia.github.io/nvidia-container-runtime/):

For debian-based distributions:
```
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | \
  sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-runtime.list
sudo apt-get update
```

[Note](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#setting-up-nvidia-container-toolkit:~:text=container%2Dtoolkit.list-,Note,-Note%20that%20in) that the downloaded list file may contain URLs that do not seem to match the expected value of distribution which is expected as packages may be used for all compatible distributions.

2) b) Install the nvidia-container-runtime package with:

```
sudo apt-get install nvidia-container-runtime
```

2) c) Docker Engine setup

We choose the setup via [Daemon configuration file](https://github.com/NVIDIA/nvidia-container-runtime#daemon-configuration-file):

See <sup>[1](#myfootnote1)</sup> for pros and cons of this method.

Create the daemon.json file in your home directory and specify the NVIDIA runtime there. This method allows you to control the NVIDIA runtime only for your user's Docker containers and doesn't require administrative privileges.

```
sudo tee /etc/docker/daemon.json <<EOF
{
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
EOF
sudo pkill -SIGHUP dockerd
```

You can optionally reconfigure the default runtime by adding the following to /etc/docker/daemon.json:
```
"default-runtime": "nvidia"
```

To verify whether the command was executed properly, you can verify if the content of `/etc/docker/daemon.json` 

2) d) Restart the Docker daemon to apply the new configuration. You can use the following command:
   ```
   sudo systemctl restart docker
   ```

2) e) After restarting, verify that the NVIDIA runtime is properly configured by running the following command:
   ```
   sudo docker info
   ```

   The output of sudo docker info should indicate that the NVIDIA runtime is available as one of the configured runtimes. The `Runtimes` section in the output should show something like 
   ``` 
   Runtimes: io.containerd.runc.v2 nvidia runc
   Default Runtime: runc
    ```


If the Docker daemon restarts without any errors and the `docker info` command shows the expected configuration, it indicates that the command was executed properly and the NVIDIA runtime is configured correctly.


## Step 3

Install singularity

<br>

<br>

<a name="myfootnote1">1</a>: *Docker Engine setup with Daemon configuration file.*

*Pros:*

- Allows fine-grained control over the Docker runtime c onfiguration.
 
- Changes can be made without administrative privileges by modifying the daemon.json file in the user's home directory (~/.docker/daemon.json).
 
- Does not affect the Docker service globally, only applies to the user's Docker runtime configuration.

*Cons:*

- Requires manually restarting the Docker daemon (dockerd) or sending a signal to the daemon (pkill -SIGHUP dockerd) for the changes to take effect.
- Changes are user-specific and only apply to the Docker runtime for that user.
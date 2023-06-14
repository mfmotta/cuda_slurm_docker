# Slurm cluster for parallel computing with CUDA on Google Cloud Platform. 

<br>

The setup involves four main parts:

1) A custom slurm cluster built using Terraform
2) An NVIDIA docker image for the OS distribution and the chosen CUDA toolkit.
3) A [Packer](https://github.com/SchedMD/slurm-gcp/tree/master/packer) to convert the docker image into a slurm-cluster compliant image.
4) Singularity Container Platform for packaging and executing workloads in a container format. 
<br>

---

<ol>
<li> <b> Slurm Cluster </b>
<br>

Set up a custom cluster on GCP with [Terraform](https://developer.hashicorp.com/terraform). 

Please refer to [my branch](https://github.com/mfmotta/slurm-gcp/tree/mm_branch/terraform/slurm_cluster/examples/slurm_cluster/cloud/full) (based on [SchedMD/slurm-gcp](https://github.com/SchedMD/slurm-gcp)) for an example.
    
The tutorial [Installing apps in a Slurm cluster on Compute Engine](https://cloud.google.com/architecture/installing-apps-slurm-clusters-compute-engine) is also helpful to understand how slurm clusters work on GCP.
</li>

<li> <b> Docker Image </b>
<br>
    
We need a Dockerfile to build an image for the CUDA development environment and the chosen OS ditribution.

We will use an image to install CUDA 11.6.0 and Ubuntu 20.04: `nvidia/cuda:11.6.0-devel-ubuntu20.04`. However, 
```
    docker pull nvidia/cuda:11.6.0-devel-ubuntu20.04
```
won't work without a few requirements, see [hub.docker.com/nvidia/cuda](https://hub.docker.com/r/nvidia/cuda#:~:text=Deprecated%3A%20%22latest%22%20tag). and [[sources](#sources)]:

<ol type='a'> 

<li> Installation of container runtime for OS distribution:


Add the NVIDIA Container Runtime repository and its GPG key to the package manager's configuration on your distribution's system
with [the corresponding instructions](https://nvidia.github.io/nvidia-container-runtime/). 

For debian-based distributions, they are:
```
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | \
sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.list | \
sudo tee /etc/apt/sources.list.d/nvidia-container-runtime.list
sudo apt-get update
```

---
<font size ="1"> &rarr; [Note](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#setting-up-nvidia-container-toolkit:~:text=container%2Dtoolkit.list-,Note,-Note%20that%20in): The downloaded list file may contain URLs that do not seem to match the expected value of distribution which is expected as packages may be used for all compatible distributions.</font>

---

<br>
</li>


<li> Install the nvidia-container-runtime package with:
```
sudo apt-get install nvidia-container-runtime
```
</li>

<br>
<li>
Docker Engine setup

We choose the setup via [Daemon configuration file](https://github.com/NVIDIA/nvidia-container-runtime#daemon-configuration-file) (see [pros and cons](#pros-and-cons) of this method).

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

To check whether the command was executed properly, you can verify the content of /etc/docker/daemon.json
</li>

<li> 
Restart the Docker daemon to apply the new configuration. You can use the following command:

```
sudo systemctl restart docker
```
</li>

<li> 
After restarting, verify that the NVIDIA runtime is properly configured by running the following command:
    
```
sudo docker info
```
The output of sudo docker info should indicate that the NVIDIA runtime is available as one of the configured runtimes. The "Runtimes" section in the output should show something like 
    
```
Runtimes: io.containerd.runc.v2 nvidia runc
Default Runtime: runc
```
If the Docker daemon restarts without any errors and the `docker info` command shows the expected configuration, it indicates that the command was executed properly and the NVIDIA runtime is configured correctly.
</li>
    
<li> Now you are able to 
    
```
docker pull nvidia/cuda:11.6.0-devel-ubuntu20.04
```
Now that we have our docker image we can modify it to be used by slurm.
</li>

</ol>

</li>

<br>
<li> <b> Custom slurm-cluster compliant image </b>
    
To create a slurm-cluster compliant image, a custom Slurm image can be created. You have to

Requirements: Packer and Ansible are used to orchestrate custom image creation, see (Packer)[https://github.com/SchedMD/slurm-gcp/tree/master/packer] for details. 
<!-- Note: you need a [hashicorp keyring](https://developer.hashicorp.com/terraform/tutorials/gcp-get-started/install-cli#:~:text=Install%20the%20HashiCorp%20GPG%20key.) --which you should already have after having created the cluster. But the comand to get the keyring is included below -->

Install Packer with:

```
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee      /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install packer
```
See https://developer.hashicorp.com/packer/downloads for other OS.

And run:

``` 
pip3 install ansible~=2.7
```
</li>
    
Now follow the instructions from [/slurm-gcp/docs/images.md](https://github.com/SchedMD/slurm-gcp/blob/master/docs/images.md#custom-image), and modify the pkrvars file and point source_image your image docker image produced in the previous step, and change the source_image_project_id [to match your OS](https://github.com/SchedMD/slurm-gcp/blob/3c5a3e570137e9ce8f33d19d1dfe46772c5eb66e/docs/images.md#supported-operating-systems),  i.e.
```
source_image_project_id = "ubuntu-os-cloud "
source_image = /var/lib/docker/overlay2
```
    
<br>
<li> <b> Singularity </b>
    
    For this step we recommend the GCP tutorial [Deploying containerized workloads to Slurm on Compute Engine](https://cloud.google.com/architecture/deploying-containerized-workloads-slurm-cluster-compute-engine) and the documentation of the [Singularity Container Platform](https://sylabs.io/singularity/)

</li>

</ol>

<br>

<br>

---
<br>

#### Sources:

nvidia images: https://hub.docker.com/r/nvidia/cuda

nvidia container runtime: https://github.com/NVIDIA/nvidia-container-runtime

---
<br>

#### Pros and cons
of Docker Engine setup with Daemon configuration file.

*Pros:*

- Allows fine-grained control over the Docker runtime c onfiguration.
 
- Changes can be made without administrative privileges by modifying the daemon.json file in the user's home directory (~/.docker/daemon.json).
 
- Does not affect the Docker service globally, only applies to the user's Docker runtime configuration.

*Cons:*

- Requires manually restarting the Docker daemon (dockerd) or sending a signal to the daemon (pkill -SIGHUP dockerd) for the changes to take effect.
- Changes are user-specific and only apply to the Docker runtime for that user.

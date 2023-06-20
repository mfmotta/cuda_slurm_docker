# Slurm cluster for parallel computing with CUDA on Google Cloud Platform. 

<br>

The setup involves four main parts:


1) NVIDIA docker image for the OS distribution and the chosen CUDA toolkit.
2) Slurm Packer to convert the docker image into a slurm-cluster compliant image.
3) Custom slurm cluster configuration using Terraform.
4) Optional : Singularity Container Platform for packaging and executing workloads in a container format. 

<br>

---

<ol>

<li> <b> Docker Image </b>

<br>
    
We want an image for the CUDA development environment in a specific OS ditribution, in this case, CUDA 11.6.0 and Ubuntu 20.04: ``nvidia/cuda:11.6.0-devel-ubuntu20.04``. This image is already available at https://hub.docker.com/r/nvidia/cuda, and [this repository](https://gitlab.com/nvidia/container-images/cuda) can be used to create custom images with different speficicatons. 

Here we will perform the minimum steps to create our custom image. Notice that, even though ``nvidia/cuda:11.6.0-devel-ubuntu20.04`` is available, ``docker pull nvidia/cuda:11.6.0-devel-ubuntu20.04``
won't work without the steps below, see [hub.docker.com/nvidia/cuda](https://hub.docker.com/r/nvidia/cuda#:~:text=Deprecated%3A%20%22latest%22%20tag).

Setup:

<ol type='a'> 

<li> Installation of container runtime for OS distribution:


Add the NVIDIA Container Runtime repository and its GPG key to the package manager's configuration on your distribution's system
with [the corresponding instructions](https://nvidia.github.io/nvidia-container-runtime/). 

For debian-based distributions:
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
Docker Engine setup:

<br>

We choose the setup via [Daemon configuration file](https://github.com/NVIDIA/nvidia-container-runtime#daemon-configuration-file) (see [pros and cons](#pros-and-cons) of this method).

Create the ``daemon.json`` file in your home directory and specify the NVIDIA runtime there. This method allows you to control the NVIDIA runtime only for your user's Docker containers and doesn't require administrative privileges.
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

You can optionally reconfigure the default runtime by adding the following to ``/etc/docker/daemon.json``:
```
"default-runtime": "nvidia"
```

To check whether the command was executed properly, you can verify the content of ``/etc/docker/daemon.json``.
</li>

<li> 
Restart the Docker daemon to apply the new configuration:

```
sudo systemctl restart docker
```
</li>

<li> 
After restarting, verify that the NVIDIA runtime is properly configured:
    
```
sudo docker info
```
The output of ``sudo docker info`` should indicate that the NVIDIA runtime is available as one of the configured runtimes. The "Runtimes" section in the output should show something like 
    
```
Runtimes: io.containerd.runc.v2 nvidia runc
Default Runtime: runc
```
If the Docker daemon restarts without any errors and the `docker info` command shows the expected configuration, it indicates that the command was executed properly and the NVIDIA runtime is configured correctly.
</li>
    
<li> Now you should be able to 
    
```
docker pull nvidia/cuda:11.6.0-devel-ubuntu20.04
```

Now that we have our docker image we need to modify it to be used by slurm.
</li>

</ol>

</li>

<br>
<li> <b> Custom slurm-cluster compliant image </b>

This step requires Packer and Ansible, which are used to orchestrate the custom image creation, see [SchedMD/slurm-gcp/packer](https://github.com/SchedMD/slurm-gcp/tree/master/packer) for details. 
    
Install Packer with:

```
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee      /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install packer
```
See https://developer.hashicorp.com/packer/downloads for other OS.

And Ansible with:

``` 
pip3 install ansible~=2.7
```
</li>
    
To create a [slurm-cluster compliant image]((https://github.com/SchedMD/slurm-gcp/blob/master/docs/images.md#custom-image)), cd into ``slurm-gcp/packer``and modify ``example.pkrvars.hcl``, ``variables.pkr.hcl``, and ``main.pkr.hcl``. Our packer configuration files introduce the following main changes:

In ``example.pkrvars.hcl``:

```
docker_image = "your-image-name:your-image-tag"
```

In ``variables.pkr.hcl``:

```
packer {
  required_plugins {
    docker = {
      version = ">= 0.0.7"
      source = "github.com/hashicorp/docker"
    }
  }
}
```

In ``main.pkr.hcl``:

<pre><code>#source

source "docker" "my_source" {
  image  = var.docker_image
  commit = true
}

#build:
<span style="color: red; font-family: Courier, font-size:6px;">
<s>- sources = ["sources.googlecompute.image"] </s> </span>  <span style="color: green; font-family: Courier font-size:6px;">
+ sources = ["sources.docker.my_source"]</span> 
</code></pre>


Now you can build the custom image with:

```
packer init 
packer validate -var-file=example.pkrvars.hcl 
packer build -var-file=example.pkrvars.hcl 
```

<br>

<li> <b> Slurm Cluster </b>

<br>

Set up a custom cluster on GCP with [Terraform](https://developer.hashicorp.com/terraform). 

Please refer to my guide [Slurm  Cluster on Google Cloud Platform](https://github.com/mfmotta/slurm-gcp#slurm-cluster-on-google-cloud-platform) for an example.
    
The tutorial [Installing apps in a Slurm cluster on Compute Engine](https://cloud.google.com/architecture/installing-apps-slurm-clusters-compute-engine) is also helpful to understand how slurm clusters work on GCP.
</li>
    
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

# Novel HPC FAQ

## SSH Access

Within the LAN: `ssh <usrname>@login.novel.hpc`. You will be prompted to login via Microsoft SSO.

Hint: `man ssh-copy-id` for how to use `ssh-copy-id` or see `man ssh` AUTHENTICATION section for more detail, and `man ssh-keygen`.

## Login Environment

The cluster uses the module environment, run `module avail` to see available modules.

Utilities such as `tmux`, `nano`, `vim` and `git` are provided by default. Even the graphical editor `pluma` is included (which requires ssh X-forwaring i.e. ssh -X ...).

## Worker Node Environment and NVIDIA Enroot

All jobs should take advantage of the [`enroot`](https://github.com/NVIDIA/enroot) container environment, although `tmux`, `nano`, `vim` and `git` are available.
Python Conda is available through the module environment, but enroot is preferable.

As Enroot uses SquashFS, it's not possible to perform commands that would modify the containerised system (such as apt-get etc.). To install dependencies after the creation of the SquashFS filesystem, you need to rely upon userspace software, such as Python's `venv` system:

It's best to create and interact with venvs exclusively from within the Enroot environment, so python versions aren't mixed.

```bash
# Start an interactive job
login $ srun --time=03:00:00 --job-name=interactive --partition=gpu --nodelist=hpc-novel-gpu03 --gres=gpu:nvidia_rtx_a6000:1 --mem=59G --cpus-per-task=8 --pty bash
# Enter the Enroot environment of your choice (example pytorch container here)
gpu03 $ enroot start --mount="$HOME:$HOME" /home/shared/nvproj002/pytorch_n_friends.sqsh
# CD to your project
gpu03 $ cd ~/my-project
# Create a virtual environment in the current directory, and activate it
gpu03 $ python3 -m venv .venv
gpu03 $ source .venv/bin/activate
# Install your dependencies
(.venv) gpu03 $ pip3 install tqdm numpy matplotlib etc...
(.venv) gpu03 $ pip3 install -r requirements.txt
# Exit the Enroot container
(.venv) gpu03 $ exit # Or ^D
# Exit the node and release the resources
gpu03 $ exit # Or ^D
login $

# The virtual environment created within enroot is persistent where it was
# created (i.e. at ~/my-project/.venv) and can be "reactivated" and modified at any time
```

## Create Enroot SquashFS Images

> [!CAUTION]
> Building sqfs images is currently not working properly... input welcome. The below guide used to work, but now produces images that fail.

During the building of a Dockerfile, or the creation of a squahfs, access to the filesystem of the cluster can become slow. If you can, use the image Villanelle compiled at `/home/shared/nvproj002/pytorch_n_friends.sqsh` which has nano, git and PyTorch 2.7.0 (for CUDA 12.6).

### From Docker Hub to SquashFS

```bash
ENROOT_SQUASH_OPTIONS='-comp lz4 -noD' enroot import docker://ubuntu
```

### From Dockerfile to SquashFS

> [!IMPORTANT]
> It takes a *very* long time to build Dockerfiles on the HPC. Every layer your build will freeze for several minuets, it will feel like it's crashed; it hasn't. A simple container, such as my example, will take up to 30 minuets. This is likely because of the filesystem (NFS), or the configuration of it.

```bash
# Make your working directory, and copy example
mkdir dockerdir && cd dockerdir
cp /home/shared/nvproj002/pytorch_n_friends.Dockerfile.ci .
# Build (please report the time to the Teams chat)
# -t is the name Podman will associate with the image
time podman build -t pytorch_n_friends -f pytorch_n_friends.Dockerfile.ci .
# Convert the image (this cmd creates pytorch_n_friends.sqsh)
# This will take 7+ mins!
ENROOT_SQUASH_OPTIONS='-comp lz4 -noD' enroot import podman:/pytorch_n_friends
```

### Test Your Image

```bash
# Example interactive job
srun --time=03:00:00 --job-name=interactive --partition=gpu --nodelist=hpc-novel-gpu03 --gres=gpu:nvidia_rtx_a6000:1 --mem=59G --cpus-per-task=8 --pty bash
enroot start --mount="$HOME:$HOME" ~/pytorch_n_friends.sqsh
```

### Using Enroot

Section below explains how to use Enroot in jobs (running scripts within Enroot).

## Submitting Jobs

The cluster uses SLURM for accessing resources.

### Interactive Jobs

An interactive job could be used to explore an environment, but it should not be used for running scripts.
```bash
# Example interactive job
srun --time=03:00:00 --job-name=interactive --partition=gpu --nodelist=hpc-novel-gpu03 --gres=gpu:nvidia_rtx_a6000:1 --mem=59G --cpus-per-task=8 --pty bash
```

### Non-Interactive Jobs

It's best to use an sbatch file:
```bash
#!/bin/bash

#SBATCH --nodes=1
#SBATCH --cpus-per-task=8 # This is vCPUs
#SBATCH --mem=59G
#SBATCH -o logs/%j.out
#SBATCH -e logs/%j.err
#SBATCH --ntasks-per-node=1
#SBATCH --partition=gpu
#SBATCH --nodelist=hpc-novel-gpu01
#SBATCH --gres=gpu:1
#SBATCH --job-name="myname"

#SBATCH --mail-type=ALL
#SBATCH --mail-user=<associated-emails-only>
#SBATCH --time=06:00:00
# Or
# #SBATCH --qos=long 
# #SBATCH --time=2-00:00:00 # i.e. 2 days, you can go up to 7 with long QOS

# ADD DEBUGGING ECHO COMMANDS HERE
module purge

echo "Starting job at $(date)"
echo "Working directory: $(pwd)"

nvidia-smi

srun enroot start --mount="$HOME:$HOME" /home/shared/nvproj002/pytorch_n_friends.sqsh $HOME/project/launch.sh
```

Example of the ~/project/launch.sh (must have executable bit)
```
cd ~/project && source .venv/bin/activate && python train.py
```

Then you can execute the sbatch:
```bash
sbatch your_sbatch.sh

# Or override options
sbatch --nodelist=hpc-novel-gpu02 --gres=gpu:2 --mem=119GB --cpus-per-task=32 your_sbatch.sh
```

## Hardware

To get up-to-date hardware information run `scontrol show nodes <nodename>`, or use commands such as `lscpu` in interactive jobs. There are also 4 CPU-only nodes.

The only active GPUs are NVIDIA RTX A6000s (48GiB VRAM), which have 3D (OpenGL, Vulkan etc.) capabilities. All nodes are running the NVIDIA driver version 575 which comes with CUDA 12.9.

| Hostname          | CPU                          | RAM    | GPU                  |
| :---------------- | :--------------------------- | :----- | :------------------- |
| `hpc-novel-gpu01` | AMD EPYC 7453 x2 (112 vCPUs) | 950GiB | 8x RTX A6000 (48GiB) |
| `hpc-novel-gpu02` | AMD EPYC 7453 x2 (112 vCPUs) | 480GiB | 4x RTX A6000 (48GiB) |
| `hpc-novel-gpu03` | AMD EPYC 7343 x1 (32 vCPUs)  | 240GiB | 4x RTX A6000 (48GiB) |
| `hpc-novel-gpu04` | AMD EPYC 7713 x2 (256 vCPUs) | 950GiB | 8x RTX A6000 (48GiB) |


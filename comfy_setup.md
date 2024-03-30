
# AWS-specific Setup (deprecated, note that A10G is not enough VRAM to run well)
Sign into AWS console

Create EC2 instance using AMI
`Deep Learning Base OSS Nvidia Driver GPU AMI (Ubuntu 20.04) 20240308` (ami-05c66b46e53abca58)

Run on a g5.xlarge, 75GB root storage
Ensure SSM access via IAM role

Set up an SSH-config bookmark forwarding port 8188 (For comfyui)

Create an EBS volume of 200GB and mount to `/dev/sdg` . This is where we will store comfy, the env, and the model files, and we can persist this volume when we stop the instance to save costs.

SSH into the instance

Update and upgrade
```
sudo apt-get update && sudo apt-get upgrade -y --with-new-pkgs && sudo apt-get autoremove -y && sudo apt-get clean
```

Mount and format data drive (TODO)
```
# find the volume you added, in this case /dev/nvme2n1
lsblk

# sudo mkfs -t ext4 /dev/nvme2n1

sudo mkdir /workspace
sudo chown ubuntu:ubuntu /workspace

# make an fstab backup
sudo cp /etc/fstab /etc/fstab.orig

# Get your UUID for the device
sudo blkid /dev/nvme2n1
# Run the following after replacing UUID with the appropriate value
# echo "UUID=b7490733-9b3a-4100-9723-69427a0362cd /workspace ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
# Mount
sudo mount -a
sudo chown ubuntu:ubuntu /workspace --recursive
```

# Runpod-specific setup
Create a pod and a network storage volume of 200GB
Add a public key
SSH in via 
```
ssh abcdefg1234567.runpod.io -i ~/.ssh/tj_runpod.pem -L 8188:localhost:8188
```

Notes - I had originally tried doing this on networked storage to persist across sessions, but it was way too slow both to run things from or even to copy over prebuilt environments etc from. Ended up being easier to do all this setup on a new node each time, and just use the networked storage for the model files (since downloading them from the runpod datacenter locations takes 15min each)

# First time setup

Install Conda
(best to do on main system drive for speed, IO is slow on networked /workspace drive)
```
wget https://github.com/conda-forge/miniforge/releases/download/23.11.0-0/Mambaforge-23.11.0-0-Linux-x86_64.sh
bash Mambaforge-23.11.0-0-Linux-x86_64.sh
# follow instructions and install to /root/mambaforge
```

Initial Setup
```
mkdir github
```

Download ComfyUI
```
cd github
git clone https://github.com/comfyanonymous/ComfyUI
cd ComfyUI
```

Download Model Files (first time, to networked storage)
```
mkdir models/svd
cd models/svd
# SVD regular
wget https://huggingface.co/stabilityai/stable-video-diffusion-img2vid/resolve/main/svd.safetensors
wget https://huggingface.co/stabilityai/stable-video-diffusion-img2vid/resolve/main/svd_image_decoder.safetensors
# SVD XT
wget https://huggingface.co/stabilityai/stable-video-diffusion-img2vid-xt/resolve/main/svd_xt.safetensors
wget https://huggingface.co/stabilityai/stable-video-diffusion-img2vid-xt/resolve/main/svd_xt_image_decoder.safetensors
# SVD XT 1.1
wget https://huggingface.co/stabilityai/stable-video-diffusion-img2vid-xt-1-1/resolve/main/svd_xt_1_1.safetensors
cd ../..
```

Download Model files (subsequent times, from networked storage to local)
```
cp /workspace/github/ComfyUI/models/svd models/ --recursive
cp models/svd/* models/checkpoints
```

Create conda environment and install dependencies
```
mkdir /env
# note that we want python 3.11 since pypi doesnt have a prebuilt xformers wheel available for 3.12 so build takes many hours.
conda create -p /env/comfy python=3.11

conda activate /env/comfy
# make sure we're using the conda pip and not system pip
which pip
pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu121
pip install -r requirements.txt
```

Download custom video node
```
cd custom_nodes/
git clone https://github.com/thecooltechguy/ComfyUI-Stable-Video-Diffusion
cd ComfyUI-Stable-Video-Diffusion/
python install.py
cd ../..
```

Download custom RIFE node
```
cd custom_nodes/
git clone https://github.com/Fannovel16/ComfyUI-Frame-Interpolation
cd ComfyUI-Frame-Interpolation/
python install.py
cd ../..
```

Download custom video helpers node
```
cd custom_nodes/
git clone https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite
cd ComfyUI-VideoHelperSuite/
pip install -r requirements.txt
cd ../..
```

Download comfyui essentials (image resize, etc)
```
cd custom_nodes/
git clone https://github.com/cubiq/ComfyUI_essentials
cd ComfyUI_essentials/
pip install -r requirements.txt
cd ../..
```

Download Anything Everywhere
```
cd custom_nodes/
git clone https://github.com/chrisgoringe/cg-use-everywhere
cd ..
```

```
conda deactivate
```


# Running it (on any clean runpod node, after first time setup above is done)
Make sure your node has enough storage space to copy everything over from the networked storage drive (~200GB should suffice)

Activate mamba on a new node
```
wget https://github.com/conda-forge/miniforge/releases/download/23.11.0-0/Mambaforge-23.11.0-0-Linux-x86_64.sh
bash Mambaforge-23.11.0-0-Linux-x86_64.sh
# etc
```

Install screen on node
```
apt-get update
apt-get install screen -y
```

Start ComfyUI in a persistent screen
```
screen -S comfy
cd /github/ComfyUI
conda activate /env/comfy
python main.py --listen
```
https://{POD_ID}-{INTERNAL_PORT}.proxy.runpod.net


Starter workflow for Comfy
https://comfyworkflows.com/workflows/bf3b455d-ba13-4063-9ab7-ff1de0c9fa75
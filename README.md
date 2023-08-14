# gpu_passthrough
Notes on setting up GPU passthrough on Proxmox. 

Goal: Allow Jellyfin container (running inside a Virtual Machine) to use an Nvidia GPU for transcoding video
- Host: Optiplex 9020 SFF running Proxmox 8.0.2
- VM: Debian 12 Bookworm
- GPU: Nvidia Quadro P400

## In Proxmox
* enable IOMMU on host
* Blacklist modules so host does not use GPU or set in BIOS
* Create VM. Just make sure you select 'q35' for Machine Type and 'OVMF(UEFI)' for BIOS
  ![alt text](gpu_vmconfig.png?raw=true)
* Passthrough PCI GPU (Hardware -> Add PCI Device)
  ![alt text](gpu_pci_device.png?raw=true)
* Start VM, enter BIOS and Disable SecureBoot (F2 -> Device Manager -> Secure Boot Configuration)
  ![alt text](gpu_secureboot.png?raw=true)
* Go through and complete the Debian installation. Nothing special required here.

## Inside the VM

### Initial Steps

1. Check if VM even detects the GPU is attached first. If not, look back over the previous steps.
```
root@media:~$ lspci | grep -i p400
01:00.0 VGA compatible controller: NVIDIA Corporation GP107GL [Quadro P400] (rev a1)
```
2. Install some useful packages  
```apt install htop curl wget vim rsync sudo```

3. Add account to sudoers  
```usermod -aG sudo conor```

4. From Workstation, add your SSH keys  
```ssh-copy-id conor@<VM-IP-ADDRESS>```

Once that's done you can jump off the Proxmox console and just SSH to the VM from your workstation.

### Install Nvidia drivers

1. Install kernel headers

```sudo apt install linux-headers-amd64```

2. Make sure these repositories are present in /etc/apt/sources.list

```
deb http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware
```   

3. Now install the nvidia driver from those repos

```sudo apt install nvidia-driver firmware-misc-nonfree```

4. Reboot the VM

```sudo systemctl reboot```

5. Once the VM is back up, check if the nvidia driver is loaded with ```modinfo nvidia-current```. You should see:
   ```
   conor@gputest2:~$ sudo modinfo nvidia-current | head -15
   filename:       /lib/modules/6.1.0-10-amd64/updates/dkms/nvidia-current.ko
   firmware:       nvidia/525.125.06/gsp_tu10x.bin
   firmware:       nvidia/525.125.06/gsp_ad10x.bin
   alias:          char-major-195-*
   version:        525.125.06
   supported:      external
   license:        NVIDIA
   srcversion:     B91689BE2C82F40FE520B73
   alias:          pci:v000010DEd*sv*sd*bc06sc80i00*
   alias:          pci:v000010DEd*sv*sd*bc03sc02i00*
   alias:          pci:v000010DEd*sv*sd*bc03sc00i00*
   depends:        drm
   retpoline:      Y
   name:           nvidia
   vermagic:       6.1.0-10-amd64 SMP preempt mod_unload modversions
   ```

7. Install some more useful packages

```sudo apt install nvtop nvidia-detect ```
```
conor@gputest2:~$ nvidia-smi
Sun Aug  6 21:30:43 2023
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.125.06   Driver Version: 525.125.06   CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Quadro P400         On   | 00000000:01:00.0 Off |                  N/A |
| 34%   44C    P8    N/A /  30W |      1MiB /  2048MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

Really at this point if you didn't care about running Jellyfin in a container, you could install Jellyfin via the system packages and use the options in the WebGUI to enable transcoding on the GPU.

#### Docker

1. Install docker and docker-compose
```
sudo apt install docker docker-compose -y
```
2. Install the Nvidia container toolkit. This will allow for GPU support in Docker.
```
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey > /etc/apt/keyrings/nvidia-docker.key
curl -s -L https://nvidia.github.io/nvidia-docker/debian11/nvidia-docker.list > /etc/apt/sources.list.d/nvidia-docker.list
sed -i -e "s/^deb/deb \[signed-by=\/etc\/apt\/keyrings\/nvidia-docker.key\]/g" /etc/apt/sources.list.d/nvidia-docker.list
apt update
apt install install nvidia-container-toolkit
systemctl restart docker
```
3. Install nvidia-docker2. You need this to be able to pass the 'runtime: nvidia' environment variable to Docker
```
sudo apt install nvidia-docker2
```

Before we look at Jellyfin. Test if you can pass the GPU through to a container via  
```docker run --gpus all nvidia/cuda:12.1.1-runtime-ubuntu22.04 nvidia-smi```  
This is going to spin up a container using the base cuda image and run nvidia-smi. You should see this:
```
conor@media:~$ sudo docker run --gpus all nvidia/cuda:12.1.1-runtime-ubuntu22.04 nvidia-smi

==========
== CUDA ==
==========

CUDA Version 12.1.1

Container image Copyright (c) 2016-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.

This container image and its contents are governed by the NVIDIA Deep Learning Container License.
By pulling and using the container, you accept the terms and conditions of this license:
https://developer.nvidia.com/ngc/nvidia-deep-learning-container-license

A copy of this license is made available in this container at /NGC-DL-CONTAINER-LICENSE for your convenience.

Mon Aug 14 16:49:33 2023
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.125.06   Driver Version: 525.125.06   CUDA Version: 12.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Quadro P400         On   | 00000000:01:00.0 Off |                  N/A |
| 34%   38C    P8    N/A /  30W |      1MiB /  2048MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

#### Jellyfin
I used the image from lscr.io

My docker compose file looks like this: (should really use a volume for the configs but whatever)
```
---
version: "2.1"
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - JELLYFIN_PublishedServerUrl=192.168.0.223 #optional
    volumes:
      - /opt/docker/configs/jellyfin:/config
      - /mnt/media:/media
      - /mnt/media_2:/media_2
    ports:
      - 8096:8096
      - 8920:8920 #optional
      - 7359:7359/udp #optional
      - 1900:1900/udp #optional
    restart: unless-stopped
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
```
Bring that up with ```docker-compose up``` and check if you are able to access the WebGUI at ```http://<VM-IP-ADDRESS>:8096```  
Once in Jellyfin, navigate to Dashboard -> Playback -> Enable Hardware Transcoding.
![alt text](gpu_jellyfin.png?raw=true)
That's it done!

## Testing

Here is the VM playing a ~20GB 4K HEVC video file. CPUs are pinned to 100% (and it's loud!)
 ![alt text](jellyfin_no_gpu.png?raw=true)

After enabling Hardware Acceleration, the CPUs are much less stressed and the GPU is handling the video transcoding.
 ![alt text](jellyfin_with_gpu.png?raw=true)
 

## Troubleshooting
```
conor@gputest2:~$ nvidia-smi
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver
```

Make sure you definitely disabled SecureBoot on the VM as above.

## References
1. https://wiki.debian.org/NvidiaGraphicsDrivers
2. https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
3. https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
4. https://www.server-world.info/en/note?os=Debian_12&p=nvidia&f=2
5. https://github.com/NVIDIA/nvidia-docker/issues/1268#issuecomment-632692949

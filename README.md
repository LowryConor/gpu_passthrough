# gpu_passthrough
Notes on setting up GPU passthrough on Proxmox. 

Goal: Allow Jellyfin container (running inside a VM) to use an Nvidia GPU for transcoding video
- Host: Optiplex 9020 running Proxmox 7.4-3
- VM: Debian 12
- GPU: Nvidia Quadro P400

## Proxmox
* Upload debian 12 ISO  
* enable IOMMU
* Blacklist modules so host does not use GPU
* Create VM
* Disable SecureBoot (F2 -> Device Manager -> Secure Boot Configuration)
* Passthrough PCI GPU (Hardware -> Add PCI Device)
* Check if VM detects GPU

## Inside the VM

### Initial Steps
1. Install some useful packages  
```apt install htop curl wget vim rsync sudo```

2. Add account to sudoers  
```usermod -aG sudo conor```

3. From Workstation, add your SSH keys  
```ssh-copy-id conor@<VM-IP-ADDRESS>```

At this point, best to take a VM snapshot in Proxmox to easily rollback before we start installing any Nvidia stuff.  
TODO: Screenshot  
Once that's done you can jump off the Proxmox console and just SSH to the VM from your workstation.

### Install Nvidia drivers
Following https://wiki.debian.org/NvidiaGraphicsDrivers for the next steps.

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
Install Install nvidia-docker2 and the Nvidia container toolkit  
From https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html  
At time of writing, the method on that page doesn't support Debian 12.  
So just use https://www.server-world.info/en/note?os=Debian_12&p=nvidia&f=2

Before we look at Jellyfin. Test if you can pass the GPU through to a container via  
```docker run --gpus all nvidia/cuda:12.1.1-runtime-ubuntu22.04 nvidia-smi```  
This is going to spin up a container using the base cuda image and run nvidia-smi. You should see this:

Congrats if that's working

#### Jellyfin
I used the image from lscr.io rather than the offical Jellyfin image.  

My docker compose file looks like this:
```
---
version: "2.1"
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=911
      - PGID=110
      - TZ=Etc/UTC
      - JELLYFIN_PublishedServerUrl=192.168.0.110 #optional
    volumes:
      - /opt/jellyfin:/config
      - /opt/media:/data/movies
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
Bring that up with ```docker compose up``` and check if you are able to access the WebGUI at ```http://<VM-IP-ADDRESS>:8096```  
Once in Jellyfin, navigate to Dashboard -> Playback -> Enable Hardware Transcoding.  
That's it done!

## Testing

TODO: nvtop and htop screenshots here from playback with transcoding on/off

## Troubleshooting
```
conor@gputest2:~$ nvidia-smi
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver
```

Make sure you definitely disabled SecureBoot on the VM as above.

## References
1. https://wiki.debian.org/NvidiaGraphicsDrivers
2. https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

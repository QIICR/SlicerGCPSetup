These instructions are based on the notes from the [NA-MIC Project week 31 - GCP setup for Slicer project](https://github.com/NA-MIC/ProjectWeek/blob/master/PW31_2019_Boston/Projects/SlicerGCP/README.md). The original notes were copied here for more convenient maintenance and refinement.

# Slicer in Google Cloud Platform (GCP) with GPU support

## Objective

<!-- Describe here WHAT you would like to achieve (what you will have as end result). -->

Replicate Slicer running GCP machine with instructions and write them down for the public

## PLEASE READ: Important notes

* The Google Cloud Platform costs real money once your free trial is over.  **Be sure to shut down anything you aren't using** or your credit card will eventually be charged.
* Be careful with your login information.  If someone takes over your account they **could run up a huge bill that you will be responsible for paying**.
* Depending on your specific situation, not all of the options or steps may be available or applicable (e.g., as a member of organization, your organization administrator may have disabled some of the options discussed).
* Unless you are not concerned about billing, remember to **SHUT DOWN THE MACHINE** when you aren't using it! You are billed continuously while the VM instance is running.

## Instructions

1. Sign up for 300$ free credit on GCP
2. Go to https://console.cloud.google.com/home
3. Select left sidebar "Compute Engine --> VM instances"
4. Create Instance with the configuration as you wish
5. Machine type —> Customize and select GPU 
7. Select “Ubuntu 18.04” boot disk
8. Finish creation of VM
9. (Optional) Create instance templates for repeated creation

### Important: GPU usage prerequisites**

1. Go to sidebar —> IAM & admin —> Quotas
2. Select metrics —> None and search for GPU
3. Select GPUs (all regions)
4. Check and click EDIT Quotas
5. Enter your information and click next
6. Set limit to 2
7. Request process might take up to 2 business days, but if you send them an email, they could be faster with it (at least for me it was)

You have two options to access VNC:
* **Option 1**: by directly connecting to the noVNC port on the GCP VM instance - you will need to adjust your firewall settings. **Important**: this approach is not secure - it will allow anyone to connect to the VM!
* **Option 2**: by tunneling the connection through an SSH channel - this approach is easier to implement, and will restrict access to the VM instance to authorized users only

### Option 1: Direct access to noVNC port on VM instance

Configure Firewall to open the noVNC port:

1. Under VM instance configuration: Firewall --> allow HTTP and HTTPS
1. Select left sidebar "VPC network --> Firewall rules"
2. Select "CREATE FIREWALL RULE"
3. Set Source ip ranges: 0.0.0.0/0
4. Protocols and ports: tcp: 6080

### Option 2: Access noVNC port via SSH tunnel

Configure prerequisites on your machine:

1. Install gcloud SDK as described [here](https://cloud.google.com/sdk/).
2. Set up gcloud with your GCP credentials:
```
$ gcloud init
```

### Configure the VM instance

1. Start VM by clicking
2. Get terminal access to the VM instance by either clicking on "SSH" button in the VM instances list, or executing the following command in the local terminal on your computer (considering you installed the GCP SDK, as discussed above):
```
$ gcloud compute ssh <VM instance name>
```
3. Install the prerequisites (after the `sudo xinit` command, interrupt it with Ctrl+C and proceed with the next steps)
```
sudo apt-get update
sudo apt install ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo apt install xinit
sudo apt-get install x11vnc
sudo apt-get install xterm
sudo xinit
sudo nvidia-xconfig
```

Execute the following and take not of the BusID

```
nvidia-xconfig --query-gpu-info
```

Open the X11 configuration file
```
sudo vim /etc/X11/xorg.conf
```

and insert the following lines including the BusID you retrieved earlier
```
Section "Device"
   Identifier     "Device0"
   Driver         "nvidia"
   VendorName     "NVIDIA Corporation"
   BusID          "PCI:0:4:0"
EndSection
```

Prepare for running noVNC:
```
sudo apt-get install python
git clone https://github.com/novnc/noVNC
```

### Server-side: Start X11, VNC and noVNC

Each reboot (e.g. after doing 'start' on the google cloud console). The commands below are set up so you can cut and paste them into the ssh terminal from the google interface, but if you want to debug more easily them you might want to paste each in its own terminal. 
```
sudo xinit -- +extension GLX &
./noVNC/utils/launch.sh --vnc localhost:5900 &
while true; do x11vnc -forever -display :0; sleep 1; done
```

Note that the `while true ...` part in the instructions above is needed to address the possible intermittent crashes of x11vnc. You can improve stability by building `x11vnc` from source (see Troubleshooting section).

### Server-side: Download and unpack Slicer for linux

Here using a specific revision, but any version should work

```
wget http://slicer.kitware.com/midas3/download/item/435293/Slicer-4.10.2-linux-amd64.tar.gz
tar xvzf Slicer-4.10.2-linux-amd64.tar.gz
```
### Client-side: Connect VNC

Note: this is a very raw linux machine and you are running as root.  There is also a user account under your name that is automatically created by the google VM boot process.  Pretty much anything from the last few decades of linux development should run the same here as it does on a local workstation.

#### Option 1: Direct access to noVNC port on VM instance

1. Connect to http://{VM_External_IP}:6080/vnc.html

```
cd Slicer-4.10.2-linux-amd64
./Slicer
```


### Option 2: Access noVNC port via SSH tunnel

1. Tunnel noVNC port:
```
 gcloud compute ssh <your VM name> --project <your GCP project name> -- -L 6080:localhost:6080
```
2. Open the connection in your browser: http://localhost:6080/vnc.html

# TODO
If anyone works on these issues please write them up and let us know:
* Add instructions for setting up TLS / HTTP (e.g. with [letsencrypt.org](https://letsencrypt.org/)
* Add instructions for setting up reverse proxy with OAuth
* Add a window manager and other utilities to the X environment, (e.g. with [OpenBox](http://openbox.org), as is [done here](https://github.com/pieper/SlicerDockers/tree/master/x11))
  * running `sudo apt-get install openbox && openbox-session` in the terminal window is one way to start.  A lot of things won't work out of the box but you can configure the files in `/opt/xdg/openbox`.  Also you can access the NVidia X server settings to change the screen resolution.
* Describe other VNC options
* Come up with similar instructions for AWS and Azure (and other computer rental providers).
* Consider setting up a multi-user system with multiple logins to be used in 'time sharing' for collaboration and resource sharing.
* Explore reliability (sometime instances reboot unexpectedly)
* Explore the most cost-effective options (e.g. preemptable instances and GCP vs AWS and others for running tasks)

## Troubleshooting

### Sporadic x11 server disconnects

If you have trouble with the x11 server disconnecting when openning menus or resizing files, you are probably hitting [this bug](https://github.com/LibVNC/x11vnc/issues/61) which is not yet fixed in ubuntu.

You can replace with a patched version like this (as root):
```
curl "https://drive.google.com/uc?id=1FCTxYPAPf58AqchST0SLYfZFZoVANCfL&export=download" -o x11vnc -L
cp x11vnc /usr/bin/x11vnc
```

* For key repeat: `sudo apt-get install x11-xserver-utils` and then `xset r rate 300 10` (you also need run `xset r on` twice to override the [-norepeat](http://www.karlrunge.com/x11vnc/x11vnc_opts.html) option of x11vnc)

You can also build `x11vnc` from source using the instructions below.

1. Install prerequisites (as discussed [here](https://askubuntu.com/questions/496549/error-you-must-put-some-source-uris-in-your-sources-list)):
```
sudo cp /etc/apt/sources.list /etc/apt/sources.list~
sudo sed -Ei 's/^# deb-src /deb-src /' /etc/apt/sources.list
sudo apt-get update
```
2. Install `x11vnc` build dependencies, checkout source, configure and build (as in the [`x11vnc` build instructions](https://github.com/LibVNC/x11vnc#building-x11vnc)).
```
sudo apt-get build-dep x11vnc
git clone https://github.com/LibVNC/x11vnc.git
cd x11vnc
autoreconf -fiv
make
```

The binary will be in the `src` directory!

### Alternative VNC servers

Some recipes, such as [this one](https://medium.com/google-cloud/linux-gui-on-the-google-cloud-platform-800719ab27c5), recommend using `vncserver`, which is a wrapper around `xvnc`. Based on [this post on NVIDIA forum](https://devtalk.nvidia.com/default/topic/1031651/nvidia-driver-vnc-and-glx-issue/), `xvnc` does not support GLX, and will not work with Slicer.

If you experiment with alternative VNC implementations, please share your experience via PR!

### Debugging

If something is not working, you can debug individual components.

On the server:
1. Start `xinit` in the foreground mode and check if there are no errors.
2. Start `x11vnc` in a separate terminal window, and check there are no errors.
3. Check that `x11vnc` is listening on port 5900 after startup:
`$ nc localhost 5900`

On the client:
1. If you can adjust the firewall settings, you can check if you can connect to `x11vnc` directly bypassing noVNC. If you are on mac, do NOT use the default macOS VNC client! We confirmed that [Chicken](https://sourceforge.net/projects/chicken/) open source VNC client can establish connection under the same conditions where default macOS client cannot.

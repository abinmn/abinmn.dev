---
title: Setting up Android Emulator Containers in Virtual Machine
date: 2024-01-27
description: Setup table of content in Hugo blog awesome theme
---

In a recent project at work, I had to create an ephemeral Android Emulator for automated end-to-end testing within our CI pipeline. 

Initially, I explored deploying the emulator within the Kubernetes cluster that hosts the CI/CD runners. Given the ephemeral nature of Kubernetes, the simplicity of deploying pods and easy communication from the other CI/CD runner pods, this seemed to be a logical choice.

[Android-emulator-container-scripts](https://github.com/google/android-emulator-container-scripts/) is an official tool from Google that helps to create custom emulator container images. Below `docker run` command is from the quick start guide. They expose two ports - for `adb`(5555) and a gRPC based web interface (8554)
 
```bash
docker run \
  -e ADBKEY="$(cat ~/.android/adbkey)" \
  --device /dev/kvm \
  --publish 8554:8554/tcp \
  --publish 5555:5555/tcp  \
  us-docker.pkg.dev/android-emulator-268719/images/30-google-x64:30.1.2
```

While the `docker run` command seemed familiar, a notable addition was `--device /dev/kvm`. Despite my initial lack of understanding regarding this flag's purpose, I proceeded to test the container image in the Kubernetes cluster by launching a pod.

```bash
kubectl run android-emulator \
--image=us-docker.pkg.dev/android-emulator-268719/images/30-google-x64:30.1.2 
```


![](https://i.imgur.com/2yWIsrE.png)

Though the pod started, it exited with an error that hardware acceleration is required to start the emulator.

The [--device](https://docs.docker.com/engine/reference/commandline/run/#device) flag exposes a device in the host to the docker container.  In this case, the container needs access to the `/dev/kvm` device. So what is this KVM that's preventing the emulator pods from coming up?

KVM or [Kernel Virtual Machine](https://www.youtube.com/watch?v=BgZHbCDFODk) is a VM management tool that helps in creating Virtual Machines on Linux. Android Emulators are based on [QEMU](https://www.qemu.org/docs/master/about/index.html, an open-source machine emulator, which in turn utilizes the KVM infrastructure for virtualisation. To create an emulator, access to a host's KVM infrastructure is necessary.

Unfortunately, the current Kubernetes setup (Azure Kubernetes Service in this case) didn't have access to a KVM environment and using k8s was out of the question.

## Setting up emulators in Virtual Machines  

The next practical step was to spin up a Virtual Machine and run the emulator there. There was confusion about how to run the emulator, which is a virtual machine, inside another virtual machine. 

Turns out, most of the cloud providers support *nested virtualisation*, which allows creating nested virtual machines. A VM with nested virtualisation support was provisioned in Azure and verified KVM was available by running `ls /dev/kvm`.

If KVM is not available, you can install KVM it on your VM. The steps to install vary with distros. If you are running your VM in the cloud, you might need to check your cloud provider's documentation to see if KVM is supported on the machine type you choose (I'm using an Azure D-Series machine).

In VMs, you could either install the emulator directly or run the containerized version. Emulator containers checked all the boxes of my requirements — I could spin up multiple instances of emulators that use different Android versions, with low maintenance overhead, while keeping the emulator ephemeral.


## Creating Custom Android Image
After KVM availability is verified, install Docker in the Virtual Machine. You can follow the [official docker installation guide](https://docs.docker.com/engine/install/) to install Docker. Docker is a prerequisite for [android-emulator-container-scripts](https://github.com/google/android-emulator-container-scripts/). This tool helps in creating custom images with your preferred Android API version, architecture, Google APIs/Play store enabled and so on.


```bash
# android-container-script installation

## Install Python 3.11 & Git
sudo apt update
sudo apt install python3.11 python3.11-pip python3-venv git
python3 --version # Verify Python Version

## Installing emu-docker
git clone https://github.com/google/android-emulator-container-scripts.git
source ./configure.sh
emu-docker licenses --accept
emu-docker interactive
```

`emu-docker interactive` will show an interactive screen where you could choose the base image you want to use, the emulator version and whether to send usage statistics to Google.

![](https://i.imgur.com/Yjm9Iub.png)


Use the `docker run` command to start the emulator.
```bash
docker run \
  -e ADBKEY="$(cat ~/.android/adbkey)" \
  --device /dev/kvm \
  --publish 5555:5555/tcp  \
  <docker-image-name>
```

Here, we are publishing Port `5555`, the default `adb` port. `adb` clients could connect to this port to communicate with the emulator.

The container image also exposes a gRPC endpoint in port `8554`. Through this port, you could use your emulator in a web browser — a detailed guide is available in the [repo](https://github.com/google/android-emulator-container-scripts/tree/master?tab=readme-ov-file#running-the-emulator-on-the-web).

## Emulator Segmentation Fault in RHEL/CentOS

The Android emulator might complain of a Segmentation Fault and exit if you are running it on Red Hat Enterprise Linux, CentOS, AlmaLinux or any machine with SELinux enabled. This is because the emulator tries to execute from the heap, and the SELinux policy blocks it.

You could view the SELinux messages by reading the `/var/log/messages` file and check for any SELinux error. If it's the case, you could enable executing from heap by running `sudo setsebool -P selinuxuser_execheap 1` 

In summary, setting up Android Emulator Containers involved overcoming challenges with Kubernetes and opting for Virtual Machines with nested virtualisation. Using Google's container scripts and Docker, custom emulators were created, allowing flexibility in choosing Android versions and configurations. The process also required understanding and accessing Kernel Virtual Machine (KVM) infrastructure, and considerations for SELinux in certain environments. Overall, the journey provided me insights into efficiently deploying ephemeral emulators within a virtualised setting.

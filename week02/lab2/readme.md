# Customer Image creation  in Softlayer
With IBM® Cloud Virtual Servers image templates, you can capture a device's image to quickly replicate its configuration with minimal changes in the order process.

Image templates provide an imaging option for all Virtual Servers, regardless of operating system. Image templates allow you to capture an image of an existing virtual server and create a new one based on the captured image. 

When you request to create an image, the automated system of IBM Cloud™ takes your server offline. While the server is offline, a compressed backup of your data is created, the configuration information is recorded, and the image template is stored on the IBM Cloud SAN. During the deployment stage of the image template, the automated system constructs a new server that is based on the data that is gathered from the selected image. The deployment process makes adjustments for volume, restores the copied data, and then makes final configuration changes (for example, network configurations) for the new host.

### Provision the VM 
Provison a VSI with P-100 GPUs using Ubuntu 16 as the OS this will be the base virtual server for the image creation

```
# replace the things in <> with your own values
ibmcloud sl vs create --datacenter=dal13 --hostname=<hostname> --domain=<domain> --os=UBUNTU_16_64 --flavor AC1_16X120X25 --billing=hourly --san --disk=25 --disk=2000 --network 1000 --key=<your SL key>

# for instance, this is what I did:
ibmcloud sl vs create --datacenter=dal13 --hostname=p100 --domain=dima.com --os=UBUNTU_16_64 --flavor AC1_16X120X25 --billing=hourly --san --disk=25 --disk=2000 --network 1000 --key=p305
``` 
### Install cuda
As of right now, 10.1 is the latest version.  Check https://developer.nvidia.com/cuda-toolkit  for the latest.
```
wget https://developer.nvidia.com/compute/cuda/10.1/Prod/local_installers/cuda-repo-ubuntu1604-10-1-local-10.1.105-418.39_1.0-1_amd64.deb
dpkg -i cuda-repo-ubuntu1604-10-1-local-10.1.105-418.39_1.0-1_amd64.deb


# the cuda 10.1 key
apt-key add /var/cuda-repo-10-1-local-10.1.105-418.39/7fa2af80.pub

# install it!
apt-get update
apt-get install -y cuda
```
If you have a dependency on cuda 10.0 (e.g. DeepStreamSDK), you will have to install these instead:
```
wget https://developer.nvidia.com/compute/cuda/10.0/Prod/local_installers/cuda-repo-ubuntu1604-10-0-local-10.0.130-410.48_1.0-1_amd64
dpkg -i cuda-repo-ubuntu1604-10-0-local-10.0.130-410.48_1.0-1_amd64 . 

# the cuda 10.0 key
apt-key add /var/cuda-repo-10-0-local-10.0.130-410.48/7fa2af80.pub

# install it!
apt-get update
apt-get install -y cuda
```

### Install docker
Validate these at https://docs.docker.com/install/linux/docker-ce/ubuntu/
```
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
	
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"	

apt-get update


# Dima validated on 01/13/19 that this below is still required; sigh
# apt-get install docker-ce=5:18.09.0~3-0~ubuntu-xenial
# As of 2/24/19 this works now
apt-get install -y docker-ce

# verify

docker run hello-world
```

### Install nvidia-docker (version 2)
First, add the package repositories
```
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
  sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
apt-get update
```

Now, install nvidia-docker2 and reload the Docker daemon configuration
```
apt-get install -y nvidia-docker2
pkill -SIGHUP dockerd
```
Test nvidia-smi with the latest official CUDA image
```
docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi
```
Hopefully, you will see your GPUs.  
### Prepare the second disk
What is it called?
```
fdisk -l
```
You should see your large disk, something like this
```
Disk /dev/xvdc: 2 TiB, 2147483648000 bytes, 4194304000 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
In this case, our disk is called /dev/xvdc.  Your disk may be named differently.  Format it:
```
# first
mkdir -m 777 /data
mkfs.ext4 /dev/xvdc
```

Add to /etc/fstab
```
# edit /etc/fstab and all this line:
/dev/xvdc /data                   ext4    defaults,noatime        0 0
```
Mount the disk
```
mount /data
```
### Move the working Docker directory
By default, docker stores its images under /var/lib/docker , which will quickly fill up.  So,
```
service docker stop
cd /var/lib
cp -r docker /data
rm -fr docker
ln -s /data/docker ./docker
service docker start
```

	
## Image creation
Access the IBM® Cloud infrastructure customer portal [portal](https://control.softlayer.com/) by using your unique credentials.
1- From the Devices menu, select Device List.
2- Click the virtual server that you want to use to create an image template.
3- Check the Passwords tab of the Device Details page. Ensure that any passwords listed on the Device Details page match the actual operating system passwords and any other software add-on passwords. If passwords do not match, virtual servers that are created from this image template fail.
4- From the Actions menu, select Create Image Template.
5- Enter the new name for the image in the Image Name field.
6- Enter any necessary notes for the image in the Note field.
7- Select the Agree check box when all information is entered.
8- Click Create Template to create the image template.

## Ordering an instance from an image template
Access the IBM® Cloud infrastructure customer portal [portal](https://control.softlayer.com/) by using your unique credentials.
1- access the Image Templates page by selecting Devices > Manage > Images.
2- Click the Actions menu for the image template that you want to use and select the type of virtual server that you want to order.
3- On the Configure your Cloud Server page, complete all of the relevant information.
4- Click the Add to Order button to continue.
5- On the Checkout page, complete any advanced system configuration.
6- Click the Cloud Service terms and the Third-Party Service Agreement check boxes.
7- Confirm or enter your payment information and click Submit Order. You are redirected to a screen with your provisioning order number. You can print the screen because it's also your provisioning order receipt.

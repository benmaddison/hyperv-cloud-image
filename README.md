# Ubuntu Cloud-Image on Windows Server Hyper-V 2008R2

Using cloud images with hyper-v is not straightforward, so we are
documenting the process here, in case someone else wants to do this.

A better approach might be to not need to do this in the first place :-)

## The short version

The standard Ubuntu cloud-images (`*.img`) won't boot on Hyper-V 2008R2.
The customised kernels, etc, appear to be available in the Azure images
(`*.vhd.zip`). Unfortunately, these have `cloud-init` configured to use
the `Azure` datasource only.

We want to use the `NoCloud` datasource, so we had to mount and customise
the Azure image to play nicely.

## Prep the image

We'll assume that you are doing this from the root directory of a clone of this repo.
Some commands assume that you're on Ubuntu. Adjust commands for your OS as appropriate.

1. Get the release that you want (you'll need the raw and vhd versions):
```bash
wget -P images/ \
    https://cloud-images.ubuntu.com/releases/18.04/release/ubuntu-18.04-server-cloudimg-amd64.img
wget -P images/ \
    https://cloud-images.ubuntu.com/releases/18.04/release/ubuntu-18.04-server-cloudimg-amd64.vhd.zip
unzip ubuntu-18.04-server-cloudimg-amd64.vhd.zip -d images/
```

2. (*Optional*) Convert the image to a dynamic VHD:
```bash
qemu-img convert images/bionic-server-cloudimg-amd64.vhd \
    -O vpc -o subformat=dynamic \
    images/bionic-server-cloudimg-amd64-dynamic.vhd
```

3. Customise `meta-data.yml` and `user-data.yml` if necessary.

4. Create a `NoCloud` compatible iso image:
```bash
cloud-localds cidata.iso user-data.yml meta-data.yml
```

5. Create a boot image from the **raw** image:
```bash
qemu-img create \
    -f qcow2 -b ubuntu-18.04-server-cloudimg-amd64.img \
    images/boot.img
```

6. Boot a VM from your created boot image, with the `NoCloud` drive and your 
   dynamic VHD image as additional block devices:
```bash
sudo kvm -m 1024 -net nic -net user,hostfwd=tcp::2222-:22 \
    -drive file=images/boot.img,if=virtio \
    -drive file=cidata.iso,if=virtio \
    -drive file=images/bionic-server-cloudimg-amd64-dynamic.vhd,if=virtio
```

7. (*In another terminal*) Connect to your VM, using the password defined in `user-data.yml`:
```bash
ssh -p 2222 ubuntu@localhost
```

8. Mount the VHD image and make the necessary changes:
```bash
sudo mount /dev/vdc1 /mnt/
# disable the walinuxagent service
sudo rm /mnt/etc/systemd/system/multi-user.target.wants/walinuxagent.service
# disable the azure-specific cloud-init config
sudo rm /mnt/etc/cloud/cloud.cfg.d/90-azure.cfg
echo "datasource_list: [ NoCloud, None ]" | sudo tee /mnt/etc/cloud/cloud.cfg.d/90_dpkg.cfg > /dev/null
# disable the azure-specific netplan customisations
sudo rm /mnt/etc/netplan/90-hotplug-azure.yaml
# unmount the vhd and shutdown the vm
sudo umount /mnt
sudo shutdown -h now
```

You should now have a VHD image in `images/bionic-server-cloudimg-amd64-dynamic.vhd`
that can be used in a VMM 2008R2 template.



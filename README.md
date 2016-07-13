# convert-sandbox-vagrant

# Objective
This tutorial is intended to describe the process of converting the Hortonworks HDP 2.5 Technical Preview sandbox, which is a VirtualBox virtual machine, into a Vagrant box.  

# Overview
Given the limited resources of a single virtual machine, it can be difficult to run and manage multiple demo environments within a single virtual machine.  It is a common desire to add additional software components to the sandbox.  Adding additional components often means having to manually manage port forwarding rules in VirtualBox or you have to add an additional network interface to the virtual machine to directly access it via an IP address using either host-only or bridged networking.  Given the steps involved, it is common to do this once and try to reuse the same virtual machine.  This may not always be ideal.

Using Vagrant, you can quickly create a clean sandbox with a simple command like `vagrant init HDP25TP && vagrant up`.  Vagrant makes it easy to connect to the virtual machine via `vagrant ssh`.  Vagrant will also auto-mount the host vagrant/project directory to /vagrant on the virtual machine.  This makes it incredibly easy to add files or data to the virtual machine by simply copying the data to the mounted directory.

The addition of the Vagrant plugin vagrant-vbguest automates the installation and maintenance of the VirtualBox Guest Additions software.  Out of the box, the sandbox does not provide the VirtualBox Guest Additions.  You can add the additions yourself, however, it is easy for the additions to stop working whenvever you update the virtual machine operating system kernel via `yum upgrade` or whenever you update the installed version of VirtualBox.  The plugin will automatically handle this for you everytime you run `vagrant up`.

# Scope
This tutorial was created and validated on Mac OS X.  However, the overall process should work for Windows.  This tutorial has been tested with the following configuration:
* Mac OS X 10.11.5
* VirtualBox 5.0.24
* Vagrant 1.8.4
* vagrant-vbguest 0.12.0 

# Prerequisites
* VirtualBox for Mac OS X installed
  * [VirtualBox Link](https://www.virtualbox.org/wiki/Downloads)​
* Vagrant software for Mac OS X installed
  * [Vagrant Link](https://www.vagrantup.com/downloads.html)
* Vagrant plugin vagrant-vbguest installed
  * [Vagrant Plugin Link](https://github.com/dotless-de/vagrant-vbguest)
* Download Hortonworks HDP 2.5 Technical Preview Sandbox
  * [Sandbox Link](http://hortonworks.com/downloads/#tech-preview)

# Steps
## Prepare The Sandbox Virtual Machine
Before creating the Vagrant box from the HDP sandbox, there are some configuration changes we need to complete.

* Import the Hortonworks HDP 2.5 Technical Preview sandbox OVA file into VirtualBox. You should be able to use the GUI or the command line.
  * **NOTE: When I use the command line, the eth0 network device would not work properly.  I believe this is a VirtualBox issue as this problem does not happen when you use the GUI to import the OVA file.**
  ```bash
  $ VBoxManage import ./"Hortonworks Sandbox with HDP 2.5 Technical Preview.ova" --vsys 0 --eula accept

  0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
  Interpreting /Users/username/Downloads/VirtualBox Appliances/./Hortonworks Sandbox with HDP 2.5 Technical Preview.ova...
  OK.
  Disks:
  vmdisk1	52428800000	-1	http://www.vmware.com/interfaces/specifications/vmdk.html#streamOptimized	Hortonworks Sandbox with HDP 2.5 Technical Preview-disk1.vmdk	-1	-1

  Virtual system 0:
  0: Suggested OS type: "RedHat_64"
      (change with "--vsys 0 --ostype <type>"; use "list ostypes" to list all possible values)
  1: Suggested VM name "Hortonworks Sandbox with HDP 2.5 Technical Preview"
      (change with "--vsys 0 --vmname <name>")
  2: Number of CPUs: 4
      (change with "--vsys 0 --cpus <n>")
  3: Guest memory: 8192 MB
      (change with "--vsys 0 --memory <MB>")
  4: USB controller
      (disable with "--vsys 0 --unit 4 --ignore")
  5: Network adapter: orig NAT, config 2, extra slot=0;type=NAT
  6: IDE controller, type PIIX4
      (disable with "--vsys 0 --unit 6 --ignore")
  7: IDE controller, type PIIX4
      (disable with "--vsys 0 --unit 7 --ignore")
  8: Hard disk image: source image=Hortonworks Sandbox with HDP 2.5 Technical Preview-disk1.vmdk, target path=/Users/username/VirtualBox VMs/Hortonworks Sandbox with HDP 2.5 Technical Preview/Hortonworks Sandbox with HDP 2.5 Technical Preview-disk1.vmdk, controller=6;channel=0
      (change target path with "--vsys 0 --unit 8 --disk path";
      disable with "--vsys 0 --unit 8 --ignore")
  0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
  Successfully imported the appliance.
  ```
* Start the imported virtual machine.  You can use the GUI or the command line.
  ```
  $ VBoxManage startvm "Hortonworks Sandbox with HDP 2.5 Technical Preview"

  Waiting for VM "Hortonworks Sandbox with HDP 2.5 Technical Preview" to power on...
  VM "Hortonworks Sandbox with HDP 2.5 Technical Preview" has been successfully started.
  ```

* The sandbox requires you to reset the root password upon initial login. Change the password when prompted to a secure password. CentOS will not let you use "hadoop" at this stage.

* For convenience, you can reset the password back to "hadoop" so that it is consistent with the VM banner displayed after bootup. This is very handy after a couple of months has passed and you can't remember what you set the password to.
Change the password back to "hadoop"
  ```
  # passwd
  ```

* For a new sandbox, the Ambari admin user password needs to be reset to enable the admin account. I recommend you set it something easy to remember like 'admin' or 'password'.
  * **NOTE: You would not use a simple password for production systems.**
  ```
  # ambari-admin-password-reset
  ```

* Add the vagrant user. This user is what Vagrant expects by default.
  ```
  # useradd vagrant -d /home/vagrant
  ```

* Set the vagrant user password to "vagrant".
  ```
  # passwd vagrant
  ```

* The vagrant user needs to be able to run sudo.
  * **NOTE: You can also create a file named "vagrant" in /etc/sudoers.d, however, the requiretty line is in /etc/sudoers so it may be more convenient to simply modify /etc/sudoers.**
  ```
  # visudo
  ```

  * Add an additional "env_keep" setting after the existing "env_keep" settings.
  ```
  Defaults env_keep += "SSH_AUTH_SOCK"
  ```

  * Add vagrant no password setting at the end of the file.
  ```
  vagrant ALL=(ALL) NOPASSWD:ALL
  ```

  * Ensure the "Defaults requiretty" line is commented out.
  ```
  #Defaults requiretty
  ```

  * Save the file
  ```
  !wq
  ```

* Now switch to the vagrant user
  ```
  # su - vagrant
  ```

* Test sudo as the vagrant user.  You should not be prompted for a password.
  ```
  $ sudo ls
  ```

* Create a .ssh directory for the vagrant user
  ```
  $ cd
  $ mkdir .ssh
  ```

* Download the vagrant public key and save it as "authorized_keys" in the .ssh directory
  ```
  $ wget --no-check-certificate https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub -O /home/vagrant/.ssh/authorized_keys
  ```

* Change the permissions of the .ssh directory and authorized_key files or ssh will not work properly.
  ```
  $ chmod 0700 .ssh
  $ chmod 0600 .ssh/authorized_keys
  ```
  
* Shutdown the virtual machine
  ```
  $ sudo init 0
  ```
## Create the Vagrant Box
If everything worked correctly, you are ready to create a Vagrant box using the modified virtual machine.

* Make a directory for creating the box
  ```
  $ mkdir create-sandbox && cd create-sandbox
  ```

* Initialize a default Vagrantfile
  ```
  $ vagrant init
  ```

* Edit the Vagrantfile and replace everythng in the file with the following:
  ```ruby
  # -*- mode: ruby -*-
  # vi: set ft=ruby :

  vagrant.configure("2") do |config|
    config.vm.box = "HDP25TP" # name of the base vagrant box, should line up with the --name option used with vagrant box add command
    config.vm.box_check_update = false # don't update the box version on boot
    config.vm.network :private_network, type: "dhcp", :adapater=>2 # add a private network to avoid having to forward ports
    config.vm.provider "virtualbox" do |vb|
      vb.customize ["modifyvm", :id, "--cpuexecutioncap", "75"] # set maximum host cpu usage to 75%
      vb.cpus = "4" # set the number of cpus to 4
      vb.memory = "8192" # set the memory to 8GB
    end
  end
  ```

* Get the unique ID of the virtual machine:
  ```
  $ VBoxManage list vms

  "Hortonworks Sandbox HDP 2.4 IoT Demo" {c9db5b78-554f-41d5-beba-a27d54ee42db}
  "Hortonworks Sandbox HDP 2.4 NiFi Demo" {3abe96ef-258b-4395-b9bd-590b1685132d}
  "CentOS 6.8 Minimal" {bc5178e9-061f-47c8-897d-c424aa3a50de}
  "hdp24_ldap" {b8e1f3ad-6c06-4e8a-849a-0249611a607a}
  "hdp24_sandbox" {125e8108-21e6-48ce-87f8-a065a50cde3a}
  "default" {3a471444-fb24-40d3-9954-c53d9ea03895}
  "Hortonworks Sandbox with HDP 2.5 Technical Preview" {e250b938-bece-414a-ba8a-77972959b5f8}
  ```

* Look for the ID value inside the {} for the virtual machine named "Hortonworks Sandbox with HDP 2.5 Technical Preview"
  ```
  "Hortonworks Sandbox with HDP 2.5 Technical Preview" {e250b938-bece-414a-ba8a-77972959b5f8}
  ```

* Package the vagrant box using the ID from the previous step.
  * **NOTE: Specifying the --vagrantfile option ensures the settings are applied at boot without having to specify them in every Vagrantfile you create. The --output option specifies the name of the box file.**
  ```
  $ vagrant package --base e250b938-bece-414a-ba8a-77972959b5f8 --output hdp25tp.box --vagrantfile Vagrantfile

  ==> e250b938-bece-414a-ba8a-77972959b5f8: Clearing any previously set forwarded ports...
  ==> e250b938-bece-414a-ba8a-77972959b5f8: Exporting VM...
  ==> e250b938-bece-414a-ba8a-77972959b5f8: Compressing package to: /Users/username/Vagrant/create-sandbox/hdp25tp.box
  ==> e250b938-bece-414a-ba8a-77972959b5f8: Packaging additional file: Vagrantfile
  ```

* Add the newly created box to Vagrant
  * **NOTE: The --name option can be anything you like.  Use something intuitive.**
  ```
  $ vagrant box add hdp25tp.box --name HDP25TP

  ==> box: Box file was not detected as metadata. Adding it directly...
  ==> box: Adding box 'HDP25TP' (v0) for provider:
  box: Unpacking necessary files from: file:///Users/username/Vagrant/create-sandbox/hdp25tp.box
  ==> box: Successfully added box 'HDP25TP' (v0) for 'virtualbox'!
  ```

* Create a demo project directory
  ```
  $ cd /vagrant-projects
  $ mkdir demo1 && cd demo1
  ```

* Initialize the Vagrant environment
  * **NOTE: If you specify the name used with the --name option above, you won't have to edit the Vagrantfile.**
  ```
  $ vagrant init HDP25TP

  A `Vagrantfile` has been placed in this directory. You are now
  ready to `vagrant up` your first virtual environment! Please read
  the comments in the Vagrantfile as well as documentation on
  `vagrantup.com` for more information on using Vagrant.
  ```

* Start the vagrant machine
  * **NOTE: During the first boot of a new vagrant virtual machine, you will notice the VirtualBox Guest Additions will be installed.**
  ```
  $ vagrant up

  Bringing machine 'default' up with 'virtualbox' provider...
  ==> default: Importing base box 'HDP25TP'...
  ==> default: Matching MAC address for NAT networking...
  ==> default: Setting the name of the VM: demo1_default_1468377822096_63592
  ==> default: Clearing any previously set network interfaces...
  ==> default: Preparing network interfaces based on configuration...
  default: Adapter 1: nat
  default: Adapter 2: hostonly
  ==> default: Forwarding ports...
  default: 22 (guest) => 2222 (host) (adapter 1)
  ==> default: Running 'pre-boot' VM customizations...
  ==> default: Booting VM...
  ==> default: Waiting for machine to boot. This may take a few minutes...
  default: SSH address: 127.0.0.1:2222
  default: SSH username: vagrant
  default: SSH auth method: private key
  default:
  default: Vagrant insecure key detected. Vagrant will automatically replace
  default: this with a newly generated keypair for better security.
  default:
  default: Inserting generated public key within guest...
  default: Removing insecure key from the guest if it's present..
  ==> default: Machine booted and ready!
  [default] No installation found.
  Loaded plugins: fastestmirror, riorities
  Setting up Install Process
  Determining fastest mirrors
  * base: mirror.vcu.edu
  * epel: lug.mtu.edu
  * extras: mirror.vcu.edu
  * updates: mirrors.cat.pdx.edu
  Package kernel-devel-2.6.32-642.1.1.el6.x86_64 already installed and latest version
  Package gcc-4.4.7-17.el6.x86_64 already installed and latest version
  Package binutils-2.20.51.0.2-5.44.el6.x86_64 already installed and latest version
  Package 1:make-3.81-23.el6.x86_64 already installed and latest version
  Package 4:perl-5.10.1-141.el6_7.1.x86_64 already installed and latest version
  Package bzip2-1.0.5-7.el6_0.x86_64 already installed and latest version
  Nothing to do
  Copy iso file /Applications/VirtualBox.app/Contents/MacOS/VBoxGuestAdditions.iso into the box /tmp/VBoxGuestAdditions.iso
  Installing Virtualbox Guest Additions 5.0.24 - guest version is unknown
  Verifying archive integrity... All good.
  Uncompressing VirtualBox 5.0.24 Guest Additions for Linux............
  VirtualBox Guest Additions installer
  Copying additional installer modules ...
  Installing additional modules ...
  Removing existing VirtualBox non-DKMS kernel modules[ OK ]
  Building the VirtualBox Guest Additions kernel modules
  Building the main Guest Additions module[ OK ]
  Building the shared folder support module[ OK ]
  Building the graphics driver module[ OK ]
  Doing non-kernel setup of the Guest Additions[ OK ]
  Starting the VirtualBox Guest Additions Installing the Window System drivers
  Could not find the X.Org or XFree86 Window System, skipping.
  [ OK ]
  ==> default: Checking for guest additions in VM...
  ==> default: Configuring and enabling network interfaces...
  ==> default: Mounting shared folders...
  default: /vagrant => /Users/username/Vagrant/demo1
  ```

* SSH into the vagrant machine
  ```
  $ vagrant ssh
  ```

* Verify the network interfaces:
  * **NOTE: You should note there are two ethernet devices. The first device, eth0, is configured with NAT for internet access. The second device, eth1, is configured with host-only networking. Note the IP address of of eth1. This is the IP address you will use to interact with the sandbox web interfaces instead of setting up all of the port forwarding.  The IP address for eth1 will depend on your specific VirtualBox settings.  You can modify these setting to change the host-only network to use different IP addresses.**
  ```
  $ ifconfig -a

  eth0      Link encap:Ethernet  HWaddr 08:00:27:8B:B4:D2
            inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
            UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
            RX packets:725 errors:0 dropped:0 overruns:0 frame:0
            TX packets:494 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1000
            RX bytes:80100 (78.2 KiB)  TX bytes:63763 (62.2 KiB)
            Interrupt:19 Base address:0xd020

  eth1      Link encap:Ethernet  HWaddr 08:00:27:C8:EB:D8
            inet addr:172.28.128.3  Bcast:172.28.128.255  Mask:255.255.255.0
            UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
            RX packets:3 errors:0 dropped:0 overruns:0 frame:0
            TX packets:7 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1000
            RX bytes:1770 (1.7 KiB)  TX bytes:1266 (1.2 KiB)

  lo        Link encap:Local Loopback
            inet addr:127.0.0.1  Mask:255.0.0.0
            UP LOOPBACK RUNNING  MTU:65536  Metric:1
            RX packets:3 errors:0 dropped:0 overruns:0 frame:0
            TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:0
            RX bytes:165 (165.0 b)  TX bytes:165 (165.0 b)
  ```

* On the host machine, `ls` the vagrant project directory
  ```
  $ ls /vagrant-projects/demo1

  Vagrantfile
  ```

* Create a test file on the host machine
  ```
  touch /vagrant-projects/demo1/local-file.txt
  ```

* On the vagrant virtual machine, verify you can see the new file
  ```
  $ ls /vagrant
  
  Vagrantfile local-file.txt
  ```

* Create a test file on the Vagrant virtual machine
  ```
  $ touch /vagrant/vm-file.txt
  ```

* On the host machine, verify you can see the new file
  ```
  $ ls /vagrant-projects/demo1/

  Vagrantfile local-file.txt vm-file.txt
  ```

* Exit the vagrant virtual machine
  ```
  $ exit
  ```

* Shutdown the Vagrant virtual machine
  ```
  $ vagrant halt
  ```

During the Vagrant setup process, Vagrant will have cleared the forwarded ports for the Vagrant virtual machine.  This also cleared the ports of the sandbox virtual machine we initially imported into VirtualBox.  The sandbox virtual machine is no longer needed, so you can delete it from VirtualBox.

## Review
If all of the steps completed successfully, you now have the ability to create demo virtual machines as needed with a couple of simple commands.

```
mkdir <demo directory> && cd <demo directory>
vagrant init HDP25TP
vagrant up
vagrant ssh
```

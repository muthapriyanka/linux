
1. For each member in your team, provide 1 paragraph detailing what parts of the lab that member implemented / researched.

#### **Priyanka Jadhav's Contribution**
Priyanka Jadhav managed the setup of the GCP environment by resizing disk space, configuring virtual machine resources, and installing required dependencies for the kernel build. She also handled the cloning of the Linux kernel repository, generating certificates for secure boot, and compiling the initial kernel. Priyanka documented the steps involved in setting up the environment, ensuring clarity for reproducibility. She worked collaboratively on analyzing kernel exits, contributing to the logging and monitoring of VM operations.

#### **Priyanka Mutha's Contribution**
Priyanka Mutha focused on modifying the kernel source code, including updates to the __vmx_handle_exit function. She worked on rebuilding the kernel, installing the updated modules, and updating the GRUB configuration for seamless kernel integration. Additionally, Priyanka managed the reboot and verification process for the new kernel. She shared responsibility for analyzing the frequency of kernel exits, ensuring accurate documentation of findings, and addressing challenges encountered during the nested VM testing phase.


2. Describe in detail the steps you used to complete the assignment

Prerequisite: Ensure GCP Disk Size greater than 20 GB

Steps to increase size :

Increase Disk Size and core (GCP)

Stop the instance:

gcloud compute instances stop <INSTANCE_NAME> --zone <ZONE>

Resize the disk to ensure sufficient space:

gcloud compute disks resize DISK_NAME --size=SIZE_GB --zone=ZONE

Upgrade the machine type for additional CPU cores:

gcloud compute instances set-machine-type <INSTANCE_NAME> --zone <ZONE> --machine-type n1-standard-2

Restart the instance:

gcloud compute instances start <INSTANCE_NAME> --zone <ZONE>


Step 1: Kernel Build Test

1.1 Fork the Linux GitHub Repository

●	Go to https://github.com/torvalds/linux and click the Fork button.

●	Fork the repository into your own GitHub account.

1.2 Clone the Repository in Your Outer VM

●	SSH into your outer VM.

●	Install Git : sudo apt install git -y

●	Clone your forked repository:

git clone https://github.com/<your_username>/linux.git

cd linux

1.3 Configure the Kernel Build

●	Install the necessary tools to build the Linux kernel:

sudo apt update

sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev

●	Configure the kernel:

make menuconfig

●	Install bc (Basic Calculator Tool): Run the following command to install bc:

sudo apt update

sudo apt install bc -y

1.4 Build the Kernel

●	Generate a dummy certificate:

mkdir -p debian/certs

Generate Certificate:

openssl req -new -x509 -newkey rsa:2048 -sha256 -nodes -days 365 -keyout debian/certs/debian-uefi-certs.key -out debian/certs/debian-uefi-certs.pem -subj "/CN=Dummy Debian Secure Boot Certificate"

Generate a Signing Key

Generate a private key and CSR for signing:

openssl req -new -newkey rsa:2048 -days 3650 -nodes -keyout debian/certs/debian-uefi.key -out debian/certs/debian-uefi.csr

Self-sign the certificate for 10 years:

openssl x509 -req -days 3650 -in debian/certs/debian-uefi.csr -signkey debian/certs/debian-uefi.key -out debian/certs/debian-uefi-certs.pem

●	Install lz4 using apt:

sudo apt update

sudo apt install lz4 -y

●	Build the kernel :

make -j$(nproc)

Sudo make modules_install

Sudo make install

Generate the Initial RAM Disk

sudo update-initramfs -c -k $(make kernelrelease)

●	The kernel files will be installed in /boot.

1.5 Take a Snapshot

●	Before rebooting, take a snapshot of your outer VM to avoid any issues:

○	Use your cloud platform or virtualization tool's snapshot feature.

1.6 Test the New Kernel

●	Reboot into the new kernel:

sudo update-grub

sudo reboot

●	Verify the running kernel version:

uname -r

Ensure it matches the version of the kernel you built.

Ensure that the necessary KVM modules are present:

ls /lib/modules/$(uname -r)/kernel/arch/x86/kvm/



Step 2: Modify the KVM Code

Step 2.1: Locate the Exit Handler

The function that handles VM exits in the KVM source code is architecture-dependent.

Note : For AMD processors (SVM), the exit handler is typically svm_handle_exit

For Intel processors (VMX), the equivalent function is likely _vmx_handle_exit, found in arch/x86/kvm/vmx.c.

●	Use grep to locate the function handle_exit or a similarly named function. This function processes VM exits.

1. Update the Exit Handler Logic

In the __vmx_handle_exit function, insert your logic to:

Increment counters for the total exits and specific exit types.
Log statistics every 10,000 exits.

To make the logs more informative, map common exit reasons to human-readable names:

4. Rebuild and Install the Kernel

Clean up the build directory to ensure no stale files:

make clean

Rebuild and install the kernel:

1.	Rebuild the Kernel:

make -j$(nproc)

2.	Install Kernel Modules:

sudo make modules_install

3.	Install the Kernel:

sudo make install

sudo update-grub

sudo reboot



Step 3 : Test the Changes

1.	 Boot the Inner VM using your modified KVM code:

sudo qemu-system-x86_64 -enable-kvm -hda focal-server-cloudimg-amd64.img -m 512 -nographic -netdev user,id=net0,hostfwd=tcp::2222-:22 -device e1000,netdev=net0

2.	Generate VM Exits in the Inner VM:

Perform activities like:

○	High CPU usage: while true; do :; done

○	Disk I/O: dd if=/dev/zero of=testfile bs=1M count=100 rm testfile

○	Network traffic: ping -c 100 google.com

3.   Check Kernel Logs: On the outer VM, view the statistics in the kernel logs:

cd /linux

sudo dmesg | grep "KVM Exit"



3. Comment on the frequency of exits – does the number of exits increase at a stable rate? Or are there more exits performed during certain VM operations? Approximately how many exits does a full VMboot entail?

Stable Rate vs. Peaks

●	The frequency of exits varies during different VM operations:

●	MSR WRITE exits occur consistently in high numbers.

●	EXTERNAL INTERRUPT exits are regular but less frequent.

●	Memory-related exits like EPT MISCONFIG occur during specific operations.

Exits During Specific Operations

●	Memory Operations: Memory management triggers exits like EPT MISCONFIG and EPT VIOLATION.

●	Interrupt Handling: Frequent external interrupts are observed.

●	Rare Events: Exits like UNKNOWN EXIT and HLT occur during unique cases.

Total Exits During a Full VM Boot

The total number of exits is approximately 1,000,000, including:

●	MSR WRITE: Hundreds of thousands.

●	EXTERNAL INTERRUPT: Several thousand.

●	Other Exit Types: Cumulatively smaller contributions.



4. Of the exit types, which are the most frequent? Least?

Most Frequent Exits

1.	MSR WRITE: 669,483 occurrences.

2.	EXTERNAL INTERRUPT: 5,975 occurrences.

Least Frequent Exits

1.	UNKNOWN EXIT: 1–2 occurrences.

2.	EXCEPTION NMI: 11 occurrences.

3.	HLT (Halt Instruction): 1 occurrence.


![App Screenshot](https://github.com/muthapriyanka/linux/blob/master/Screenshots/Assignment2_result.png)

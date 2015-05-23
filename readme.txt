###############################################################################
# Copyright (c) 2015 Intel Corporation
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
###############################################################################


This readme covers instructions for building and launching iKGT framework and 
example usage (integrity) components.

=============================================================================
Content of the Source Package
=============================================================================

ikgt/
¦ 
+---Readme/
¦ 
+---xmon/
¦   +---Makefile
¦   +---common/            /* Common header files */
¦   ¦   +---include/       
¦   ¦ 
¦   +---core/		   /* generic xmon core */
¦   ¦   +---common/
¦   ¦   +---include/
¦   ¦   +---vmexit/
¦   ¦   +---vmx/
¦   ¦   +---
¦   +---api/               
¦   ¦     
¦   +---package/           /* rpm/deb packages, scripts */
¦   ¦ 
¦   +---loader/            /* pre-os xmon loader */
|   ¦   
¦   +---plugin/            /* xmon-plugin for supporting integrity usecase */
¦       +---ikgt-plugin      
¦ 
+---example-usage/
¦ 
¦   +---integrity/
¦       +---Makefile      
¦       +---policy/        /* example policy (.json) and install script */
¦       +---driver/        /* example driver to configure policy        */
¦       +---handler/       /* example vmx-root policy plugin module     */
¦      

=============================================================================
Build environment
=============================================================================

Ubuntu 12.04/14.04 64-bit
Gcc version 4.6.3 (Ubuntu/Linaro 4.6.3-1ubuntu5)

=============================================================================
Target machine Configuration
=============================================================================

The target machine is where iKGT is to be installed.

(1) System Configuration

    - Intel Core i processors (3rd generation or newer)
    - OS: Linux (e.g. Ubuntu)
    - System memory: at least 2GB

    Note: For target machines with less than 2GB RAM, xmon launching may fail.
          Please refer to "Customization" section for more details.

=============================================================================
Building iKGT binaries
=============================================================================

On a Linux build machine,

(1) Clone the ikgt source files into a working directory (e.g. project) from
    GitHub
   $ mkdir project
   $ cd project
   $ repo init -u https://github.com/01org/ikgt-manifest.git
   $ repo sync

(2) Build xmon binaries:
   $ cd ikgt/example-usage/integrity
   $ make

 Output files can be found in the directories shown below:

    ikgt/xmon/bin/linux/release/ikgt_pkg.bin  - contains xmon loader, 
                                                xmon and policy handler
    ikgt/usage/integrity/driver/ikgt_agent.ko - sample driver


=============================================================================
Installing/running iKGT vmx-root component
=============================================================================

Currently, xmon can be launched by the target system which uses GRUB loader
(e.g. Ubuntu). To install xmon,

(1) Type below commands:
   $ cd ikgt/xmon
   $ sudo make install

This step will copy the xmon binaries to /boot folder and update the GRUB
configuration file for xmon boot.

(2) Enable the GRUB menu countdown display so that it gives chance
    during boot to select boot entry.

   $ sudo vi /etc/default/grub
   comment the following two lines with #
        GRUB_HIDDEN_TIMEOUT=0
        GRUB_HIDDEN_TIMEOUT_QUIET=true

   modify the timeout value if necessary.
        GRUB_TIMEOUT=20

   Save the changes and quit.

This step is needed only when you want to have the chance to select the boot
entry on the GRUB menu during the boot.

(3) Reboot the system
   $ sudo reboot

Linux should come up with xmon running under it in vmx-root

=============================================================================
Verifying if iKGT is up and running
=============================================================================

Check cpuid and verify Intel VT-x is being used.

    $ ./check_vtx.sh

    If it returns "VTx is not available", it implies the processor VT-x feature
    is being used.

Check if xmon is running by executing the following utility:

    $ ikgt/xmon/package/check_ikgt

    If xmon is running, the utility will print out "iKGT is running".
    Xmon runs silently under the existing OS de-privileging it.

=============================================================================
Uninstalling iKGT vmx-root component
=============================================================================
On the target machine,

    $ cd ikgt/xmon
    $ sudo make uninstall

   This command will delete "ikgt_pkg.bin" in /boot and "20_linux_xmon"
   in /etc/grub.d, restore the changes to grub configuration files.

=============================================================================
Setting up configfs and installing ikgt_agent.ko
=============================================================================

Follow these steps to set up the configfs filesystem under /config

(1) $ sudo mkdir /config

If configfs driver is not installed:
    (2) sudo insmod \
        /lib/modules/<installed-kernel-version>/kernel/fs/configfs/configfs.ko

(3) $ sudo mount -t configfs none /config

(4) $ sudo insmod ikgt_agent.ko

After successful installation, the driver will create /config/ikgt_agent as its 
configuration space. The resource to be monitored and policy actions can now 
be specified by creating directories and files in this space. 

============================================================================
Installing example policy
=============================================================================
Use python script to parse policy file and create configfs directories.
The python script is under ikgt/example-usage/integrity/policy/parse_policy.py. 

sudo python parse_policy.py -f <policy_file> -b <base_dir> 
where 
   <policy_file> is the JSON file,
   <base_dir> is the base directory under which the resource directories 
              are to be created

    $ sudo python parse_policy.py -f policy.json -b /config/ikgt_agent

You can check the new entries in configfs by executing following command
    $ tree /config
       
Above command should create directories and files based on the contents 
of .json file. The example policy enables monitoring of following resources 
with actions as shown below:
		CR0:WP  -  LOG & ALLOW
		CR0:PG  -  LOG & SKIP
		CR4:PAE -  LOG & SKIP
		MSR:EFER - LOG & SKIP


============================================================================
Testing policy enforcement
=============================================================================

After the policy is successfully installed, all attempts to modify monitored 
resources will be controlled as per the actions specified against them.

For example, if the OS tries to modify CR0:WP, the event will be logged
but will be allowed. Similarly, if the OS tries to modify EFER, the event will 
be logged and the violating instruction will be skipped.

The contents of the log can be seen in /config/ikgt-agent/log/log.txt

$cat /config/ikgt_agent/log/log.txt

            0,5,28,700,FFFFFFFF81050414,0

For detailed instructions on how to set up configfs space and how to interpret 
log entries, please refer to iKGT-user-guide.


=============================================================================
Customization
=============================================================================

Currently the loading address for xmon loader is located at 0x10000000.
The address may not available for some systems. To avoid unexpected behavior,
verify if this address is available or not on your system as following.

(1) Boot to GRUB menu

(2) Press 'c' to grub command line when GRUB boot menu appears

(3) Type the following command:

   # lsmmap

(4) If 0x10000000 is not within an available RAM range, change this
    hardcoded value in xmon/loader/pre_os/build_xmon_pkg_linux.sh to an
    address that within any of the "available RAM" regions from lsmmap command.

    Note: whenever ikgt/loader/pre_os/build_xmon_pkg_linux.sh is modified,
          it is required to rebuild loader by rebuilding ikgt_pkg.bin.


=============================================================================
Known problems
=============================================================================

(1) Dependency on *.h may not always be checked.
    Use "make clean" before rebuilding after modification of any
    header file.

=============================================================================
Release Note
=============================================================================

(1) The current release v1.0 only supports monitoring of CR0, CR4 and limited 
    set of MSRs. For details of which specific bits in CR0/CR4 and which MSRs 
    are supported, refer to cr0_bits, cr4_bits, and msr_regs in driver/cr0.c, 
    driver/cr4.c, and driver/msr.c respectively.

(2) Monitoring of certain bits in CR0 and CR4 may cause system instability 
    on certain platforms. For example, CR4.SMAP bit is supported only on 
    5th generation of Core i processors and enabling of the SMAP bit on 
    system that has older version of the Core i processors may hang the 
    system.
 

End of file

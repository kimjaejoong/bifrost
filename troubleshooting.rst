===============================
Troubleshooting
===============================
***********
Firewalling
***********

Due to the nature of firewall settings and customizations, Bifrost does **not** change any local firewalling on the node.   Users must ensure that their firewalling for the node running bifrost is such that the nodes that are being booted can connect to the following ports::

    67/UDP for DHCP requests to be serviced
    69/UDP for TFTP file transfers (Initial iPXE binary)
    6301/TCP for the Ironic API
    8080/TCP for HTTP File Downloads (iPXE, Ironic-Python-Agent)

If you encounter any additional issues, use of tcpdump is highly recommended while attempting to deploy a single node in order to capture and review the traffic exchange between the two nodes.

*****************
NodeLocked Errors
*****************

This is due to node status checking thread in Ironic, which is a locking action as it utilizes IPMI.  The best course of action is to retry the operation.  If this is occurring with a high frequency, tuning might be required.

Example error:

    NodeLocked: Node 00000000-0000-0000-0000-046ebb96ec21 is locked by host $HOSTNAME, please retry after the current operation is completed.

*********************************************
Unexpected/Unknown failure with the IPA Agent
*********************************************

New image appears not to be deploying
=====================================

When deploying a new image with the same previous name, it is necessary to purge the contents of the TFTP master_images folder which caches the image file for deployments.  The default location for this folder is /tftpboot/master_images.

Additionally, a playbook has been included that can be used prior to a re-installation to ensure fresh images are deployed.  This playbook can be found at playbooks/cleanup-deployment-images.yaml

Building an IPA image
=====================

Troubleshooting issues involving IPA can be time consuming.  The IPA developers **HIGHLY** recommend that users build their own custom IPA images in order to inject things such as SSH keys, and turn on agent debugging which must be done in a custom image as there is no mechanism to enable debugging via the kernel command line at present.

IPA's instructions on building a custom image can be found at http://git.openstack.org/cgit/openstack/ironic-python-agent/tree/imagebuild/coreos/README.rst

This essentially boils down to the following steps:

1. `git clone https://git.openstack.org/openstack/ironic-python-agent`
2. `cd ironic-python-agent`
3. `pip install -r ./requirements.txt`
4. If you don't already have docker installed, execute: `sudo apt-get install docker docker.io`
5. `cd imagebuild/coreos`
6. Edit oem/cloudconfig.yml and add "--debug" to the end of the ExecStart setting for the ironic-python-agent.service unit.
7. Execute `make` to complete the build process.

Once your build is completed, you will need to copy the images written to the UPLOAD folder to /httpboot.  If your utilizing the default file names, a simple `cp UPLOAD/* /httpboot/` will do the trick.  Since you have updated the image to be deployed, you will need to purge the contents of /tftpboot/master_images for the new image to be utilized for the deployment process.

Obtaining IPA logs via the console
==================================

1) By default, in testing mode Bifrost sets the agent journal to be logged to the console, hover testing mode is geared more towards local Virtual machine testing.  In order to get get the IPA agent to at least detail information to the screen, you will want to set the following setting in ironic.conf::

    agent_pxe_append_params=nofb nomodeset vga=normal console=ttyS0 systemd.journald.forward_to_console=yes

2) Once set, restart the ironic-conductor service, e.g. `service ironic-conductor restart` and attempt to redeploy the node.  You will want to view the system console occurring.  If possible, you may wish to use ipmitool and write the output to a log file.

Gaining access via SSH to the node running IPA
==============================================

If you wish to SSH into the node in order to perform any sort of post-mortem, you will need to do the following:

1) Set an sshkey="ssh-rsa AAAA....." value on the agent_pxe_append_params setting in /etc/ironic/ironic.conf

2) You will need to short circuit the ironic conductor process.  An ideal place to do so is in ironic/drivers/modules/agent.py in the reboot_to_instance method.  Temporarily short circuiting this step will force you to edit the MySQL database if you wish to re-deploy the node, but the node should stay online after IPA has completed deployment.

3) ssh -l core <ip-address-of-node>

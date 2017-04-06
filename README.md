# Pre-Defined vRO workflows for Puppet Enterprise

This repo contains a number of pre-defined vRO workflows for use with Puppet Enterprise. It leverages the Puppet Enterprise vRA plugin and a vRO package to manage tags.

* [vRA plugin](https://solutionexchange.vmware.com/store/products/puppet-plugin-2-0-for-vrealize-automation) by [Puppet](https://puppet.com)
* [vRealize Orchestrator 7 VAPI vSphere tags workflow](https://oliverleach.wordpress.com/2016/07/19/vrealize-orchestrator-7-vapi-vsphere-tags-workflow/) by [Oliver Leach](https://twitter.com/oliverleach)

In addition to this readme, my blog articles ([part one](https://rnelson0.com/2017/01/27/making-the-puppet-vrealize-automation-plugin-work-with-vrealize-orchestrator/) and [part two](http://rnelson0.com/2017/04/06/vrealize-orchestrator-workflows-for-puppet-enterprise)) provide additional depth and details if necessary.

## Initial Setup

Start by installing your vRealize Orchestrator instance.
The workflows were designed with v7.0 and tested thoroughly with v7.2.
These versions or newer should be used when possible.
Ensure your vRO instance communicates with your vCenter properly before continuing (don't forget to license it or it will expire after 30 days!).

Visit the vRO Control Center (sample URL: https://vro.example.com:8283/vco-controlcenter) and select *Manage Plugins* near the bottom.
Click *Browse…* at the top, point it to the downloaded plugin, and click *Install*.
Follow the prompts to restart the vRO instance.
If you have any vRO Clients open, you MUST close them and re-open them after the restart, as the plugin adds new Types that are not loaded dynamically.
You’ll just get weird errors when working with Puppet workflows until you re-launch the client.

Launch the vRO client (sample URL: https://vro.example.com:8281/vco).
Follow [these instructions](https://oliverleach.wordpress.com/2016/07/19/vrealize-orchestrator-7-vapi-vsphere-tags-workflow/) to install [the workflow](https://github.com/oliverleach/virtualdevops/raw/master/com.virtualdevops.tags.package).

In either the *Run* or *Design* view, switch to the *Workflow* tab.
You will be importing workflows; I recommend creating a folder such as *$ORGNAME Workflows* or *Puppet Workflows* for organization.
Right click on the target folder for the workflows and choose *Import Workflow…*.
Select the first workflow and import it.
Repeat these actions for all of the provided workflows.

## Add your Puppet Master

Next, we need to add at least one master, specifically that clients can download the Puppet Enterprise packages from.
This will create a Puppet-type object corresponding with our master that other plugins will leverage.
This lets us set up a single set of credentials and interaction methods rather than repeating it in every workflow.
We need to provide the hostname, IP, and ssh port.
Then we need a user/pass that works via ssh and whether that user needs to use sudo.
For most of us, this means the username cannot be root, because we don’t permit root logins over the network (if you allow root logins, I encourage you to disable that; your auditing and compliance teams will thank you!).

The hostname is what both vRO and the clients will access – clients specifically in the `curl <URL> | bash` step – and you should ensure the hostname/ip provided is accessible from all network segments properly.
You could add the same master with different display names and DNS/IP entries for agents who can’t reach the normal DNS/IP, as long as vRO can access the alternative DNS/IP as well.

The workflow is found under *Library* -> *Puppet* -> *Configuration* -> *Add a Puppet Master*.
Provide the quested information.

In order to use sudo, your master may require additional configuration.
After creating the SSH/RBAC user/password, ensure the user has `NOPASSWD` sudo access by adding a file to `/etc/sudoers.d` (I suggest the puppet module [saz/sudo](https://forge.puppet.com/saz/sudo).
For a user called `vro-workflows`, this syntax will work:

    vro-workflows ALL=NOPASSWD: /opt/puppetlabs/bin/puppet, /opt/puppetlabs/bin/facter, /bin/cp, /bin/kill

You may use the *Update a Puppet Master* workflow to change any settings, or the *Remove a Puppet Master* workflow to start from scratch.

No further configuration is required.
You can use some of other Puppet Enterprise workflows to ensure your configuration is correct before moving on to the workflows.

## Workflows
This repo contains a number of "common function" workflows as well as two opinionated workflows: **Provision new Linux VM with Puppet** and **Decommission VM**.
The provisioning workflow spins up a Linux VM from a template, installs the Puppet Enterprise client, signs the cert on the master, and runs puppet.
It also adds a vCenter Tag to the VM, for example with [tag-based Veeam backups](https://rnelson0.com/2016/06/29/tag-based-veeam-backups/).

The decommission workflow purges the client node from the master before deleting the VM.

As these workflows are opinionated, they are only intended to be a starting point.
It is unlikely that most environments can use the workflows as-is, thus I have documented the required customizations and the suggested areas of alteration.

### Randomly select Datastore from Datastore Cluster
VC:StoragePods and VC:Datastores are incompatible datatypes.
This workflow converts the former into a random member, which is equivalent to the later type.

Customization: None

### Simplified Linux
An extension of the `Clone, Linux with single NIC` workflow, with extraneous details removed or pre-populated with defaults.

Customization:
* Attribute `pool`: Starting point for the `network` presentation item. Set it or remove the presentation filter.
* Attribute `dnsServerList`: Default DNS servers for the VMs. This must be an array of strings.
* Attribute `dnsDomain`: Default DNS domain for the VMs. Public servers are provided but they will be unable to resolve any local DNS domains.
* Attribute `dhcp` (optional): Toggle to yes if using DHCP. That value is untested, it may require other adjustments.
* Attribute `network_start_point` (optional):
  * Set to the default Network Folder where you would start searching for (distribute) port groups, OR
  * Remove the *Specify a root object to be shown in the chooser…* property in the `network` Presentation item.

### Linux VM provisioning checklist
Provides the user with a checklist of external systems to update before proceeding.
Throws an exception if any checklist item is false.
This checklist assumes none of these external systems are automated. If they are, you can add workflows for those automated systems and remove them from the checklist, or remove the checklist entirely.

Customization:
* Rename or update labels for existing items as required.
* Remove items that are already automated in your environment.

### Provision new Linux VM with Puppet
Combines the above workflows with some built-in workflows to properly deploy from template, add to puppet, and add a backup tag to the VM.

Customization:
* Attributes `vmfolders_start`, `dsfolders_start`: Starting points for the `vm_template/folder` and `datastore` presentation items, respectively. Set them or remove the presentation filters.
* Attribute `puppet_master`: Select the appropriate puppet master.
* Attribute `ssh_pass`: Provide the root password that newly created VMs have.
* Attribute `endpoint`: Select the VAPI endpoint.
* Input `network`: Select the type `VC:DistributedVirtualPortgroup` (default) when using Distributed vSwitches or `VC:Network` if using Standard vSwitches.
* Presentation `VM Details/tagName`: Enter the default value you use, or remove the input and related workflow bindings.
* If you have automated any systems in the provisioning checklist, integrate those workflows as appropriate, removing the provisioning checklist item if all systems are automated.

### Decommission VM
Powers off and deletes a VM. If it was added to puppet, runs `puppet node purge $vm_name` to clean up the certificates.
Relies on a checklist to ensure non-automated cleanup is completed before processing the workflow; as with the provisioning checklist, adjust as necessary.

Customization:
* Attribute `puppetMaster`: Select the correct puppet master object.
* Attribute `domain_name`: Default domain name for VMs that are being decommissioned.

Ansible Role: QCOW2builder
==========================

Red Hat and the open source community provide an optimized image for
virtualization.  This is can be downloaded from [Red Hat](access.redhat.com)
as a compressed qcow2 image.  These have a title such as "Red Hat Enterprise
Linux X.x KVM Guest Image" and are typically much smaller than a full Binary
DVD download.  Because of this, this role will utilize the smaller size, make
some modifications and create a VM based on passed parameters.

I prefer to pass the variables "into" the role from the playbook versus by
includeing variable files.  This is because I hope to make the role usable by
other roles.  I don't know if this logic makes sense or not, but I am
essentially attempting to remove the variables from the role itself.  (Maybe I
should at least include defaults that simply don't work?  I think that if I put
them in a file in the repo, then others will assume that is where they should
go and that is not my intention.)

At a high level, this role does the following:

1. Create a copy of the already-downloaded qcow2 image to work on
2. Install some requisite packages on the Ansible host (we'll be utilizing
   `delegate_to` quite a bit)
3. Make some changes prior to creating the VM from this image to include:
   1. Remove `cloud-init` (*we won't be using that and it causes the VM to take
      forever to boot*)
   2. Inject an SSH key of the user using this role
4. Use `virt-install` to create the VM with all the specified configurations
   and by using the supplied qcow2 image as the boot disk

The result is a freshly built and running VM.  The longterm vision is that
this role will be used in other roles...

Important Notes
---------------

* This could be changed to prompt for a password to use for root and insert that
  into the image, but using keys avoids the need to modify `sshd_config`
  
      virt-customize -a $IMAGE --root-password password:pASSword!

* This is currently developed to be used on a system locally.  In order to use
  it with Tower or AWX will take some refactoring
* This role is far from idempotent at this point.  During testing I have found
  the following commands helpful to clean up:

      sudo virsh destroy <vm.name>;
      sudo virsh undefine <vm.name> --remove-all-storage

* My example playbook prompts the user for passwords to use in the VM...sort of
  breaking unattended automation, but that's how I do it right now.

Requirements
------------

I couldn't escape a few requirements:

* You'll need to have your libvirt environment configured to your liking.  I'm
  working on creating my own preferences in a separate role...to be continued...
* A locally downloaded KVM qcow2 image of the distribution you wish to install
* Because I inject the ssh key of the user calling ansible, you will need to
  ensure that you have created a key locally and actually that it is called
  `~/.ssh/id_rsa.pub`

Role Variables
--------------

All of these variables should be considered **required** however, just know that
there is currently no sanity checking:

* `vm` *I've created a bit of a nested list here so that the variables can be
  used like `vm.name`*
  * `name`
    * the hostname of the new VM
  * `os_type` *this could probably be hard coded since it will always be Linux*
  * `os_variant` *this will be specific to what image you're using.  the way to
    find acceptable values varies, but I used `osinfo-query os`*
  * `network` *another layer, but with the idea that I could add IP info, etc*
    * `name`
      * the libvirt network the VM will be on
* `qcow2`
  * `name`
    * the filename of the qcow2 KVM image to be used
  * `location`
    * the absolute path to the image to be used ***this is local to the system
      calling the playbook***


Example Playbook
----------------

Playbook with configuration options specified:

```yaml
- hosts: localhost
  connection: local
  roles:
    - role: QCOW2builder
      vm:
#        name: rhel69
        name: rhel74
        memory: 2048
        os_type: Linux
#        os_variant: rhel6.9
        os_variant: rhel7.4
        network:
          name: default # this is the libvirt network the vm will be on
      qcow2:
#        name: rhel-server-6.9-update-9-x86_64-kvm.qcow2
        name: rhel-server-7.4-update-4-x86_64-kvm.qcow2
        location: /depot/images/libvirt/ # this must be local
```

Inclusion
---------

* I'm imagining this role as a part of other roles, such as to create a 
  Satellite server.  So I have included a fairly generic `meta/main.yml` file
  which allows for something similar to:

        ansible-galaxy install -p ./roles -r requirements.yml

    with `requirements.yml` containing:

        ---
        # get the QCOW2builder role from github
        - src: https://github.com/dswhitley/ansible-role-QCOW2builder.git
          scm: git
          name: QCOW2builder

References
----------

* [How to provision a Red Hat Enterprise Linux virtual machine for Microsoft Azure](https://access.redhat.com/articles/uploading-rhel-image-to-azure)
* [How do I install the qcow2 image provided in the RHEL 7 downloads?](https://access.redhat.com/solutions/641193)

License
-------

Red Hat, the Shadowman logo, Ansible, and Ansible Tower are trademarks or
registered trademarks of Red Hat, Inc. or its subsidiaries in the United
States and other countries.

All other parts of this project are made available under the terms of the [MIT
License](LICENSE).
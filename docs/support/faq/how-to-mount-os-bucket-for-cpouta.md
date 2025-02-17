# How can I mount my Allas S3 bucket to a VM running in cPouta

Combining cPouta cloud environment and Allas storage environment allows you to build scalable data management environments. This document shows one example how you can combine these two services by mounting a bucket from Allas to an Ubuntu or a Centos7 based virtual machine running in cPouta.

[TOC]

## Installing OpenStack, s3cmd and s3fs

### In Ubuntu

* After launching an Ubuntu based virtual machine in cPouta, open a terminal connection to the VM and update it with the command:

	```sh
	sudo apt update
	```

!!! warning
    Older versions of Ubuntu will have older and deprecated versions of python. It is recommended to use the latest Ubuntu version available in Pouta.

* Then install OpenStack client by:

	```sh
	sudo apt install python3-pip python3-dev python3-setuptools
	sudo pip install --upgrade pip
	sudo pip install python-openstackclient
	```

* Next, install  **s3cmd** and **s3fs** commands to your VM.

	```sh
	sudo apt install s3cmd s3fs
	```

### In Centos7

* After launching a Centos7 based virtual machine in cPouta, open a terminal connection to the VM and update it with the command:

	```sh
	sudo yum update
	```

* OpenStack and s3cmd can then be installed by:

	```sh
	sudo yum install python-pip python-devel wget
	sudo pip install python-openstackclient
	sudo pip install --upgrade \
	    --requirement https://raw.githubusercontent.com/platform9/support-locker/master/openstack-clients/requirements.txt \
	    --constraint https://raw.githubusercontent.com/openstack/requirements/stable/pike/upper-constraints.txt
	sudo yum install s3cmd
	```

* In the case of Centos7, s3fs needs to be compiled locally. To do this we need to first to intall some tools needed for compilation:

	```sh
	sudo yum install automake fuse fuse-devel gcc-c++ git libcurl-devel libxml2-devel make openssl-devel
	```

* Then download the s3fs-fuse from the git repository and install it:

	```sh
	git clone https://github.com/s3fs-fuse/s3fs-fuse.git
	cd s3fs-fuse/
	./autogen.sh
	./configure
	make
	make  all-recursive
	sudo make install
	```

## Configuring and using Allas

### Use s3cmd to read and write files

Once you have openstack, s3cmd and s3fs installed, download and execute the **poutaos_configure** tool to configure _s3cmd_ so that it uses your cPouta project. You can also use this tool to switch between different Allas projects 
if you have several of them.

```
wget https://a3s.fi/tools/poutaos_configure
chmod u+x poutaos_configure
./poutaos_configure
```

The _poutaos_configure_ will first ask you for your CSC username and password, you can see which is your CSC username in your [My CSC profile](https://my.csc.fi/profile) page, you csan also change your password there. Then it will list your Allas projects and ask you to fill up the project to be used. Finally it will ask you for the **chunk size**, it is recommended to leave the default.

After this you can use the storage area of your Allas project with _s3cmd_ commands. Now you can see, download and upload files in this bucket with _s3cmd_.

* List all your buckets:

```sh
$ s3cmd ls s3://
2022-10-17 07:03  s3://data-europe
2020-09-17 11:12  s3://images-sky
2020-11-06 13:56  s3://case_1
```

* Let's assume you already have a bucket called **case_1** in Allas and that you have some data objects (i.e. files) in this bucket.

```sh
$ s3cmd ls s3://case_1
2022-10-17 07:14     67213268  s3://case_1/file1.txt
```

* To retrieve the file:

```sh
s3cmd get s3://case_1/file1.txt
```

* To upload a new file:

```sh
s3cmd put file2.txt s3://case_1/
```

This is the **recommended way** to use Allas with the S3 protocol from the command line. However, it is also possible to mount the bucket to your VM so that it is shown as  "mounted disk". You can use `s3fs` for that.

### Use s3fs to mount a folder intyo your VM


1. To do this, create first an empty directory (like **os_case_1**) to be used as a mount point:

	```sh
	mkdir os_case_1
	```

	!!! info
	    Any empty directory can be used as a mount point

1. Create a `.passwd-s3fs` file in your home directory. The format of the file must be: `ACCESS_KEY_ID:SECRET_ACCESS_KEY` and have _600_ permissions.

	```sh
	$ openstack ec2 credentials list -f value | grep $OS_PROJECT_ID | tail -1 |\
	   awk '{print $1":"$2}' >.passwd-s3fs
	Password:
	$ chmod 600 .passwd-s3fs
	```

1. then use the _s3fs_ command to mount the bucket.

	```sh
	s3fs case_1 os_case_1 -o passwd_file=~/.passwd-s3fs -o url=https://a3s.fi/ \
		-o use_path_request_style -o umask=0333,uid=$(id -u)
	```

	!!! info 
	    The uid value returned by `id -u` should be 1000 for the default user

	!!! info
	    The umask value `0333` mounts the files in **read-only mode**. If you want to mount them in read-write mode, use `0027` instead

1. And after this you should be able to see the objects of the mounted bucket as files. Try for example the command:

	```sh
	ls -l os_case_1
	```

	The output should be the same as with `s3cmd ls s3://case_1`

1. When you are done you can unmount the folder by:

	```sh
	sudo umount os_case_1
	```


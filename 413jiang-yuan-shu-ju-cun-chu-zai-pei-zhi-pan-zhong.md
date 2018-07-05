### 将元数据存储在配置盘中 {#将元数据存储在配置盘中}

您可以让OpenStack将元数据写入到一个特殊的“配置盘”中，实例启动时可以挂载这个盘来读取配置信息。一般情况下，这些信息是由元数据服务提供的。注意，这里的元数据和用户数据不同。

这个功能的一个用例是在没有DHCP的情况下给实例分配IP地址。例如，您可以通过配置盘将IP地址配置传给实例，实例挂载上这个盘以后，读取其中的IP信息，before you configure the network settings for the instance.

#### 使用本功能的要求，和本功能的几个注意事项 {#使用本功能的要求，和本功能的几个注意事项}

要使用配置盘，您的主机和镜像必须满足以下要求。

**对主机的要求：**

* 以下虚拟机管理器支持配置盘：libvirt，XenServer，Hyper-V和VMware。
* 如果要在libvirt，XenServer或者VMware中使用配置盘，您必须先在每个compute host上安装
  `genisoimage`
  。否则，实例无法正常启动。 您需要用
  `mkisofs_cmd`
  来标记您安装genisoimage程序的路径。如果genisoimage和
  `nova-compute`
  服务在一个路径下，就不用设置了。
* 如果您要在Hyper-V下使用配置盘，您必须用
  `mkisofs_cmd`
  来标记您安装的mkisofs.exe的全路径。此外，您还要在
  `hyperv`
  中设置
  `qemu_img_cmd`
  值，将其指向
  `qemu-img`
  这个命令的安装位置。

**对镜像的要求：**

* 带有新版本cloud-init包的镜像能够自动获取到配置盘中的元数据。0.7.1版本的cloud-init包能用在Ubuntu和Fedora系的操作系统上，比如说RHEL。
* 如果镜像中没有安装cloud-init包，您必须定制一下这个镜像：写一个能执行各种动作的脚本，比如在启动的时候挂载配置盘，从盘中读数据，然后做一些诸如导入公钥的动作。您可以在本文档中读到更多配置盘中数据格式的内容。

**几个注意事项：**

* 不要依赖配置盘中的EC2元数据，因为这些内容在新版本中可能会被移除。例如，不要依赖`ec2`文件中的文件。

* 在您创建可读取配置盘的镜像时，如果有`openstack`文件夹下有多个文件夹，您一定要选择用户支持的最高版本的API（以日期标注）。比如说，如果您的镜像支持2012-03-05，2012-08-05和2013-04-13版本，先尝试使用2013-04-13版本，如果该版本不存在，再尝试前面的版本。

#### 启用和访问配置盘 {#启用和访问配置盘}

1. 如果要启用配置盘，将
   `--config-driver true`
   参数传给
   `nova boot`
   命令即可。

在下面的例子里，我们启用了配置盘，将用户数据，两个文件，以及两个键/值元数据对传给了它，这些都可以在配置盘中获取到。

```
$ nova boot --config-drive true --image my-image-name --key-name mykey \
  --flavor 1 --user-data ./my-user-data.txt myinstance \
  --file /etc/network/interfaces=/home/myuser/instance-interfaces \
  --file known_hosts=/home/myuser/.ssh/known_hosts \
  --meta role=webservers --meta essential=false

```

您也可以把Compute服务配置成每次都使用配置盘。在`/etc/nova/nova.conf`文件中配置如下条目：

```
force_config_drive=true

```

> 注意： 如果某位用户将`--config-drive true`参数传递给了`nova boot`命令，连管理员也没法禁用。

1. 如果您的实例支持通过标签来访问磁盘，您可以以
   `/dev/disk/by-label/configurationDriveVolumeLabel`
   来挂载配置盘。在下面的例子中，配置盘的标签是
   `config-2`
   ：

```
# mkdir -p /mnt/config
# mount /dev/disk/by-label/config-2 /mnt/config

```

> 注意： 为了对配置盘提供支持，您使用的CirrOS至少在0.3.1版本以上。 如果您的客户机不用`udev`，是不会有`/dev/disk/by-label`的。 您可以用`blkid`命令来查找配置盘对应的块设备。比如，如果您用CirrOS镜像，在`m1.tiny`的配置下启动了实例，配置盘应该是`/dev/vdb`：
>
> ```
> # blkid -t LABEL="config-2" -odevice
> /dev/vdb
> # mkdir -p /mnt/config
> # mount /dev/vdb /mnt/config
>
> ```

#### 配置盘中的内容 {#配置盘中的内容}

下面这个例子中，配置盘中的内容如下：

```
ec2/2009-04-04/meta-data.json
ec2/2009-04-04/user-data
ec2/latest/meta-data.json
ec2/latest/user-data
openstack/2012-08-10/meta_data.json
openstack/2012-08-10/user_data
openstack/content
openstack/content/0000
openstack/content/0001
openstack/latest/meta_data.json
openstack/latest/user_data

```

配置盘中有哪些内容取决于您使用`nova boot`时传入了哪些参数。

#### OpenStack元数据格式 {#openstack元数据格式}

下面的内容展示了`openstack/2012-08-10/meta_data.json`和`openstack/latest/meta_data.json`文件。这两个文件是完全一样的。为了方便阅读，内容已经做过排版。

```
{
    "availability_zone": "nova",
    "files": [
        {
            "content_path": "/content/0000",
            "path": "/etc/network/interfaces"
        },
        {
            "content_path": "/content/0001",
            "path": "known_hosts"
        }
    ],
    "hostname": "test.novalocal",
    "launch_index": 0,
    "name": "test",
    "meta": {
        "role": "webservers",
        "essential": "false"
    },
    "public_keys": {
        "mykey": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAgQDBqUfVvCSez0/Wfpd8dLLgZXV9GtXQ7hnMN+Z0OWQUyebVEHey1CXuin0uY1cAJMhUq8j98SiW+cU0sU4J3x5l2+xi1bodDm1BtFWVeLIOQINpfV1n8fKjHB+ynPpe1F6tMDvrFGUlJs44t30BrujMXBe8Rq44cCk6wqyjATA3rQ== Generated by Nova\n"
    },
    "uuid": "83679162-1378-4288-a2d4-70e13ec132aa"
}

```

请注意您使用`nova boot`参数时配置的参数`--file /etc/network/interfaces=/home/myuser/instance-interfaces`产生的效果。这个文件的内容被保存在了`openstack/content/0000`文件中，路径信息指定为`/etc/network/interface/`，保存在`meta_data.json`中。

#### EC2元数据格式 {#ec2元数据格式}

下面的内容展示了`ec2/2009-04-04/meta_data.json`和`ec2/latest/meta_data.json`文件。这两个文件是完全一样的。为了方便阅读，内容已经做过排版。

```
{
    "ami-id": "ami-00000001",
    "ami-launch-index": 0,
    "ami-manifest-path": "FIXME",
    "block-device-mapping": {
        "ami": "sda1",
        "ephemeral0": "sda2",
        "root": "/dev/sda1",
        "swap": "sda3"
    },
    "hostname": "test.novalocal",
    "instance-action": "none",
    "instance-id": "i-00000001",
    "instance-type": "m1.tiny",
    "kernel-id": "aki-00000002",
    "local-hostname": "test.novalocal",
    "local-ipv4": null,
    "placement": {
        "availability-zone": "nova"
    },
    "public-hostname": "test.novalocal",
    "public-ipv4": "",
    "public-keys": {
        "0": {
            "openssh-key": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAgQDBqUfVvCSez0/Wfpd8dLLgZXV9GtXQ7hnMN+Z0OWQUyebVEHey1CXuin0uY1cAJMhUq8j98SiW+cU0sU4J3x5l2+xi1bodDm1BtFWVeLIOQINpfV1n8fKjHB+ynPpe1F6tMDvrFGUlJs44t30BrujMXBe8Rq44cCk6wqyjATA3rQ== Generated by Nova\n"
        }
    },
    "ramdisk-id": "ari-00000003",
    "reservation-id": "r-7lfps8wj",
    "security-groups": [
        "default"
    ]
}

```

#### 用户数据 {#用户数据}

`openstack/2012-08-10/user_data`，`openstack/latest/user_data`，`ec2/2009-04-04/user-data`和`ec2/latest/user-data`这四个文件只有在您使用`nova boot`文件时传入了`--user-data`参数和用户数据文件时才会出现。

#### 配置盘格式 {#配置盘格式}

默认的配置盘格式时`ISO 9660`。如果要显式地指定`ISO 9660`格式，请在`/etc/nova/nova.conf`文件中添加以下配置：

```
config_drive_format=iso9660

```

默认情况下，您只能将配置盘以硬盘的形式装载在实例上，而不能用光盘的形式。如果要以CD的形式装载，请在`/etc/nova/nova.conf`文件中添加以下配置：

```
config_drive_cdrom=true

```

为了提供对旧设备的支持，您还可以将配置盘设置为`VFAT`格式。您可能不会需要用到`VFAT`，因为`ISO 9660`已被绝大多数操作系统所支持。然而，要使用`VFAT`格式，请在`/etc/nova/nova.conf`文件中添加以下配置：

```
config_drive_format=vfat

```

如果您选择了`VFAT`，配置盘的大小将会是64 MB。

> 注意： 在当前的OpenStack版本中（Liberty），给用到`config_drive`的本地盘做热迁移的功能被禁用了，因为libvirt在复制只读盘时有bug。然而，如果我们使用`VFAT`格式作为`config_drive`的格式，热迁移的功能是可用的。



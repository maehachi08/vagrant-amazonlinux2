# create amazonlinux2 vagrant box

* https://aws.amazon.com/jp/blogs/news/amazon-linux-2-release/
* https://aws.amazon.com/jp/amazon-linux-2/

2017/12/15 に AmazonLinux2 がリリースされました。

従来のAmazonLinuxと同様にDockerイメージに加え、オンプレミスでの開発を可能にするために、VMware、Microsoft Hyper-V、Oracle VM VirtualBoxの仮想マシンイメージを提供しています。

## 本レポジトリでのゴール

1. Oracle VM VirtualBoxの仮想マシンイメージ(VDI) から vagrant boxを作成
   * https://qiita.com/aibax/items/7fd9a874cb7e88f95488　を参考にしています
2. 作成したAmazonLinux2 vagrant boxをimport
3. 作成したAmazonLinux2 vagrant boxから仮想マシンを起動できる Vagrantfile(v2) の作成
   * `vbguest plugins` でKernel updateを実行する
   * `ansible_local` によるprovisioningを実行する

## amazonlinux2 VirtualBox machine image download

* https://cdn.amazonlinux.com/os-images/latest/

ブラウザなら上のURLへアクセスすると最新リリースのダウンロードページへリダイレクトします。
`virtualbox/` をクリックすると vdi拡張子のファイルが表示されますのでダウンロードしましょう。

余談ですが、curl でリダイレクト後のURLを取得したい場合は以下のようなコマンドで取れました。

```
echo "`curl -s -L -I -o /dev/null -w '%{url_effective}' https://cdn.amazonlinux.com/os-images/latest/`virtualbox/"
```

## create config for to create vagrant box

### create config

* meta-data

```
local-hostname: amznlinux2
```

* user-data
   * vagrantユーザのssh-authorized-keys: https://github.com/hashicorp/vagrant/blob/master/keys/vagrant.pub

```
# https://cdn.amazonlinux.com/os-images/2017.12.0.20180222/README.cloud-init
#cloud-config
# vim:syntax=yaml
users:
  - default
  - name: vagrant
    groups: wheel
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    plain_text_passwd: vagrant
    ssh-authorized-keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key
    lock_passwd: false

chpasswd:
  list: |
    root:vagrant
  expire: False

packages:
  # To build of VirtualBox Guest Additions
  - kernel-devel
  - kernel-headers
  - gcc
  - make
  - perl
  - bzip2
  - mod_ssl

runcmd:
  # cloud-init による起動時の root のパスワードリセット処理を無効化
  - sed -i 's/.*root:RANDOM/#&/g' /etc/cloud/cloud.cfg.d/99_onprem.cfg

  # Vagrant によるディストリビューションの識別のための互換設定
  # Amazon Linux をサポートしていないため、RedHat Enterprise Linux として認識させる
  - ln -s /etc/system-release /etc/redhat-release

  # Install Ansible
  - amazon-linux-extras install ansible2
```

### create seed.iso

* MacOsX

```
brew install cdrtools
mkisofs -output seed.iso -volid cidata -joliet -rock user-data meta-data
```

* CentOS

```
docker run -it -v $(pwd):/data centos sh
yum install -y genisoimage
genisoimage  -output seed.iso -volid cidata -joliet -rock user-data meta-data
```

## Launch VirtualBox Machine

  1. vdi拡張子のImageを指定して仮想マシンを作成
     * 仮想マシンの名前は `amazonlinux2` とする(後述のvagrant packageコマンドで名前を指定している)
  2. 光学ドライブにseed.isoを挿入
  3. 仮想マシンを起動
  4. cloud-init の処理が完了しログインプロンプトが表示されたら仮想マシンを停止

## create box image

```
vagrant package --base amazonlinux2 --output amazonlinux2.box
vagrant box add --name amazonlinux2 amazonlinux2.box
```

---

```
[web] No installation found.
The guest's platform ("amazon") is currently not supported, will try generic Linux method...
Copy iso file /Applications/VirtualBox.app/Contents/MacOS/VBoxGuestAdditions.iso into the box /tmp/VBoxGuestAdditions.iso
Mounting Virtualbox Guest Additions ISO to: /mnt
mount: /dev/loop0 is write-protected, mounting read-only
Installing Virtualbox Guest Additions 5.2.8 - guest version is unknown
Verifying archive integrity... All good.
Uncompressing VirtualBox 5.2.8 Guest Additions for Linux........
VirtualBox Guest Additions installer
Copying additional installer modules ...
Installing additional modules ...
VirtualBox Guest Additions: Building the VirtualBox Guest Additions kernel modules.
This system is currently not set up to build kernel modules.
Please install the Linux kernel "header" files matching the current kernel
for adding new hardware support to the system.
    VirtualBox Guest Additions: Starting.
    VirtualBox Guest Additions: Building the VirtualBox Guest Additions kernel modules.
    This system is currently not set up to build kernel modules.
    Please install the Linux kernel "header" files matching the current kernel
    for adding new hardware support to the system.
        An error occurred during installation of VirtualBox Guest Additions 5.2.8. Some functionality may not work as intended.
        In most cases it is OK that the "Window System drivers" installation failed.
        Redirecting to /bin/systemctl start vboxadd.service
        Job for vboxadd.service failed because the control process exited with error code. See "systemctl status vboxadd.service" and "journalctl -xe" for details.
        Unmounting Virtualbox Guest Additions ISO from: /mnt

```

## Launch amazonlinux2 vagrant vm


```
vagrant up


```

```
[MacBookAir] $ vagrant up
Bringing machine 'web' up with 'virtualbox' provider...
==> web: Resuming suspended VM...
==> web: Booting VM...
==> web: Waiting for machine to boot. This may take a few minutes...
    web: SSH address: 127.0.0.1:2222
    web: SSH username: vagrant
    web: SSH auth method: private key
==> web: Machine booted and ready!
==> web: Machine already provisioned. Run `vagrant provision` or use the `--provision`
==> web: flag to force provisioning. Provisioners marked to run always will still run.

[MacBookAir] $ vagrant ssh

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
No packages needed for security; 33 packages available
Run "sudo yum update" to apply all updates.

$ id
uid=1000(vagrant) gid=1000(vagrant) groups=1000(vagrant),10(wheel)

$ sudo su -

# id
uid=0(root) gid=0(root) groups=0(root)

# hostnamectl
   Static hostname: amznlinux2.localdomain
         Icon name: computer-vm
           Chassis: vm
        Machine ID: d36630f22ae14377b6514347b8921897
           Boot ID: 09d9bfadcb1841b3a8bbdb3e0748b23b
    Virtualization: kvm
  Operating System: Amazon Linux 2 (2017.12) LTS Release Candidate
       CPE OS Name: cpe:2.3:o:amazon:amazon_linux:2
            Kernel: Linux 4.9.81-44.57.amzn2.x86_64
      Architecture: x86-64
```



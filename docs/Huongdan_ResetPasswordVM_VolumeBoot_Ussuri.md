# Hướng dẫn Reset password cho các máy ảo boot từ Volume trên hệ thống OpenStack 
### Mục tiêu của hướng dẫn:
```
 - Hỗ trợ khách hàng có nhu cầu phục hồi password cho các máy ảo: Windows và Linux.
```

#### <i>Chú ý: </i>
 - Hướng dẫn thực hiện trên phiên bản OpenStack Ussuri, OS CentOS 8.3
 - Đã test thành công với các máy ảo Windows Server 2k12 Standard, Ubuntu 18.04, CentOS 7.3. Cách thực hiện đối với các OS khác tương tự.

## 1. SystemRescueCD
### 1.1. Tải SystemRescueCD
```sh
wget https://osdn.net/projects/systemrescuecd/storage/releases/7.01/systemrescue-7.01-amd64.iso
```
### 1.2. Trên máy chủ tạo image, tải gói `syslinux`. Đây là 1 gói các lightweight master boot record để khởi động OS.
```sh
yum install syslinux -y
```

### 1.3. Sử dụng công cụ `isohybrid`, cho phép ISO image có thể được boot qua BIOS từ disk storage device (USB)
```sh
isohybrid systemrescue-7.01-amd64.iso
```

### 1.4. Đưa ISO SystemRescueCD lên Glance
```sh
openstack image create --file systemrescue-7.01-amd64.iso --public systemrescuecd-amd64
```
### 1.5. Đặt các metadata cho image
```sh
openstack image set  --property hw_cdrom_bus=ide --property hw_disk_bus=ide --property hw_rescue_bus=ide --property hw_rescue_device=cdrom systemrescuecd-amd64
```
Tham khảo về việc set metadata cho rescue image: https://specs.openstack.org/openstack/nova-specs/specs/ussuri/implemented/virt-rescue-stable-disk-devices.html


## 2. Cấu hình trên OpenStack
Để thực hiện việc reset password máy ảo, ta sẽ sử dụng 1 tính năng của nova là Rescue VM. Tính năng này sẽ đặt máy ảo vào trạng thái Rescue, trong đó VM sẽ boot vào 1 image được chỉ định. Chi tiết về tính năng này có thể đọc tại [đây](https://docs.openstack.org/ocata/user-guide/cli-reboot-an-instance.html)
### 2.1. Cấu hình `shutdown_timeout` trên OpenStack Controller
Khi thực hiện Rescue, VM sẽ được soft reboot trước khi boot vào SystemRescueCD, nếu thời gian soft_reboot lâu hơn thời gian mặc định của nova (60s), VM sẽ bị hard reboot. Để quá trình reset password thành công, phải đảm bảo quá trình soft reboot này của VM thành công tốt đẹp. 
Để thực hiện điều đó, cấu hình giá trị `shutdown_timeout` cao hơn mức mặc định của nova.  Sửa file `/etc/nova.conf`, đặt thời gian timeout là 5p (300s):
```sh
[DEFAULT]
shutdown_timeout = 300
```
Khởi động lại các service của nova:
```sh
cd /etc/init/; for i in $(ls nova-* | cut -d \. -f 1 | xargs); do sudo service $i restart; done
```

## 3. Thực hiện Reset password VM
### 3.1. Đưa máy ảo về trạng thái Rescue. **Lưu ý:** máy ảo phải đang ở trạng thái Active (Running) trước khi chuyển về Rescue.

*Lưu ý*: tính năng rescue Volume-backed VM đã hỗ trợ ở bản Ussuri, tuy nhiên phải chỉ định sử dụng API compute 2.87 (mặc định là API version 2). 
Để chỉ định API version:
 - Khi chạy command line, thêm biến môi trường `OS_COMPUTE_API_VERSION=2.87`
 - Khi chạy API, thêm header  `OpenStack-API-Version: compute 2.87` hoặc `X-OpenStack-Nova-API-Version: 2.87`

Trên host OpenStack Controller, thực hiện

```sh
openstack server rescue --image systemrescuecd-amd64 win2k12
```

Trong đó:
 - `systemrescuecd-amd64`: tên image SystemRescueCD đã tạo ở trên
 - `win2k12`: tên máy ảo cần reset password

Trên horizon, trạng thái của máy ảo như sau:
![resetpassword](/images/resetpassword/rp_1.png)

### 3.2. Truy cập giao diện console của máy ảo, sẽ thấy giao diện của SystemRescueCD.
![resetpassword](/images/resetpassword/rp_2.png)

Lựa chọn option đầu tiên *Boot SystemRescueCD using default options*. VM sẽ boot vào cmd của RescueCD
![resetpassword](/images/resetpassword/rp_3.png)

### 3.3. Đối với máy ảo Windows
#### 3.3.1. List các ổ đĩa của máy ảo
```sh
fdisk -l
```

Kết quả:

![resetpassword](/images/resetpassword/rp_4.png)

Ta thấy ổ đĩa `/dev/vdb2` là ổ đĩa chứa OS máy ảo

### 3.3.2. Mount ổ đĩa chứa OS vào thư mục `/mnt/windows` để thao tác
```sh
mkdir /mnt/windows
ntfs-3g /dev/vdb2 /mnt/windows
```

Lưu ý: nếu xuất hiện lỗi `The NTFS partition is in an unsafe state. Please resume and shutdown Windows fully (no hibernation or fast restarting), or mount the volume read-only with the 'ro' mount option.`. có nghĩa là máy ảo đã không được shutdown tốt đẹp, lúc này cần khởi động lại máy ảo và shutdown lại để đảm bảo máy ảo shutdown thành công.

### 3.3.3. Chuyển tới thư mục chứa SAM database của Windows. Lưu ý: với Windows Server 2k12, SAM database được đặt tại `/Windows/System32/config`, các OS Windows khác cần kiểm tra lại đường dẫn trươc khi thao tác
```sh
cd /mnt/windows/Windows/System32/config
```

### 3.3.4. Dùng lênh `chntpw` để reset password của Administrator
```sh
chntpw SAM
```
Nếu muốn reset password của 1 user, sử dụng lệnh:
```sh
chntpw -u Guest SAM
```

Lựa chọn option 1 - Xóa password máy ảo, khi này máy ảo sẽ tự động login và khách hàng sẽ tự đặt lại password
![resetpassword](/images/resetpassword/rp_5.png)

Sau khi reset password xong, thoát và save

![resetpassword](/images/resetpassword/rp_6.png)

### 3.3.5. Trên host Controller, gỡ bỏ trạng thái Rescue của máy ảo
```sh
openstack server unrescue win2k12
```

### 3.3.5. Máy ảo boot vào Windows mà không hỏi password, tại đây khách hàng chủ động đặt password mới
![resetpassword](/images/resetpassword/rp_7.png)


### 3.4. Đối với máy ảo Linux
#### 3.4.1. List các ổ đĩa của máy ảo
```sh
fdisk -l
```
Kết quả:

![resetpassword](/images/resetpassword/rp_8.png)

Ta thấy ổ đĩa `/dev/vdb1` là ổ đĩa chứa OS máy ảo

### 3.4.2. Mount ổ đĩa chứa OS vào thư mục `/mnt/root` để thao tác
```sh
mkdir /mnt/root
mount /dev/vdb1 /mnt/root
```

### 3.4.3. Thay đổi `root directory` từ SystemRescueCD sang thư mục `/mnt/root`
```sh
chroot /mnt/root /bin/bash
```

### 3.4.4. Đặt password mới cho `root` máy ảo
```sh
passwd root
```


### 3.4.5. Trên host Controller, gỡ bỏ trạng thái Rescue của máy ảo
```sh
openstack server unrescue u1804
```

### 3.4.5. Máy ảo boot vào OS , khách hàng login với password mới vừa thay đổi


## Done

# Tham khảo:

- https://www.linux-magazine.com/Online/Features/Resetting-Passwords-with-SystemRescueCd
- https://ibm-blue-box-help.github.io/help-documentation/troubleshooting/Troubleshooting_FAQ/
- https://docs.openstack.org/ocata/user-guide/cli-reboot-an-instance.html
- https://www.howtogeek.com/howto/14369/change-or-reset-windows-password-from-a-ubuntu-live-cd/
- https://maideveloper.com/blog/how-to-reset-linux-root-password
- https://specs.openstack.org/openstack/nova-specs/specs/ussuri/implemented/virt-rescue-stable-disk-devices.html

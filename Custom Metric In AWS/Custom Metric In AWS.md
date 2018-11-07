## 0. Đặt vấn đề
Trong AWS, `cloudwatch` là một công cụ rất mạnh để quản lý resource. Tuy nhiên, đa phần các vấn đề xảy ra đối với các dự án phần mềm sẽ liên quan đến memory hoặc disk I/O. Thật không may các công cụ quản lý các resource liên quan đến memory hay disk I/O lại không được cung cấp sẵn trong `cloudwatch`. Nó sẽ kéo theo một vài hệ lụy sau

  * Làm thế nào để alert cho admin biết rằng memory của anh ta đang có vấn đề
  * Làm thế nào để auto scale khi memory quá tải
  * Khi fulldisk thì làm sao báo được cho admin
  * Vân vân và mây mây
  
 Chắc hẳn các bạn đã làm việc với cloudwatch đề hiểu rằng các resoure được quản lý dựa vào con số ví dụ như là phần trăm - một số thực dấu phẩy động. Điều này làm chúng ta nảy ra một ý tưởng rằng, nếu truyền cho cloudwatch lượng phần trăm memory đang sử dụng cloudwatch sẽ sử dụng được nó. Vậy chỉ cần biết:
  * Lượng phần trăm memory đang sử dụng
  * Cách gửi nó lên cloudwatch
  
Là chúng ta sẽ biết cách giải quyết được các hệ lụy đã đề cập ở trên và còn nhiều thứ hay ho hơn thế nữa. Chúng ta bắt tay vào giải quyết nó nào.

## 1.Cài đặt aws cli
```sh
pip install awscli --upgrade --user 
```

## 2. Config aws cli
Bắt buộc phải có `access key` và `sceret key` để config aws cli có quyền access đến cloudwatch của bạn. Nếu bạn chưa biết cách lấy `access key` và `sceret key` thì hãy tham khảo trên AWS

```sh
aws configure
```

## 3. Setting aws path để chạy crontab
Thông thường, chỉ cần chạy crontab là chúng ta có thể thự hiện được các command cần thiết trong khi quản lý một server. Tuy nhiên, do aws cli viết bằng python nên khi chúng ta cài đặt nó sẽ được set vào một vị trí nào đó trên hệ thống của bạn. Bên cạnh đó crontab lại chỉ có quyền access vào `/usr/bin/`, có nghĩa là nếu aws cli của bạn không được cài đặt ở path nói trên thì crontab sẽ không thể nào thực thi được câu lệnh liên quan đến đó. Chúng ta tiến hành kiểm tra path của aws cli. Nếu bạn không biết aws cli của bạn được nằm ở đâu bạn hãy dùng lệnh sau:
```sh
which aws
```
Sau khi có kết quả rồi chúng ta hãy thực hiện liên kết như dưới nếu aws cli không nằm trong `/usr/bin/` :
```
sudo ln -s ~/.local/bin/aws /usr/bin/aws
```

## 4. Tạo script
Sau khi đã link aws vào `/usr/bin/aws` chúng ta có thể tạo scipt để quản lý resource của hệ thống và gửi nó đến cho cloudwatch. Dưới đây là một ví dụ điển hình:
```sh
#!/bin/bash
USEDMEMORY=$(free -m | awk 'NR==2{printf "%.2f\t", $3*100/$2 }')
TCP_CONN=$(netstat -an | wc -l)
TCP_CONN_PORT_80=$(netstat -an | grep 80 | wc -l)
USERS=$(uptime |awk '{ print $6 }')
IO_WAIT=$(iostat | awk 'NR==4 {print $5}')
 
aws cloudwatch put-metric-data --metric-name memory-usage --dimensions Instance=i-0dda07a0c88e4e09e  --namespace "Custom" --value $USEDMEMORY
aws cloudwatch put-metric-data --metric-name Tcp_connections --dimensions Instance=i-0dda07a0c88e4e09e  --namespace "Custom" --value $TCP_CONN
aws cloudwatch put-metric-data --metric-name TCP_connection_on_port_80 --dimensions Instance=i-0dda07a0c88e4e09e  --namespace "Custom" --value $TCP_CONN_PORT_80
aws cloudwatch put-metric-data --metric-name No_of_users --dimensions Instance=i-0dda07a0c88e4e09e  --namespace "Custom" --value $USERS
aws cloudwatch put-metric-data --metric-name IO_WAIT --dimensions Instance=i-0dda07a0c88e4e09e  --namespace "Custom" --value $IO_WAIT
```
Trong đó hãy truyền vào id của instance bạn muốn quản lý vào command phía trên. Ví dụ ở trên đang quản lý trạng thái của instance có id là `i-0dda07a0c88e4e09e`. Đến đây bạn đã hiểu vì sao phải configure aws cli rồi chứ?

## 5. Tạo cronjob quản lý
Giả sử bạn lưu script bạn tạo phía trên ở `/home/centos/usage.sh`. Hãy tạo crontab như dưới để forward thông tin 1p một lần sang `cloudwatch`
```sh
*/1 * * * * /home/centos/usage.sh
```

## 6. Xác nhận thành quả
Login vào aws console và xác nhận thành quả mình vừa làm ra:

### Memory Usage
![memory.png](resources/C5556578CCE001667031B821DC9F1A1C.png =1440x692)

### TCP Connection
![TCP Connection.png](resources/2CDAAA0D24E02C0B7922010A1F7588A8.png =1440x694)

### IO
![IO.png](resources/39FE135565F418189C6AEAAAF61C3C13.png =1440x691)

### Accessing User
![Access.png](resources/7A8317DEC2F3E7C6DFAA038701C800C3.png =1440x693)

### Accessing on TCP 80
![TCP.png](resources/0B8F02348271A85007889E3D0B79A5C2.png =1440x694)

## 7. Setting Alarm
Sau khi đã thực hiện xong việc forward tình trạng resource cho `cloudwatch`. Chúng ta sẽ tiếp tục setting alert cho một trong các resource ấy. Ở đây tôi sẽ ví dụ khi làm việc với `memory usage`

* Tất nhiên bạn phải access vào cloudwatch trước tiên rồi
![Select metric.png](resources/FEA025CA37770378CE2D7AC5AE8A871B.png =1440x732)

* Sau khi chọn lựa metric bạn hãy setting giá trị để nó alert
![Alert.png](resources/9391948AE19F5C3CF29F3D2889CECC83.png =1440x699)

Từ đây bạn sẽ làm được vô vàn thứ hay ho với Alarm đã set ở đây.
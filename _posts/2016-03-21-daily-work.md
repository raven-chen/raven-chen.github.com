## 2016-3-21

Today, I try to deploy our new notification API on demo server and test send notification via APNS, Same code and same data, works fine in local but on demo server I alway got 

``` sh
[2016-03-21 07:05:55] [ERROR] Rpush::Daemon::TcpConnectionError, Errno::ETIMEDOUT, Connection timed out - connect(2) for "gateway.sandbox.push.apple.com" port 2195
```

when sending notification. After 2 hours investigation, I talked to SA and checked [APNS documentation](https://developer.apple.com/library/ios/technotes/tn2265/_index.html#//apple_ref/doc/uid/DTS40010376-CH1-TNTAG41). The reason is our server forbid unusual port to be used which is needed by APNS, 2195 and 2196..

## 2016-3-22



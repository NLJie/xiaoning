
## 打印共享内存信息

`io -1 -l 128 -r 0x47400000`


`busybox mount -o nolock -o tcp -o rsize=32768,wsize=32768 192.168.1.69:/tftpboot /mnt/sdcard`


```
ifconfig eth0 192.168.1.18 netmask 255.255.255.0
route add default gw 192.168.1.1

busybox mount -o nolock -o tcp -o rsize=32768,wsize=32768 192.168.1.69:/tftpboot /mnt/sdcard
```

# **MCU CAN功能的使用**

## 硬件特性

- 支持 CAN FD；
- 速率最高支持 5M bps 的最高速率传输；
- 支持 8~64 位传输(4的倍数)；

1. 首先在 `main.c` 的 `SystemModuleInit` 初始化 canfd 接口，目前只支持 can0的接口；

```c
CanfdAppInit(0, CANFD_BPS_1MBAUD, CANFD_MODE_FD);

初始化CAN0，配置速率1Mbps，模式使用CANFD;
```

## 发送接口的使用
```c
/**
 * @brief CANFD 应用层发送一帧.
 * @param[in] channel: CANFD 通道.
 * @param[in] pMsg: 消息指针.
 * @retval HAL_ERROR or HAL_OK.
 */
HAL_Status CanfdAppSend(uint8_t channel, CanfdMsg_s *pMsg);
```

- `mcu_project/fq_project/fq_pro/src/CanfdApp.c` 文件里面的 `void CanfdTestSendTask(void)` 这个函数测试可以进行发送一帧数据测试；
## 接收接口的使用

接收方式一：（轮询接收）：

```c
/**
 * @brief CANFD 应用层接收一帧消息（从接收队列取出）.
 * @param[in] channel: CANFD 通道.
 * @param[out] pMsg: 消息指针.
 * @retval 0=成功, -1=失败（无数据或参数错误）.
 */
int CanfdAppRecv(uint8_t channel, AppCanfdMsg_s *pMsg);
```

- `mcu_project/fq_project/fq_pro/src/CanfdApp.c` 文件里面的 `void CanfdAppRecvTask(void)` 这个任务函数测试可以接收数据帧，默认将接收到的数据写入到了共享内存；也可以不写入共享内存根据实际需求做；

接收方式二：（中断接收）


- CanfdAppSetRxCallback
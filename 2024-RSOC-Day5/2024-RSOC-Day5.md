# RT-Thread软件包的使用和MQTT协议（搭配阿里云平台）
**本文是关于RT-Thread丰富软件包的介绍和阿里云物联网平台的使用，由于本人也是初次接触的一名小白，文中如有不合适之处欢迎指正。**
## RT-Thread的软件包
**RT-Thread 软件包是运行在 RT-Thread 物联网操作系统平台之上的通用代码库，非常简单易用，适合用来进行快速开发。这就相当于我们不需要重新去写底层设备复杂的驱动了，我们只需要调用官方提供的API接口就可以进行开发了，或者说我们只需要关注应用层了，这就极大的提高了效率。下面我将以通过软件包来实现温湿度采集这个简单的例子来详细说明。**
## 温湿度传感器---AHT10(I2C设备)
**假如我们不使用软件包想要驱动一个I2C设备，我们需要做一些什么呢**
1. 找到传感器命令
2. 找到I2C设备
3. 利用API编写驱动，包括写命令`write_reg()`、读命令`read()`、获取参数命令…
**显而易见在不使用软件包的情况下我们的工作量就很大，代码如下**
```C
#include <rtthread.h>
#include <rtdevice.h>
#include <board.h>

#define DBG_TAG "main"
#define DBG_LVL DBG_LOG
#include <rtdbg.h>

#define AHT_I2C_BUS_NAME "i2c1"  // AHT20 挂载的I2C总线
#define AHT_ADDR 0x38            // AHT20 I2C地址
#define AHT_CALIBRATION_CMD 0xBE // AHT20 初始化命令
#define AHT_NORMAL_CMD 0xA8      // AHT20 正常工作模式命令
#define AHT_GET_DATA_CMD 0xAC    // AHT20 获取结果命令

// I2C_BUS设备指针，用于等会寻找与记录AHT挂载的I2C总线
struct rt_i2c_bus_device *i2c_bus = RT_NULL;

// AHT命令的空参数
rt_uint8_t Parm_Null[2] = {0, 0};

// 写命令（主机向从机传输数据）
rt_err_t write_reg(struct rt_i2c_bus_device *Device, rt_uint8_t reg, rt_uint8_t *data)
{
    // 代写入的数据
    // 数组大小为3的原因：buf[0]--命令（即上面的AHT_CALIBRATION_CMD、AHT_NORMAL_CMD、AHT_GET_DATA_CMD
    //                   buf[1]/buf[2]为命令后跟的参数，AHT有些命令后面需要加上参数，具体可查看数据手册
    rt_uint8_t buf[3];

    // 记录数组大小
    rt_uint8_t buf_size;

    // I2C传输的数据结构体
    struct rt_i2c_msg msgs;

    buf[0] = reg;
    if (data != RT_NULL)
    {
        buf[1] = data[0];
        buf[2] = data[1];
        buf_size = 3;
    }
    else
    {
        buf_size = 1;
    }

    msgs.addr = AHT_ADDR;   // 消息要发送的地址：即AHT地址
    msgs.flags = RT_I2C_WR; // 消息的标志位：读还是写，是否需要忽视ACK回应，是否需要发送停止位，是否需要发送开始位(用于拼接数据使用)...
    msgs.buf = buf;         // 消息的缓冲区：待发送/接收的数组
    msgs.len = buf_size;    // 消息的缓冲区大小：待发送/接收的数组的大小

    // 传输信息
    // 这里i2c.core层提供给我们三个API去进行I2C的数据传递：
    /*
     * 1.发送API
     * rt_size_t rt_i2c_master_send(struct rt_i2c_bus_device *bus,
                             rt_uint16_t               addr,
                             rt_uint16_t               flags,
                             const rt_uint8_t         *buf,
                             rt_uint32_t               count)

       2.接收API
       rt_size_t rt_i2c_master_recv(struct rt_i2c_bus_device *bus,
                             rt_uint16_t               addr,
                             rt_uint16_t               flags,
                             rt_uint8_t               *buf,
                             rt_uint32_t               count)
       3.传输API
       rt_size_t rt_i2c_transfer(struct rt_i2c_bus_device *bus,
                          struct rt_i2c_msg         msgs[],
                          rt_uint32_t               num)
      * 实际上1跟2最后都会调用回3，大家可以按照自己需求进行调用
    */
    if (rt_i2c_transfer(Device, &msgs, 1) == 1)
    {
        return RT_EOK;
    }
    else
    {
        return RT_ERROR;
    }
}

// 读数据（从机向主机返回数据）
rt_err_t read_reg(struct rt_i2c_bus_device *Device, rt_uint8_t len, rt_uint8_t *buf)
{
    struct rt_i2c_msg msgs;

    msgs.addr = AHT_ADDR;   // 消息要发送的地址：即AHT地址
    msgs.flags = RT_I2C_RD; // 消息的标志位：读还是写，是否需要忽视ACK回应，是否需要发送停止位，是否需要发送开始位(用于拼接数据使用)...
    msgs.buf = buf;         // 消息的缓冲区：待发送/接收的数组
    msgs.len = len;         // 消息的缓冲区大小：待发送/接收的数组的大小

    // 传输函数，上面有介绍
    if (rt_i2c_transfer(Device, &msgs, 1) == 1)
    {
        return RT_EOK;
    }
    else
    {
        return RT_ERROR;
    }
}

// 读取AHT的温湿度数据
void read_temp_humi(float *Temp_Data, float *Humi_Data)
{
    // 根据数据手册我们可以看到要读取一次数据需要使用到的数组大小为6
    rt_uint8_t Data[6];

    write_reg(i2c_bus, AHT_GET_DATA_CMD, Parm_Null); // 发送一个读取命令，让AHT进行一次数据采集
    rt_thread_mdelay(500);                           // 等待采集
    read_reg(i2c_bus, 6, Data);                      // 读取数据

    // 根据数据手册进行数据处理
    *Humi_Data = (Data[1] << 12 | Data[2] << 4 | (Data[3] & 0xf0) >> 4) * 100.0 / (1 << 20);
    *Temp_Data = ((Data[3] & 0x0f) << 16 | Data[4] << 8 | Data[5]) * 200.0 / (1 << 20) - 50;
}

// AHT进行初始化
void AHT_Init(const char *name)
{
    // 寻找AHT的总线设备
    i2c_bus = rt_i2c_bus_device_find(name);

    if (i2c_bus == RT_NULL)
    {
        rt_kprintf("Can't Find I2C_BUS Device"); // 找不到总线设备
    }
    else
    {
        write_reg(i2c_bus, AHT_NORMAL_CMD, Parm_Null); // 设置为正常工作模式
        rt_thread_mdelay(400);

        rt_uint8_t Temp[2]; // AHT_CALIBRATION_CMD需要的参数
        Temp[0] = 0x08;
        Temp[1] = 0x00;
        write_reg(i2c_bus, AHT_CALIBRATION_CMD, Temp);
        rt_thread_mdelay(400);
    }
}

// AHT设备测试线程
void AHT_test(void)
{
    float humidity, temperature;

    AHT_Init(AHT_I2C_BUS_NAME); // 进行设备初始化

    while (1)
    {
        read_temp_humi(&temperature, &humidity); // 读取数据
        rt_kprintf("humidity   : %d.%d %%\n", (int)humidity, (int)(humidity * 10) % 10);
        rt_kprintf("temperature: %d.%d\n", (int)temperature, (int)(temperature * 10) % 10);
        rt_thread_mdelay(1000);
    }
}
MSH_CMD_EXPORT(AHT_test, AHT_test); // 将命令到出到MSH列表
```
**使用软件包的话就很简单了，我们只需要在env工具中打开板上外设开关，之后在使用`pkgs -update`下载到本地即可，如下图所示。**
![](pictures\1.png)

**接下来就展示使用软件包后编写的代码喽。**
```C
#define AHT_I2C_BUS_NAME "i2c3"

void AHT_TEST(void)
{
    unsigned int count = 1;

    aht10_device_t AHT = aht10_init(AHT_I2C_BUS_NAME);

    float Temp, Humi;

    while (count > 0)
    {

        Humi = aht10_read_humidity(AHT);
        Temp = aht10_read_temperature(AHT);

        rt_kprintf("Tem: %.2f\n",Temp);
        rt_kprintf("Humi: %.2f \%\n",Humi);
        rt_thread_mdelay(1000);
        count++;
    }
}
MSH_CMD_EXPORT(AHT_TEST,AHT_TEST);
```
**然后烧录到板子上就可以实时显示温湿度了，如下图。**
![](pictures\2.png)

*OK,开始下一节，相信你已经能够感受到软件包的强大了！*

## MQTT协议（搭配阿里云平台）

MQTT（Message Queuing Telemetry Transport）是一种轻量级、基于发布-订阅模式的消息传输协议，适用于资源受限的设备和低带宽、高延迟或不稳定的网络环境。它在物联网应用中广受欢迎，能够实现传感器、执行器和其它设备之间的高效通信。

### 运行框架

**Client：**客户端，即我们使用的设备。

使用MQTT的程序或设备。客户端总是通过网络连接到服务端。它可以

- 发布应用消息给其它相关的客户端。
- 订阅以请求接受相关的应用消息。
- 取消订阅以移除接受应用消息的请求。
- 从服务端断开连接。

**Server：**服务端

作为发送消息的客户端和请求订阅的客户端之间的中介。服务端

- 接受来自客户端的网络连接。
- 接受客户端发布的应用消息。
- 处理客户端的订阅和取消订阅请求。
- 转发应用消息给符合条件的已订阅客户端。

**Topic Name：**主题名

附加在应用消息上的一个标签，服务端已知且与订阅匹配。服务端发送应用消息的一个副本给每一个匹配的客户端订阅。

**Subscription：** 订阅

订阅相应的主题名来获取对应的信息。

**Publish：**发布

### 阿里云环境搭建

#### 步骤一

进入阿里云物联网平台，然后开通公共实例，如下图
![](pictures\3.png)

#### 步骤二

在设备管理中新建一个产品，如下图
![](pictures\4.png)

#### 步骤三

查看我们新建的产品，然后将自定义Topic中第一个改成发布和订阅，因为我们想要让我们的板子既可以给云平台发布消息也想从云平台上订阅消息，如下图
![](pictures\5.png)

#### 步骤四

在设备管理中新建一个设备，如下图
![](pictures\6.png)

#### 步骤五

在env工具中使能RW007 wifi模块，因为上传云平台肯定少不了wifi呀，然后在env中找到阿里云的软件包，修改如下图中框住的四行，就是产品名称，产品密钥，设备名称，设备密钥，然后通过`pkgs --update`命令下载到本地。
![](pictures\7.png)

#### 步骤六

这就是最后一步喽，修改mqtt-example.c,这一步也是稍微有点复杂的一步了，根据你的需求自由发挥，你可以通过修改代码将任何传感器采集到的数据上传云端，我就展示一个简单一点的，将一个固定的温度值上传云端，修改的代码如下图所示
![](pictures\8.png)

其实改的也不多，就两行，一行是你要发布的主题，另一行就是你要上传的东西，我上传的比较简单，就是一个温度值。

#### 结果展示
将代码烧录到板子上后，进行如下操作，如图所示
![](pictures\9.png)
同时在阿里云平台上的设备物模型数据中也能看到温度值的变化，如下图
![](pictures\10.png)


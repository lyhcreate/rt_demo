# 通过小爱同学控制灯的开关，获取今天的天气

当你看到这个要实现的功能，你肯定会好奇要该怎么样连接小爱同学

我这里实现的方法是通过mqtt连接到巴法云平台，然后通过手机上的米家app连接巴法云平台，这样子就能通过手机唤醒小爱同学控制星火一号开发板了

所以接下来的步骤是

1、使星火一号开发板连接WiFi，连接上了网才能有接下来的步骤
2、去巴法云官网注册账号，注册mqtt设备云 ，在米家app上绑定巴法云平台，然后用mqtt.fx测试能否正常使用
3、在rtthread-studio上去使能paho mqtt、webclient、cjson ，更改里面配置 ，并测试板子是否能连接上巴法云
4、去心知天气注册账号，测试里面api能否使用
5、修改webclient代码
6、都测试好之后整合到一起




## wifi设置

rw007设置（上面有具体路径）
![image](https://github.com/user-attachments/assets/9276fcd7-b66b-45ef-b164-a88ddab5396f)



## 巴法云设置
注册mqtt设备云 教程 http://t.csdnimg.cn/WADsV
![image](https://github.com/user-attachments/assets/c2e77934-8890-431c-932c-f78d02b84bdc)

打开米家app，底部–我的—其他平台设备---->点击添加—>找打巴法，登录你的巴法云账号，如果巴法云控制台有创建设备，设备就会自动同步过去了。（如果没同步到，再次点击底部的同步设备即可）


mqtt.fx调试步骤 https://bemfa.com/m/mqttfx.html
1连接设置
![image](https://github.com/user-attachments/assets/db0b3c29-ae92-442c-88ff-70ae8273ec29)

![image](https://github.com/user-attachments/assets/0a176b8c-6ca5-43ea-8cf9-dc8a7023e6ab)

连接
![image](https://github.com/user-attachments/assets/96394bd4-f75d-4323-8657-f04f30a6e08e)

订阅与推送
![image](https://github.com/user-attachments/assets/8d54d131-b3ac-406c-a567-686ca62e92f2)

巴法云接入文档
https://cloud.bemfa.com/docs/src/mqtt.html

用的软件包
![image](https://github.com/user-attachments/assets/4fa9a4dd-71d1-4999-9aab-1ece3db4b362)

paho软件包设置，注意订阅主题数填3
![image](https://github.com/user-attachments/assets/24e9d09d-914d-42e0-b66b-d111e9c3e53e)

mqtt参考例程
```
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

#include <rtthread.h>

#define DBG_ENABLE
#define DBG_SECTION_NAME    "mqtt.sample"
#define DBG_LEVEL           DBG_LOG
#define DBG_COLOR
#include <rtdbg.h>

#include "paho_mqtt.h"

/**
 * MQTT URI farmat:
 * domain mode
 * tcp://iot.eclipse.org:1883
 *
 * ipv4 mode
 * tcp://192.168.10.1:1883
 * ssl://192.168.10.1:1884
 *
 * ipv6 mode
 * tcp://[fe80::20c:29ff:fe9a:a07e]:1883
 * ssl://[fe80::20c:29ff:fe9a:a07e]:1884
 */
#define MQTT_URI                "tcp://iot.eclipse.org:1883"   // 配置测试服务器地址
#define MQTT_USERNAME           "admin"   
#define MQTT_PASSWORD           "admin"
#define MQTT_SUBTOPIC           "/mqtt/test"                   // 设置订阅主题
#define MQTT_PUBTOPIC           "/mqtt/test"                   // 设置推送主题
#define MQTT_WILLMSG            "Goodbye!"                     // 设置遗言消息

/* 定义 MQTT 客户端环境结构体 */
static MQTTClient client;
static int is_started = 0;

/* MQTT 订阅事件自定义回调函数 */
static void mqtt_sub_callback(MQTTClient *c, MessageData *msg_data)
{
    *((char *)msg_data->message->payload + msg_data->message->payloadlen) = '\0';
    LOG_D("mqtt sub callback: %.*s %.*s",
               msg_data->topicName->lenstring.len,
               msg_data->topicName->lenstring.data,
               msg_data->message->payloadlen,
               (char *)msg_data->message->payload);
}

/* MQTT 订阅事件默认回调函数 */
static void mqtt_sub_default_callback(MQTTClient *c, MessageData *msg_data)
{
    *((char *)msg_data->message->payload + msg_data->message->payloadlen) = '\0';
    LOG_D("mqtt sub default callback: %.*s %.*s",
               msg_data->topicName->lenstring.len,
               msg_data->topicName->lenstring.data,
               msg_data->message->payloadlen,
               (char *)msg_data->message->payload);
}

/* MQTT 连接事件回调函数 */
static void mqtt_connect_callback(MQTTClient *c)
{
    LOG_D("inter mqtt_connect_callback!");
}

/* MQTT 上线事件回调函数 */
static void mqtt_online_callback(MQTTClient *c)
{
    LOG_D("inter mqtt_online_callback!");
}

/* MQTT 下线事件回调函数 */
static void mqtt_offline_callback(MQTTClient *c)
{
    LOG_D("inter mqtt_offline_callback!");
}

static int mqtt_start(int argc, char **argv)
{
    /* 使用 MQTTPacket_connectData_initializer 初始化 condata 参数 */
    MQTTPacket_connectData condata = MQTTPacket_connectData_initializer;
    static char cid[20] = { 0 };

    if (argc != 1)
    {
        rt_kprintf("mqtt_start    --start a mqtt worker thread.\n");
        return -1;
    }

    if (is_started)
    {
        LOG_E("mqtt client is already connected.");
        return -1;
    }
     /* 配置 MQTT 结构体内容参数 */
    {
        client.isconnected = 0;
        client.uri = MQTT_URI;

        /* 产生随机的客户端 ID */
        rt_snprintf(cid, sizeof(cid), "rtthread%d", rt_tick_get());
        /* 配置连接参数 */
        rt_memcpy(&client.condata, &condata, sizeof(condata));
        client.condata.clientID.cstring = cid;
        client.condata.keepAliveInterval = 30;
        client.condata.cleansession = 1;
        client.condata.username.cstring = MQTT_USERNAME;
        client.condata.password.cstring = MQTT_PASSWORD;

        /* 配置 MQTT 遗言参数 */
        client.condata.willFlag = 1;
        client.condata.will.qos = 1;
        client.condata.will.retained = 0;
        client.condata.will.topicName.cstring = MQTT_PUBTOPIC;
        client.condata.will.message.cstring = MQTT_WILLMSG;

         /* 分配缓冲区 */
        client.buf_size = client.readbuf_size = 1024;
        client.buf = rt_calloc(1, client.buf_size);
        client.readbuf = rt_calloc(1, client.readbuf_size);
        if (!(client.buf && client.readbuf))
        {
            LOG_E("no memory for MQTT client buffer!");
            return -1;
        }

        /* 设置事件回调函数 */
        client.connect_callback = mqtt_connect_callback;
        client.online_callback = mqtt_online_callback;
        client.offline_callback = mqtt_offline_callback;

        /* 设置订阅表和事件回调函数*/
        client.messageHandlers[0].topicFilter = rt_strdup(MQTT_SUBTOPIC);
        client.messageHandlers[0].callback = mqtt_sub_callback;
        client.messageHandlers[0].qos = QOS1;

        /* 设置默认的订阅主题*/
        client.defaultMessageHandler = mqtt_sub_default_callback;
    }

    /* 运行 MQTT 客户端 */
    paho_mqtt_start(&client);
    is_started = 1;

    return 0;
}

/* 该函数用于停止 MQTT 客户端并释放内存空间 */
static int mqtt_stop(int argc, char **argv)
{
    if (argc != 1)
    {
        rt_kprintf("mqtt_stop    --stop mqtt worker thread and free mqtt client object.\n");
    }

    is_started = 0;

    return paho_mqtt_stop(&client);
}

/* 该函数用于发送数据到指定 topic */
static int mqtt_publish(int argc, char **argv)
{
    if (is_started == 0)
    {
        LOG_E("mqtt client is not connected.");
        return -1;
    }

    if (argc == 2)
    {
        paho_mqtt_publish(&client, QOS1, MQTT_PUBTOPIC, argv[1]);
    }
    else if (argc == 3)
    {
        paho_mqtt_publish(&client, QOS1, argv[1], argv[2]);
    }
    else
    {
        rt_kprintf("mqtt_publish <topic> [message]  --mqtt publish message to specified topic.\n");
        return -1;
    }

    return 0;
}

/* MQTT 新的订阅事件自定义回调函数 */
static void mqtt_new_sub_callback(MQTTClient *client, MessageData *msg_data)
{
    *((char *)msg_data->message->payload + msg_data->message->payloadlen) = '\0';
    LOG_D("mqtt new subscribe callback: %.*s %.*s",
               msg_data->topicName->lenstring.len,
               msg_data->topicName->lenstring.data,
               msg_data->message->payloadlen,
               (char *)msg_data->message->payload);
}

/* 该函数用于订阅新的 Topic */
static int mqtt_subscribe(int argc, char **argv)
{
    if (argc != 2)
    {
        rt_kprintf("mqtt_subscribe [topic]  --send an mqtt subscribe packet and wait for suback before returning.\n");
        return -1;
    }
	
	if (is_started == 0)
    {
        LOG_E("mqtt client is not connected.");
        return -1;
    }

    return paho_mqtt_subscribe(&client, QOS1, argv[1], mqtt_new_sub_callback);
}

/* 该函数用于取消订阅指定的 Topic */
static int mqtt_unsubscribe(int argc, char **argv)
{
    if (argc != 2)
    {
        rt_kprintf("mqtt_unsubscribe [topic]  --send an mqtt unsubscribe packet and wait for suback before returning.\n");
        return -1;
    }
	
	if (is_started == 0)
    {
        LOG_E("mqtt client is not connected.");
        return -1;
    }

    return paho_mqtt_unsubscribe(&client, argv[1]);
}

#ifdef FINSH_USING_MSH
MSH_CMD_EXPORT(mqtt_start, startup mqtt client);
MSH_CMD_EXPORT(mqtt_stop, stop mqtt client);
MSH_CMD_EXPORT(mqtt_publish, mqtt publish message to specified topic);
MSH_CMD_EXPORT(mqtt_subscribe,  mqtt subscribe topic);
MSH_CMD_EXPORT(mqtt_unsubscribe, mqtt unsubscribe topic);
#endif /* FINSH_USING_MSH */
```

复制后把前面修改成下图所示 主题改成你巴法云设置的主题
![image](https://github.com/user-attachments/assets/e52593fa-a416-411b-bdb8-544585146185)

测试板子是否能连接上巴法云，在这之前你要手动连接wifi，命令使wifi join 你的wifi名字 密码

成功后你就可以在MQTT 订阅事件自定义回调函数 添加你想控制的东西，比如灯
我的代码是
```
    LOG_D("mes is %s",msg_data->message->payload);

    strcpy(flag,(char *)msg_data->message->payload);
    LOG_D("flag:%c",flag[1]);
    if(flag[1]=='n'){
        LOG_D("led on");
       rt_pin_write(GPIO_LED_B, PIN_LOW);
       

    }
    if(flag[1]=='f') {

        LOG_D("led off");
       rt_pin_write(GPIO_LED_B, PIN_HIGH);
       

    }
```
根据参考例程可以知道，我们是要对msg_data->message->payload这里面的数据进行判断
所以可以定义一个字符数组接受数据，然后进行判断控制

# 实现连接管理
connmgr_run()：实现连接管理（ofproto/connmgr.c）
connmgr顾名思义（connect manager，连接管理器），主要完成OVS网桥的连接管理。每一个网桥ofproto都有一个connmgr实体来管理连接。connmgr主要完成两种连接管理，一种是主动连接（即与控制器连接，作为客户端），一种是被动连接（主要提供ofctl等工具连接请求服务和snoop服务，作为服务端）。connmgr结构体主要内容见下面，all_conns是主动连接，services和snoops等提供被动连接服务。
参考https://blog.csdn.net/qq_20817327/article/details/106533029?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-5.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-5.nonecase
# openflow 连接建立
在bridge_reconfigure()函数中调用bridge_configure_remotes 进行openflow 连接
的相关处理，主要创建两个对象：ofconn 作为客户端负责和远端conntroller 主动建立连
接；ofservice 作为服务器提供被动式的监听服务

```
main (vswitchd/ovs-vswitchd.c:127)
    +
    +--bridge_init
    +
    +--bridge_run
    +    + 
    +    +--bridge_reconfigure
    +    +    +
    +    +    +--bridge_configure_remotes
    +    +    +    +
    +    +    +    +--bridge_get_controllers(br, &controllers) /* 从表里获取配置信息 */
    +    +    +    +
    +    +    +    +--ofproto_set_controllers(br->ofproto, &ocs);
    +    +    +    +    +
    +    +    +    +    +--connmgr_set_controllers(p->connmgr, controllers)
    +    +    +    +    +    +
    +    +    +    +    +    +--ofservice_create(struct connmgr *mgr, const char *target,const struct ofproto_controller *c)
    +    +    +    +    +    +    +
    +    +    +    +    +    +    +--rconn_create   /* 主动连接 */
    +    +    +    +    +    +    +
    +    +    +    +    +    +    +--rconn_connect
    +    +    +    +    +    +    +
    +    +    +    +    +    +    +--pvconn_open   /* 被动连接 */
```

char *name = ofconn_make_name(mgr, target);
"ubuntu_br<->tcp:121.229.17.188:6653"

#0  vconn_stream_connect (vconn=0x9cc600) at lib/vconn-stream.c:119
#1  0x00000000005bf7df in vcs_connecting (vconn=0x9cc600) at lib/vconn.c:399
#2  0x00000000005bfe96 in vconn_connect (vconn=0x9cc600) at lib/vconn.c:555
#3  0x00000000005bf5b6 in vconn_run (vconn=0x9cc600) at lib/vconn.c:274
#4  0x000000000059398f in rconn_run (rc=0x9cc4e0) at lib/rconn.c:631
#5  0x000000000047a7d4 in ofservice_run (ofservice=0x9cc750) at ofproto/connmgr.c:1992
#6  0x0000000000476ac5 in connmgr_run (mgr=0x9a8860, handle_openflow=0x42f6df <handle_openflow>) at ofproto/connmgr.c:367
#7  0x000000000041fad7 in ofproto_run (p=0x9a8310) at ofproto/ofproto.c:1826
#8  0x000000000040edd3 in bridge_run__ () at vswitchd/bridge.c:2965
#9  0x00000000004082a1 in bridge_reconfigure (ovs_cfg=0x97b990) at vswitchd/bridge.c:723
#10 0x000000000040f0a7 in bridge_run () at vswitchd/bridge.c:3044
#11 0x0000000000414b70 in main (argc=6, argv=0x7fffffffe568) at vswitchd/ovs-vswitchd.c:127

发送OFPRAW_OFPT_ECHO_REQUEST
```
#0  vconn_send (vconn=0x9cc650, msg=0x9cb7f0) at lib/vconn.c:669
#1  0x00000000005946c9 in try_send (rc=0x9cc430) at lib/rconn.c:1128
#2  0x0000000000593e62 in rconn_send__ (rc=0x9cc430, b=0x9cb7f0, counter=0x0) at lib/rconn.c:760
#3  0x0000000000593849 in run_ACTIVE (rc=0x9cc430) at lib/rconn.c:574
#4  0x0000000000593ab4 in rconn_run (rc=0x9cc430) at lib/rconn.c:660
#5  0x0000000000478e2b in ofconn_run (ofconn=0x9d6390, handle_openflow=0x42f6df <handle_openflow>) at ofproto/connmgr.c:1298
#6  0x0000000000476a2d in connmgr_run (mgr=0x9a8820, handle_openflow=0x42f6df <handle_openflow>) at ofproto/connmgr.c:355
#7  0x000000000041fad7 in ofproto_run (p=0x9a82d0) at ofproto/ofproto.c:1826
#8  0x000000000040edd3 in bridge_run__ () at vswitchd/bridge.c:2965
#9  0x000000000040efc9 in bridge_run () at vswitchd/bridge.c:3023
#10 0x0000000000414b70 in main (argc=6, argv=0x7fffffffe568) at vswitchd/ovs-vswitchd.c:127

```
接收OFPTYPE_HELLO   
收消息入口vconn_recv(struct vconn *vconn, struct ofpbuf **msgp) 
```
#0  vcs_recv_hello (vconn=0x9cc9f0) at lib/vconn.c:462   进入时vconn->state=2
#1  0x00000000005bfeb2 in vconn_connect (vconn=0x9cc9f0) at lib/vconn.c:563
#2  0x00000000005bf5b6 in vconn_run (vconn=0x9cc9f0) at lib/vconn.c:274
#3  0x000000000059398f in rconn_run (rc=0x9cc7f0) at lib/rconn.c:631
#4  0x000000000047a7d4 in ofservice_run (ofservice=0x9cca40) at ofproto/connmgr.c:1992
#5  0x0000000000476ac5 in connmgr_run (mgr=0x9a7d00, handle_openflow=0x42f6df <handle_openflow>) at ofproto/connmgr.c:367
#6  0x000000000041fad7 in ofproto_run (p=0x9a9100) at ofproto/ofproto.c:1826
#7  0x000000000040edd3 in bridge_run__ () at vswitchd/bridge.c:2965
#8  0x000000000040efc9 in bridge_run () at vswitchd/bridge.c:3023
#9  0x0000000000414b70 in main (argc=6, argv=0x7fffffffe568) at vswitchd/ovs-vswitchd.c:127

```
p *counter_vconn_received_get()

```
#0  do_recv (vconn=0x9cc810, msgp=0x7fffffffe100) at lib/vconn.c:653
#1  0x00000000005bf9ed in vcs_recv_hello (vconn=0x9cc810) at lib/vconn.c:456
#2  0x00000000005bfeb2 in vconn_connect (vconn=0x9cc810) at lib/vconn.c:563
#3  0x00000000005bf5b6 in vconn_run (vconn=0x9cc810) at lib/vconn.c:274
#4  0x000000000059398f in rconn_run (rc=0x9cc610) at lib/rconn.c:631
#5  0x000000000047a7d4 in ofservice_run (ofservice=0x9cc860) at ofproto/connmgr.c:1992
#6  0x0000000000476ac5 in connmgr_run (mgr=0x9a80c0, handle_openflow=0x42f6df <handle_openflow>) at ofproto/connmgr.c:367
#7  0x000000000041fad7 in ofproto_run (p=0x9a7b70) at ofproto/ofproto.c:1826
#8  0x000000000040edd3 in bridge_run__ () at vswitchd/bridge.c:2965
#9  0x000000000040efc9 in bridge_run () at vswitchd/bridge.c:3023
#10 0x0000000000414b70 in main (argc=6, argv=0x7fffffffe568) at vswitchd/ovs-vswitchd.c:127
```
一开始vconn->state = 0，调用vcs_connecting(vconn)，将状态vconn->state = VCS_SEND_HELLO
再调用vcs_send_hello(vconn)，发送完HELLO报文后vconn->state = VCS_RECV_HELLO，如果此时已经收到HELLO报文（SDN conntroller先发送HELLO报文），再调用vcs_recv_hello(vconn)，vconn->state = VCS_CONNECTED
```
int
vconn_connect(struct vconn *vconn)
{
    enum vconn_state last_state;

    do {
        last_state = vconn->state;
        switch (vconn->state) {
        case VCS_CONNECTING:
            vcs_connecting(vconn);
            break;

        case VCS_SEND_HELLO:
            vcs_send_hello(vconn);
            break;

        case VCS_RECV_HELLO:
            vcs_recv_hello(vconn);
            break;

        case VCS_CONNECTED:
            return 0;

        case VCS_SEND_ERROR:
            vcs_send_error(vconn);
            break;

        case VCS_DISCONNECTED:
            ovs_assert(vconn->error != 0);
            return vconn->error;

        default:
            OVS_NOT_REACHED();
        }
    } while (vconn->state != last_state);

    return EAGAIN;
}
```
发送request过程
```
ofproto_run
    +
    +--connmgr_run
    +    + 
    +    +--ofservice_run
    +    +    +
    +    +    +--rconn_run
    +    +    +    +
    +    +    +    +--vconn_run                                       1
    +    +    +    +    +
    +    +    +    +    +--vconn_connect(vconn)              1.1
    +    +    +    +    +    +    VCS_CONNECTING
    +    +    +    +    +    +--vcs_connecting(vconn)       1.2
    +    +    +    +    +    +   VCS_SEND_HELLO
    +    +    +    +    +    +--vcs_send_hello(vconn)        1.3
    +    +    +    +    +    +   VCS_RECV_HELLO
    +    +    +    +    +    +--vcs_recv_hello(vconn)         1.4   
    +    +    +    +                VCS_CONNECTED
    +    +    +    +--run_CONNECTING                          2
    +    +    +    +          S_ACTIVE      
    +    +    +    +--run_ACTIVE(struct rconn *rc)           3
    +    +    +    +    +     S_IDLE      
    +    +    +    +    +      /* Send an echo request. */
    +    +    +    +    +--rconn_send__(struct rconn *rc, struct ofpbuf *b,struct rconn_packet_counter *counter)   
    +    +    +    +    +    +
    +    +    +    +    +    +--try_send(struct rconn *rc)
    +    +    +    +    +    +    +
    +    +    +    +    +    +    +--vconn_send(rc->vconn, msg)
    +    +    +    +
    +    +    +    +--run_IDLE                                         4
    +    +    +    +    +
    +    +    +    +    +--do_tx_work(struct rconn *rc)
```
收到OFPTYPE_ECHO_REPLY
```
#0  handle_openflow (ofconn=0x9cc520, msgs=0x7fffffffe210) at ofproto/ofproto.c:8584
#1  0x0000000000478eee in ofconn_run (ofconn=0x9cc520, handle_openflow=0x42f6df <handle_openflow>) at ofproto/connmgr.c:1318
#2  0x0000000000476a2d in connmgr_run (mgr=0x9a7ca0, handle_openflow=0x42f6df <handle_openflow>) at ofproto/connmgr.c:355
#3  0x000000000041fad7 in ofproto_run (p=0x9a90e0) at ofproto/ofproto.c:1826
#4  0x000000000040edd3 in bridge_run__ () at vswitchd/bridge.c:2965
#5  0x000000000040efc9 in bridge_run () at vswitchd/bridge.c:3023
#6  0x0000000000414b70 in main (argc=6, argv=0x7fffffffe568) at vswitchd/ovs-vswitchd.c:127
```
每隔一段时间
![b0f7da8584538a42943f44dc41b4b2c0d66cfcf0](b0f7da8584538a42943f44dc41b4b2c0d66cfcf0.png)
# ovs处理openflow消息的流程
![f2df46376b216ed2bf6306f9810692eca2bde8e0](f2df46376b216ed2bf6306f9810692eca2bde8e0.png)
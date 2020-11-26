# OVS 代码架构
![c423b0e9acfba8cb068c22951fc4a786e829e4f2](c423b0e9acfba8cb068c22951fc4a786e829e4f2.png)

struct ofproto抽象了OpenFlow交换机，一个ofproto实例就是一个OpenFlow交换机(网桥)，
相关的数据结构(ofproto/ofproto-provider.h):
struct ofproto: 表示一个OpenFlow交换机(OVS网桥)，所有的流/端口操作都针对ofproto
struct ofport: 表示ofproto内的一个端口
struct rule: 表示ofproto内的一个OpenFlow流
struct ofgroup:表示ofproto内的一个OpenFlow 1.1+组

结构体ofproto代表一个ovs网桥，其含有of表、meter表、port、连接管理connmgr等属性，相比bridge结构体，s属于更为切实的一个ovs网桥的属性集合体
# ofproto 数据结构
![7a6e2fc804c60b514a6df0403b79a1ca7d7c67b6](7a6e2fc804c60b514a6df0403b79a1ca7d7c67b6.png)
# ofproto 创建流程

```

main (vswitchd/ovs-vswitchd.c:127)
    +
    +--bridge_init
    +
    +--bridge_run
    +    + 
    +    +--bridge_reconfigure
    +    +     +    遍例all_bridges，创建ofprotos，保存到网桥
    +    +     +--ofproto_create(br->name, br->type, &br->ofproto)
    +    +     +    +
    +    +     +    +--ofproto_class_find__  找到ofproto_dpif_class
    +    +     +    +
    +    +     +    +--ofproto = class->alloc()      分配ofproto_dpif，相当于派生类对象
    +    +     +    +
    +    +     +    +--ofproto->connmgr = connmgr_create   创建openflow连接管理
    +    +     +    +
    +    +     +    +--ofproto->ofproto_class->construct(ofproto)  初始化ofproto_dpif
    +    +     + 
    +    +     +--bridge_run__
    +    +     +    +
    +    +     +    +--ofproto_run(br->ofproto)
    +    +     +    +    +   ofproto/ofproto-dpif.c文件里
    +    +     +    +    +--p->ofproto_class->run(p) 
```
创建网桥
bridge_reconfigure(const struct ovsrec_open_vswitch *ovs_cfg)
```C
    /* Finish pushing configuration changes to the ofproto layer:
     *
     *     - Create ofprotos that are missing.
     *
     *     - Add ports that are missing. */
    HMAP_FOR_EACH_SAFE (br, next, node, &all_bridges) {
        if (!br->ofproto) {
            int error;

            error = ofproto_create(br->name, br->type, &br->ofproto);
            if (error) {
                VLOG_ERR("failed to create bridge %s: %s", br->name,
                         ovs_strerror(error));
                shash_destroy(&br->wanted_ports);
                bridge_destroy(br, true);
            } else {
                /* Trigger storing datapath version. */
                seq_change(connectivity_seq_get());
            }
        }
    }
```
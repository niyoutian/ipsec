# OVS ����ܹ�
![c423b0e9acfba8cb068c22951fc4a786e829e4f2](c423b0e9acfba8cb068c22951fc4a786e829e4f2.png)

struct ofproto������OpenFlow��������һ��ofprotoʵ������һ��OpenFlow������(����)��
��ص����ݽṹ(ofproto/ofproto-provider.h):
struct ofproto: ��ʾһ��OpenFlow������(OVS����)�����е���/�˿ڲ��������ofproto
struct ofport: ��ʾofproto�ڵ�һ���˿�
struct rule: ��ʾofproto�ڵ�һ��OpenFlow��
struct ofgroup:��ʾofproto�ڵ�һ��OpenFlow 1.1+��

�ṹ��ofproto����һ��ovs���ţ��京��of��meter��port�����ӹ���connmgr�����ԣ����bridge�ṹ�壬s���ڸ�Ϊ��ʵ��һ��ovs���ŵ����Լ�����
# ofproto ���ݽṹ
![7a6e2fc804c60b514a6df0403b79a1ca7d7c67b6](7a6e2fc804c60b514a6df0403b79a1ca7d7c67b6.png)
# ofproto ��������

```

main (vswitchd/ovs-vswitchd.c:127)
    +
    +--bridge_init
    +
    +--bridge_run
    +    + 
    +    +--bridge_reconfigure
    +    +     +    ����all_bridges������ofprotos�����浽����
    +    +     +--ofproto_create(br->name, br->type, &br->ofproto)
    +    +     +    +
    +    +     +    +--ofproto_class_find__  �ҵ�ofproto_dpif_class
    +    +     +    +
    +    +     +    +--ofproto = class->alloc()      ����ofproto_dpif���൱�����������
    +    +     +    +
    +    +     +    +--ofproto->connmgr = connmgr_create   ����openflow���ӹ���
    +    +     +    +
    +    +     +    +--ofproto->ofproto_class->construct(ofproto)  ��ʼ��ofproto_dpif
    +    +     + 
    +    +     +--bridge_run__
    +    +     +    +
    +    +     +    +--ofproto_run(br->ofproto)
    +    +     +    +    +   ofproto/ofproto-dpif.c�ļ���
    +    +     +    +    +--p->ofproto_class->run(p) 
```
��������
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
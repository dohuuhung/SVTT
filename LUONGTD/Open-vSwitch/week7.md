# ovs-vswitchd
## [1. Kiến trúc](#architech)
## [2. Unit tests](#test)
---
## <a name="architech"></a> 1. Kiến trúc
### 1.1. Hàm main trong file vswitchd/ovs-vswitchd.c:

```c
int main()
{
    /* step.1. khởi tạo bridge module, được thực hiện trong vswitchd/bridge.c
    Module bridge sẽ lấy một số tham số cấu hình từ ovsdb */
    bridge_init();

    /* step.2. deamon loop */
    while (!exiting) {
        /* step.2.1. Xử lý control messages từ OpenFlow Controller và CLI */
        bridge_run()
          |
          |--dpdk_init()
          |--bridge_init_ofproto() 			// khởi tạo bridge
          |--bridge_run__()
              |
              |--for(datapath types):		/* Mỗi datapath sẽ thực hiện công việc của mình bằng cách chạy ofproto_type_run(), 
              								vswitchd sẽ gọi vào kiểu thực thi type_run() cụ thể của kiểu dữ liệu đó. */
                     ofproto_type_run(type)

              |--for(all_bridges):			/* Mỗi bridge sẽ thực hiện công việc của nó bằng cách chạy ofproto_run()
              								vswitchd sẽ gọi vào hàm run() cụ thể của class ofproto.*/
                     ofproto_run(bridge) 	// xử lý message từ OpenFlow Controller

        unixctl_server_run(unixctl); 		/* Nhận control message từ CLI (ovs-appctl <xxx>) */

        netdev_run(); 						/* Performs periodic work needed by all the various kinds of netdevs
											Xử lý các loại netdev khác nhau 
											*/

        /* step.2.2. đợi cho events đến */
        bridge_wait();
        unixctl_server_wait(unixctl);
        netdev_wait();

        /* step.2.3. block cho đến khi events đến */
        poll_block();
    }
}
```

### 1.2. Dependancy graph của vswitchd:

![](images/vswitchd/understand_1.png)

## <a name="test"></a> 2. Unit tests
Danh sách các test cho ```ovs-vswitchd```:
```sh
$ make check TESTSUITEFLAGS="-l -k ovs-vswitchd"
```
```sh
 NUM: FILE-NAME:LINE     TEST-GROUP-NAME
      KEYWORDS

 537: unixctl-py.at:20   unixctl ovs-vswitchd exit - Python2
      python unixctl
 538: unixctl-py.at:21   unixctl ovs-vswitchd exit - Python3
      python unixctl
 539: unixctl-py.at:38   unixctl ovs-vswitchd list-commands - Python2
 540: unixctl-py.at:39   unixctl ovs-vswitchd list-commands - Python3
 541: unixctl-py.at:84   unixctl ovs-vswitchd arguments - Python2
 542: unixctl-py.at:85   unixctl ovs-vswitchd arguments - Python3
 846: ovs-vswitchd.at:7  ovs-vswitchd detaches correctly with empty db
 847: ovs-vswitchd.at:36 ovs-vswitchd -- stats-update-interval
 848: ovs-vswitchd.at:69 ovs-vswitchd -- start additional ovs-vswitchd process
 849: ovs-vswitchd.at:94 ovs-vswitchd -- switch over to another ovs-vswitchd process
 850: ovs-vswitchd.at:134 ovs-vswitchd -- invalid database path
 851: ovs-vswitchd.at:159 ovs-vswitchd -- set service controller
 852: ovs-vswitchd.at:176 ovs-vswitchd -- Compatible with OVSDB server - w/o monitor_cond
 853: ovs-vswitchd.at:197 ovs-vswitchd - do not create sockets with unsafe names
 854: ovs-vswitchd.at:230 ovs-vswitchd - set datapath IDs
 2454: ovn.at:5297        ovn -- ovs-vswitchd restart
      vswitchd
```
### Test 0: detaches correctly with empty db
```sh
dnl The OVS initscripts never make an empty database (one without even an
dnl Open_vSwitch record) visible to ovs-vswitchd, but hand-rolled scripts
dnl sometimes do.  At one point, "ovs-vswitchd --detach" would never detach
dnl and use 100% CPU if this happened, so this test checks for regression.
AT_SETUP([ovs-vswitchd detaches correctly with empty db])
on_exit 'kill `cat ovsdb-server.pid ovs-vswitchd.pid`'

dnl Create database.
touch .conf.db.~lock~
AT_CHECK([ovsdb-tool create conf.db $abs_top_srcdir/vswitchd/vswitch.ovsschema])

dnl Start ovsdb-server. Don't initialize database.
AT_CHECK([ovsdb-server --detach --no-chdir --pidfile --log-file --remote=punix:$OVS_RUNDIR/db.sock], [0], [ignore], [ignore])
AT_CAPTURE_FILE([ovsdb-server.log])

dnl Start ovs-vswitchd.
AT_CHECK([ovs-vswitchd --detach --no-chdir --pidfile --enable-dummy --disable-system --log-file], [0], [], [stderr])
AT_CAPTURE_FILE([ovs-vswitchd.log])

dnl ovs-vswitchd detached OK or we wouldn't have made it this far.  Success.
OVS_APP_EXIT_AND_WAIT([ovs-vswitchd])
OVS_APP_EXIT_AND_WAIT([ovsdb-server])
AT_CLEANUP
```

### Test 1: stats-update-interval
```sh
AT_SETUP([ovs-vswitchd -- stats-update-interval])
OVS_VSWITCHD_START([add-port br0 p1 -- set int p1 type=internal])
ovs-appctl time/stop

dnl at the beginning, the update of rx_packets should happen every 5 seconds.
ovs-appctl time/warp 11000 1000
OVS_VSCTL_CHECK_RX_PKT([p1], [0])
AT_CHECK([ovs-appctl netdev-dummy/receive p1 'eth(src=50:54:00:00:00:09,dst=50:54:00:00:00:0a),eth_type(0x0800),ipv4(src=10.0.0.2,dst=10.0.0.1,proto=1,tos=0,ttl=64,frag=no),icmp(type=8,code=0)'])
ovs-appctl time/warp 11000 1000
OVS_VSCTL_CHECK_RX_PKT([p1], [1])

dnl set the stats update interval to 100K ms, the following 'recv' should not be updated.
AT_CHECK([ovs-vsctl set O . other_config:stats-update-interval=100000])
ovs-appctl time/warp 51000 1000
for i in `seq 1 5`; do
    AT_CHECK([ovs-appctl netdev-dummy/receive p1 'eth(src=50:54:00:00:00:09,dst=50:54:00:00:00:0a),eth_type(0x0800),ipv4(src=10.0.0.2,dst=10.0.0.1,proto=1,tos=0,ttl=64,frag=no),icmp(type=8,code=0)'])
done

OVS_VSCTL_CHECK_RX_PKT([p1], [1])
dnl advance the clock by 100K ms, the previous 'recv' should be updated.
ovs-appctl time/warp 100000 1000
OVS_VSCTL_CHECK_RX_PKT([p1], [6])

dnl now remove the configuration. 'recv' one packet.  there should be an update after 5000 ms.
AT_CHECK([ovs-vsctl clear O . other_config])
AT_CHECK([ovs-appctl netdev-dummy/receive p1 'eth(src=50:54:00:00:00:09,dst=50:54:00:00:00:0a),eth_type(0x0800),ipv4(src=10.0.0.2,dst=10.0.0.1,proto=1,tos=0,ttl=64,frag=no),icmp(type=8,code=0)'])
ovs-appctl time/warp 11000 1000
OVS_VSCTL_CHECK_RX_PKT([p1], [7])

OVS_VSWITCHD_STOP
AT_CLEANUP
```

### Test 2: khởi động ovs-vswitchd process phụ
```sh
dnl ----------------------------------------------------------------------
AT_SETUP([ovs-vswitchd -- start additional ovs-vswitchd process])
OVS_VSWITCHD_START

# start another ovs-vswitchd process.
ovs-vswitchd --log-file=fakelog --unixctl=unixctl2 --pidfile=ovs-vswitchd-2.pid &
on_exit 'kill `cat ovs-vswitchd-2.pid`'

# sleep for a while
sleep 5

# stop the process.
OVS_APP_EXIT_AND_WAIT_BY_TARGET(["`pwd`"/unixctl2], [`pwd`/ovs-vswitchd-2.pid])

# check the fakelog, should only see one ERR for reporting
# the existing ovs-vswitchd process.
AT_CHECK([test `grep ERR fakelog | wc -l` -eq 1])

AT_CHECK([tail -n1 fakelog | sed -e 's/^.*ERR|//; s/pid [[0-9]]*//'], [0], [dnl
another ovs-vswitchd process is running, disabling this process () until it goes away
])

OVS_VSWITCHD_STOP
AT_CLEANUP

```

### Test 3: chuyển sang ovs-vswitchd process khác
```sh
dnl ----------------------------------------------------------------------
AT_SETUP([ovs-vswitchd -- switch over to another ovs-vswitchd process])
OVS_VSWITCHD_START

# start a new ovs-vswitchd process.
ovs-vswitchd --log-file=fakelog --enable-dummy --unixctl=unixctl2 --pidfile=ovs-vswitchd-2.pid &
on_exit 'kill `cat ovs-vswitchd-2.pid`'

# sleep for a while.
sleep 5

# kill the current active ovs-vswitchd process.
kill `cat ovs-vswitchd.pid`

sleep 5

# check the creation of br0 on the new ovs-vswitchd process.
AT_CHECK([grep "bridge br0" fakelog | sed -e 's/port [[0-9]]*$/port/;
s/datapath ID [[a-z0-9]]*$/datapath ID/;s/^.*INFO|//'], [0], [dnl
bridge br0: added interface br0 on port
bridge br0: using datapath ID
])

# stop the process.
OVS_APP_EXIT_AND_WAIT_BY_TARGET(["`pwd`"/unixctl2], [ovs-vswitchd-2.pid])

# check the fakelog, should not see WARN/ERR/EMER log other than the one
# for reporting the existing ovs-vswitchd process and the one for killing
# the process.
AT_CHECK([sed -n "
/|ERR|another ovs-vswitchd process is running/d
/|WARN|/p
/|ERR|/p
/|EMER|/p" fakelog
])

# cleanup.
kill `cat ovsdb-server.pid`
AT_CLEANUP

```

### Test 4: invalid database path
- Bắt đầu một tiến trình ovs-vswitchd với invalid db path
- Dừng tiến trình này lại
- Kiểm tra file log (không có WARN/ERR/ỂM log nào thì test đúng)
```sh
AT_SETUP([ovs-vswitchd -- invalid database path])

# start an ovs-vswitchd process with invalid db path.
ovs-vswitchd unix:invalid.db.sock --log-file --enable-dummy --pidfile &
on_exit 'kill `cat ovs-vswitchd.pid`'

# sleep for a while.
sleep 10

# stop the process.
OVS_APP_EXIT_AND_WAIT([ovs-vswitchd])

# should not see this log (which indicates high cpu utilization).
AT_CHECK([grep "wakeup due to" ovs-vswitchd.log], [ignore])

# check the log, should not see any WARN/ERR/EMER log.
AT_CHECK([sed -n "
/|WARN|/p
/|ERR|/p
/|EMER|/p" ovs-vswitchd.log
])

AT_CLEANUP
```

### Test 5: set service controller
```sh
AT_SETUP([ovs-vswitchd -- set service controller])
AT_SKIP_IF([test "$IS_WIN32" = "yes"])
OVS_VSWITCHD_START

AT_CHECK([ovs-vsctl set-controller br0 punix:$(pwd)/br0.void])
OVS_WAIT_UNTIL([test -e br0.void])

AT_CHECK([ovs-vsctl set-controller br0 punix:$(pwd)/br0.void/../overwrite.file])
OVS_WAIT_UNTIL([test -n "`grep ERR ovs-vswitchd.log | grep overwrite.file`"])

OVS_VSWITCHD_STOP(["/Not adding Unix domain socket controller/d"])
AT_CLEANUP
```

### Test 6: Tương thích với OVSDB server - w/o monitor_cond
```sh
dnl OVSDB server before release version 2.5 does not support the monitor_cond
dnl method.  This test defeatures the OVSDB server to simulate an older
dnl OVSDB server and make sure ovs-vswitchd can still work with it
AT_SETUP([ovs-vswitchd -- Compatible with OVSDB server - w/o monitor_cond])
OVS_VSWITCHD_START

dnl defeature OVSDB server -- no monitor_cond
AT_CHECK([ovs-appctl -t ovsdb-server ovsdb-server/disable-monitor-cond])

sleep 1

AT_CHECK([ovs-vsctl add-port br0 p0  -- set interface p0 type=internal])
AT_CHECK([ovs-vsctl add-port br0 p1  -- set interface p1 type=internal])

dnl ovs-vswitchd should still 'see' ovsdb change with the 'monitor' method
AT_CHECK([ovs-appctl dpif/show | tail -n +3], [0], [dnl
    br0 65534/100: (dummy-internal)
    p0 1/1: (dummy-internal)
    p1 2/2: (dummy-internal)
])
OVS_VSWITCHD_STOP
AT_CLEANUP

```

### Test 7: Tạo sockets với unsafe names
```sh
AT_SETUP([ovs-vswitchd - do not create sockets with unsafe names])
OVS_VSWITCHD_START

# On Unix systems, test for sockets with "test -S".
#
# On Windows systems, we simulate a socket with a regular file that contains
# a TCP port number, so use "test -f" there instead.
if test $IS_WIN32 = yes; then
   S=f
else
   S=S
fi

# Create a bridge with an ordinary name and make sure that the management
# socket gets creatd.
AT_CHECK([ovs-vsctl add-br a -- set bridge a datapath-type=dummy])
AT_CHECK([test -$S a.mgmt])

# Create a bridge with an unsafe name and make sure that the management
# socket does not get created.
mkdir b

cat >experr <<EOF
ovs-vsctl: Error detected while setting up 'b/c'.  See ovs-vswitchd log for details.
ovs-vsctl: The default log directory is "$OVS_RUNDIR".
EOF
AT_CHECK([ovs-vsctl add-br b/c -- set bridge b/c datapath-type=dummy], [0], [], [experr])
AT_CHECK([test ! -e b/c.mgmt])

OVS_VSWITCHD_STOP(['/ignoring bridge with invalid name/d'])
AT_CLEANUP
```

### Test 8: set datapath IDs
```sh
AT_SETUP([ovs-vswitchd - set datapath IDs])
OVS_VSWITCHD_START([remove bridge br0 other-config datapath-id])

# Get the default dpid and verify that it is of the expected form.
AT_CHECK([ovs-vsctl --timeout=10 wait-until bridge br0 datapath-id!='[[]]'])
AT_CHECK([ovs-vsctl get bridge br0 datapath-id], [0], [stdout])
orig_dpid=$(tr -d \" < stdout)
AT_CHECK([sed 's/[[0-9a-f]]/x/g' stdout], [0], ["xxxxxxxxxxxxxxxx"
])
AT_CHECK_UNQUOTED([ovs-ofctl show br0 | strip_xids | head -1], [0], [dnl
OFPT_FEATURES_REPLY: dpid:$orig_dpid
])

# Set a dpid with 16 hex digits.
AT_CHECK([ovs-vsctl set bridge br0 other-config:datapath-id=0123456789abcdef])
AT_CHECK([ovs-vsctl --timeout=10 wait-until bridge br0 datapath-id=0123456789abcdef])
AT_CHECK([ovs-ofctl show br0 | strip_xids | head -1], [0], [dnl
OFPT_FEATURES_REPLY: dpid:0123456789abcdef
])

# Set a dpif with 0x prefix.
AT_CHECK([ovs-vsctl set bridge br0 other-config:datapath-id=0x5ad515c0])
AT_CHECK([ovs-vsctl --timeout=10 wait-until bridge br0 datapath-id=000000005ad515c0])
AT_CHECK([ovs-ofctl show br0 | strip_xids | head -1], [0], [dnl
OFPT_FEATURES_REPLY: dpid:000000005ad515c0
])

# Set invalid all-zeros dpid and make sure that the default reappears.
AT_CHECK([ovs-vsctl set bridge br0 other-config:datapath-id=0x00])
AT_CHECK([ovs-vsctl --timeout=10 wait-until bridge br0 datapath-id=$orig_dpid])
AT_CHECK_UNQUOTED([ovs-ofctl show br0 | strip_xids | head -1], [0], [dnl
OFPT_FEATURES_REPLY: dpid:$orig_dpid
])

OVS_VSWITCHD_STOP
AT_CLEANUP
```
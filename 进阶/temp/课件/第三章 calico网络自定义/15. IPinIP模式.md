您可以配置每个IP池不同封装配置。然而,你不能一个IP池内混合封装类型。

- Configure IP in IP encapsulation for only cross subnet traffic
- Configure IP in IP encapsulation for all inter workload traffic
- Configure VXLAN encapsulation for only cross subnet traffic
- Configure VXLAN encapsulation for all inter workload traffic

1. IPv4/6 地址支持

IP in IP和 VXLAN只支持IPv4地址。

2. 最佳实践

Calico 只有一个选项来选择性地封装流量 ,跨越子网边界。我们建议使用`IP in IP `的`cross subnet`选项把开销降到最低。

注意：切换封装模式会导到正在连接的进程中断。

3. 针对仅跨子网的流量配置IP in IP

IP in IP封装可以选择性的执行， 并且仅用于通过子网边界的通信量  。

开启这个功能，设置`ipipMode`为`CrossSubnet`

```shell
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ippool-ipip-cross-subnet-1
spec:
  cidr: 192.168.0.0/16
  ipipMode: CrossSubnet
  natOutgoing: true
```

4. 针对workload间的流量配置IP in IP的封装

`ipipMode`设置`Always`

```shell
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ippool-ipip-1
spec:
  cidr: 192.168.0.0/16
  ipipMode: Always
  natOutgoing: true
```


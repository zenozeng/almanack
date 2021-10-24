# MinIO：基于 deterministic hashing 的分布式对象存储

MinIO 是一套开源的分布式对象存储，基于 deterministic hashing，架构因此非常简洁，无需 rebalancing 和额外的 metadata 存储，代价是扩容时需要以添加新的 Pool 的方式进行。综合来看对于对象存储来说，是一个非常划算的 trade off。

同时 Erasure Coding 的设计也比 Replication 更经济，比如 Replication 3 副本 1PB 物理存储只能产生 0.33 PB 可用存储，使用 EC:4，1PB 物理存储可以产生 0.75PB 可用存储，同时每个对象可以容忍四块磁盘 offline。

本文会概要描述一下 MinIO 的核心概念、对象写入过程。

## Core Concepts

### Disk

一个卷，可以是一个本地目录（通常每个 volume mount 各自的磁盘），在 Kubernetes 下通常是一个 Local PV。

### Erasure Coding

MinIO 的核心。可以仔细阅读：[https://docs.min.io/minio/baremetal/concepts/erasure-coding.html](https://docs.min.io/minio/baremetal/concepts/erasure-coding.html)

Erasure Coding 将每个文件对象拆分为 K 个 data blocks 和 N 个 parity blocks，parity blocks 被用于 data blocks 丢失或损毁的时候重新构建数据。

现在假设每个文件对象被拆分为 K + N = 16 个 blocks：

如果 Erasure Coding 设置为 EC:4（MinIO 默认值，每个 Object 有 4 个 parity blocks），则：

- 文件对象会被拆分为 K=12 个 data blocks 和 N=4 个 parity blocks
- Storage Ratio 为 0.75（1PB 物理容量产生 0.75 PB 可用容量）
- 允许在 4 个 blocks offline 的时候进行读操作
- 允许在 4 个 blocks offline 的时候进行写操作

如果 Erasure Coding 设置为 EC:8（每个 Object 有 8 个 parity blocks） ，则：

- 文件对象会被拆分为 K=8 个 data blocks 和 N=8 个 parity blocks
- Storage Ratio 为 0.5
- 允许在 8 个 blocks offline 的时候进行读操作
- 允许在 7 个 blocks offline 的时候进行写操作（8个的话刚好是16的一半，因此写操作需要 9 个 blocks 来避免脑裂问题）

### Erasure Sets 以及 Server Pool

一个 Erasure Set 由一组 disks 组成，对于每个文件对象，MinIO 会将 K + N 个 blocks 随机且均匀得在 disks 中分布，且每个 disk 最多仅有 1 个 data block 或 parity block。

在指定了 Erasure Set 的 Size 和 Erasure Coding 设置后，K 由 Erasure Set Size - N 决定。

一个 Server Pool 包含多个 MinIO Server，且每个 server 管理几个 disks。

Erasure Set Size 则由当前 Server Pool 的总 disks 数量、服务数量确定，通常为 16。

### Cluster

由多个 Server Pool 组成，Bucket 是在 Cluster 这个层次的概念

## 对象写入过程

### 服务启动及 Disk 确定过程

1. 服务从启动参数获取和确定每个 endpointPool 的 endpoints 以及 Erasure Set 的数量、每个 Erasure Set 的 Disk 数量
   1. 启动参数形如：minio server http://host{1...n}/export{1...m}（n 个 server，m 个 disk）
2. 初始化 Server Pools，每个 Server Pool 是一个 *erasureSets 对象，通过 newErasureSets 初始化
   1. 调用 newErasureSets，传入 endpoints（数组长度为 pool 下 disks 总数） 与 storageDisks
3. erasureSets 初始化，erasureSets 包含一个  Erasure Set 数组，每个 Erasure Set 是一个 *erasureObjects 对象
3. 结合以下代码：我们可以得知，getEndpoints() 结果是一个 m * n 的数组
   1. 形如：
      1. host1/export1
      1. host2/export1
      1. host3/export1
      1. host4/export1
      1. host1/export2
      1. host2/export2
      1. host3/export2
      1. host4/export2
      1. ...
   2. 因此每个 Erasure Set 里的 blocks 会均匀在各个 Server 之间均匀分布，比如对于 4 servers，8 drives per server，Erasure Set Size 为 16，16 个 blocks 以 4 * 4 形式均匀分布在 4 servers 上，一个 server 故障最多导致一个 Erasure Set 中 4 个 blocks 掉线。

```go
// https://github.com/minio/minio/blob/13e41f2c6899173a495e8a019e69bdd9797534ba/cmd/endpoint-ellipses_test.go#L409
		{
			"http://minio{2...3}/export/set{1...64}",
			endpointSet{
				[]ellipses.ArgPattern{
					[]ellipses.Pattern{
						{
							Prefix: "",
							Suffix: "",
							Seq:    getSequences(1, 64, 0),
						},
						{
							Prefix: "http://minio",
							Suffix: "/export/set",
							Seq:    getSequences(2, 3, 0),
						},
					},
				},
				nil,
				[][]uint64{{16, 16, 16, 16, 16, 16, 16, 16}},
			},
			true,
		},

// https://github.com/minio/minio/blob/13e41f2c6899173a495e8a019e69bdd9797534ba/cmd/endpoint-ellipses.go#L209
// Returns all the expanded endpoints, each argument is expanded separately.
func (s endpointSet) getEndpoints() (endpoints []string) {
	if len(s.endpoints) != 0 {
		return s.endpoints
	}
	for _, argPattern := range s.argPatterns {
		for _, lbls := range argPattern.Expand() {
			endpoints = append(endpoints, strings.Join(lbls, ""))
		}
	}
	s.endpoints = endpoints
	return endpoints
}


// https://github.com/minio/minio/blob/13e41f2c6899173a495e8a019e69bdd9797534ba/cmd/erasure-sets.go#L342
func newErasureSets(ctx context.Context, endpoints Endpoints, storageDisks []StorageAPI, format *formatErasureV3, defaultParityCount, poolIdx int) (*erasureSets, error)
	// ...
	for i := 0; i < setCount; i++ {
		var lockerEpSet = set.NewStringSet()
		for j := 0; j < setDriveCount; j++ {
			endpoint := endpoints[i*setDriveCount+j]
			// Only add lockers only one per endpoint and per erasure set.
			if locker, ok := erasureLockers[endpoint.Host]; ok && !lockerEpSet.Contains(endpoint.Host) {
				lockerEpSet.Add(endpoint.Host)
				s.erasureLockers[i] = append(s.erasureLockers[i], locker)
			}
			disk := storageDisks[i*setDriveCount+j]
			if disk == nil {
				continue
			}
    // ...
}
```
参见：

- [https://docs.min.io/docs/distributed-minio-quickstart-guide.html](https://docs.min.io/docs/distributed-minio-quickstart-guide.html)

### PutObject 过程

func (er erasureObjects) putObject：[https://github.com/minio/minio/blob/d693431183d2c6e85831eff6e77f45376cbe306c/cmd/erasure-object.go#L748](https://github.com/minio/minio/blob/d693431183d2c6e85831eff6e77f45376cbe306c/cmd/erasure-object.go#L748)


1. 确定对象的 Erasure Coding（EC:N，如 EC:2, EC:4, EC:8 等等），由集群配置确定，请求的 x-amz-storage-class Header 可以覆盖
1. 获取当前 erasureObjects（Erasure Set） 对应的 disks
1. 如果 disks 存在 Offline 情况，增大 EC:N
1. 当前的 parity blocks 为 N，按照当前 Erasure Set Size - N 得到 data blocks 数量为 K
1. 将对象路径作为 key，K + N 作为 cardinality，通过 hashOrder 函数（基于 CRC32）计算 blocks 的分布，对于一个 object，一个 disk 最多只有一个 block
1. 写入 K 个 data blocks，N 个 parity blocks

```go
// https://github.com/minio/minio/blob/d693431183d2c6e85831eff6e77f45376cbe306c/cmd/erasure-metadata-utils.go#L102
// hashOrder - hashes input key to return consistent
// hashed integer slice. Returned integer order is salted
// with an input key. This results in consistent order.
// NOTE: collisions are fine, we are not looking for uniqueness
// in the slices returned.
func hashOrder(key string, cardinality int) []int {
	if cardinality <= 0 {
		// Returns an empty int slice for cardinality < 0.
		return nil
	}

	nums := make([]int, cardinality)
	keyCrc := crc32.Checksum([]byte(key), crc32.IEEETable)

	start := int(keyCrc % uint32(cardinality))
	for i := 1; i <= cardinality; i++ {
		nums[i-1] = 1 + ((start + i) % cardinality)
	}
	return nums
}
```

## Erasure Code 计算

[https://min.io/product/erasure-code-calculator](https://min.io/product/erasure-code-calculator)

## References

- [https://docs.min.io/minio/k8s/core-concepts/core-concepts.html](https://docs.min.io/minio/k8s/core-concepts/core-concepts.html)
- [https://docs.min.io/minio/baremetal/concepts/erasure-coding.html](https://docs.min.io/minio/baremetal/concepts/erasure-coding.html)
- [https://blog.min.io/no-rebalancing-object-storage/](https://blog.min.io/no-rebalancing-object-storage/)
- [https://blog.min.io/databases-on-object-storage/](https://blog.min.io/databases-on-object-storage/)
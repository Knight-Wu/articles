# 参考
https://archive.fo/2015.07.08-082503/http://www.boundary.com/blog/2012/01/flake-a-decentralized-k-ordered-unique-id-generator-in-erlang/
# 需求
如果 uuid 的需求是不需要其他节点的协同, 而且按照时间顺序递增, 那么实际上这个 id 的生成是需要两个条件, 一个是单调递增的时钟, 另一个是位置信息(worker id)
# 如何解决时钟回拨的问题
如果是在进程运行的时候, 可以保留一个当前时间id , 后续生成的时间 id 不能小于它, 小于就证明时钟回拨, 继续用上次的时间戳, 只能应对临时的时钟回拨, 如果一直发生回拨, 要用完也需要很多年;</br> 如果是进程重启了, 还不知道有什么根本解决办法, 可以和其他节点通信去校准. 
# twitter snowflake id 
## 问题
### 需要不重复的 worker id
就意味着需要节点的协同和这个不重复的 worker id 的保证, 需要依赖其他组件, 不能单独离线生成
### id 长度不够
64 bit 的整型, 需要容纳 worker id , timestamp id , seq id 相对来说不够长. 
## snowflake id 中 epoch 的作用
epoch 为过去某天的unix 时间戳, 当前时间 id = 当前时间戳 - epoch 所代表的时间, 就是为了让当前时间 id 一开始的时候比较小, 如果不减去 epoch , 那么一开始就很大就这个时间 id 就会浪费很多uuid.

# flake id
## 参考
https://archive.fo/2015.07.08-082503/http://www.boundary.com/blog/2012/01/flake-a-decentralized-k-ordered-unique-id-generator-in-erlang/
![image](https://github.com/Knight-Wu/articles/assets/20329409/c8fcbbf5-8948-4a40-9295-76edc1f8a2e9)
## 好处
worker id 取自 mac addr, 不需要节点协同; timestamp 可以取毫秒也不需要任何的截断和减去 epoch
## mac address 为什么是不重复的
* 理论上
  
MAC地址是IEEE在管理，一个标准的MAC地址格式是12个16进制数字两两一组，分为6组。因为一个16进制数字是4位，所以MAC地址是组48位的二进制数字。理论上MAC地址可以有2^48次方个，也就是281474976710656个(实际上可用的没这么多，下面讲)，这个数字虽然没有IPv6的个数那么夸张，不过比起IPv4还是宽余太多，足够厂商霍霍好久了。
</br>6组16进制数字里，前3组是厂商识别码(OUI)，由IEEE分配(厂商出钱买)，后3组厂商自己定，所以如果一个厂商购买了一个OUI，那么理论上它可以制造出2^24个网络设备(16777216个，1600万真彩色，实际可制造设备数目要少于这个数字，比如一个路由器的WAN/LAN，无线各占不同的MAC)，有些出货上亿的手机厂商怎么办？简单，再买OUI就是。
* 本地的 mac 地址和世界上的 mac 地址是否冲突
![image](https://github.com/Knight-Wu/articles/assets/20329409/bf6795b2-2e5f-4383-9502-3b94fc1097ee)

MAC地址冲突实际上几乎不会冲，参考IPv4，实际上可用的地址早已经被分配完毕，但是因为有NAT技术存在，IPv4同样还在继续使用，我们的设备实际上大都在一个个大大小小的不同的局域网之中，比如家庭网络里，那么多设备在自己所处局域网里的ip都是192.168.x.x，也没见和其他家的同ip的设备冲突。</br>
同理，MAC主要是被使用在OSI网络第二层，第三层就是ip协议为主了，MAC在本局域网内和本地ip映射，只要在自己网段内没有相同的就行。一旦跨网段，那么即使有两个同样的mac地址也无所谓了。

* 山寨的设备怎么办

现实情况48位的地址数量足够大，几乎是2^48。（注：因为有些地址不能用，比如组播地址01-00-5e和广播地址ff-ff-ff-ff-ff-ff等，所以只能是几乎）。就算是水货网卡，冒用了正品的OUI，也很难造成冲突。买正品和水货用在同一个组织的同一个广播域里，太反常，太小概率，几乎是1/2^48，做水货也会有意避开使用竞品组织的OUI。

* 虚拟设备呢, 例如在 k8s 里面部署的设备

猜想应该有 k8s 保证, 应该也是类似局域网的感觉, 

# es docId
在每个 shard 内唯一, shard 里面有多个 seg 在被同时写, 
</br>
参考 es 源码, 不是非常理解. 
```
/*
 * Copyright Elasticsearch B.V. and/or licensed to Elasticsearch B.V. under one
 * or more contributor license agreements. Licensed under the Elastic License
 * 2.0 and the Server Side Public License, v 1; you may not use this file except
 * in compliance with, at your election, the Elastic License 2.0 or the Server
 * Side Public License, v 1.
 */

package org.elasticsearch.common;

import java.util.Base64;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

/**
 * These are essentially flake ids but we use 6 (not 8) bytes for timestamp, and use 3 (not 2) bytes for sequence number. We also reorder
 * bytes in a way that does not make ids sort in order anymore, but is more friendly to the way that the Lucene terms dictionary is
 * structured.
 * For more information about flake ids, check out
 * https://archive.fo/2015.07.08-082503/http://www.boundary.com/blog/2012/01/flake-a-decentralized-k-ordered-unique-id-generator-in-erlang/
 */

class TimeBasedUUIDGenerator implements UUIDGenerator {

    // We only use bottom 3 bytes for the sequence number. Paranoia: init with random int so that if JVM/OS/machine goes down, clock slips
    // backwards, and JVM comes back up, we are less likely to be on the same sequenceNumber at the same time:
    private final AtomicInteger sequenceNumber = new AtomicInteger(SecureRandomHolder.INSTANCE.nextInt());

    // Used to ensure clock moves forward:
    private final AtomicLong lastTimestamp = new AtomicLong(0);

    private static final byte[] SECURE_MUNGED_ADDRESS = MacAddressProvider.getSecureMungedAddress();

    static {
        assert SECURE_MUNGED_ADDRESS.length == 6;
    }

    // protected for testing
    protected long currentTimeMillis() {
        return System.currentTimeMillis();
    }

    // protected for testing
    protected byte[] macAddress() {
        return SECURE_MUNGED_ADDRESS;
    }

    @Override
    public String getBase64UUID() {
        final int sequenceId = sequenceNumber.incrementAndGet() & 0xffffff;
        long currentTimeMillis = currentTimeMillis();

        long timestamp = this.lastTimestamp.updateAndGet(lastTimestamp -> {
            // Don't let timestamp go backwards, at least "on our watch" (while this JVM is running). We are
            // still vulnerable if we are shut down, clock goes backwards, and we restart... for this we
            // randomize the sequenceNumber on init to decrease chance of collision:
            long nonBackwardsTimestamp = Math.max(lastTimestamp, currentTimeMillis);

            if (sequenceId == 0) {
                // Always force the clock to increment whenever sequence number is 0, in case we have a long
                // time-slip backwards:
                nonBackwardsTimestamp++;
            }

            return nonBackwardsTimestamp;
        });

        final byte[] uuidBytes = new byte[15];
        int i = 0;

        // We have auto-generated ids, which are usually used for append-only workloads.
        // So we try to optimize the order of bytes for indexing speed (by having quite
        // unique bytes close to the beginning of the ids so that sorting is fast) and
        // compression (by making sure we share common prefixes between enough ids),
        // but not necessarily for lookup speed (by having the leading bytes identify
        // segments whenever possible)

        // Blocks in the block tree have between 25 and 48 terms. So all prefixes that
        // are shared by ~30 terms should be well compressed. I first tried putting the
        // two lower bytes of the sequence id in the beginning of the id, but compression
        // is only triggered when you have at least 30*2^16 ~= 2M documents in a segment,
        // which is already quite large. So instead, we are putting the 1st and 3rd byte
        // of the sequence number so that compression starts to be triggered with smaller
        // segment sizes and still gives pretty good indexing speed. We use the sequenceId
        // rather than the timestamp because the distribution of the timestamp depends too
        // much on the indexing rate, so it is less reliable.

        uuidBytes[i++] = (byte) sequenceId;
        // changes every 65k docs, so potentially every second if you have a steady indexing rate
        uuidBytes[i++] = (byte) (sequenceId >>> 16);

        // Now we start focusing on compression and put bytes that should not change too often.
        uuidBytes[i++] = (byte) (timestamp >>> 16); // changes every ~65 secs
        uuidBytes[i++] = (byte) (timestamp >>> 24); // changes every ~4.5h
        uuidBytes[i++] = (byte) (timestamp >>> 32); // changes every ~50 days
        uuidBytes[i++] = (byte) (timestamp >>> 40); // changes every 35 years
        byte[] macAddress = macAddress();
        assert macAddress.length == 6;
        System.arraycopy(macAddress, 0, uuidBytes, i, macAddress.length);
        i += macAddress.length;

        // Finally we put the remaining bytes, which will likely not be compressed at all.
        uuidBytes[i++] = (byte) (timestamp >>> 8);
        uuidBytes[i++] = (byte) (sequenceId >>> 8);
        uuidBytes[i++] = (byte) timestamp;

        assert i == uuidBytes.length;
        // 15 bytes,from low to high, seqId 2 bytes, timestamp 4 bytes, changes every ~65 secs and so on , mac addr 6 byte, timestamp 1 bytes changes every 256 millsecs, seqId one byte, timestamp one byte, changes every millsecs
        return Base64.getUrlEncoder().withoutPadding().encodeToString(uuidBytes);
    }
}

```

# baidu snowflake uid
![image](https://github.com/Knight-Wu/articles/assets/20329409/9608a1ad-19c8-46fd-87ff-7a0c14b1a858)

https://github.com/baidu/uid-generator

## 问题
最好是把 worker id 和delta second 换位置, 因为当前是 worker id 每次重启加 1, 如果时钟发生回拨, delta seconds 变小了, 那么就有可能跟之前的重复
![image](https://github.com/Knight-Wu/articles/assets/20329409/b1854b62-0dfb-4870-8826-0d4971639e87)



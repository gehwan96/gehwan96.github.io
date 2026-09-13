---
title: OS별 InetAddress 클래스에서 가져오는 IP가 달라지는 원인 분석
pubDatetime: 2022-07-02T09:00:00+09:00
description: InetAddress.getLocalHost()가 OS와 네트워크 인터페이스 순서에 따라 루프백 IP를 반환하는 원인을 분석하고 NetworkInterface로 해결합니다.
tags:
  - incident-retro
  - java
---
## 개요

서비스 로직 개선간 특이한 문제를 발견했고 해당 문제를 분석하고 해결한 과정을 공유하겠습니다.

비록 해당 코드는 팀에서 Spring Security 적용을 nginx 필터링으로 변경하여 반영되지 않았으나, 재밌는 이슈라고 생각해 해당 글을 작성해봤습니다.

## 문제

기존 테스트 로직은 자신의 로컬 ip로 요청할 경우 허용하지 않는 로직이였으며, 실패되는 결과를 기댓값으로 하는 테스트였습니다.

이 때, 제 PC에서만 특정 로직에서 로컬 ip가 아닌 루프백 ip를 반환받아 허용되지 않아야 될 요청이 허용되는 문제였습니다.

내 PC

```java
// 결과값 127.0.0.1
final String ip = InetAddress.getLocalHost().getHostAddress();
```

팀원들 PC

```java
// 결과값 개인 PC IP  ex) 10.x.x.x
final String ip = InetAddress.getLocalHost().getHostAddress();
```

어이없었던 부분은 내가 실행을 하면 간헐적으로 성공하며 대부분 실패하는 문제가 발생했습니다. (5번 중 1번은 테스트 로직이 정상적으로 동작)

다른 팀원분들께 문의를 해도 정상적으로 동작하는 코드라고 답변을 받아 어떤 문제가 있어 해당 로직이 일부 인원에게는 에러가 발생하지 않는지를 분석하였고, 유추한 내용은 정상적으로 동작하는 인원과 에러가 발생하는 인원에겐 운영체제의 차이가 있음을 확인했습니다.

---

## 분석과정

### 이슈 확인

해당 이슈를 찾다보니 Stack Overflow에서도 여러명이 겪은 이슈였기에 좀 더 자세한 설명이 있는지 확인하였고 apache issue에서 다음 이슈에 대해 구체적으로 논의한 것을 알게 되었습니다.

[https://issues.apache.org/jira/browse/JCS-40](https://issues.apache.org/jira/browse/JCS-40)

> On Windows the address returned by InetAddress.getLocalHost() is fairly consistent; typically the server's LAN address. On Windows systems with multiple network cards (e.g. Internet, WAN, LAN, VPN, multi-homed), the address returned is ambiguous however.
> On Linux, the address returned by InetAddress.getLocalHost() seems to depend on the order in which the OS lists network interfaces, which really should be irrelevant.
> Furthermore the behaviour can vary between Linux distributions.
> Linux always exposes the loopback address (127.0.0.1) as a virtual network card, as if it was a physical NIC. On servers using DHCP, the method usually returns the loopback address.
> On servers configured with static IP addresses, depending on OS ordering, the method sometimes returns the LAN address but sometimes returns the loopback (127.0.0.1) address.
> InetAddress.getLocalHost() makes no attempt to prioritize LAN/non-loopback addresses in its selection.

정리해서 요약하자면, 현재 해당 메소드에는 다음과 같은 문제가 있었습니다.

- 다중 네트워크 카드를 이용할 경우, Windows 시스템에서 InetAddress.getLocalHost() 실행시 반환되는 주소가 일관성이 없다.
- 고정 IP 주소로 구성된 서버에서는 OS의 정책에 따라 LAN 주소를 반환하는 경우도 있지만 루프백(127.0.0.1) 주소를 반환하는 경우가 발생한다.

앞서 본 InetAddress.getLocalHost()는 리스트의 맨 처음 인덱스 IP 주소를 반환하기 때문에 다중 네트워크 카드를 이용할 경우 이 리스트의 순서가 변경되어 일관성 없는 IP를 반환할 수 있게 되는 문제를 알 수 있었습니다.

또한, OS의 정책, 캐시 사용 유무에 따라 반환되는 IP 주소값이 달라질 수 있음을 알 수 있습니다.

그럼 InetAddress.getLocalHost()는 어떻게 구현 되었길래 일관성 없는 주소를 가져오는지 궁금했고, 어떻게 해결해야 할지 방법을 찾기 위해 코드 분석을 진행하였습니다.

---

### 코드 분석

**InetAddress.getLocalHost() 코드**

```java
public static InetAddress getLocalHost() throws UnknownHostException {

        SecurityManager security = System.getSecurityManager();
        try {
            String local = impl.getLocalHostName();

            if (security != null) {
                security.checkConnect(local, -1);
            }

            if (local.equals("localhost")) {
                return impl.loopbackAddress();
            }

            InetAddress ret = null;
            synchronized (cacheLock) {
                long now = System.currentTimeMillis();
                if (cachedLocalHost != null) {
                    if ((now - cacheTime) < maxCacheTime) // Less than 5s old?
                        ret = cachedLocalHost;
                    else
                        cachedLocalHost = null;
                }

                // we are calling getAddressesFromNameService directly
                // to avoid getting localHost from cache
                if (ret == null) {
                    InetAddress[] localAddrs;
                    try {
                        localAddrs =
                            InetAddress.getAddressesFromNameService(local, null);
                    } catch (UnknownHostException uhe) {
                        // Rethrow with a more informative error message.
                        UnknownHostException uhe2 =
                            new UnknownHostException(local + ": " +
                                                     uhe.getMessage());
                        uhe2.initCause(uhe);
                        throw uhe2;
                    }
                    cachedLocalHost = localAddrs[0];
                    cacheTime = now;
                    ret = localAddrs[0];
                }
            }
            return ret;
        } catch (java.lang.SecurityException e) {
            return impl.loopbackAddress();
        }
    }
```

디버깅을 해보니 다음 로직에서 문제가 있다는 것을 발견했습니다.

```java
   if (ret == null) {
        InetAddress[] localAddrs;
        try {
            localAddrs =
                    InetAddress.getAddressesFromNameService(local, null);
        } catch (UnknownHostException uhe) {
            // Rethrow with a more informative error message.
            UnknownHostException uhe2 =
                    new UnknownHostException(local + ": " +
                            uhe.getMessage());
            uhe2.initCause(uhe);
            throw uhe2;
        }
        cachedLocalHost = localAddrs[0];
        cacheTime = now;
        ret = localAddrs[0];
    }
```

해당 내용을 볼 경우

```java
localAddrs = InetAddress.getAddressesFromNameService(local, null);
```

```java
 private static InetAddress[] getAddressesFromNameService(String host, InetAddress reqAddr)
        throws UnknownHostException
    {
        InetAddress[] addresses = null;
        boolean success = false;
        UnknownHostException ex = null;

        // Check whether the host is in the lookupTable.
        // 1) If the host isn't in the lookupTable when
        //    checkLookupTable() is called, checkLookupTable()
        //    would add the host in the lookupTable and
        //    return null. So we will do the lookup.
        // 2) If the host is in the lookupTable when
        //    checkLookupTable() is called, the current thread
        //    would be blocked until the host is removed
        //    from the lookupTable. Then this thread
        //    should try to look up the addressCache.
        //     i) if it found the addresses in the
        //        addressCache, checkLookupTable()  would
        //        return the addresses.
        //     ii) if it didn't find the addresses in the
        //         addressCache for any reason,
        //         it should add the host in the
        //         lookupTable and return null so the
        //         following code would do  a lookup itself.
        if ((addresses = checkLookupTable(host)) == null) {
            try {
                // This is the first thread which looks up the addresses
                // this host or the cache entry for this host has been
                // expired so this thread should do the lookup.
                for (NameService nameService : nameServices) {
                    try {
                        /*
                         * Do not put the call to lookup() inside the
                         * constructor.  if you do you will still be
                         * allocating space when the lookup fails.
                         */

                        addresses = nameService.lookupAllHostAddr(host);
                        success = true;
                        break;
                    } catch (UnknownHostException uhe) {
                        if (host.equalsIgnoreCase("localhost")) {
                            InetAddress[] local = new InetAddress[] { impl.loopbackAddress() };
                            addresses = local;
                            success = true;
                            break;
                        }
                        else {
                            addresses = unknown_array;
                            success = false;
                            ex = uhe;
                        }
                    }
                }

                // More to do?
                if (reqAddr != null && addresses.length > 1 && !addresses[0].equals(reqAddr)) {
                    // Find it?
                    int i = 1;
                    for (; i < addresses.length; i++) {
                        if (addresses[i].equals(reqAddr)) {
                            break;
                        }
                    }
                    // Rotate
                    if (i < addresses.length) {
                        InetAddress tmp, tmp2 = reqAddr;
                        for (int j = 0; j < i; j++) {
                            tmp = addresses[j];
                            addresses[j] = tmp2;
                            tmp2 = tmp;
                        }
                        addresses[i] = tmp2;
                    }
                }
                // Cache the address.
                cacheAddresses(host, addresses, success);

                if (!success && ex != null)
                    throw ex;

            } finally {
                // Delete host from the lookupTable and notify
                // all threads waiting on the lookupTable monitor.
                updateLookupTable(host);
            }
        }

        return addresses;
  }
```

로직을 통해 lookupTable을 가져오고 이후 lookupTable에서 첫번째 인덱스를 반환(localAddrs[0])하는 것을 알 수 있습니다. 리스트는 다음과 같은 방식으로 호출되고 있습니다.

**InetAddress.getAddressesFromNameService() 리스트 조회 과정**

1. 호스트가 lookupTable에 있는지 확인합니다.
2. lookupTable 확인 결과에 따라 분기합니다.
   - 2-1) checkLookupTable()이 호출될 때 호스트가 lookupTable에 없으면 checkLookupTable()이 lookupTable에 호스트를 추가하고 null을 반환합니다. (다음 로직 진행)
   - 2-2) checkLookupTable()이 호출될 때 호스트가 lookupTable에 있으면 addressCache에서 조회를 해 주소를 반환합니다.
   - 2-3) 어떤 이유로든 addressCache에서 주소를 찾지 못한 경우 lookupTable에 호스트를 null로 추가하고 null을 반환합니다. (다음 로직 진행)
3. NameService 조회
   - 3-1) checkLookupTable()을 통해 주소를 찾지 못한 경우 (**2-1, 2-3**), NameService를 통해 **호스트를 찾아 주소를 반환**합니다.
   - 3-2) NameService를 통해서 주소를 찾지 못했을 경우, **빈 리스트 또는 루프백 주소를 반환**합니다.

이로 인해 InetAddress.getAddressesFromNameService()를 호출해 가져온 localAddrs 리스트가 Window와 Mac의 IP 테이블 관리로 인해 일치하지 않을 경우 다른 IP가 나올 수도 있는 것을 알게 되었습니다.

### 해결

해결 방안으로는 다음과 같은 방안이 있습니다.

**해결방안**

1. **etc/hosts 의 ip 주소값을 설정한다.**
   해당 해결 방안은 설정이 필요하기 때문에 해당 코드를 이용하는 모든 컴퓨터에서 설정을 해줘야 된다는 점에서 문제가 있습니다. 상황에 따라서 다음 방안이 제일 간단한 방법이 아닐까 생각됩니다.
   (ex) 혼자만 쓰는 코드, 작업 환경이 변하지 않을 경우)
2. **NetworkInterface를 이용해 코드를 구현한다.**
   기존의 InetAddress.getLocalHost()에 문제가 존재하기 때문에 NetworkInterface 인터페이스를 이용한 ip를 호출하는 로직을 직접 구현해 사용합니다.

해당 서비스는 서비스가 동작하는 환경 및 개발자의 개발 환경에서만 정상적으로 동작할 경우 큰 이슈가 없다는 점을 고려하여 Mac, Window, Linux 세 개의 운영체제 환경만을 고려하였습니다. 추가로, 기존 서비스에 영향을 주지 않기위해 2번 해결방안을 이용하기로 했습니다.

기존 문제는 Linux나 Window가 아닌 Mac OS에서 발생한 문제였기 때문에 Mac의 네트워크 인터페이스를 이용한 해결 방안을 고민하였고, MAC OS와 다른 운영체제의 차이점을 네트워크 인터페이스 구성에서 확인할 수 있었습니다.

#### Mac 네트워크 구성

mac의 터미널에서 ifconfig 명령어를 사용할 경우, 다음과 같은 내용을 볼 수 있습니다.

![macOS ifconfig 출력 예시](/images/inetaddress-localhost-ip-by-os/img-01.png)

여기서 맨 처음 lo0, gif0가 무엇인지 궁금해 찾아본 결과 네트워크 인터페이스 이름을 각각 종류별로 나누어 관리하고 있는 것을 알 수 있었습니다. 대표적인 네트워크 인터페이스 이름으로는 다음과 같은 것들이 존재합니다.

- lo0 : 루프백
- en : 이더넷 (en0가 mac 로컬호스트 이더넷 인터페이스 명칭)
- utun : VPN and Back to My Mac에서 생성 관련 참고

Mac의 경우 en0에 이더넷 로컬호스트 ip가 존재하기 때문에 이를 이용해 로컬호스트 ip를 조회하고, 조회가 되지 않을 경우, InetAddress.getLocalHost()을 호출하는 방향으로 구현 방향을 잡았습니다.

```java
    private InetAddress getLocalHost() throws SocketException, UnknownHostException {
        try{
            NetworkInterface networkInterface = NetworkInterface.getByName("en0");
            for (Enumeration<InetAddress> addresses = networkInterface.getInetAddresses(); addresses.hasMoreElements(); ) {
                InetAddress address = addresses.nextElement();
                if (address instanceof Inet4Address && !address.isLoopbackAddress()) {
                    return address;
                }
            }
        } catch(NullPointerException e){
            return InetAddress.getLocalHost();
        }
        return InetAddress.getLocalHost();
    }
```

기본적으로 wifi 및 로컬호스트 ip를 제공하는 en0를 가져오면 실제 로컬 호스트를 가져올 수 있습니다. 하지만 반환되는 ip값이 루프백 ip이거나 IPv6일 수 있기 때문에 해당 경우의 수를 고려하여 다음 조건문을 추가했습니다.

다만 주의할 것은 NetworkInterface.getByName("en0")시 네트워크 인터페이스에 다음 명칭이 존재하지 않을경우 NPE를 발생시킨다는 점이었습니다. Window일 경우 해당 인터페이스 명이 없을수도 있어 NPE가 발생할 수 있기 때문에, NPE일 경우 기존에 사용했던 InetAddress.getLocalHost();을 반환하도록 구현했습니다.

## 결론

다중 네트워크 카드를 컴퓨터 및 노트북에서 사용할 경우, InetAddress.getLocalHost()는 일관성 있는 ip 주소를 반환하지 못했습니다. 또한, 고정 ip를 사용할 경우 OS의 정책에 따라 ip 주소가 다르게 반환될 수 있습니다.

추후에 InetAddress 관련 네트워크 클래스를 이용할 경우에는 해당 이슈를 참고하여 기존 메소드에 이슈가 있는지 먼저 확인하시고 사용하는 것을 추천드립니다.

감사합니다.

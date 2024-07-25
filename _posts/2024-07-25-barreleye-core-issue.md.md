---
title: '멀티 코어 환경에서 싱글 코어 환경으로 배포 시 발생한 문제들'
tags: 
  - 프로젝트
  - 배럴아이
---

## 들어가기 앞서

배럴아이를 멀티 코어 환경에서 개발한 후, 싱글 코어 사양의 클라우드 환경으로 배포할 때, 이전에 발생하지 않았거나 드물게 발생했던 여러 기능 장애들이 예기치 않게 많이 발생했다. 원인을 찾기 어려웠던 몇 가지 주요 사례와 그 해결 방법을 공유하려고 한다.

## 멀티 코어 환경에서 싱글 코어 환경으로 배포 시 발생한 문제들

### 1. 메시지 덮어쓰기 문제로 인한 블록 분기

여러 인접 노드를 통해 동시에 메시지를 수신할 때, Read Buffer에서 메시지가 덮어씌워져 한 쪽 노드의 메시지가 폐기되어 블록 분기가 발생하는 문제가 있었다.

**해결 방법:** 비트코인과 유사한 긴 체인을 따르는 방법을 도입하여 문제를 해결했다. 긴 체인을 따르는 방법이란 여러 체인 중 가장 긴 체인을 신뢰하고 따르는 방식으로, 블록체인 네트워크에서 주로 사용된다. 이를 통해 네트워크 내의 블록 분기를 최소화하고 일관성을 유지할 수 있다.

### 2. 동기화 중단 문제

신규 노드가 네트워크에 처음 접속하여 인접 노드를 통해 체인 데이터를 동기화하는 과정 중, 인접 노드가 블록 마이닝에 성공하거나 블록 검증 등의 이유로 다른 노드와 통신하게 되면, Read Buffer에 메시지가 덮어씌어져 동기화 과정이 멈추는 현상이 발생했다.

**해결 방법:** 동기화 중에는 양측 노드 모두 블록 마이닝과 제3 노드와의 통신을 일시적으로 중단한 후, 두 노드 간의 동기화 완료 후 네트워크에서 발생한 체인 데이터들을 주기적으로 동기화하도록 조치했다.

```go
func (n *Node) checkBlockSyncTimeout() {
    n.isCheckingTimeout = true
    
    for n.miningRestartTime > time.Now().UnixNano() {
        time.Sleep(10 * time.Second) 
    }
    
    n.miningStopped = false
    n.isCheckingTimeout = false
}

```

### 3. 접속 시도 중 Read/Write 문제

처음 네트워크에 접속하는 신규 노드가 인접 노드와 피어 등록을 해야 할 때, 싱글 코어 환경에서는 접속 시도 중에 Read()가 완료되기 전에 인접 노드가 먼저 Write()해버리면서 접속이 실패하는 문제가 있었다.

**해결 방법:** 인접 노드에서 Write()하기 직전에 Sleep()을 사용하여 쓰레드를 대기 상태로 만들어 CPU 점유권을 넘겨주도록 유도하여, 접속 노드가 먼저 Read()를 수행할 수 있도록 수정했다.

```go
free:
    for {
        select {
        case peer := <-n.peerCh:
            n.peerMap[peer.conn.RemoteAddr()] = peer

            go peer.readLoop(n.rpcCh)

            time.Sleep(1 * time.Second) // CPU 점유권을 넘겨줌

            if err := n.sendChainInfoRequestMessage(peer); err != nil {
                _ = n.Logger.Log("err", err)
                continue
            }

            // ...생략
        }
    }

```

## 런칭 후 사용자 대응

런칭 이후, 많은 사람들이 GitHub 및 이메일을 통해 상장 계획 및 [노드 구동 방법](https://github.com/barreleye-labs/barreleye/issues/94)에 대해 문의를 보냈다. 특히, Barreleye Scan의 Faucet 기능을 통해 Barrel 코인을 주기적으로 획득하는 봇들이 등장했다. 이들은 채굴량에 비해 너무 많은 양을 가져갔기 때문에 여러 가지 제한을 두어야 했다.

### 1. Faucet 시간 제한

IP당 한 시간에 한 번씩만 요청할 수 있도록 제한을 두었지만, IP를 변경하면서 이 제한을 우회하는 경우가 발생했다. 이러한 상황을 해결하기 위해 IP별로 Faucet 요청 시간을 기록하고, 제한 시간을 설정했다.

```go
type Server struct {
    ServerConfig
    txChan      chan *types.Transaction
    bc          *core.Blockchain
    privateKey  *types.PrivateKey
    faucetLimit map[string]int64 // ip => unix time.
}

func (s *Server) handleFaucetRequest(c *fiber.Ctx) error {
    ip := c.IP()
    if remainTime, ok := s.faucetLimit[ip]; ok {
        if remainTime > time.Now().Unix() {
            return c.JSON(http.StatusBadRequest, ResponseBadRequest("faucet time limit"))
        }
    }

    // Faucet logic here...

    // Update faucet limit
    s.faucetLimit[ip] = time.Now().Unix() + 3600
    return c.JSON(http.StatusOK, ResponseOK("Faucet request successful."))
}


```

<br/>
 
다음 이미지는 사용자가 IP당 한 시간에 한 번씩만 Faucet을 사용할 수 있도록 제한된 상황을 보여준다. 제한 시간이 지나지 않은 상태에서 Faucet을 사용하려고 하면 “Faucet can only be used once per hour”라는 메시지가 표시된다.
 
![3](https://github.com/user-attachments/assets/eaede910-d000-44a4-8d9f-b8bd3259e919)

 <br/>
 
### 2. Faucet 갯수 제한

계정당 일정 잔액 이하에서만 Faucet 요청을 가능하도록 프론트엔드와 백엔드 모두에서 로직을 추가했다. 



```go
if toInfo != nil && toInfo.Balance >= 10 {
    return c.JSON(http.StatusBadRequest, ResponseBadRequest("The account already has sufficient balance of 10 Barrel or more."))
}
```

 <br/>
 
사용자가 Faucet을 통해 Barrel 코인을 요청하고 있으며, 계정 잔액이 10 Barrel 이하인 경우 요청이 가능함을 보여준다.
 
<img alt="1" src="https://github.com/user-attachments/assets/f8793e2f-5ecf-41b7-8f0c-2f5b567300c3">

<br/>
 
계정 잔액이 10 Barrel 이상인 경우 요청이 거부되는 메시지를 보여주고 있다.

<img alt="2" src="https://github.com/user-attachments/assets/9e7a7329-f3d5-472c-bdac-a3554ef685f3">

 <br/>
 
하지만, 이들은 특정 계정으로 받으면서 다른 계정으로 지속적으로 Transfer하여 잔액 제한을 회피하거나 계정을 매번 생성하여 코인을 받아가는 다양한 방법을 시도했다. 
이러한 상황으로 인해 지급되는 코인의 양을 대폭 축소하는 조치를 취했다.

코드는 배럴아이 깃허브에서 볼 수 있다. [Github](https://github.com/barreleye-labs/barreleye)

## 끝으로

이번 경험을 통해 멀티 코어 환경과 싱글 코어 환경 간의 차이에 대해 많은 것을 배웠다. 특히, 분산 시스템에서의 동기화 문제와 리소스 관리의 중요성을 다시 한번 깨달았다.
 <br/> <br/>
또한, 사용자와의 지속적인 소통과 피드백 수집이 시스템 안정성과 성능 개선에 큰 도움이 된다는 것을 느꼈다. <br/> 
이메일과 깃허브를 통해 상장 계획이나 구동방법에 대해서 물어보시는 연락이 많이 왔는데 이것이 개발 후 하나의 즐거움이였다.구동 방법이 어렵다면  앞으로도 이러한 경험을 바탕으로 더욱 견고하고 효율적인 시스템을 개발하기 위해 노력할 것이다.
<br/>

배럴아이를 구동하면서 어려운 점이 있거나 에러사항이 있다면 GitHub 이슈나 메일을 통해 알려주세요. rusvo님이 올려주신 [Github Issue](https://github.com/barreleye-labs/barreleye/issues/94)

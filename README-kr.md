# 이더리움 스마트 컨트렉트 보안 모범 사례들

이 문서는 중급 솔리디티 프로그래머들을 위해 보안 고려사항들의 기초 지식을 제공합니다.
이것은 [ConsenSys Diligence](https://media.consensys.net/introducing-consensys-diligence-cf38f83948c)와 폭넓은 이더리움 커뮤니티들에 의해 유지됩니다.

[![Join the chat at https://gitter.im/ConsenSys/smart-contract-best-practices](https://badges.gitter.im/ConsenSys/smart-contract-best-practices.svg)](https://gitter.im/ConsenSys/smart-contract-best-practices?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

## 어디서부터 시작하죠?

* [일반적인 철학 (General Philosophy)](#General_Philosophy) 스마트 컨트렉트 보안 사고방식에 대해 설명합니다.
* [솔리디티 권장사항(Solidity Recommendations)](#recommendations) 바람직한 코드 패턴의 예시를 담았습니다.
* [알려진 공격(Known Attacks)](#known_attacks) 다른 클래스들 간의 취약성을 피하는 방법을 설명합니다.
* [소프트웨어 엔지니어링(Software Engineering)](#software_engineering) 몇개의 아키텍쳐의 개요와 위험완화를 위한 디자인 접근법의 개요입니다.
* [문서와 절차(Documentation and Procedures)](#documentation_procedures) 외부 개발자들과 감사원들(Auditors)을 위한 시스템 문서 작성법 연습 개요입니다.
* [보안 도구(Security Tools)](#security_tools) 취약성 탐지와 코드 수준 발전을 위한 툴 리스트 입니다.
* [보안 알림(Security Notifications)](#security_notifications) 최신 상태를 유지하기 위한 정보 소스 리스트 입니다.
* [토큰(Tokens)](#tokens) 토큰과 관련된 연습 개요 입니다.

## 기여자(Contributions)를 환영합니다!

작은 수정이나, 새로운 섹션 추가를 자유롭게 풀 리퀘스트(pull request)해주세요. 만약에 새로운 컨텐츠 내용을 작성하신다면, [contributing](./docs/about/contributing.md)페이지 안에 스타일 안내 방식으로 언급해주세요.  

필요한 보완이나 업데이트 주제들은 [issues](https://github.com/ConsenSys/smart-contract-best-practices/issues)에서 볼 수 있습니다. 만약 당신의 아이디어에 대해 토론하기 원한다면, [Gitter](https://gitter.im/ConsenSys/smart-contract-best-practices)에서 우리와 채팅 할 수 있습니다.

글이나 블로그 포스팅을 하신다면, [bibliography](#bibliography)에 추가 해 주세요.

<a name="General_Philosophy"></a>
# 일반적인 철학 (General Philosophy)
이더리움과 복잡한 블록체인 프로그램들은 새롭고 아주 실험적입니다. 그러므로, 당신은 새로운 버그들이나 보안 위험을 발견하는 것과 같은 보안 환경에 대해 끊임없는 변화를 예상하셔야 합니다. 이 문서의 보안 연습은 기초적인 단계의 보안이므로 스마트 컨트렉트 개발자로서는 더 공부하셔야 합니다.

스마트 컨트렉트 프로그래밍은 당신이 해오던 방식과 다른 엔지니어링 사고 방식이 필요합니다. 실패 비용(The cost of failure)이 높을 수도 있고, 변경하기 까다롭고, 하드웨어 프로그래밍 또는 금융 서비스 프로그래밍과 유사한 방법으로 웹이나 모바일 환경에서 만들어야 합니다. 그러니 알려진 취약성 방어기법으로는 충분하지 않아서 새로운 개발 철학에 대해 학습 필요성을 느끼실 것 입니다:

## 실패를 대비

컨트렉트 안에 에러 중 사소하지 않은 건 없습니다. 그러므로 당신의 코드는 버그 대응과 취약성을 유연하게 대처해야 합니다.

  - 컨트렉트에 무언가 잘못 되어가고 있으면 멈춤 ('써킷 브레이커')
  - 위험이 있는 자금의 액수를 관리 (비율 제한, 최대 사용량)
  - 버그들 수정과 개선을 위한 효과적인 업그레이드 방법

## 신중한 공개

전체 제품 공개 전에 항상 버그를 잡을 수 있도록 하세요.

  - 철저하게 테스트 컨트렉트를 작성하고, 새로운 공격 벡터를 발견하면 테스트 항목에 추가
  - [버그 바운티](#bounties)를 제공하고 알파 테스트넷 공개 때 부터 시작
  - 단계별 공개, 각 단계별 테스트와 사용량 증가

## 컨트렉트의 단순함 유지

복잡성은 에러 발생 가능성을 증가시킵니다.

  - 컨트렉트 구조를 반드시 간단하게 구성
  - 코드를 모듈화해 컨트렉트와 함수를 작게 유지
  - 이미 사용된 툴들 또는 이용 가능한 코드 사용(예. 랜덤 숫자 생성을 스스로 돌리지 마세요)
  - 가능한 퍼포먼스 명확성 선호  
  - 분산화가 필요한 시스템 부분은 블록체인만 사용

## 최신 버전으로 유지

자원 리스트의 다음 부분으로 새로운 보안 개발 관심 유지를 합니다.

  - 발견된 새로운 버그에 대한 컨트렉트 확인
  - 가능한 툴이나 라이브러리를 최신버전으로 업그레이드
  - 유용하다고 인정된 새로운 보안 기술 채택

## 블록체인의 특징을 이해

대부분의 프로그래밍은 이더리움 프로그래밍과 관련된 경험일텐데, 몇가지 위험을 아셔야 합니다.

  - 외부(external) 컨트렉트 호출(call)을 아주 주의, 악성코드가 실행되거나 통제 흐름이 변경될 수도 있음
  - 공공(public) 함수는 공공의 성질을 가지고 있으며, 악의적으로 호출 될 수 있음을 이해. 개인적인(private) 데이터도 아무나 볼 수 있음
  - 가스 비용과 블록 가스 제한을 명심

## 펀더멘탈 트레이드오프 (Fundamental Tradeoffs) : 단순함과 복잡함 경우

스마트 컨트렉트 시스템 보안과 구조를 평가할때 몇가지 펀더멘탈 트레이드오프가 고려됩니다. 스마트 컨트렉트 시스템의 상충 관계에 대한 적절한 균형을 확인하기 위해 일반적으로 추천 합니다.

소프트웨어 엔지니어링 측면에서 이상적인 스마트 컨트렉트 시스템은 모듈화이며, 복제 대신 코드를 재사용하며, 컴포넌트가 업그레이드 가능하게 하는 것입니다. 안전한 아키텍쳐(secure architecture)측면에서 이상적인 스마트 컨트렉트 시스템은 유난히 복잡한 스마트 컨트렉트 시스템의 경우를 포함해 이런 사고방식을 공유하는 것입니다.

하지만, 중요한 예외사항으로 소프트웨어 엔지니어링 모범 규칙과 보안을 동일시 하지 않는 것입니다. 모든 경우에, 아래 컨트렉트 시스템 관점 사이에서 최적의 방법을 섞어 적절한 균형을 얻는 것입니다:

- 업그레이드 불가능과 업그레이드 가능
- 단일 방식과 모듈 방식
- 복제와 재사용

### 업그레이드 불가능과 업그레이드 가능

여러 리소스들을 사용하는 동안에 Killable과 같이 유연성이 강조된 형태거나, 업그레이드가 가능하거나 수정가능한 패턴은 *펀더멘탈 트레이드오프* 의 유연성(malleability)과 보안성(security) 사이에 있습니다.

유연하게 정의된 패턴은 복잡성과 잠재적인 공격면을 추가하게 됩니다. 단순함은 스마트 컨트렉트 시스템의 짧은 시간동안 사전 정의된 아주 제한적인 기능의 복잡한 성능을 뛰어넘는데 효과적입니다. 예를 들면, 한정된 시간동안 관리자 없이 토큰 세일을 하는 컨트렉트 시스템과 같습니다.

### 단일 방식과 모듈 방식

단일 방식 컨트렉트는 지역적으로 정의된 식별가능와 조회가능이 모두 포함 되어 있습니다. 단일 방식으로 존재하는 몇개의 스마트 컨트렉트 시스템은 높은 관심을 가지게 되며, 극도로 지역적인 데이타와 흐름을 가지는 인수(argument)가 생깁니다 - 예를 들면, 코드 최적화(optimizing code) 효율성 검토 같은 경우 입니다.

여기서 고려되는 다른 상충관계(tradeoffs)와 마찬가지로, 보안 사례들(best practices)은 간단하면서 오래가지 못하는 소프트웨어 엔지니어링 사례와 멀어지는 경향이 있고, 더 복잡하면서 빈번하게 계속되는 컨트렉트 시스템 경우의 소프트웨어 엔지니어링 사례 쪽으로 기울게 됩니다.

### 복제와 재사용

소프트 엔지니어링 스마트 컨트렉트 시스템은 합당한 재사용을 최대화하는 것을 바라고 있습니다. 솔리디티 컨트렉트 코드 재사용 방법은 여러가지가 있습니다. 전에 디플로이(deployed) 된 *당신이 가지고 있는* 증명된 컨트렉트를 사용하는 것은 일반적으로 코드를 재사용하기 위한 안전한 방법 입니다.

복제는 당신이 가지고 있는 컨트렉트 중 전에 디플로이 되었던 컨트렉트를 사용할 수 없는 경우에 빈번하게 의존하게 됩니다. 복제 없이 재사용 가능한 안전한 코드 패턴을 찾아 제공하기 위해 [라이브 립스(Live Libs)](https://github.com/ConsenSys/live-libs)와 [제플린 솔리디티(Zeppelin Solidity)](https://github.com/OpenZeppelin/zeppelin-solidity)는 노력하고 있는 중 입니다. 모든 컨트렉트 보안 분석은 스마트 컨트렉트 시스템 목표의 자금 위험과 신뢰할만한 수준으로 사전에 인증(established)받지 못한 재사용 코드도 모두 포함 되어야 합니다.

<a name="recommendations"></a>
# 솔리디티 스마트 컨트렉트 보안 권장사항들
이 페이지는 스마트 컨트렉트를 작성할때 일반적으로 따르는 솔리디티 패턴 몇가지를 보여줍니다.

## 프로토콜 별 권장사항들

아래 권장사항들은 이더리움의 컨트렉트 시스템 개발에 적용됩니다.

## 외부 호출(External Calls)

### 외부 호출를 구성할때 주의하세요

믿을 수 없는 컨트렉트의 호출은 몇가지 기대하지 않은 위험이나 에러를 야기할 수 있습니다. 외부 호출은 다른 컨트렉트에 의존하는 컨트렉트 _또는_ 컨트렉트 내부의 악성 코드를 실행 시킬 수 있습니다. 보통 말하는 그런 모든 외부 호출은 잠재적인 보안 위험으로 처리해야 합니다. 그게 불가능해 지거나 외부 호출을 제거할 수 없다면, 이 섹션의 뒷부분에 있는 권장사항을 이용해 위험을 최소화 하세요.

<a name="mark-untrusted-contracts"></a>
### 믿을 수 없는 컨트렉트 표시

외부에 있는 컨트렉트들, 변수들, 메소드들, 컨트렉트 인터페이스들과 상호작용을 할 때 명확하게 하는 방법으로 그들과 상호작용은 잠재적으로 불안전 합니다. 이것은 당신이 소유한 함수의 외부 호출 컨드렉트를 적용한 것 입니다.

```sol
// bad
Bank.withdraw(100); // 신뢰할 수 있는지 없는지 불명확함

function makeWithdrawal(uint amount) { // 이 함수가 잠재적으로 불안한지에 대해 불명확함
    Bank.withdraw(amount);
}

// good
UntrustedBank.withdraw(100); // 신뢰할수 없는 외부 호출
TrustedBank.withdraw(100); // 외부에 있지만 XYZ 회사에서 유지하고 있는 믿을 수 있는 은행 컨트렉트

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```

### 외부 호출 이후에 상태 변화를 피함

*로우 콜(raw calls)* [`someAddress.call()`과 같은 유형] 또는 *컨트렉트 콜(contract calls)* [`ExternalContract.someMethod()`과 같은 유형] 은 악성 코드가 실행 될 수 있습니다. `ExternalContract`가 악성 코드가 아니여도, *그것* 이 어떤 컨트렉트에 의해 호출되면 악성코드가 실행 될 수 있습니다.

한가지 더 큰 위험은 악성코드가 통제 흐름을 뺏을 수 있으며, 경합조건(race conditions)으로 이어지게 됩니다 ([경합 조건(Race Conditions)](#race-conditions)에서 이 문제에 대해 논의하고 있습니다).

만약 당신이 신뢰할 수 없는 외부 컨트렉트를 호출한다면, *호출 이후에 상태를 바꾸는 것을 피하세요*. 이 패턴은 때때로 [확인-효과-작동 패턴 (checks-effects-interactions pattern)](http://solidity.readthedocs.io/en/develop/security-considerations.html?highlight=check%20effects#use-the-checks-effects-interactions-pattern) 이라고도 합니다.

### `send()`와 `transfer()`, `call.value()()`의 사이 상충관계 인지 필요

이더를 보낼때 `someAddress.send()`와 `someAddress.transfer()`, `someAddress.call.value()()` 사이에서 서로 상충되는 관계를 아셔야 합니다.

- `someAddress.send()`와 `someAddress.transfer()`는 [재진입(reentrancy)](#reentrancy)에 대응하는 *안전함* 이 고려되어야 합니다. 이러한 메소드들이 코드를 실행시키는 순간, 호출 당한 컨트렉트는 사용료로 현재 이벤트 로그 용도으로 충분한 2,300 가스만 받습니다.
- `x.transfer(y)`와 `require(x.send(y));`는 같지만, 후자는 전송이 실패하면 자동적으로 회귀(revert)됩니다.
- `someAddress.call.value(y)()` 는 이더와 실행 코드를 전송하게 됩니다. 실행 코드는 재진입에 반하는 *불안전* 값을 전송하는 타입으로 실행을 위해 가능한 모든 가스가 주어집니다.

`send()` 또는 `transfer()`을 사용하는 것은 재진입을 방지하지만 2,300 가스보다 더 필요로 하는 폴백 함수(fallback function)가 있는 컨트렉트는 비용적으로 부적합하게 된다. 사용할 가스의 량을 설정하는 `someAddress.call.value(ethAmount).gas(gasAmount)()` 를 사용하는 것도 가능합니다.

한가지 패턴으로 *푸시(push)* 컴포넌트를 위해 `send()` 또는 `transfer()`를 사용하거나 *풀(pull)* 컴포넌트를 위해 `call.value()()`를 사용하는 것과 같은, 즉 [*푸시(push)* and *풀(pull)*](#favor-pull-over-push-for-external-calls) 메카니즘을 사용해 이런 상충관계의 균형잡기를 시도해야 합니다.

`send()` 또는 `transfer()`의 배타적인 사용은 짚고 넘어갈 가치가 있는데 값 전송을 위해 재진입에 대응하는 안전한 컨트렉트를 스스로 만들 것이 아니라 그것들의 명확한 값을 만들어 전송해야 합니다.

### 외부 호출의 에러 다루기

솔리디티는 가공되지 않은 주소에서 작동하는 로우-레벨 호출 메소드를 제공합니다: `address.call()`과 `address.callcode()`, `address.delegatecall()`, `address.send()`. 이러한 로우-레벨 메소드들은 예외처리(exception)로 보내지 않고, 호출이 예외처리에 걸리면 `false` 를 반환하게 됩니다.
다른 한편으로, *컨트렉트 콜(contract calls)* (가령, `ExternalContract.doSomething()`)은 자동적으로 예외처리(throw)를 한다(예를 들어, `ExternalContract.doSomething()`는 `doSomething()`이 예외처리 하면 `throw` 하게 된다).

만약에 로우-레벨 호출 메소드를 사용한다면, 반환 값 확인을 하면서 호출이 실패할 가능성을 명심해 두세요.


```sol
// bad
someAddress.send(55);
someAddress.call.value(55)(); // 이것은 두배로 위험하고, 남은 가스 전부를 사용하면서 결과를 확인할 수 없습니다.
someAddress.call.value(100)(bytes4(sha3("deposit()"))); // 보증금(deposit)이 예외처리를 던지면, 로우 콜(raw call)은 false만 반환하고 트렌젝션은 복구되지 않습니다.

// good
if(!someAddress.send(55)) {
    // 실패 코드가 몇개 있음
}

ExternalContract(someAddress).deposit.value(100);
```
<a name="favor-pull-over-push-for-external-calls"></a>
### 외부 호출시 *pull* 보다 *push* 를 선호

외부 호출은 우연하게 또는 고의적으로 실패 할 수 있습니다. 이런 실패의 피해를 최소화하기 위해, 호출의 수신자(recipient)에 의해 시작할 수 있는 트렌젝션을 담고 있는 각각의 외부 호출은 분리 시키는 것이 종종 더 낫습니다. 이것은 특히 결제쪽과 관련있는데, 사용자가 스스로 자금을 인출하는 것 보다는 자동으로 그들에게 자금이 보내지게 하는 것이 더 낫습니다. (이것은 [가스 제한 관련문제(problems with the gas limit)](#dos-with-block-gas-limit) 가능성도 줄입니다.) 단일 트렌젝션에서 여러 개의 `send()` 호출을 묶지 마세요.

```sol
// bad
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() payable {
        require(msg.value >= highestBid);

        if (highestBidder != 0) {
            highestBidder.transfer(highestBid); // 만약 이 호출이 지속적으로 실패하면, 아무도 입찰 할 수 없음
        }

       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}

// good
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() payable external {
        require(msg.value >= highestBid);

        if (highestBidder != 0) {
            refunds[highestBidder] += highestBid; // 이 사용자가 요구할 수 있는 환불내용을 기록
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        msg.sender.transfer(refund);
    }
}
```

## 만들어진 컨트렉트의 잔액이 없을 것이라고 추정하지 마세요

공격자(attacker)는 생성 전의 컨트렉트 주소에 웨이(wei)를 보낼 수 있습니다. 컨트렉트의 최초 상태 잔액이 0 이라고 가정하시면 안됩니다. [issue 61](https://github.com/ConsenSys/smart-contract-best-practices/issues/61) 을 보시면 더 자세한 내용이 있습니다.

## 온체인(on-chain)데이터는 공공(public)이라는 것을 기억하세요

많은 어플리케이션들은 작동을 위해서 특정 시점에 사적인 데이터 제출을 요구합니다. 게임들 (예. on-chain rock-paper-scissors) 과 경매 메카니즘들 (예. sealed-bid second-price auctions) 은 예시들 중에서 두가지 큰 카테고리 입니다. 만약 당신이 프라이버시와 관련있는 어플리케이션을 제작하는 경우, 사용자 정보를 너무 빠르게 게시(publish) 하지 않도록 주의하세요.

예시:

* 가위-바위-보 게임에서, 양쪽의 플레이어는 그들이 의도한 첫번째 움직임 해시를 제출 해야하고, 두 플레이어 모두 이동을 요구해야 합니다; 만약 제출된 움직임과 일치하지 않는다면 해시는 버리게 됩니다.
* 경매에서, 플레이어들은 초기단계 (입찰가 보다 큰 예치금과 함께) 에서 그들이 제시한 값의 해시를 제출해야 하며, 두번째 단계에서 그들이 제시한 입찰가를 제출하게 된다.
* 난수 생성기 (random number generator) 를 필요로 하는 어플리케이션 개발을 할때, 순서는 항상 (1) 플레이어가 움직임을 제출하고, (2) 난수가 생성되고, (3) 플레이어가 지불(paid out)하는 방식이어야 한다. 난수 생성 방법은 그 자체가 활발한 연구 분야 입니다; 현재 최고의 방법 (best-in-class solutions) 으로 http://btcrelay.org 를 통해 인증된 비트코인 블록 헤더를 포함하는 법, 해시-커밋-리빌 스킴 (hash-commit-reveal schemes, 예. 한 파티가 숫자를 생성하고, 해시를 게시하여 값을 "커밋"한 다음, 나중에 값을 나타냅니다.) 그리고  [RANDAO](http://github.com/randao/randao) 입니다.
* 만약 당신이 빈번한 일괄 경매 (frequent batch auction) 절차를 구현하는 경우, 해시-커밋 스킴 (hash-commit scheme) 이 바람직 합니다.

## 2개 집단 또는 N개 집단의 컨트랙트들에서 몇몇 참여자들이 "행동을 거부(drop-offline)"하고 다시는 돌아오지 않을 가능성이 있다는 것을 알고 있어야 합니다.

자금을 빼낼 다른 방법이 없는 특정 행동을 수행하는 특정 집단에 의존하는 환불이나 요청 절차들을 만들지 마세요. 예를 들어, 가위바위보 게임의 흔한 실수는 두 플레이어가 그들의 행동을 제출하지 전까지 지불을 하지 않는 것입니다; 그러나, 악의적인 플레이어는 단순히 행동을 제출하지 않음으로써 상대방이 "상심하게" 만들 수 있습니다 - 사실, 플레이어가 상대방의 노출된 행동을 보고 그들이 지도록 결정한다면, 상대방은 그들의 행동을 제출할 이유가 전혀 없습니다. 이 문제는 상태 채널 합의 컨텍스트(context)에서 다시 발생하게 됩니다. 이런 상황들이 문제인 경우, (1) 초대받지 않은 참여자들을 피할 수 있는 방법(아마, 시간 제한을 통해)을 제공하고, (2) 모든 경우에 있어서 의도한대로 정보를 제출하는 참여자들을 위해 경제적 장려금을 추가하는 것을 고려해야 합니다.

## 솔리디티 관련 권장 사항

다음 권장 사항은 솔리디티에 특정한 내용이지만, 다른 언어들로 스마트 컨트랙트를 개발하는 것에도 도움이 될 수 있습니다.

## `assert()`를 사용하여 불변성을 강제로 적용시키세요.

표명 보호(assert guard)는 표명이 실패했을 경우 작동합니다 - 불변해야하는 속성이 변경되었을 경우. 예를 들어, 토큰 발행 컨트랙트에서 토큰 대 이더 발행 비율은 고정되어 있어야 합니다. 당신은 `assert()`를 사용해 이 비율이 고정되어 있다는 사실이 모든 경우에 같다는 것을 검증할 수 있습니다. 표명 보호는 컨트랙트를 중단시키거나 업그레이드를 허용하는 것과 같은 경우에 다른 기법들과 함께 자주 사용되어야만 합니다.(그렇지 않는다면, 표명이 항상 실패하는 경우 당신은 그 상황에 갇히게 될 것입니다.)

예시:

```sol
contract Token {
    mapping(address => uint) public balanceOf;
    uint public totalSupply;

    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        totalSupply += msg.value;
        assert(this.balance >= totalSupply);
    }
}
```

컨트랙트가 `deposit()` 함수를 통하지 않아도 [이더가 강제로 전송](#remember-that-ether-can-be-forcibly-sent-to-an-account)될 수 있기 때문에 표명은 잔액에 대한 등식에 엄격하지 *않습니다*!

## `assert()`와 `require()`를 적절히 사용하세요.

솔리디티 0.4.10에서 `assert()`와 `require()`가 소개되었습니다. `require(조건)`는 입력에 대한 유효성 검사에 사용됩니다. 모든 사용자의 입력에 대해 행해지며, 조건이 거짓(false)일 경우 회귀(revert)하게 됩니다. `assert(조건)` 또한 조건이 거짓일 경우 회귀하게 되지만 불변성에 대해서만 사용하게 됩니다: 내부 오류 또는 당신의 컨트랙트가 유효하지 않은 상태에 이르렀는지를 확인합니다. 이 패러다임을 따르게 되면 유효하지 않은 명령어에 도달하는 것을 정식 분석 도구들로 검증할 수 있습니다.

## 정수 나눗셈의 라운딩(Rounding)을 조심하세요.

모든 정수의 나눗셈은 가장 근접한 정수로 내림합니다. 만약 당신이 더 정교한 값을 원한다면, 승수(multiplier)을 사용하거나 분자(numerator)와 분모(denominator)를 모두 저장하는 것을 고려하세요.

(미래에는, 솔리디티는 고정소숫점 방식을 가질 것이며, 라운딩을 더욱 쉽게 만들 것입니다.)

```sol
// bad
uint x = 5 / 2; // 결과는 2 입니다. 모든 정수의 나눗셈은 가장 근접한 정수로 내림합니다.
```

곱셈을 사용하는 것은 내림을 방지할 수 있습니다. 이 승수는 미래에 x를 사용시 계산을 위해 필요합니다:

```sol
// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

분자와 분모를 저장하는 것은 당신이 `분자/분모`의 결과를 체인 외부에서 계산할 수 있다는 것을 의미합니다:
```sol
// good
uint numerator = 5;
uint denominator = 2;
```

<a name="remember-that-ether-can-be-forcibly-sent-to-an-account"></a>
## 이더가 강제로 전송될 수 있다는 것을 기억하세요.

컨트렉트 계좌를 철저하게 확인하는 변하지 않는(invariant) 코딩을 주의하세요.

공격자는 강제로 웨이(wei)를 어떤 계좌에라도 보낼 수 있으며 강제로 이더가 전송되는 것을 막을 수 없습니다 (`revert()`를 실행하는 폴백 함수라도 이것을 방지할 수 없습니다).

공격자는 컨트랙트를 생성하고, 그것에 1 wei를 보내고 `selfdestruct(victimAddress)`를 호출함으로써 강제로 이더를 전송할 수 있습니다. 아무런 코드도 `victimAddress`에서 호출되지 않으므로, 방지할 수 없습니다.

## 추상 컨트랙트와 인터페이스 사이의 트레이드오프(tradeoff)를 알아야 합니다.

인터페이스와 추상 컨트랙트는 둘 다 스마트 컨트랙트를 위한 사용자 정의 및 재사용이 가능한 접근을 제공합니다. 솔리디티 0.4.11에 도입된 인터페이스는 추상 컨트랙트와 유사하지만 구현된 함수를 가질 수 없습니다. 또한 인터페이스는 일반적으로 추상 컨트랙트를 더 실용적으로 만들어주는 저장소에 대한 접속 또는 다른 인터페이스로부터의 상속과 같은 기능들이 불가능하다는 한계점을 가지고 있습니다. 하지만 인터페이스는 구현에 앞서 컨트랙트를 설계하는데 확실히 유용합니다. 추가로 만약 컨트랙트가 추상 컨트랙트로부터 상속받았다면 구현되지 않은 함수들을 오버라이드(override)을 통해 반드시 구현해야 하며 그렇지 않는다면 이 컨트랙트 또한 추상 컨트랙트가 된다는 사실을 명심해야 합니다.

## 폴백 함수의 간결하게 유지하세요.

[폴백 함수](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function)는 컨트랙트가 인자 없이 메시지를 보냈을 경우 호출됩니다 (또는 일치하는 함수가 없을 경우). 그리고 `.send()` 또는 `.transfer()`로부터 호출되었을 때 2,300 가스만을 사용해 접근할 수 있습니다. 만약 당신이 `.send()` 또는 `.transfer()`로부터 이더를 받는 것이 가능하게 되는 것을 원한다면, 당신이 폴백 함수 내에서 할 수 있는 가장 많은 것은 이벤트를 로그로 남기는 것입니다. 더 많은 연산이나 가스가 필요하다면 적당한 함수를 사용하세요.

```sol
// bad
function() payable { balances[msg.sender] += msg.value; }

// good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { LogDepositReceived(msg.sender); }
```

## 폴백 함수의 데이터 길이를 확인하세요.

[폴백 함수](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function)는 평범한 이더 전송을 위해서 뿐만 아니라 다른 함수가 일치하지 않을 때에도 호출되기 때문에, 폴백 함수가 이더를 받는 것을 기록하기 위한 목적만 사용되었을 경우 데이터가 비어있는지 확인해야만 합니다. 그렇지 않으면, 호출자는 당신의 컨트랙트가 부정확하게 사용되었고 존재하지 않는 함수들이 호출되었는지를 알 수 없습니다.

```sol
// bad
function() payable { LogDepositReceived(msg.sender); }

// good
function() payable { require(msg.data.length == 0); LogDepositReceived(msg.sender); }
```

## 함수와 상태 변수들의 가시성을 분명하게 표시하세요.

함수와 상태 변수들의 가시성을 분명하게 표시해야 합니다. 함수들은 `external`, `public`, `internal` 또는 `private`으로 명시될 수 있습니다. 이 가시성들 사이의 차이점을 이해해주세요. 예를 들어, `public`이 아닌 `external`로 충분할 수도 있습니다. 상태 변수에는 `external`의 사용이 불가능합니다. 가시성을 분명하게 표시하는 것은 누가 함수를 호출했는지 또는 변수에 접근할 수 있는지에 대한 틀린 가정들을 찾아내는 것을 쉽게 만듭니다.

```sol
// bad
uint x; // 상태 변수는 internal이 기본값입니다, 그러나 분명히 표시해야만 합니다.
function buy() { // 기본값은 public입니다.
    // public 코드
}

// good
uint private y;
function buy() external {
    // 오직 외부에서만 호출 가능함
}

function utility() public {
    // 외부뿐만 아니라 내부에서도 호출이 가능함: 이 코드를 수정하기 위해서는 두가지 경우에 대해 모두 고려해야 함
}

function internalAction() internal {
    // 내부 코드
}
```

## 프라그마(pragma)를 특정한 컴파일러 버전으로 고정하세요.

컨트랙트는 테스트에서 가장 많이 사용되었던 컴파일러 버전 그리고 플래그와 같은 버전으로 배포되어야만 합니다. 프라그마를 고정하는 것은 컨트랙트가 의도치 않게 배포되는 것을 막아줍니다. 예를 들어, 최신 컴파일러는 발견되지 않은 버그들로 인한 더 많은 위험을 가지고 있습니다. 또한 컨트랙트는 다른 사람에 의해 배포되어야 하며 프라그라는 원래 저자가 의도한 컴파일러 버전을 나타냅니다.

```sol
// bad
pragma solidity ^0.4.4;


// good
pragma solidity 0.4.4;
```

### 예외

라이브러리나 EthPM 패키지 내의 컨트랙트의 경우와 같이, 컨트랙트가 다른 개발자들의 사용을 위한 경우 프라그마 구문이 뜰 수 있습니다. 그렇지 않으면 개발자는 로컬에서 컴파일을 하기 위해서 프라그마를 수동으로 업데이트해야 할 것입니다.

## 함수와 이벤트를 구별하세요.

함수와 이벤트 사이에 발생할 수 있는 혼란을 막기 위해, 이벤트의 이름을 대문자로 시작하는 것과 접두사 (우리는 Log를 추천합니다)를 사용하는 것을 장려합니다. 함수의 이름은 생성자(constructor)를 제외하고는 항상 소문자로 시작해야 합니다.

```sol
// bad
event Transfer() {}
function transfer() {}

// good
event LogTransfer() {}
function transfer() external {}
```

## 새로운 솔리디티 구성을 선호합니다.

`selfdestruct` (`suicide` 대신) 그리고 `keccak256` (`sha3` 대신)과 같은 구성/별명 (alias)을 선호합니다. 또한 `require(msg.sender.send(1 ether))` 같은 양식들은 `msg.sender.transfer(1 eter)`와 같이 `transfer()`를 사용해서 단순하게 만들 수 있습니다.

## '내장 함수'의 섀도잉이 가능하다는 것을 알아야 합니다.

이제 솔리디티 내의 내장 글로벌 함수들의 [섀도잉](https://en.wikipedia.org/wiki/Variable_shadowing)이 가능합니다.섀도잉은 컨트랙트가 `msg` 그리고 `revert()`와 같은 내장 함수들의 기능을 오버라이드하는 것을 가능하게 합니다. 비록 섀도잉이 [의도된 기능](https://github.com/ethereum/solidity/issues/1249)이긴 하지만, 이는 컨트랙트의 사용자들이 컨트랙트의 진짜 동작을 오해하게 할 수 있습니다.

```sol
contract PretendingToRevert {
    function revert() internal constant {}
}

contract ExampleContract is PretendingToRevert {
    function somethingBad() public {
        revert();
    }
}
```

컨트랙트 사용자들 (그리고 감사원들)은 그들이 사용하고자 하는 어플리케이션의 모든 스마트 컨트랙트 소스 코드를 잘 알고 있어야 합니다.

## `tx.origin`를 사용하지 마세요.

`tx.origin`을 인가(authorization)에 절대로 사용하지 마세요. 다른 컨트랙트는 (사용자가 인스턴스를 위해 자금을 가질수 있는) 컨트랙트를 호출할 수 있는 메서드를 가지고 있을 수 있고 당신의 컨트랙트는 그 트랜잭션을 `tx.origin`의 당신의 주소로 인가할 것입니다.

```
pragma solidity 0.4.18;

contract MyContract {

    address owner;

    function MyContract() public {
        owner = msg.sender;
    }

    function sendTo(address receiver, uint amount) public {
        require(tx.origin == owner);
        receiver.transfer(amount);
    }

}

contract AttackingContract {

    MyContract myContract;
    address attacker;

    function AttackingContract(address myContractAddress) public {
        myContract = MyContract(myContractAddress);
        attacker = msg.sender;
    }

    function() public {
        myContract.sendTo(attacker, msg.sender.balance);
    }

}
```

당신은 인가를 위해 `msg.sender`를 사용해야 합니다 (만약 다른 컨트랙트가 당신의 컨트랙트를 호출한다면 `msg.sender`는 그 컨트랙트의 주소가 될 것이며 그 컨트랙트를 호출한 사용자의 주소가 되지 않습니다).

더 자세한 내용은 이곳에서 확인하세요: [Solidity docs](https://solidity.readthedocs.io/en/develop/security-considerations.html#tx-origin)

인가에 대한 문제 외에도, `tx.origin`이 미래에 이더리움 프로토콜에서 사라질 가능성이 있습니다. 그러므로 `tx.origin`을 사용하는 코드는 미래의 출시버전과는 호환되지 않을 것입니다 ([비탈릭: 'tx.origin이 계속해서 사용가능하거나 의미가 있을거라 가정하지 마십시오.'](https://ethereum.stackexchange.com/questions/196/how-do-i-make-my-dapp-serenity-proof/200#200)).

또한 `tx.origin`을 사용하게 되면 당신은 컨트랙트들 간의 상호운용성을 제한할 수 있습니다. 왜냐하면 tx.origin을 사용하는 컨트랙트는 컨트랙트가 `tx.origin`이 될 수 없기 때문에 다른 컨트랙트에 의해 사용될 수 없습니다.

<a name="timestamp-dependence"></a>
## 타임스탬프 의존성 (Timestamp Dependence)

컨트랙트의 중요한 함수를 실행하기 위해, 특히 송금을 포함하는 동작일 경우, 타임스탬프를 사용할 때 3가지 중요한 고려사항이 있습니다.

### Gameability

블록의 타임스탬프는 채굴자에 의해 조작될 수 있음을 인지해야 합니다. 이 [컨트랙트](https://etherscan.io/address/0xcac337492149bdb66b088bf5914bedfbf78ccc18#code)에 대해 생각해보세요:

```sol

uint256 constant private salt =  block.timestamp;

function random(uint Max) constant private returns (uint256 result){
    //무작위성을 위해 최고의 시드를 받음
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5) ;
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y;
    uint256 h = uint256(block.blockhash(seed));

    return uint256((h / x)) % Max + 1; //1과 최대값 사이의 무작위 숫자
}
```

컨트랙트가 무작위 숫자 시드를 위해 타임스탬프를 사용할 때, 채굴자는 실제로 블록의 유효성을 검사하는 시점으로부터 30초 이내에 타임스탬프를 게시할 수 있으므로, 사실상 채굴자들은 복권에서 그들의 기회를 더 유리하게 할 수 있도록 설정값을 미리 연산할 수 있습니다. 타임스탬프는 무작위가 아니며 해당 컨텍스트에서 사용해서는 안됩니다.

### *30초 법칙(30-second Rule)*
타임스탬프의 사용을 평가하는데 사용되는 일반적인 경험 법칙:
#### 만약 컨트랙트 함수가 [30초](https://ethereum.stackexchange.com/questions/5924/how-do-ethereum-mining-nodes-maintain-a-time-consistent-with-the-network/5931#5931)가 지연되는 것을 허용할 수 있다면 `block.timestamp`를 사용해도 안전합니다.
만약 당신의 시간 종속적인 이벤트의 규모가 30초간 다를 수 있고 무결성을 유지한다면, 타임스탬프를 사용해도 안전합니다. 경매 종료, 등록 기간 등이 포함됩니다.

### `block.number`를 타임스탬프로 사용하지 마세요.

컨트랙트가 [이것과]((https://github.com/SpankChain/old-sc_auction/blob/master/contracts/Auction.sol)) 같이 토큰 판매 종료를 나타내기 위해 `auction_complete` 수정자를 만들 때
```sol
modifier auction_complete {
    require(auctionEndBlock <= block.number     ||
          currentAuctionState == AuctionState.success ||
          currentAuctionState == AuctionState.cancel)
        _;}
```
`block.number`와 *[average block time](https://etherscan.io/chart/blocktime)* 은 시간을 계산하는데 사용될 수 있습니다. 그러나 이것은 블록 타임이 바뀔수 있기 때문에 미래를 보장할 수 없습니다 ([fork reorganisations](https://blog.ethereum.org/2015/08/08/chain-reorganisation-depth-expectations/) 및 [difficulty bomb](https://github.com/ethereum/EIPs/issues/649)과 같은 경우). 판매 기간 동안, 12분 법칙은 신뢰할 수 있는 시간의 추정치를 만들 수 있게 합니다.

## 다중 상속 주의 (Multiple Inheritance Caution)

솔리디티에서 다중 상속을 활용할 때, 컴파일러가 상속 그래프를 어떻게 구성하는지를 이해하는 것은 중요합니다.

```sol

contract Final {
    uint public a;
    function Final(uint f) public {
        a = f;
    }
}

contract B is Final {
    int public fee;

    function B(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 3;
    }
}

contract C is Final {
    int public fee;

    function C(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 5;
    }
}

contract A is B, C {
  function A() public B(3) C(5) {
      setFee();
  }
}
```
A가 배포되었을 때, 컴파일러는 다음과 같이 상속을 왼쪽에서 오른쪽으로 선형화할 것입니다:

**C -> B -> A**

선형화의 결과는 C가 가장 깊게 파생되었기 때문에 fee 값으로 5를 넘겨줍니다. 이 사례는 명백해 보이지만, C가 주요한 기능을 숨기고, 부울 절을 다시 정렬하고, 개발자가 부당하게 이용될 수 있는 컨트랙트를 작성하게 만드는 경우를 상상해보세요. 현재 정적 분석은 숨겨져 있는 함수들에 대한 문제점을 발견하지 못합니다. 때문에 반드시 직접 검사해야만 합니다.

보안과 상속에 대해 더 알고 싶다면, 이 [기사](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/)를 확인하세요.

기여를 돕기 위해, 솔리디티 깃허브는 모든 상속 관련 문제들을 다루는 [프로젝트](https://github.com/ethereum/solidity/projects/9#card-8027020)를 가지고 있습니다.

## 지금은 불가능하지만 역사적으로 있었던 권장사항들

솔리디티 발전 또는 프로토콜의 변화로 인해 더 이상 사용하지 않는 권장사항들 입니다. 후배개발자(posterity)들과 경각(awareness)을 위해 이곳에 기록되었습니다.

### 0으로 나누기를 조심하세요 (솔리디티 < 0.4)

0.4 버전 이전의 솔리디티는 0으로 나누기를 했을 때 [0을 반환](https://github.com/ethereum/solidity/issues/670) 하고 예외를 `throw`하지 않았습니다. 솔리디티 버젼이 최소 0.4 버전인지 확인하세요.

<a name="known_attacks"></a>
# 알려진 공격들(Known Attacks)

다음은 당신이 스마트 컨트랙트를 작성할 때 인지하고 방지해야 하는 알려진 공격들의 목록입니다.

<a name="race-conditions"></a>
## 경합조건(Race Conditions)

컨트랙트를 외부(external)에서 호출 시 발생하는 가장 큰 위험 중 하나는 그것이 통제 흐름을 뺏을 수 있다는 것이며, 호출한 함수가 예상하지 못한 당신의 데이터를 변경할 수 있다는 것입니다. 이러한 종류의 버그는 많은 형태를 가지고 있을 수 있으며, DAO의 붕괴를 만들어낸 두 개의 주요 버그들은 이런 종류의 버그였습니다.

<a name="reentrancy"></a>
### 재진입(Reentrancy)

이미 알려진 이 버그의 첫 번째 버전은 함수의 첫 번째 호출이 종료되기 전에 재귀 호출되는 함수를 포함하고 있습니다. 이것은 파괴적인 방향으로 상호작용하는 다른 함수의 호출을 야기할 수 있습니다.

```sol
// 안전하지 않음
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // 이 부분에서 호출자의 코드가 실행되며, withdrawBalance를 다시 호출할 수 있음
    userBalances[msg.sender] = 0;
}
```

사용자의 잔액은 함수의 마지막까지 0이 되지 않기 때문에, 두 번째(그리고 이후) 호출은 계속해서 성공할 것이며, 잔액을 다시 반복해서 인출(withdraw)하게 됩니다. 매우 유사한 버그는 DAO 공격의 취약성들 중 하나였습니다.

예시에서 주어진 것처럼, 이 문제를 피하기 위한 가장 좋은 방법은 [`call.value()()` 대신 `send()`를 사용하는 것]입니다. 이렇게 하면 외부 코드가 호출되는 것을 막을 수 있습니다.

그러나 만약 당신이 외부 호출을 제거하지 못한다면, 이 공격을 막기위한 가장 간단한 방법은 당신이 해야하는 모든 내부(internal) 작업을 마치기 전까지 외부 함수를 호출하지 않는 것입니다:

```sol
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    require(msg.sender.call.value(amountToWithdraw)()); // 사용자의 잔액은 이미 0이기 때문에 이후의 호출들은 아무것도 인출하지 못함
}
```

만약 당신이 `withdrawBalance()`를 호출하는 다른 함수를 가지고 있다면, 같은 공격의 잠재적인 목표가 될 수 있음에 유의하세요. 그러므로 당신은 신뢰할 수 없는 컨트랙트를 호출하는 함수들 그 자체를 신뢰할 수 없는 것으로 취급해야 합니다. 가능성 있는 해결책(solution)들에 대한 자세한 내용은 아래를 참조하십시오.

### 함수간 경합 조건(Cross-function Race Conditions)

또한 공격자는 같은 상태(state)를 공유하는 두 가지 다른 함수들을 사용해 유사한 공격을 할 수 있습니다.

```sol
// 안전하지 않음
mapping (address => uint) private userBalances;

function transfer(address to, uint amount) {
    if (userBalances[msg.sender] >= amount) {
       userBalances[to] += amount;
       userBalances[msg.sender] -= amount;
    }
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // 이 부분에서 호출자의 코드가 실행되며, transfer()를 호출할 수 있음
    userBalances[msg.sender] = 0;
}
```

이 경우, 공격자는 코드가 `withdrawBalance` 내에서 외부 호출로 실행되었다면 `transfer()`를 호출합니다. 공격자들의 잔액은 0이 되지 않기 때문에, 그들은 이미 인출을 받았음에도 불구하고 토큰을 전송할 수 있습니다. 이 취약성 또한 DAO 공격에 사용되었습니다.

같은 주의 사항의 같은 해결책으로 해결이 가능합니다. 또한 이 예시에서는 두 함수 모두 같은 컨트랙트의 일부임을 유의하세요. 그러나 여러 컨트랙트들이 상태를 공유한다면 같은 버그가 컨트랙트들 간에 발생할 수 있습니다.

### 경합 조건 해결책의 위험요소(Pitfalls in Race Condition Solutions)

경합 조건은 여러 함수들, 심지어는 여러 컨트랙트들 간에 발생할 수 있기 때문에 재진입을 막는 것을 목표로 하는 해결책들은 충분하지 않습니다.

대신 우리는 모든 내부 작업을 우선으로 마친 후, 외부 함수를 호출하는 것을 추천합니다. 만약 당신이 이 규칙을 잘 지킨다면, 당신은 경합 조건을 피할 수 있습니다. 그러나 당신은 외부 함수를 너무 빨리 호출하는 것을 피하는 것 뿐만 아니라 외부 함수를 호출하는 함수를 호출하는 것도 피해야 합니다. 다음 예시는 안전하지 못한 경우입니다:

```sol
// 안전하지 않음
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function withdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function getFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // 받는 사람들은 각자 보너스를 한번만 요구할 수 있음

    rewardsForA[recipient] += 100;
    withdraw(recipient); // 이 부분에서 호출자는 getFirstWithdrawalBonus를 다시 실행할 수 있음
    claimedBonus[recipient] = true;
}
```

`getFirstWithdrawalBonus()`가 외부 컨트랙트를 직접 호출하지 않음에도 불구하고 `withdraw()`내의 호출은 경합 조건을 취약하게 만들기에 충분합니다. 그러므로 당신은 `withdraw()` 또한 신뢰할 수 없는 것처럼 취급해야 합니다.

```sol
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function untrustedWithdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function untrustedGetFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // 받는 사람들은 각자 보너스를 한번만 요구할 수 있음

    claimedBonus[recipient] = true;
    rewardsForA[recipient] += 100;
    untrustedWithdraw(recipient); // claimedBonus를 true로 설정합니다. 그래서 재진입(reentry)이 불가능함
}
```

재진입을 불가능하도록 수정하는 것 뿐만 아니라, [신뢰할 수 없는 함수들은 표시가 되었습니다](#mark-untrusted-contracts). 이것과 같은 양식은 모든 수준에서 반복됩니다: `untrustedGetFirstWithdrawalBonus()` 함수가 외부 컨트랙트를 호출하는 `untrustedWithdraw()`를 호출하기 때문에, 당신은 또한 반드시 `untrustedGetFirstWithdrawalBonus()`를 안전하지 않은 것으로 대해야 합니다.

종종 추천되는 또 다른 해결책은 [뮤텍스(mutex)](https://en.wikipedia.org/wiki/Mutual_exclusion)입니다. 이것은 당신이 몇몇 상태를 "잠금"할 수 있도록 함으로써 "잠금"의 소유자에 의해서만 변경하는 것이 가능하게 합니다. 쉬운 예시는 다음과 같습니다.

```sol
// 주석(Note): 이것은 가장 기본적인 예시이며 뮤텍스는 큰 규모의(substantial) 로직 그리고/또는 공유된 상태에서 특히 유용합니다.
mapping (address => uint) private balances;
bool private lockBalances;

function deposit() payable public returns (bool) {
    require(!lockBalances);
    lockBalances = true;
    balances[msg.sender] += msg.value;
    lockBalances = false;
    return true;
}

function withdraw(uint amount) payable public returns (bool) {
    require(!lockBalances && amount > 0 && balances[msg.sender] >= amount);
    lockBalances = true;

    if (msg.sender.call(amount)()) { // 일반적으로는 안전하지 않지만, 뮤텍스가 이것을 안전하게 함
      balances[msg.sender] -= amount;
    }

    lockBalances = false;
    return true;
}
```

만약 사용자가 첫 번째 호출이 끝나기 전에 `withdraw()`를 다시 호출하려고 한다면, "잠금"은 이것이 영향을 미치기 전에 막을 것입니다. 이것은 효율적인 패턴이 될 수 있지만, 당신이 함께 동작해야 하는 여러 컨트랙트들을 가지고 있다면 난해하게 됩니다. 다음 예시는 안전하지 못합니다:

```sol
// 안전하지 않음
contract StateHolder {
    uint private n;
    address private lockHolder;

    function getLock() {
        require(lockHolder == 0);
        lockHolder = msg.sender;
    }

    function releaseLock() {
        require(msg.sender == lockHolder);
        lockHolder = 0;
    }

    function set(uint newState) {
        require(msg.sender == lockHolder);
        n = newState;
    }
}
```

공격자는 `getLock()` 함수를 호출할 수 있지만, 그런 다음 `releaseLock()`을 호출할 수 없습니다. 만약 그들이 그렇게 한다면, 컨트랙트는 영원히 잠기게 됩니다. 그리고 추가 변경은 일어나지 않을 것입니다. 만약 당신이 경합 조건으로부터 보호하기 위해 뮤텍스를 사용한다면, 당신은 "잠금"을 요청할 수 있는 방법이 없다는 것과 다시는 풀 수 없다는 것을 주의해서 보장해야 합니다.

<a name='footnote-race-condition-terminology'></a>

<div style='font-size: 80%; display: inline;'>* 몇몇 사람들은 이더리움이 현재에는 진정한 병행성을 가지고 있지 않기 때문에 <i>경합 조건</i>이라는 단어의 사용에 반대할 수도 있습니다. 하지만 자원(resources)을 차지하기 위한 논리적으로 서로 다른 프로세스(processes)의 기본적인 특징들이 여전히 존재하며 같은 종류의 위험들과 잠재적인 해결책이 적용되어 있습니다.</div>

<a name="transaction-ordering-dependence-tod-front-running"></a>
## 트랜잭션-순서 의존성(TOD : Transaction-Ordering Dependence) / 선행매매(Front Running)

위의 예시들은 공격자가 악의적인 코드를 *단일 트랜잭션 내*에서 실행하는 것을 포함하고 있는 경합 조건들의 예시입니다. 다음은 블록체인에 내재되어 있는 다른 종류의 경합 조건입니다: 실제로 *트랜잭션들의 순서*(블록 내의)는 쉽게 조작될 수 있습니다.

트랜잭션은 멤풀(Mempool)내에 잠시 머무르게 되기 때문에, 누군가는 트랜잭션이 블록에 포함되기 전에 어떤 동작이 일어나게 될지 알 수 있습니다. 이것은 몇 개의 토큰을 구매하기 위한 트랜잭션이 보일 수 있는 탈중앙화되된 시장(market)이나, 다른 트랜잭션이 포함되기 전에 시장 주문이 먼저 수행되는 것과 같은 문제를 만들어 낼 수 있습니다. 이것을 방지하는것은 특정 컨트랙트 그 자체에 이르기 때문에 어렵습니다. 예를 들면 시장에서는 집단 경매(batch auction)을 구현하는 것이 더 낫습니다(또한 이것은 잦은 빈도의 구매에 대한 걱정을 하지 않아도 됩니다). 또 다른 방법은 선-커밋(pre-commit) 구조를 사용하는 것입니다("자세한 내용은 나중에 제출하겠습니다").

## 타임스탬프 의존성(Timestamp Dependence)

블록의 타임스탬프는 채굴자에 의해 조작될 수 있음을 알고 있어야 합니다. 그리고 모든 직, 간접적인 타임스탬프의 사용은 고려해야 합니다.

타임스탬프 의존성에 관련된 디자인 고려사항들을 위한 [권장 사항](#timestamp-dependence) 부분을 참조하세요.

## 정수 오버플로우와 언더플로우

간단한 토큰 전송을 살펴보겠습니다:

```sol
mapping (address => uint256) public balanceOf;

// 안전하지 않음
function transfer(address _to, uint256 _value) {
    /* 보내는 사람이 충분한 잔액을 가지고 있는지를 확인함 */
    require(balanceOf[msg.sender] >= _value);
    /* 새로운 잔액을 더하고 뺌 */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}

// 안전함
function transfer(address _to, uint256 _value) {
    /* 보내는 사람의 잔액이 충분한지와 오버플로우를 확인함 */
    require(balanceOf[msg.sender] >= _value && balanceOf[_to] + _value >= balanceOf[_to]);

    /* 새로운 잔액을 더하고 뺌 */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}
```

만약 잔액이 최대 uint 값(2^256)에 다다르면, 이것은 0으로 돌아가게 됩니다. 이것은 이런 조건을 확인합니다. 이것이 관련이 있든, 없든 구현에 따르게 됩니다. 어떻게 됐든 uint 값이 매우 큰 숫자에 근접할 경우를 생각해봅시다. 어떻게 uint 변수가 상태를 변경시키고, 그리고 누가 이와 같은 변경을 할 수 있는 권한을 가지고 있는지 생각해봅시다. 만약 아무나 uint 값을 변경하는 함수를 호출할 수 있다면, 이것은 공격에 더욱 취약해집니다. 만약 관리자만이 변수의 상태를 변경할 수 있도록 접근할 수 있다면, 당신은 안전할 것입니다. 또한 만약 사용자가 한번에 1 밖에 증가시킬 수 없다면, 이런 한계값에 근접할 수 있는 실현 가능한 방법이 없기 때문에 아마 당신은 안전할 것입니다.

언더플로우도 마찬가지입니다. 만약 uint가 0보다 작아진다면, 이것은 언더플로우를 발생시킬 것이고 uint의 최대값으로 설정될 것입니다.

uint8, uint16, uint24 등과 같은 더 작은 데이터의 종류에 주의해야합니다: 이 데이터 종류들은 심지어는 더 쉽게 최대값을 만들어 낼 수 있습니다.

[20가지 이상의 오버플로우와 언더플로우](https://github.com/ethereum/solidity/issues/796#issuecomment-253578925)가 있다는 것을 알고 있어야 합니다.

### 언더플로우 심화: 저장소 조작(Storage Manipulation)
 2017 underhanded solidity contest에서 [Doug Hoyte의 출품작](https://github.com/Arachnid/uscc/tree/master/submissions-2017/doughoyte)은 선외 가작에 들었습니다. 출품작은 매우 흥미롭습니다. 왜냐하면 이것은 C언어 스타일의 언더플로우가 솔리디티 저장소에 어떻게 영향을 미칠지에 대한 문제를 제기했기 때문입니다. 이것은 위 출품작의 간단한 버전입니다:

 ```sol
contract UnderflowManipulation {
    address public owner;
    uint256 public manipulateMe = 10;
    function UnderflowManipulation() {
        owner = msg.sender;
    }

    uint[] public bonusCodes;

    function pushBonusCode(uint code) {
        bonusCodes.push(code);
    }

    function popBonusCode()  {
        require(bonusCodes.length >=0);  // 항상 진실인 명제(Tautology)
        bonusCodes.length--; // 이 부분에서 언더플로우가 발생할 수 있음
    }

    function modifyBonusCode(uint index, uint update)  {
        require(index < bonusCodes.length);
        bonusCodes[index] = update; // bonusCodes.length보다 작은 인덱스에 입력함
    }

}
```

일반적으로, 변수 `manipulateMe`의 위치는 `keccak256`을 거치지 않는다면 영향을 줄 수 없습니다. 즉, 실행이 불가능합니다. 하지만 동적 배열은 순차적으로 저장하기 때문에, 만약 악의적인 이용자가 `manipulateMe`를 변경하고자 한다면 그들은 해야하는 행동은 다음과 같습니다:

* 언더플로우를 위해 `popBonusCode`를 호출합니다(주의 사항: 솔리디티는 [내장된 pop 메서드가 없습니다](https://github.com/ethereum/solidity/pull/3743)).
* `manipulateMe`의 저장소 위치를 계산합니다.
* `modifyBonusCode`를 이용해 `manipulateMe`의 값을 조작하거나 갱신합니다.

실제로 이 배열은 즉시 수상하다고 지적되겠지만, 더 복잡한 스마트 컨트랙트 구조에 뒤덮여 있어 임의로 상수 변수에 대해 악의적인 변경을 허용할 수 있습니다.

동적 배열 사용을 고려중이라면 컨테이너 데이터 구조는 종은 예제입니다. 솔리디티 CRUD [파트 1](https://medium.com/@robhitchens/solidity-crud-part-1-824ffa69509a)와 [파트 2](https://medium.com/@robhitchens/solidity-crud-part-2-ed8d8b4f74ec) 기사는 좋은 참고 자료입니다.

<a name="dos-with-unexpected-revert"></a>

## (예상 밖의) 회귀(revert)가 있는 DoS(DoS with Unexpected revert)

간단한 경매(auction) 컨트랙트를 살펴보겠습니다:

```sol
// 안전하지 않음
contract Auction {
    address currentLeader;
    uint highestBid;

    function bid() payable {
        require(msg.value > highestBid);

        require(currentLeader.send(highestBid)); // 이전의 리더에게 환불을 해줌 그리고 만약 환불에 실패하면 회귀됨

        currentLeader = msg.sender;
        highestBid = msg.value;
    }
}
```

이전의 리더에게 환불해주려 할 때, 환불이 실패한다면 회귀하게 됩니다. 이것은 그들의 주소로 환불하려는 것을 *항상* 실패하게 함으로써 악의적인 가격 제시자가 리더가 될 수 있다는 것을 의미합니다. 이런 경우 그들은 `bid()` 함수를 호출하는 다른 사람들을 막을 수 있으며, 계속 리더로 남아있을 수 있습니다. 추천하는 방법은 이전에 설명했던 것처럼 [pull payment system](#favor-pull-over-push-for-external-calls)을 설정하는 것입니다.

또 다른 예시는 컨트랙트가 배열을 통해 사용자들에게 결제하는 것을 반복하는 것입니다(예를 들자면 크라우드펀딩 컨트랙트의 서포터들). 이것은 각각의 결제가 성공하는 것을 확실히 하는 것을 원한다면 일반적으로 사용되는 방법입니다. 만약 그렇지 않다면 그 하나는 회귀됩니다. 문제는 하나가 실패한다면 당신은 전체 지불 시스템을 회귀시켜야 한다는 것입니다. 이것은 반복이 절대 끝나지 않는다는 것을 의미합니다. 한 개의 주소가 에러를 만들어내기 때문에 그 누구도 지불을 받지 못합니다.

```sol
address[] private refundAddresses;
mapping (address => uint) public refunds;

// 좋지 않은 코드
function refundAll() public {
    for(uint x; x < refundAddresses.length; x++) { // 얼마나 많은 주소들이 참가했는지에 기반한 임의의 길이만큼 반복
        require(refundAddresses[x].send(refunds[refundAddresses[x]])) // 두 배로 나쁨, 지금 send에서 발생한 단 하나의 실패는 모든 자금을 지연시킴
    }
}
```

다시 말하지만 추천하는 해결책으로는 [pull over push payments를 장려](#favor-pull-over-push-for-external-calls)합니다.

<a name="dos-with-block-gas-limit"></a>
## 블록 가스 한도가 있는 DoS(DoS with Block Gas Limit)

아마 당신은 이전의 예제에서 또 다른 문제가 있다는 것을 눈치챘을 것입니다: 모두에게 한번에 지불한다면, 당신은 블록 가스 한도에 도달할 위험이 있습니다. 각각의 이더리움 블록은 특정 최대 계산량을 처리할 수 있습니다. 만약 당신이 그 이상을 시도한다면, 당신의 트랜잭션은 실패할 것입니다.

이것은 의도적인 공격이 없어도 문제를 일으킬 수 있습니다. 그러나 공격자가 필요한 가스량을 조작할 수 있다면 특히 나쁩니다. 이전의 예제에서 공격자는 각각 소량의 환불만을 요구하는 많은 주소들을 추가할 수 있습니다. 그러므로 공격자의 각 주소들에 환불하는데 드는 가스 비용은 가스 한도 이상일 수 있으며 환불 트랜잭션이 전혀 발생하지 않습니다.

이것은 [pull over push payments를 장려](#favor-pull-over-push-for-external-calls)하는 또 다른 이유입니다.

만약 당신이 반드시 꼭 알 수 없는 크기의 배열을 반복해야 한다면, 잠재적으로 여러 블록을 사용할 수 있도록 계획해야 합니다. 즉, 여러개의 트랜잭션이 필요합니다. 다음 예시와 같이 당신은 얼마나 진행했는지 추적할 수 있어야 하며, 그 지점으로부터 다시 시작할 수 있어야 합니다:

```sol
struct Payee {
    address addr;
    uint256 value;
}

Payee[] payees;
uint256 nextPayeeIndex;

function payOut() {
    uint256 i = nextPayeeIndex;
    while (i < payees.length && msg.gas > 200000) {
      payees[i].addr.send(payees[i].value);
      i++;
    }
    nextPayeeIndex = i;
}
```

`payOut()` 함수의 다음 반복을 기다리는 동안 다른 트랜잭션이 처리되더라도 별다른 일이 일어나지 않도록 해야합니다. 그러니 이 패턴은 반드시 필요한 경우에만 사용하세요.

## 강제로 컨트랙트에 이더를 전송(Forcibly Sending Ether to a Contract)

컨트랙트의 폴백 함수(fallback function)를 트리거링하지 않으면서 강제로 컨트랙트에 이더를 전송하는 것이 가능합니다. 이것은 중요한 로직을 폴백 함수에 위치시키거나 컨트랙트의 잔액을 기반으로 하는 계산을 만들 때 중요하게 고려해야합니다. 다음 예시를 보겠습니다:

```sol
contract Vulnerable {
    function () payable {
        revert();
    }

    function somethingBad() {
        require(this.balance > 0);
        // 무언가 악의적인 행동을 함
    }
}
```

컨트랙트 로직은 컨트랙트로의 결제를 허용하지 않기 때문에 "무언가 악의적인 행동"이 일어나는 것을 허용하지 않는 것처럼 보입니다. 그러나 컨트랙트에 이더를 강제로 전송하고 그 잔액을 0보다 크게 만드는 몇 가지 방법이 있습니다.

`selfdestruct` 컨트랙트 메서드는 사용자가 초과하는 이더를 받을 수령인을 명시하는 것을 가능하게 합니다.  `selfdestruct`는 [컨트랙트의 폴백 함수를 작동시키지 않습니다](https://solidity.readthedocs.io/en/develop/security-considerations.html#sending-and-receiving-ether).

또한 컨트랙트를 배포하기 전에 컨트랙트의 주소와 이더를 그 주소로 보내는 것을 [미리 계산하는 것](https://github.com/Arachnid/uscc/tree/master/submissions-2017/ricmoo)도 가능합니다.

컨트랙트 개발자들은 이더가 컨트랙트에 강제로 전송될 수 있다는 것을 반드시 알고 있어야 하며 컨트랙트 로직을 상황에 맞게 설계해야 합니다. 일반적으로는 당신의 컨트랙트에 돈을 보내는 출처를 차단하는것이 불가능하다는 것을 가정합니다.

## 지금은 불가능하지만 역사적으로 발생했던 공격들(Deprecated/historical attacks)

이와 같은 공격들은 프로토콜의 변경들과 솔리디티의 개선으로 인해 더 이상은 불가능합니다. 후배 개발자(posterity)들과 이에 대한 경각(awareness)을 위해 이곳에 기록되었습니다.

### 호출 깊이 공격(Call Depth Attack, 현재에는 불가능함)

[EIP 150](https://github.com/ethereum/EIPs/issues/150)의 하드포크로 Call Depth 공격은 더 이상 관련이 없습니다<sup><a href='http://ethereum.stackexchange.com/questions/9398/how-does-eip-150-change-the-call-depth-attack'>\*</a></sup> (모든 가스가 1024 호출 깊이 한도에 다다르기 이전에 소모될 것입니다).

<a name="software_engineering"></a>
# 소프트웨어 공학 기법들(Software Engineering Techniques)

[일반적인 철학](#General_Philosophy)에서 논했던 것처럼, 당신 스스로를 알려진 공격들로부터 방어하는 것만으로는 충분하지 않습니다. 블록체인 상에서의 실패 비용은 매우 높을 수 있기 때문에, 이러한 위험을 고려하여 소프트웨어를 작성하는 방법을 바꿔야 합니다.

우리가 선호하는 접근은 "실패에 대비하라"입니다. 당신의 코드가 안전한지 사전에 아는 것은 불가능 합니다. 그러나 당신은 컨트랙트가 실패하는 것을 최소한의 피해로 너그럽게 받아들일 수 있도록 설계할 수 있습니다. 이 부분은 당신이 실패에 대비하는 것에 도움을 줄 수 있는 다양한 기법들을 보여줍니다.

새로운 구성요소를 당신의 시스템에 추가하는 것은 항상 위험이 있습니다. 형편없이 설계된 고장-안전(fail-safe)은 그 자체로 취약점이 될 수 있습니다 - 다수의 잘 설계된 고장-안전 사이의 상호작용 또한 마찬가지입니다.당신이 컨트랙트에 사용하는 각 기법들에 대해 깊이 생각할 수 있어야 하며, 어떻게 그것들이 함께 동작하여 탄탄한 시스템을 구축할 수 있을지에 대해 신중히 고려해야 합니다.

### 고장난 컨트랙트 업그레이드(Upgrading Broken Contracts)

!!! 주의
     이 부분은 만기 되었습니다. 여기엔 중요한 질문들이 많이 있고, 스마트 컨트렉트 업그레이드와 관련된 리스크들이 있습니다. 최신 기술(the state of the art)에 대해 연구를 하세요. [관련 이슈](https://github.com/ConsenSys/smart-contract-best-practices/issues/164)의 토론을 환영합니다.

코드는 오류가 발견되었거나 개선이 필요할 때 변경이 필요하게 됩니다. 버그를 찾는 것은 좋지는 않지만, 버그를 다룰 수 있는 방법도 없습니다.

스마트 컨트랙트를 위해 효율적인 업그레이드 시스템을 설계하는 것은 활발히 연구중인 부문이며 이 문서 내에의 모든 문제들을 해결할 수는 없습니다. 그러나 가장 일반적으로 사용되는 두 가지 기본적인 접근법이 있습니다. 두 가지 중 더 간단한 것은 가장 최신 버전의 컨트랙트 주소를 가지고 있는 등록 컨트랙트를 만드는 것입니다. 컨트랙트 사용자를 위한 좀 더 매끄러운 방법은 최신 버전의 컨트랙트에 호출과 데이터를 전달하는 컨트랙트를 만드는 것입니다.

기법이 무엇이든, 모듈화와 구성요소 사이를 적절히 분리하는 것은 중요하며, 이와 같은 과정을 거친 코드는 변경이 되더라도 기능성을 잃거나, 데이터를 유실하거나 또는 복사에 큰 비용이 발생하지 않게 됩니다. 특히 당신의 데이터 저장공간으로부터 복잡한 로직을 분리하는 것은 대개 효과적이며, 그럼으로써 기능성을 변경하기 위해 모든 데이터를 변경하지 않아도 됩니다.

또한 단체들이 코드를 업그레이드를 결정하는 것을 위한 안전한 방법을 찾는 것도 중요합니다. 당신의 컨트랙트에 따라, 코드의 변경은 신뢰할 수 있는 단일 단체 또는 집단의 구성원들의 승인을 필요로 하거나 전체 주주들의 투표를 필요로 할 수도 있습니다. 만약 이 절차가 어느 정도의 시간을 필요로 한다면, [비상 정지(emergency stop) 또는 서킷 브레이커(circuit-breaker)](#circuit-breakers-pause-contract-functionality) 디자인 패턴과 같이 공격시 보다 더 빠르게 반응할 수 있는 다른 방법들이 있는지를 고려해야 합니다.

**예시 1: 컨트랙트의 최신 버전의 컨트랙트 주소를 저장하기 위한 등록 컨트랙트를 사용**

이 예시에서 호출은 전달되지 않기 때문에, 사용자들은 컨트랙트와 상호작용하기 전에 매번 현재 주소를 받아와야만 합니다.

```sol
contract SomeRegister {
    address backendContract;
    address[] previousBackends;
    address owner;

    function SomeRegister() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner)
        _;
    }

    function changeBackend(address newBackend) public
    onlyOwner()
    returns (bool)
    {
        if(newBackend != backendContract) {
            previousBackends.push(backendContract);
            backendContract = newBackend;
            return true;
        }

        return false;
    }
}
```

이러한 접근 방식에는 두 가지 큰 단점이 있습니다.

1. 사용자가 항상 반드시 현재 주소를 조회해야 하며, 이렇게 하지 않은 사용자들은 오래된 버전의 컨트랙트를 사용하는 위험을 감수해야 합니다.
2. 당신이 컨트랙트를 바꿀 때 컨트랙트의 데이터를 어떻게 다뤄야 할지 신중히 생각해야 합니다.

대안은 호출과 데이터를 최신 버전의 컨트랙트로 전달하는 컨트랙트를 사용하는 것입니다.

**예시 2: [`델리게이트 콜(DELEGATECALL)`을 사용](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts)해 데이터와 호출을 전달합니다.**

```sol
contract Relay {
    address public currentVersion;
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function Relay(address initAddr) {
        currentVersion = initAddr;
        owner = msg.sender; // 이 소유주는 한 명의 컨트랙트 소유주가 아닌 다중 서명된 또 다른 컨트랙트일 수 있습니다.
    }

    function changeContract(address newVersion) public
    onlyOwner()
    {
        currentVersion = newVersion;
    }

    function() {
        require(currentVersion.delegatecall(msg.data));
    }
}
```

이 접근 방식은 이전의 문제를 피할 수는 있지만 그 자체의 문제를 가지고 있습니다. 당신은 컨트랙트 안에 데이터를 저장하는 방법에 대해 극도로 신경써야만 합니다. 만약 당신의 새로운 컨트랙트가 첫 번째와 다른 형태로 저장소를 만들었다면, 결국 당신의 데이터에는 오류가 발생할 것입니다. 추가로 이 패턴의 간단한 버전은 함수로부터 값을 반환할 수 없으며, 오직 응용에 한계를 가진 값들을 전달만 할 수 있습니다. ([더 복잡한 구현들](https://github.com/ownage-ltd/ether-router)은 이 문제를 인라인(in-line) 어셈블리 코드와 반환 값들의 저장소로 해결하는 것을 시도합니다.)

당신의 접근 방식에도 불구하고, 당신의 컨트랙트를 업그레이드하는 몇 가지 방법을 가지는 것은 중요합니다. 그렇지 않으면 컨트랙트들은 불가피한 버그들이 컨트랙트들에서 발견된다면 사용할 수 없게 될 것입니다.

<a name="circuit-breakers-pause-contract-functionality"></a>
### 서킷 브레이커(Circuit Breakers, 함수의 기능을 정지시킴)

서킷 브레이커는 특정 조건이 만족되면 실행을 중지시킵니다. 그리고 새로운 버그들이 발견되었을 때 유용합니다. 예를 들어, 대부분의 동작들이 컨트랙트 내에서 버그가 별견되었다면 중단될 것입니다. 그리고 가능한 유일한 동작은 인출 뿐입니다. 당신은 특정한 신뢰할 수 있는 단체에게 서킷 브레이커를 작동시킬 수 있는 능력을 주거나, 아니면 특정 조건이 만족되었을 때 자동으로 서킷 브레이커가 작동되게 하는 프로그램 규칙을 넣을 수 있습니다.

예시:

```sol
bool private stopped = false;
address private owner;

modifier isAdmin() {
    require(msg.sender == owner);
    _;
}

function toggleContractActive() isAdmin public {
    // 당신은 사용자들의 투표와 같이 다른 동작에 기반하여 컨트랙트를 멈추는 것을 제한하는 수정자를 추가할 수 있음
    stopped = !stopped;
}

modifier stopInEmergency { if (!stopped) _; }
modifier onlyInEmergency { if (stopped) _; }

function deposit() stopInEmergency public {
    // some code
}

function withdraw() onlyInEmergency public {
    // some code
}
```

### 속도 제한(Speed Bumps, 컨트랙트의 동작을 지연시킴)

속도 제한은 동작을 느리게 함으로써, 악의적인 행동이 발생하더라도 복구할 시간을 확보합니다. 예를 들어, [The DAO](https://github.com/slockit/DAO/)는 DAO를 나누는 성공적인 요청과 이를 수행하는데 27일을 필요로 합니다. 이것은 자금이 컨트랙트 내에 보관되는 것을 보장하고, 복구 가능성을 높입니다. DAO의 경우 속도 제한에 의해 주어지는 시간 동안 할 수 있는 효율적인 행동이 없습니다. 그러나 우리의 다른 기법들과의 조합한다면, 매우 효과적일 것입니다.

예시:

```sol
struct RequestedWithdrawal {
    uint amount;
    uint time;
}

mapping (address => uint) private balances;
mapping (address => RequestedWithdrawal) private requestedWithdrawals;
uint constant withdrawalWaitPeriod = 28 days; // 4주

function requestWithdrawal() public {
    if (balances[msg.sender] > 0) {
        uint amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0; // 간결함을 위해, 우리는 모든 것을 인출할 것임;
        // 아마 예금 함수는 인출이 진행중이라면 새로운 예금을 막음

        requestedWithdrawals[msg.sender] = RequestedWithdrawal({
            amount: amountToWithdraw,
            time: now
        });
    }
}

function withdraw() public {
    if(requestedWithdrawals[msg.sender].amount > 0 && now > requestedWithdrawals[msg.sender].time + withdrawalWaitPeriod) {
        uint amountToWithdraw = requestedWithdrawals[msg.sender].amount;
        requestedWithdrawals[msg.sender].amount = 0;

        require(msg.sender.send(amountToWithdraw));
    }
}
```

### 빈도수 제한(Rate Limiting)

빈도수 제한은 큰 변화에 대해 중단 또는 승인을 요청합니다. 예를 들어, 예금자는 일정 기간이 경과한 후에 특정 금액 또는 전체 예금의 일정 비율을 인출할 수 있습니다(예를 들면 하루 이후에 최대 100 이더) - 그 기간 중의 추가 인출은 실패하거나 특별한 승인 등이 필요합니다. 또한 빈도수 제한은 일정 기간 동안 컨트랙트에 의해 발행되는 특정 양의 토큰들과 같이 컨트랙트 수준에서도 가능합니다.

[예시](https://gist.github.com/PeterBorah/110c331dca7d23236f80e69c83a9d58c#file-circuitbreaker-sol)

### 컨트랙트 공개(Contract Rollout)

컨트랙트는 상당한 그리고 장기간의 테스트 기간을 가져야만 합니다 - 상당한 돈이 위험에 놓이기 전에.

최소한 당신은 다음과 같이 해야합니다:

- 100% 테스트 범위(또는 이에 근접한)를 가진 테스트 묶음을 가지고 있어야 합니다.
- 당신만의 테스트넷에 배포해야 합니다.
- 많은 테스트와 버그 바운티를 위해 공개 테스트넷에 배포해야 합니다.
- 철저한 테스트를 통해 더 큰 규모에서 다양한 참가자들이 컨트랙트를 통해 상호작용할 수 있게 해야 합니다.
- 메인넷에 위험도를 제한한 베타 버전으로 배포해야 합니다.

##### 자동 기능 중지(Automatic Deprecation)

테스트 중 당신은 특정 기간 이후에 동작들을 막음으로써 자동 기능 중지를 만들어낼 수 있습니다. 예를 들어, 알파 컨트랙트는 몇 주간 동작을 했고, 마지막 인출 기능을 제외한 모든 기능을 자동으로 중지했습니다.

```sol
modifier isActive() {
    require(block.number <= SOME_BLOCK_NUMBER);
    _;
}

function deposit() public isActive {
    // some code
}

function withdraw() public {
    // some code
}

```
##### 사용자/컨트랙트의 이더의 양을 제한(Restrict amount of Ether per user/contract)

초기 단계에 당신은 사용자의 이더의 양을 제한할 수 있었습니다 (또는 전체 컨트랙트에 대해) - 위험을 감소시킵니다.

<a name="bounties"></a>
### 버그 바운티 프로그램(Bug Bounty Programs)

바운티 프로그램을 운영하기 위한 몇 가지 팁:

- 바운티의 보상으로 어떤 화폐가 분배될 것인지를 결정해야 합니다 (BTC 그리고/또는 이더).
- 바운티 보상을 위해 예상되는 전체 예산을 결정해야 합니다.
- 예산에서 보상의 3가지 단계를 결정합니다.
  - 최소 보상 (흔쾌히 제공 합니다)
  - 최대 보상 (보통 보상받을만 합니다)
  - 추가 범위의 보상 (매우 심각한 취약점들인 경우 보상을 제공합니다)
- 누가 바운티 심사위원을 할지 결정합니다 (3명이 이상적인 숫자로 일반적입니다).
- 리드 개발자는 바운티 심사위원 중 한 명이어야 합니다.
- 버그 보고서를 받으면, 리드 개발자는 심사위원들의 조언을 참고해 버그의 심각도을 평가합니다.
- 이 단계에서의 작업은 개인(private) 레파지토리에 있어야 합니다. 그리고 문제(issue)는 깃허브에 보관됩니다.
- 개인 레파지토리에서 버그가 수정되었다면, 개발자는 실패해야 하며 버그를 확인할 수 있는 테스트 케이스를 작성해야만 합니다.
- 개발자는 수정 사항을 구현하고 테스트가 이제 통과한다는 것을 확인해야 합니다; 필요시 추가 테스트 케이스를 작성합니다.
- 바운티 헌터에게 수정 사항을 공개합니다; 공개 레파지토리에 수정사항을 병합(merge)시키는 것도 한 가지 방법입니다.
- 바운티 헌터가 수정 사항에 대한 또 다른 피드백이 있는지 결정합니다.
- 바운티 심사위원들은 버그의 *발생가능성*과 *영향력* 두 가지의 평가를 바탕으로 보상의 규모를 결정합니다.
- 과정 중에 발생하는 정보를 바운티 참가자들에게 계속해서 제공하고, 그들에게 보상을 보내는데 지연되는 것을 막기 위해 노력합니다.

보상의 3가지 단계에 대한 예시는, [이더리움의 바운티 프로그램](https://bounty.ethereum.org)을 참고하세요:

> 지불되는 보상의 가치는 영향의 심각도에 기반해 달라집니다. 중요하지 않은 '무해한' 버그들의 보상은 0.05 BTC부터 시작합니다. 합의 문제까지 이어질 수 있는 중요한 버그들은 5BTC까지 보상이 올라가게 됩니다. 매우 심각한 취약점들의 경우 더 높은 보상이 가능합니다 (최대 25BTC 까지).

<a name="tokens"></a>
# 토큰 구현을 위한 모범 사례

토큰을 구현하는것은 다른 모범 사례를 지켜야 하지만 또한 몇 가지 고유한 고려사항도 가져야만 합니다.

## 최신 표준 준수

일반적으로 토큰의 스마트 컨트랙트는 승인되고 안정한 표준을 따라야 합니다.

현재 승인된 표준들의 예시입니다:

* [EIP20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md)
* [EIP721](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md) (대체할 수 없는 토큰)

## EIP-20에 대한 선행매매 공격을 알아야 합니다.

EIP-20 토큰의 `approve()` 함수는 인가된 소비자가 의도한 양보다 더 많이 소비할 수 있는 가능성을 만듭니다. [선행매매 공격](#transaction-ordering-dependence-tod-front-running)이 사용될 수 있으며, 이것은 `approve()` 호출이 처리되기 전후에 인가된 소비자가 `transferFrom()`을 호출할 수 있도록 합니다. 더 자세한 사항은 [EIP](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#approve)와 [이 문서](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit)에서 확인할 수 있습니다.

## 0x0 주소로의 토큰 전송 방지

이 글을 작성중인 현재, "영(0)" 주소([0x0000000000000000000000000000000000000000](https://etherscan.io/address/0x0000000000000000000000000000000000000000))는 총 8,000만 달러 이상의 가치를 가진 토큰들을 가지고 있습니다.

## 컨트랙트 주소로의 토큰 전송 방지

또한 스마트 컨트랙트의 주소와 같은 주소로의 토큰 전송을 방지하는 것을 고려해야 합니다.

이것을 열려있는 상태로 남겨둠으로써 발생하는 손실의 가능성에 대한 예시는 [EOS 토큰 스마트 컨트랙트](https://etherscan.io/address/0x86fa049857e0209aa7d9e616f7eb3b3b78ecfdb0)로 90,000개 이상의 토큰들이 컨트랙트 주소에 갇혀있습니다.

### 예시

위의 권장 사항들을 모두 구현한 예시는 다음의 수정자(modifier)를 만들어야 합니다; "받는 사람(to)" 주소가 0x0과 스마트 컨트랙트 자신의 주소가 아닌지를 검증합니다.

```sol
    modifier validDestination( address to ) {
        require(to != address(0x0));
        require(to != address(this) );
        _;
    }
```

그리고 수정자를 "transfer"과 "transferFrom" 메서드에 적용해야 합니다.

```sol
    function transfer(address _to, uint _value)
        validDestination(_to)
        returns (bool)
    {
        (... 당신의 로직 ...)
    }

    function transferFrom(address _from, address _to, uint _value)
        validDestination(_to)
        returns (bool)
    {
        (... 당신의 로직 ...)
    }
```

<a name="documentations_procedures"></a>
# 문서와 절차들

많은 자금을 가지고 있거나 반드시 필요한 컨트랙트를 출시할 때 적절한 문서를 포함시켜야 하는 것은 중요합니다. 보안과 관련된 몇 가지의 문서들은 다음 사항들을 포함해야 합니다:

## 명세 및 공개(Rollout) 계획들

- 명세, 도표, 상태 머신, 모델 그리고 기타 문서는 감사원들, 검토자들, 그리고 커뮤니티가 이 시스템이 무엇을 하고 싶은 것인지를 이해하는 것을 도와줍니다.
- 많은 버그들이 명세에서만 발견될 수 있으며, 이것들은 가장 적은 비용으로 고칠 수 있습니다.
- 공개 계획들은 [이 링크](https://github.com/ConsenSys/smart-contract-best-practices#contract-rollout)에서 나열된 세부 사항들과 예정기한을 포함해야 합니다.

## 상태(Status)

- 현재 코드가 배포된 곳
- 컴파일러 버전, 플래그(flag)가 사용된 곳, 그리고 배포된 바이트코드가 소스 코드와 일치하는지를 검증하는 순서
- 다른 단계의 롤아웃에서 사용될 컴파일러 버전 및 플래그
- 배포된 코드의 현 상태 (해결되지 않은 문제들, 성능 상태, 기타 사항들을 포함합니다.)

## 알려진 문제들

- 컨트랙트의 주요 위험요소
  - 예 : 당신은 당신의 돈 전부를 잃을 수 있습니다, 해커는 특정 결과를 위해 투표할 수 있습니다.
- 잘 알려진 버그들/한계점
- 잠재적인 공격들 및 방지책
- 잠재적인 이해충돌 (예 : Slock.it이 DAO를 이용한 것과 같이, 스스로를 이용할 것입니다.)

## 이력

- 테스트 (사용 상태, 발견된 버그들, 테스트의 길이를 포함)
- 코드를 검토한 사람들 (그리고 그들의 주요 피드백)

## 절차

- 버그가 발견되었을 때의 행동 계획 (예 : 비상 옵션들, 공지 절차, 기타 등)
- 무언가 잘못되었을 때 서서히 종료시키는 과정 (예 : 투자자들은 공격전에 남아있는 기금으로부터 당신의 잔액 비율만큼을 얻을 것입니다.)
- 책임감있는 운영 방침 공개 (예 : 버그들을 발견시 어디로 신고할 것인지, 버그 현상금(bounty) 프로그램 규칙)
- 실패시 상환청구 (예 : 보험, 위약금, 상환청구 없음)

## 연락처 정보

- 문제 발생시 누구에게 연락해야 하는 사람
- 개발자들의 이름 그리고/또는 다른 중요한 집단들
- 질문받을 수 있는 대화방

<a name="security_tools"></a>
# 보안 도구

### 정적 분석(Static Analysis)

- [Manticore](https://github.com/trailofbits/manticore) - [EVM을 지원](https://asciinema.org/a/haJU2cl0R0Q3jB9wd733LVosL)하는 동적 이진 분석 도구 입니다.
- [Mythril](https://github.com/ConsenSys/mythril) - 이더리움 블록체인을 위한 리버스 엔지니어링 및 버그 헌팅 프레임워크 입니다.
- [Oyente](https://github.com/melonproject/oyente) - [이 문서](http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf)에 기반하여 일반적인 취약점들을 찾기 위해 이더리움 코드를 분석 입니다.
- [Solgraph](https://github.com/raineorshine/solgraph) - 솔리디티 컨트랙트의 함수 제어 흐름을 시각화한 DOT 그래프를 생성하고 잠재적인 보안 취약점들을 강조합니다.
- [SmartCheck](https://tool.smartdec.net) - 보안 취약점 및 모범 사례를 위한 솔리디티 소스 코드의 정적 분석 입니다.

### 검사 방법(Test Coverage)

- [solidity-coverage](https://github.com/sc-forks/solidity-coverage) - 솔리디티 테스트를 위한 코드 검사 입니다.

### 린터(Linters)

린터는 스타일과 구성을 위해 코드 작성 규칙을 강제함으로써 코드의 품질을 개선하여 코드가 읽고 검토하기 쉽게 만듭니다.

- [Solcheck](https://github.com/federicobond/solcheck) - JS로 작성된 솔리디티 코드를 위한 린터로 eslint로부터 많은 영향을 받습니다.
- [Solint](https://github.com/weifund/solint) - 당신의 솔리디티 스마트 컨트랙트들에서 당신이 일관된 규칙을 적용하고 오류를 피할 수 있도록 도와주는 솔리디티 린터 입니다.
- [Solium](https://github.com/duaraghav8/Solium) - 또 다른 솔리디티 린터 입니다.
- [Solhint](https://github.com/protofire/solhint) - 보안과 작성 규칙 검증을 모두 제공하는 솔리디티를 위한 린터 입니다.

<a name="security_notifications"></a>
# 보안 알림

이것은 이더리움과 솔리디티에서 발견된 보안 취약점을 강조하는 출처들의 목록입니다. 보안 알림(security notifications)의 공식 출처는 이더리움 블로그이지만, 대부분의 경우 취약점들은 다른 출처들에서 먼저 공개되고 이에 대해 논의될 것입니다.

- [이더리움 블로그](https://blog.ethereum.org/): 공식 이더리움 블로그 입니다.
  - [이더리움 블로그 - 보안](https://blog.ethereum.org/category/security/): *Security* 태그가 붙은 모든 블로그 글들 입니다.
- [이더리움 깃터(Gitter)](https://gitter.im/orgs/ethereum/rooms) 대화방들 입니다.
  - [솔리디티](https://gitter.im/ethereum/solidity)
  - [Go-Ethereum](https://gitter.im/ethereum/go-ethereum)
  - [CPP-Ethereum](https://gitter.im/ethereum/cpp-ethereum)
  - [Research](https://gitter.im/ethereum/research)
- [레딧(Reddit)](https://www.reddit.com/r/ethereum)
- [네트워크 상태(Network Stats)](https://ethstats.net/)

이 모든 출처에 기록된 보안 취약점들이 당신의 컨트랙트에 영향을 줄 수 있으므로 *정기적으로* 읽는 것을 매우 추천합니다.

추가로, 아래는 보안에 관련된 글을 작성한 이더리움 주요 개발자들의 목록이며 커뮤니티로부터 더 많은 것을 위해 [참고문헌](https://github.com/ConsenSys/smart-contract-best-practices#smart-contract-security-bibliography)을 참고하세요.

- **Vitalik Buterin**: [트위터](https://twitter.com/vitalikbuterin), [깃허브](https://github.com/vbuterin), [레딧](https://www.reddit.com/user/vbuterin), [이더리움 블로그](https://blog.ethereum.org/author/vitalik-buterin/)
- **Dr. Christian Reitwiessner**: [트위터](https://twitter.com/ethchris), [깃허브](https://github.com/chriseth), [이더리움 블로그](https://blog.ethereum.org/author/christian_r/)
- **Dr. Gavin Wood**: [트위터](https://twitter.com/gavofyork), [블로그](http://gavwood.com/), [깃허브](https://github.com/gavofyork)
- **Vlad Zamfir**: [트위터](https://twitter.com/vladzamfir), [깃허브](https://github.com/vladzamfir), [이더리움 블로그](https://blog.ethereum.org/author/vlad/)

주요 개발자들을 팔로우 하는 것 이외에도, 더 넓은 블록체인 관련 보안 커뮤니티에 참여하는 것도 중요합니다 - 공개되는 보안 이슈들이나 관찰되는 사항들은 다양한 집단으로부터 발생하기 때문입니다.


# 향후 개선 사항

## 함수형 언어

!!! 주의
    이 페이지는 매우 오래되었습니다. 이 페이지의 갱신은 언제나 환영합니다.

함수형 언어는 솔리디티와 같은 절차형 언어에 대해 함수 내에서의 불변성과 강한 컴파일 시간 확인과 같은 특정한 기능들을 보장해줍니다. 함수형 언어는 결정적인 행동을 제공함으로써 오류에 대한 위험을 감소시킬 수 있습니다 (더 자세한 내용은 [여기](https://plus.google.com/u/0/events/cmqejp6d43n5cqkdl3iu0582f4k)를 참고하세요. Curry–Howard correspondence와 linear logic).

<a name="bibliography"></a>
# 스마트 컨트랙트 보안 참고문헌

이 문서의 대부분은 이미 커뮤니티에 의해 작성된 다양한 부분에서 얻은 코드, 예시 그리고 통찰을 포함하고 있습니다.
그 중 몇 가지를 이곳에 소개합니다. 자유롭게 내용을 추가해도 됩니다.

##### 이더리움 주요 개발자들이 만든 문서

- [안전한 스마트 컨트랙트를 작성하는 방법](https://chriseth.github.io/notes/talks/safe_solidity) (Christian Reitwiessner)
- [스마트 컨트랙트 보안](https://blog.ethereum.org/2016/06/10/smart-contract-security/) (Christian Reitwiessner)
- [스마트 컨트랙트 보안에 대한 고찰](https://blog.ethereum.org/2016/06/19/thinking-smart-contract-security/) (Vitalik Buterin)
- [솔리디티](http://solidity.readthedocs.io)
- [솔리디티 보안 고려사항들](http://solidity.readthedocs.io/en/latest/security-considerations.html)

##### 커뮤니티가 만든 문서

- http://forum.ethereum.org/discussion/1317/reentrant-contracts
- http://hackingdistributed.com/2016/06/16/scanning-live-ethereum-contracts-for-bugs/
- http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/
- http://hackingdistributed.com/2016/06/22/smart-contract-escape-hatches/
- http://martin.swende.se/blog/Devcon1-and-contract-security.html
- http://publications.lib.chalmers.se/records/fulltext/234939/234939.pdf
- http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour
- http://vessenes.com/ethereum-griefing-wallets-send-w-throw-considered-harmful
- http://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal
- https://blog.blockstack.org/simple-contracts-are-better-contracts-what-we-can-learn-from-the-dao-6293214bad3a
- https://blog.slock.it/deja-vu-dao-smart-contracts-audit-results-d26bc088e32e
- https://blog.vdice.io/wp-content/uploads/2016/11/vsliceaudit_v1.3.pdf
- https://eprint.iacr.org/2016/1007.pdf
- https://github.com/Bunjin/Rouleth/blob/master/Security.md
- https://github.com/LeastAuthority/ethereum-analyses
- https://github.com/bokkypoobah/ParityMultisigRecoveryReconciliation
- https://medium.com/@ConsenSys/assert-guards-towards-automated-code-bounties-safe-smart-contract-coding-on-ethereum-8e74364b795c
- https://medium.com/@coriacetic/in-bits-we-trust-4e464b418f0b
- https://medium.com/@hrishiolickel/why-smart-contracts-fail-undiscovered-bugs-and-what-we-can-do-about-them-119aa2843007
- https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc
- https://medium.com/zeppelin-blog/zeppelin-framework-proposal-and-development-roadmap-fdfa9a3a32ab
- https://pdaian.com/blog/chasing-the-dao-attackers-wake
- http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf

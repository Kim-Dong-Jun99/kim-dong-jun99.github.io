---
layout: post
title: microservice pattern.07
date: 2024-10-15 21:28 +0900
description: microservice patterns 책을 통해 학습한 내용을 정리한 글입니다.
image: https://velog.velcdn.com/images/chancehee/post/80fefca2-c70a-4740-a434-0ca140002f4a/image.png
category: ["msa"]
tags:
published: true
sitemap: true
author: kim-dong-jun99
---

## Saga 고립성 

ACID에서 I는 고립성을 의미합니다.
고립성은 동시에 실행되는 여러 트랜잭션의 결과가 어떤 순서대로 실행한 결과가 같다는 것을 의미합니다.
데이터베이스는 각 ACID 트랜잭션이 데이터에 대해서 독점적으로 접근하는 것 같은 환상을 제공합니다.
고립성 성질은 동시에 실행되는 비즈니스 로직을 작성하기 한결 편하게 해줍니다.

saga를 사용하면서 해결해야하는 과제는 saga가 고립성을 성질을 제공하지 않는 다는 점입니다.
그 이유는 saga의 로컬 트랜잭션이 만드는 변경 사항들은 커밋된 이후에 다른 saga에게 보여지기 때문입니다.
이런 행동 방식은 2가지 문제를 발생시킵니다.
1. 다른 saga가 현재 실행 중인 saga의 데이터를 변경시킬 수 있습니다.
2. 다른 saga가 현재 아직 완료되지 않은 작업의 데이터를 읽어가, 데이터의 일관성을 깨뜨릴 수 있습니다.

이런 고립성의 부족은 잠재적으로 **변칙**을 만들어낼 수 있습니다. 변칙이란 트랜잭션이 한번에 하나만 실행됐을 때와 다른 데이터를 읽거나 쓰는 것을 의미합니다.
변칙이 발생하면, 트랜잭션을 동시에 실행했을 때의 결과와 순차적으로 실행했을 때의 결과가 다릅니다.

표면적으로 봤을 때, 고립성이 부족하면, 서비스가 정상적으로 실행되지 않을 것처럼 보여집니다.
하지만 실상은 개발자들은 고립성을 줄여서 성능을 끌어올립니다. 
관계형 데이터베이스는 트랜잭션 별로 고립성 레벨을 설정할 수 있습니다.
> [mysql 고립성 레벨](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
기본 고립성 수준은 serializable transaction으로 알려진 full isolation보단 약하게 설정됩니다.
실제 데이터베이스 트랜잭션은 교과서에서 정의된 ACID 트랜잭션과는 다릅니다.

### 변칙에 대하여

고립성의 부족은 3가지 변칙을 만들어냅니다.
- **Lost updates** : 한 saga가 다른 saga에 의해서 만들어진 변경 사항을 읽지 못하고, 덮어쓸 수 있습니다.
- **Dirty Reads** : 트랜잭션 혹은 saga는 아직 변경 완료되지 않은 업데이트 내용을 읽을 수 있습니다.
- **Fuzzy/nonrepeatable reads** : saga내에서 동일한 데이터를 2번 조회했을 때 서로 다른 결과가 리턴될 수 있습니다.

위 3가지 변칙 모두 발생할 수 있지만, 첫번째 두번째 변칙이 가장 자주 일어나고, 처리하기 까다롭습니다.

**Lost updates**

다음과 같은 시나리오에서 Lost updates이 발생 가능합니다.
1. 주문 생성 saga가 주문을 생성
2. saga 실행 중에 주문 취소 saga가 주문을 취소
3. 주문 생성 saga가 주문 생성을 승인

주문 생성이 주문 취소가 만든 업데이트를 무시하고 덮어쓰게됩니다. 결과적으로 애플리케이션은 취소된 주문을 처리하게 됩니다.

**Dirty Reads**

dirty read는 한 saga가 다른 saga가 변경하는 중에 있는 데이터를 읽을 때 발생합니다.
예를 들어, 고객의 신용카드에는 한도가 존재합니다.
이때 saga가 주문을 취소하면, 다음과 같은 트랜잭션들이 실행됩니다.
- 고객 서비스 : 사용가능한 신용을 증가시킴
- 주문 서비스 : 주문의 상태를 취소로 변경
- 배송 서비스 : 배송을 취소

이때 주문 취소 saga와 주문 생성 saga가 실행되는데, 주문 취소 saga는 배송을 취소하기 너무 늦어져서 롤백되는 경우, 고객 서비스를 호출하는 트랜잭션의 순서는 다음과 같을 수 있습니다.
1. 주문 취소 saga : 사용 가능한 신용을 증가
2. 주문 생성 saga : 사용 가능한 신용을 감소
3. 주문 취소 saga : 보상 트랜잭션으로 인해 신용을 다시 감소

이 경우에 주문 생성 saga는 사용 가능한 신용을 잘못 읽어 한도 초과로 실패해야하는 주문을 허용하게 될 수 있습니다.

### 고립성의 부족에 대응하기

saga 트랜잭션 모델이 고립성을 제공하지 않기에 개발자가 이로인한 변칙을 막게 saga를 사용해야 합니다.
이전 예시들에서 이미 이런 변칙을 대응하기 위한 전략을 사용했었습니다.
주문은 `*_PENDING`이라는 상태를 사용합니다. 이런 상태를 사용하는 것도 하나의 전략입니다.
주문을 업데이트하는 saga는 주문의 상태를 `*_PENDING`으로 설정하는 것에서부터 시작합니다.
`*_PENDING` 상태는 다른 트랜잭션에서 주문이 saga에 의해서 변경 중이라는 것을 알립니다.

주문이 `*_PENDING` 상태를 사용하는 것은 [semantic lock countermeasure](https://dl.acm.org/doi/10.5555/284472.284478)의 예시입니다.
위 링크에서는 분산 트랜잭션을 사용하지 않고 멀티 데이터베이스 구조에서 트랜잭션 고립성 부족에 대응하는 방법들을 소개합니다.
이 방법들 중 대다수는 saga를 디자인할 때 유용합니다.
위 링크에서 설명하는 방법들은 다음과 같습니다.
- Semantic lock : 애플리케이션 레벨 락
- Commutative updates : 업데이트 오퍼레이션이 어느 순서로도 동작 가능하게 설계
- Pessimistic view : 비즈니스 리스크를 줄이기 위해 saga의 단계를 재정렬
- Reread value : 데이터를 덮어쓰기 전에 데이터를 다시 읽어 변하지 않았음을 검증
- Version file : 레코드에 업데이트를 기록해서 재정렬될 수 있게함
- By value : 각 요청의 비즈니스 리스크를 이용해서 동적으로 동시성 메커니즘을 선택

위 방법들에 대해서 본격적으로 알아보기 전에 saga에 구조에 대해서 먼저 정리해보겠습니다.

**saga의 구조**

saga는 3가지의 트랜잭션으로 구성됩니다.
<img width="758" alt="Screenshot 2024-10-15 at 23 41 17" src="https://github.com/user-attachments/assets/55c6f3a1-6f58-4f61-82db-713f78686bee">
- compensatable transactions : 보상 트랜잭션으로 인해 롤백될 수 있는 트랜잭션
- pivot transaction : saga의 go/no-go 포인트입니다. 만약 pivot 트랜잭션이 커밋되면, saga는 이후에 롤백될 일이 없습니다. 
- retriable transactions : pivot transaction이후 실행되는 트랜잭션으로 성공이 보장받는 트랜잭션

**countermeasure : semantic lock**

semantic lock countermeasrue를 사용하면, saga의 보상 가능한 트랜잭션은 생성하거나 업데이트하는 레코드를 마킹합니다. 마킹된 레코드는 레코드가 커밋되지 않았고, 변경될 수 있음을 의미합니다.
마킹은 다른 트랜잭션의 접근을 막는 락일 수도 있고 다른 트랜잭션이 해당 레코드를 신중하게 처리해야 한다는 경고일 수도 있습니다.
마킹은 retriable transaction(saga가 성공적으로 종료됨을 의미) 혹은 compensating transaction(롤백됨을 의미)에 의해 해제됩니다.

주문의 `state` 필드는 semantic lock의 아주 적합한 예시입니다.
`*_PENDING` 상태는 semantic lock을 구현합니다.
이 상태를 이용해 어떤 saga가 주문을 업데이트하고 있음을 알릴 수 있습니다.
주문 생성 saga의 첫번째 단계는 주문을 `APPROVAL_PENDING` 상태로 생성하는 것입니다.
그리고 최종 단계는 이 필드를 `APPROVED`로 변경합니다. 보상 트랜잭션은 필드를 `REJECTED`로 변경합니다.

락을 사용하는 것은 문제를 절반으로 줄이기만 할 뿐입니다. 
잠긴 레코드를 saga가 어떻게 처리할지 경우에 맞게 결정해야합니다.
주문 취소 커맨드가 왔을 때, 취소할 요청이 `APPROVAL_PENDING`상태일 수 있습니다.

이 시나리오를 처리하는 몇가지 방법이 존재합니다.
한가지 방법은 주문 취소 커맨드를 실패시키고, 클라이언트로 하여금 재시도하게 합니다.
이것의 이점은 구현하기 간단하는 것입니다.
단점은 재시도 로직을 구현해야 하기에 클라이언트의 복잡성이 증가합니다.

또 다른 방법은 락이 해제될 때까지 block하는 것입니다. 
semantic lock의 이점은 ACID 트랜잭션이 제공하는 고립성을 근본적으로 제공하는 것입니다.
같은 레코드를 업데이트하는 saga들은 직렬화됩니다.
이것의 또 다른 이점은, 클라이언트로 하여금 재시도의 책임을 없앱니다.
단점은 애플리케이션이 락을 관리해야합니다.
또한 데드락을 깨고 재시도하기 위한 데드락 감지 알고리즘도 구현해야 합니다.

**countermeasure : commutative updates**

한 가지 간단한 대응책은 업데이트 연산을 교환법칙이 성립하도록 설계하는 것입니다. 연산이 교환법칙을 따를 때, 어떤 순서로 실행해도 결과가 동일합니다. 예를 들어, 계좌의 `debit()`(출금)와 `credit()`(입금) 연산은 (초과 인출 검사를 무시하면) 교환 가능하다고 할 수 있습니다. 이 대응책은 잃어버린 업데이트(lost updates)를 방지하는 데 유용합니다.

예를 들어, 사가(Saga)가 출금 또는 입금과 같은 보상 가능한 트랜잭션 이후 롤백되어야 하는 상황을 생각해보세요. 보상 트랜잭션은 단순히 출금을 취소하기 위해 입금을 하거나, 입금을 취소하기 위해 출금을 실행하여 업데이트를 되돌릴 수 있습니다. 이렇게 하면 다른 사가에 의해 수행된 업데이트를 덮어쓸 가능성이 없습니다.

**countermeasrue : pessimistic view**

이 방법은 saga의 단계를 재정렬하여 dirty read로 인한 비즈니스 리스크를 최소화합니다.
앞선 시나리오에서 주문 생성 saga는 고객 신용의 한도가 초과될 수 있는 순서로 동작했습니다.
주문 취소 saga의 실행 순서를 다음과 같이 변경합니다.
1. 주문 서비스 : 주문을 취소 상태로 변경
2. 배송 서비스 : 배송을 취소
3. 고객 서비스 : 신용을 증가

앞선 시나리오에서 배송 서비스에서 주문 취소가 롤백되었기에, 위 순서로 동작하면 dirty read가 발생하지 않습니다.

**countermeasure : reread value**

reread value는 lost update을 방지합니다.
이 방법을 사용하면, 업데이트하기 전 레코드를 다시 읽어 변경되지 않았음을 검증합니다.
만약 변경되었다면, saga를 종료시키거나, 재시작할 수 있습니다.

주문 생성 saga는 이 방법을 이용해서 주문이 승인되는 중에 취소되는 것을 방지할 수 있습니다.
주문 생성 saga는 주문이 변경되지 않았으면 주문을 승인하고 만약 취소되었으면, saga를 종료하고, 보상 트랜잭션을 실행해 선행된 결과를 처리합니다.

**countermeasure : version file**

레코드에 실행된 오퍼레이션을 기록해서 재정렬할 수 있게하기에 version file이라고 불립니다.
교환법칙이 성립되지 않는 오퍼레이션을 교환가능하게 만드는 방법입니다.
주문 생성 saga가 주문 취소 saga와 동시에 실행되는 경우에, saga가 semantic lock를 사용하지 않는 한, 주문 취소 saga가 주문 생성 saga가 카드를 승인하기 전에 먼저 취소하게 될 수도 있습니다.

이를 `Accounting Serivce`가 재정렬해서 올바른 순서로 실행하게 할 수 있습니다.
이 시나리오에서, 먼저 취소 승인 요청이 오고, 카드 승인 요청이 오게됩니다. 그러면 취소 승인 요청이 먼저 왔기에, 이후에 온 카드 승인 요청을 넘어갑니다.

**countermeasure : by value**

각 요청의 프로퍼티를 확인하고, saga 혹은 분산 트랜잭션 중 사용 방식을 결정합니다.
로우 리스크 요청은 saga를 사용합니다. 하이 리스크 요청은 분산 트랜잭션을 사용합니다.

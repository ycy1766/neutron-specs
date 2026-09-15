# Network Log의 OVN ACL flow sampling

이 문서는 [영문 spec](../../specs/2027.1/ovn-acl-flow-sampling.rst)을 검토하기 위한
한글 번역본이다. 같은 브랜치의 영문 초안을 기준으로 하며, 설계 변경 시 영문과 함께
갱신한다. 초안의 제안 사항을 설명하는 문서로, 세부 설계가 upstream에서 확정되었다는
뜻은 아니다.

원문과 동일하게 [Creative Commons Attribution 3.0 Unported](http://creativecommons.org/licenses/by/3.0/legalcode)
라이선스를 따른다.

<https://bugs.launchpad.net/neutron/+bug/2165306>

ML2/OVN logging driver는 ACL logging을 사용해 패킷 이벤트를 ovn-controller로
전달한다. 이 제안은 Network Log에 출력 방식을 추가해 운영자가 OVN ACL sampling을
선택할 수 있게 한다. Neutron은 선택한 ACL에 연결된 sampling 리소스를 관리한다.
호스트의 exporter와 샘플을 수신·처리하는 시스템은 운영자가 구성한다.

## 문제 설명 (Problem Description)

OVN은 ACL에 일치하는 패킷을 sampling할 수 있으며, 이미 수립된 연결의 트래픽도
대상에 포함한다. 운영자가 OVN Northbound 데이터베이스에 직접 설정을 연결할 수는
있지만, Neutron은 그 설정을 유지하지 않는다. 예를 들어 security group의
statefulness를 변경하면 해당 규칙의 ACL이 다시 생성되면서 sampling 참조가
유실된다. Network Log를 삭제하거나 비활성화해도 해당 ACL에 수동으로 구성한
sampling은 제거되지 않는다.

Network Log에는 운영자가 관찰하려는 security group과 이벤트가 이미 기록되어 있다.
이 리소스를 확장하면 OVN driver가 해당 ACL과 함께 sampling을 유지할 수 있다.
운영자는 한 security group에는 sampling을 활성화하고 다른 security group에는
controller logging을 유지하며, 어느 요청이든 Network Log API로 제거할 수 있어야 한다.

제안 범위는 ML2/OVN의 security group ACL sampling이다. VPC flow-record API,
집계 주기, 저장 서비스를 정의하지 않는다. 샘플은 패킷을 관찰한 결과이며, 완전한
연결 기록이나 정확한 패킷·바이트 카운터가 아니다.

## 제안하는 변경 (Proposed Change)

### Network Log API

새 `logging-output-type` extension이 Log 리소스에 `output_type`을 추가한다.
값은 `packet_log`와 `flow_sample`이다. 기본값인 `packet_log`는 기존 Log와
필드를 생략한 요청에서 backend의 현재 logging 동작을 유지한다.

이 속성은 생성 요청에서 지정할 수 있고 show와 list 응답에 반환된다. 값이 같은
항목을 찾는 필터를 지원하며, 생성 후에는 변경할 수 없다. 출력 방식을 바꾸려면
새 Log를 만들고 기존 Log를 제거해야 한다. 기존 `enabled` 속성은 두 출력 방식
모두에 적용된다. 하나의 Log는 출력 방식 하나를 선택한다. 같은 ACL에 두 출력을
모두 요청하려면 별도의 Log를 사용한다.

예를 들어 운영자는 다음과 같이 sampling을 요청할 수 있다. 예제의 resource UUID는
이미 존재하는 security group을 가리킨다.

```http
POST /v2.0/log/logs
```

```json
{
    "log": {
        "name": "web-samples",
        "resource_type": "security_group",
        "resource_id": "d80f19a6-5404-4c2d-953d-7a8910d8ac60",
        "event": "ALL",
        "output_type": "flow_sample",
        "enabled": true
    }
}
```

같은 요청에서 `output_type`을 생략하면 `packet_log`가 선택된다. null, 빈 값,
알 수 없는 출력 값은 HTTP 400으로 거부한다. 기존 Log의 출력 방식을 변경하려는
요청도 HTTP 400으로 거부한다.

데이터베이스에는 expand migration으로 출력 컬럼을 추가하고 서버 기본값을
`packet_log`로 지정한다. Log versioned object에도 필드를 추가하며, 이전 버전의
객체를 읽을 때 이 기본값을 유지한다. flow-sample Log를 이전 버전의 객체로 변환할
때 출력 필드를 버려서는 안 된다. 이전 consumer가 이를 packet logging 요청으로
해석할 수 있기 때문이다.

### 초기 범위와 정책 (Initial scope and policy)

OVN security-group ACL은 Port Group에 적용된다. 대상 포트에 연결된 security group을
조회하는 것만으로 해당 ACL의 적용 범위가 그 포트로 제한되지는 않는다. 공유 security
group에는 여러 프로젝트의 포트가 포함될 수도 있다.

`flow_sample` 요청에는 `resource_type=security_group`과 명시적인 `resource_id`가
필요하다. `target_id`가 null이 아니거나 `resource_id`를 생략한 요청은 HTTP 400으로
거부한다. Sampling은 security group의 Port Group 전체를 대상으로 하며, 나중에
연결되는 포트도 포함한다. 더 좁은 선택 범위를 구현하기 위해 포트별 ACL 복사본을
추가하거나 공유 default-drop 구조를 변경하지 않는다.

기존 Network Log 정책은 administrator와 project manager를 허용한다. 새 출력
방식은 flow-sample Log 생성과 재활성화 시 관리자 권한 정책 검사를 추가로 요구한다.
공유 SG를 sampling하면 다른 프로젝트의 트래픽까지 포함할 수 있기 때문이다.
Packet-log 권한의 기존 기본값은 유지한다. 기존 Log의 조회·비활성화·삭제 정책
검사도 계속 적용하며, 이 작업을 수행하기 위해 배포 환경에서 sampling이 활성화되어
있을 필요는 없다.

추후 project manager의 요청이나 개별 포트를 지원하려면 SG 소속과 공유 상태가
바뀌어도 sampling이 요청한 범위 안에서 이루어져야 한다. Log 생성 시 SG의
프로젝트만 확인하는 것으로는 충분하지 않다.

### 배포 요구 사항과 기능 지원 정보 (Deployment requirements and capability)

Extension alias는 API가 새 필드를 이해한다는 것을 나타낸다. 기존
loggable-resources 응답은 리소스 유형별 지원 출력 방식을 설명한다. Sampling을
활성화한 배포 환경에서는 다음 내용을 포함한다.

```json
{
    "loggable_resources": [
        {
            "type": "security_group",
            "output_types": ["packet_log", "flow_sample"]
        }
    ]
}
```

Sampling은 기본적으로 비활성화한다. `flow_sample`을 지원한다고 알리려면 운영자의
명시적인 활성화, OVN logging driver, 필요한 Northbound sampling 테이블과 ACL
컬럼 지원이 있어야 한다. 또한 참여하는 모든 chassis에서 필요한 sampling 및
ACL-label 동작을 지원하는 OVN/OVS 조합을 사용해야 한다. 스키마 검사만으로는 이
dataplane 호환성을 확인할 수 없다.

초기 구현은 OVN 이외의 logging driver가 함께 로드되어 있으면 flow sampling을
지원한다고 알리지 않는다. Driver dispatch는 출력 방식을 구분하여 flow-sample
Log가 packet-only driver나 해당 agent RPC 경로로 전달되지 않도록 한다. 기존
packet-log dispatch는 유지한다.

지원하지 않는 출력 요청은 HTTP 400으로 실패한다. `enabled=false`로 생성하는
Log도 검증하며, 재활성화하기 전에 다시 검증한다. Northbound 데이터베이스에서
소유권이나 식별자 충돌을 발견하면 HTTP 409로 보고한다. 데이터베이스 연결 실패는
지원하지 않는 출력으로 보고하지 않고 logging service의 driver-error 처리
방식을 따른다.

기능 지원 정보 조회는 exporter나 collector의 정상 동작 여부를 보고하지 않는다.
운영자는 관련 bridge에 OVS `Flow_Sample_Collector_Set`과 연결된 exporter를 구성하고
샘플 전달 여부를 별도로 확인한다. Flow-based IPFIX는 sample action에 담긴 확률을
사용한다. OVS의 `IPFIX.sampling` 설정은 bridge sampling에 적용되며, 이 경로에
추가로 적용되는 sampling 비율이 아니다.

### Sampling 리소스와 소유권 (Sampling resources and ownership)

Sampling이 활성화되면 Neutron은 `acl-new`와 `acl-est`의 `Sampling_App` row와
전용 `Sample_Collector` 하나를 관리한다. 초기 collector는 두 단계와 선택된 모든
ACL에 동일한 확률을 사용한다. Log별 확률과 exporter 목적지 설정은 이번 변경
범위에 포함하지 않는다.

운영자는 서로 다른 application ID 두 개, collector ID, collector-set ID, 확률을
지정한다. Application ID와 collector ID의 범위는 1부터 255까지다. Collector-set
ID는 0이 아닌 32비트 값이다. 확률 값의 범위는 1부터 65535까지다. 활성화 플래그의
기본값은 false다. 식별자에는 클러스터 전체 리소스를 암묵적으로 점유하는 기본값을
두지 않는다.

각 application type에는 Sampling_App row가 하나만 존재할 수 있다. 다른 관리
주체가 소유한 기존 row는 현재 값이 요청한 설정과 같더라도 인수하지 않고 거부한다.
Application ID와 collector ID의 충돌도 검사한다. Collector 이름은 고유 키가
아니므로 소유한 row를 식별할 때 이름만 사용하지 않는다. 일반 `drop` sampling
application은 drop ACL에서 패킷을 sampling하는 것과 별개이며, 이 기능으로
관리하지 않는다.

Application과 collector row에는 Neutron 소유권 메타데이터를 기록한다. 기능이
활성화되어 있는 동안에는 flow-sample Log가 일시적으로 하나도 없더라도 배포 환경의
리소스로 유지한다. 마지막 Log를 제거해도 이 row들을 삭제하지 않는다. 기능을 완전히
철거할 때는 Neutron이 소유하고 다른 사용자가 더 이상 참조하지 않는 row만 제거한다.
Neutron은 호스트 로컬 OVS exporter 설정을 수정하지 않는다.

선택된 stateful `allow-related` ACL은 `sample_new`와 `sample_est` 모두를 통해
Sample을 참조한다. Stateless 허용 ACL과 drop ACL은 `sample_new`를 사용한다.
Log에 `event=ALL`을 설정해도 이 ACL들에 established-connection 이벤트가 추가되지는
않는다. 같은 ACL을 선택한 Log들은 해당 ACL의 sampling 설정을 공유한다. Sample
생성과 ACL 연결은 하나의 Northbound 트랜잭션에서 수행한다. 연결을 제거한 뒤 다른
참조가 없는 Sample은 OVSDB의 strong-reference garbage collection으로 제거할 수 있다.

Sample에는 `external_ids` 컬럼이 없으므로 해당 row에 소유권을 기록할 수 없다.
Neutron은 ACL에 연결 소유권과 예상 참조를 기록한다. 참조된 Sample을 변경하거나
연결을 제거하기 전에 Sample의 metadata와 collector를 검증한다. 외부 관리 주체의
참조이거나 소유권 기록과 실제 참조가 다르면 Neutron은 충돌을 보고하고 참조를
그대로 둔다. 검사와 갱신은 같은 IDL 트랜잭션에서 수행하고, 동시 변경이 발생하면
재시도해야 한다.

### ACL 선택과 중복 Log (ACL selection and overlapping Logs)

`ACCEPT`는 SG의 허용 ACL을 선택한다. `DROP`은 OVN logging driver가 관리하는
SG별 drop ACL을 사용하며, `ALL`은 둘 다 선택한다. 해당 drop ACL의 기존 priority와
match는 유지한다. 두 출력 중 어느 하나라도 필요로 하면 해당 ACL을 유지한다.
Flow-sample만 요청했을 때 drop ACL 생성의 부수 효과로 controller logging이
활성화되어서는 안 된다.

Drop 관찰 결과는 선택된 drop ACL을 식별한다. Security group에는 허용 규칙과
암묵적인 기본 거부가 있으므로, 이 결과가 명시적인 SG deny rule을 식별하지는 않는다.
포트가 여러 SG에 속할 때 그중 하나의 drop ACL에서 나온 관찰 결과를 해당 SG만이
거부 원인이라는 증거로 제시해서는 안 된다.

Driver는 각 ACL에 대해 출력 방식별로 활성화된 Log의 합집합을 계산한다. Packet
Log A와 sampling Log B가 있으면 두 출력이 모두 존재한다. B를 삭제하면 sampling
연결을 제거하고 A의 packet logging은 유지한다. Sampling Log B와 C가 있을 때
B를 삭제하면 C를 위해 sampling 연결을 유지한다. 해당 출력을 필요로 하는 활성
Log가 더 이상 없을 때만 출력을 제거한다.

이 계산에는 기존 packet logging 필드와 SG별 drop ACL의 생명주기도 포함한다.
Log 하나를 삭제할 때 모든 logging 필드를 독립적으로 지우는 방식으로 구현할 수는
없다. Sampling 경로에서도 ACL label을 사용하기 때문이다. Sampling을 추가하거나
제거할 때 reconciliation은 packet logging의 related-traffic 동작을 보존한다.

### 관찰 식별자 (Observation identity)

Sample metadata는 0이 아닌 32비트 식별자이며 Northbound의 고유성 제약을 따른다.
Neutron은 충돌 검사와 제한된 횟수의 재시도를 통해 이 값을 할당하고, 같은 소유
연결이 유효한 동안 재사용한다. Reconciliation을 반복하거나 neutron-server를
재시작해도 변경되지 않은 연결에 대해 새 Sample을 할당하지 않는다.

0이 아닌 ACL label은 내보내는 observation point를 결정할 때 Sample metadata보다
우선할 수 있다. 기존 packet logging이 이 label을 사용하므로 sampling을 동작시키기
위해 label을 지우지 않는다. 지원하는 dataplane 조합은 단일 collector와 register
기반 sampling을 사용하는 경우를 포함해 packet logging과 sampling이 공존할 때도
sampling을 유지해야 한다.

Observation domain에는 logical datapath 식별자도 포함된다. OVN 처리 경로와
ACL label 유무에 따라 domain의 application 부분은 new와 established 관찰을
구분할 수도 있고 0일 수도 있다. Consumer는 application ID가 항상 이 두 단계를
구분한다고 가정할 수 없다. 운영자는 내보낸 식별자를 해석하기 위해 해당 OVN의
매핑 정보가 필요하다.

이 식별자들은 Log UUID나 영구적인 SG-rule 식별자가 아니다. Packet logging을
추가하거나 제거하면 ACL label을 통해 실제 observation point가 바뀔 수 있다.
ACL 교체나 Northbound 데이터베이스 복원도 매핑을 바꿀 수 있다. 기존 연결은 설정이
변경된 뒤에도 이전 conntrack metadata를 유지할 수 있다. 이 기능은 모든 기존
연결의 즉각적인 재매핑이나 과거 트래픽의 소급 sampling을 보장하지 않는다.

### 생명주기와 복구 (Lifecycle and recovery)

SQL Log는 요청한 상태를 기록한다. SQL Log 갱신과 OVN 설정은 별도의 트랜잭션이다.
Postcommit driver 실패가 발생하면 Log는 저장되었지만 출력은 아직 적용되지 않은
상태가 될 수 있다. Client는 실패한 생성 요청을 재시도하기 전에 list와 show로
확인할 수 있다. 이번 변경은 전달 상태 리소스를 추가하지 않으며, API 성공이 외부
collector의 수신을 확인해 준다는 의미도 아니다.

Log 작업과 SG ACL 생성은 같은 출력 상태 계산을 사용한다. Driver는 SG 규칙의
생성·교체 시 sampling을 다시 적용하며, statefulness 변경에 따른 교체도 포함한다.
기존 공유 ACL이 관찰하는 트래픽은 Port Group 소속으로 결정된다.

OVN maintenance lock 아래에서 실행하는 주기 작업이 SQL에 기록된 요청 상태와
Neutron 소유의 Northbound 상태를 일치시킨다. 활성 Log와 함께 비활성화·삭제된
Log가 남긴 오래된 연결도 검사한다. 현재 존재하는 SQL row만 찾으면 삭제 후 정리
실패를 놓치게 된다. Log 리소스는 기존 OVN resource-revision maintenance 대상이
아니므로 이 reconciliation을 명시적으로 추가해야 한다.

충돌이나 동시 변경이 발생한 뒤에는 조건부 Northbound 갱신을 재시도하기 전에
현재 Log를 다시 읽는다. 재연결이나 maintenance leader 변경 시에도 저장된 상태를
재평가한다. 소유권 충돌은 해당 리소스와 함께 로그로 남겨 운영자가 해결하도록 한다.

데이터베이스 동기화는 다시 생성한 SG-rule ACL에 sampling을 복원한다. SG별
logging drop ACL도 명시적으로 reconcile해야 한다. 현재의 rule-ACL 비교 로직은
이 ACL들을 다시 만들지 않기 때문이다. 요청 상태가 같다면 어느 복구 경로를
반복하더라도 연결과 metadata 값을 유지해야 한다.

### 업그레이드와 비활성화 (Upgrade and disable)

운영자는 관련된 모든 API worker와 service가 새 Log object와 출력 필드를 이해한
뒤에 flow sampling을 활성화한다. 기본 비활성화 설정은 초기 rolling upgrade
중에 이전 worker가 flow-sample 요청 상태를 접하지 않도록 한다. 기능 지원을
알리기 전에 worker 간 설정이 일치해야 한다.

배포 옵션을 비활성화하면 새로운 flow-sample 요청과 활성화 작업을 거부한다. 기존
Log의 조회·비활성화·삭제는 계속 가능하다. 필요한 스키마를 갖춘 Northbound에
접근할 수 있으면 reconciliation이 저장된 Log 요청 상태를 유지하면서 Neutron
소유의 sampling 참조를 분리한다. 옵션을 다시 활성화하면 활성 상태의 Log를 다시
적용한다. Northbound에 접근할 수 없다면 옵션을 비활성화해도 sampling의 즉각적인
중단을 보장할 수 없다. Downgrade 전에 운영자는 기능을 비활성화하고 연결이
제거되었는지 확인해야 한다.

### 대안 (Alternatives)

배포 환경 전체에 적용하는 스위치는 API 변경을 줄일 수 있지만, 운영자가 개별
Log마다 다른 출력을 선택할 수 없다. Packet logging을 완전히 대체하면 명시적인
요청 없이 기존 배포 환경의 동작을 바꾸게 된다.

외부 도구가 Sample row와 ACL 연결을 계속 관리할 수도 있다. 다만 Neutron의 SG와
ACL 생명주기도 추적해야 한다. Logging driver가 연결을 관리하면 Log API와 같은
요청 상태를 사용할 수 있다. Exporter 구성과 샘플 처리는 계속 외부에서 담당한다.

운영자가 제공하는 전역 sampling row를 재사용하면 acl-new와 acl-est application을
다른 관리 주체가 소유한 배포 환경도 지원할 수 있다. 하지만 설정과 복구 책임이
여러 관리 주체로 나뉜다. 초기 구현은 해당 application row의 독점 소유권을 요구하며,
기존 외부 소유자가 있으면 배포 환경의 충돌로 보고한다.

## 구현 (Implementation)

구현 작업은 다음과 같다.

- neutron-lib API extension과 Log 데이터베이스 및 object 변경.
- Logging service의 출력 검증과 dispatch.
- OVN logging driver의 sampling 리소스 및 ACL 연결 관리.
- Sampling을 복원하는 maintenance와 데이터베이스 동기화 지원.

조건부 연결 관리에 필요한 ovsdbapp command에는 트랜잭션 테스트를 포함한다.
기능 지원을 알리기 전에 dataplane 호환성, 특히 ACL label 공존을 확인해야 한다.

기존 `openstack network log` 명령은 python-neutronclient OSC plugin이 제공한다.
Create에 `--output-type`을 추가하고 show, list, loggable-resources 출력에 새
필드를 표시한다. 새로운 독립 client 명령이나 SDK resource migration은 필요하지 않다.

## 테스트 (Testing)

API와 object 테스트는 출력 생략, 잘못된 값, 변경 불가 속성, 비활성 상태 생성,
재활성화, 정책, 기능 지원 정보, 기존 데이터베이스 row, 이전 버전 객체의 변환을
검증한다. Driver 테스트는 flow-sample 요청 상태가 packet-only driver나 해당
RPC 경로로 전달되지 않는지 확인한다.

OVN 기능 테스트는 Neutron 및 외부 소유의 전역 row, 식별자 충돌, 외부 ACL 참조,
공유 연결, 두 출력의 삭제 순서를 모두 다룬다. Flow-sample 전용 SG drop ACL이
controller logging을 활성화하지 않는지도 확인한다. 복구 테스트는 postcommit
실패, 동시 Log 변경, 재시작, ACL 교체, 데이터베이스 동기화, Northbound에 접근할
수 없을 때의 비활성화를 다룬다.

Dataplane 테스트는 new, established, reply 트래픽, stateless 규칙, DROP,
양쪽 ACL 방향을 다룬다. 단일 collector에서 0이 아닌 label을 사용하는 경우,
sampling 전후에 packet logging을 설정하는 경우, 출력 변경 전부터 존재하던 연결을
반드시 포함한다. 기대 결과에는 설정한 exporter로 샘플이 도달하는 것뿐 아니라
패킷 필터링과 packet-log 동작이 그대로 유지되는 것도 포함한다. 별도의 전달
테스트로 flow-based IPFIX 수신을 확인한다. Collector가 없어도 ACL의 허용·차단
결정은 바뀌어서는 안 된다.

## 문서 영향 (Documentation Impact)

API 문서는 출력 속성, 초기 selector와 정책 제한, 기능 지원 응답을 설명한다.
Logging guide는 명시적 활성화 설정, 지원하는 OVN/OVS 조합, 소유권 충돌,
운영자가 관리하는 exporter 구성, 비활성화와 downgrade 순서를 다룬다.
샘플 식별자와 통계적 관찰 결과를 Log별 기록, 정확한 카운터, collector 전달 보장과
구분해 설명한다.

## 참고 자료 (References)

- RFE: <https://bugs.launchpad.net/neutron/+bug/2165306>
- Drivers 논의:
  <https://meetings.opendev.org/meetings/neutron_drivers/2026/neutron_drivers.2026-09-11-13.00.log.html>
- OVN ACL sampling 설정 (ACL, Sample, Sample_Collector, Sampling_App):
  <https://www.ovn.org/support/dist-docs/ovn-nb.5.html>
- OVN ACL sampling의 logical flow:
  <https://docs.ovn.org/en/latest/ref/ovn-logical-flows.7.html>
- OVN sample action과 관찰 식별자:
  <https://www.ovn.org/support/dist-docs/ovn-sb.5.html>
- 호스트 IPFIX exporter 설정 (OVS IPFIX와 Flow_Sample_Collector_Set):
  <https://www.openvswitch.org/support/dist-docs/ovs-vswitchd.conf.db.5.html>
- 관련 flow-log RFE: <https://bugs.launchpad.net/neutron/+bug/2071323>

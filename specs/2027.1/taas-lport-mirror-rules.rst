..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

..
 한글 검토본이다. 업스트림(Gerrit)에는 영문본만 제출한다.
 영문 원본: 같은 경로의 wip/taas-lport-mirror-rules-spec-en 브랜치.

=========================================================
Tap-as-a-Service: lport 미러 필터링 규칙 (한글 검토본)
=========================================================

https://bugs.launchpad.net/neutron/+bug/2168008

tap mirror는 미러 대상 포트가 보내고 받는 트래픽을 전부 복제한다. 이 spec은
``lport`` 타입 tap mirror에 필터링 규칙을 추가해, 사용자가 수집기로 복제할
트래픽을 소스에서 고를 수 있게 한다. 규칙은 tap mirror의 새 하위 리소스
``rules``\ 이고, 보안 그룹 규칙과 같은 용어를 쓰며, OVN ``Mirror_Rule``
테이블(OVN 25.09 이상)로 구현한다. ML2/OVN 전용 기능이다.

``lport`` 미러 타입 자체는 RFE 2168007이 다루며, 2026-10-02 드라이버
회의에서 spec 없이 승인됐다 [1]_. 같은 회의에서 규칙은 새 API와 DB
테이블을 추가하므로 이 spec을 요청받았고, 이 spec이 답해야 할 세 가지가
제기됐다. ``tap_mirrors`` 리소스에 일부 미러 타입에만 해당하는 속성이
생겼다는 점, TaaS API가 tap service/flow(ML2/OVS)와 tap mirror(ML2/OVN)를
섞어 쓰고 있다는 점, 그리고 제대로 된 테스트가 필요하다는 점이다.


Problem Description
===================

(문제 설명)

``lport`` 미러 타입을 쓰면 프로젝트는 자기 포트의 트래픽 사본을 오버레이
안에 있는 수집 포트(IDS, 패킷 브로커)로 보낼 수 있다. 지금은 이 미러가 항상
전부를 복제한다. 보통 수집기에 필요한 양보다 훨씬 많다.

* 사본 때문에 미러 대상 포트의 트래픽이 오버레이와 수집 포트에서 두 배가
  된다.
* 여러 포트가 한 수집기로 미러되는 경우가 많아, 수집기가 받은 것의 대부분을
  버려야 한다.
* 보안 팀은 "인바운드 TCP/443과 DNS만", "10.0.9.0/24로 가는 백업 트래픽만
  제외"처럼 선택 조건을 가장 싼 곳, 즉 소스에서 표현하고 싶어 한다.

사용 사례:

* 프로젝트 구성원이 웹 서버 포트를 미러하되 수집기에서는 인바운드 TCP/443만
  보고 싶다.
* 프로젝트 구성원이 DB 포트를 미러하되 특정 서브넷으로 가는 복제 트래픽은
  빼고 싶다.
* 운영자가 테넌트 포트를 서비스 프로젝트의 수집기로 미러하고(lport 미러와
  같이 admin 전용), 인터넷 대면 프리픽스와 주고받는 트래픽만 보고 싶다.

OVN은 이미 메커니즘을 제공한다. 25.09부터 ``lport`` 타입 ``Mirror`` 행이
``Mirror_Rule`` 행(priority, match, action ``mirror``/``skip``)을 참조할 수
있다. Neutron에는 이를 위한 API가 없다.

배경: lport tap mirror (RFE 2168007)
------------------------------------

이 spec은 RFE 2168007의 ``tap-mirror-lport`` 확장 위에 쌓인다. 독자 편의를
위해 그 확장이 하는 일을 요약한다.

* ``mirror_type``\ 에 ``lport`` 값이 생기고, ``remote_port_id``\ 가 사본을
  받을 Neutron 포트를 가리킨다. ``remote_ip``\ 와 ``directions``\ 의 터널
  ID는 이 타입에서 쓰이지 않는다.
* 싱크 포트는 존재해야 하고, 호스트에 바인딩돼 있어야 하며, 미러 대상 포트와
  달라야 하고, 호출자가 admin이 아니면 같은 프로젝트여야 한다. 두 포트 중
  하나를 삭제하면 미러도 삭제된다.
* 포트당 방향별 lport 미러는 하나만 둘 수 있다. OVN이 미러·방향마다 소스
  포트에 무조건 "전부 복제" flow를 하나씩 설치하기 때문이다.
* OVN 드라이버는 복제 방향마다 OVN ``Mirror`` 행을 하나씩 만든다.
  ``IN``\ (포트로 전달되는 트래픽)은 ``tm_in_<id>``\ 에 filter ``to-lport``,
  ``OUT``\ (포트가 보내는 트래픽)은 ``tm_out_<id>``\ 에 filter
  ``from-lport``\ 다.

OVN이 lport 미러와 규칙을 구현하는 방식
---------------------------------------

아래 제안하는 동작은 ``ovn-northd`` [3]_ 에서 그대로 따라 나오므로 여기
요약한다. 논리 스위치 포트에 붙은 lport ``Mirror`` 하나마다 northd는:

* 미러 대상 포트가 속한 스위치에 싱크 포트로 전달하는 미러 포트
  ``mp-<datapath>-<sink>``\ 를 만든다. 따라서 싱크는 다른 논리 스위치에
  있어도 된다.
* ``ls_in_mirror``\ (포트가 보내는 패킷) 그리고/또는
  ``ls_out_mirror``\ (포트로 전달되는 패킷)에 해당 포트를 매치하고
  ``mirror(<mp>); next;``\ 를 실행하는 "pass" flow를 priority 100으로
  설치한다.
* ``Mirror_Rule`` 하나마다 같은 단계에 ``100 + priority`` priority로
  ``<port> && (<규칙 match>)``\ 를 매치하는 flow를 설치한다. action은
  ``mirror``\ 면 ``mirror(<mp>); next;``, ``skip``\ 이면 ``next;``\ 다.
* ``ls_out_pre_acl``\ 에 출력 포트가 미러 포트인 패킷을 곧바로
  ``ls_out_apply_port_sec``\ 로 보내는 priority 65535 flow를 설치한다.
  그래서 사본은 수집기로 가는 길에 ACL(보안 그룹)·QoS·stateful 단계를
  건너뛴다.

``ls_in_mirror``\ 는 인그레스 파이프라인의 세 번째 단계로 ACL 단계보다
앞이고, ``ls_out_mirror``\ 는 이그레스 ACL 단계 바로 뒤다. 따라서 포트가
보내는 트래픽은 보안 그룹 평가 전에 복제되고, 포트로 전달되는 트래픽은
보안 그룹 평가 후에(실제로 받는 것만) 복제된다. ``Mirror_Rule``\ 은 논리
flow 표현식 언어의 ``match``, ``action``\ (``mirror``/``skip``), 0-32767
범위의 ``priority``\ 를 가지며, OVN은 규칙을 priority와 match로 식별한다.

``gre``\ 와 ``erspanv1`` 미러 타입에는 규칙을 제안하지 않는다. OVN에서 이
둘은 ``ovn-controller``\ 가 설정하는 일반 OVS 포트 미러라 논리 파이프라인을
거치지 않으므로, match를 적용할 논리 flow 자체가 없다.


Proposed Change
===============

(제안 변경)

개요
----

새 API 확장 ``tap-mirror-rules``\ (``tap-mirror``\ 와 ``tap-mirror-lport``
필요)가 tap mirror에 ``rules`` 하위 리소스를 추가한다::

  POST   /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  GET    /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  GET    /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules/{rule_id}
  DELETE /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules/{rule_id}

하위 리소스는 QoS policy rule 모델(neutron-lib 하위 리소스 정의, 부모가
있는 컨트롤러, 규칙별 정책)을 따른다. 규칙은 불변이다. 바꾸려면 지우고
새로 만든다. 그래야 OVN 행과 일대일 매핑이 유지되고, 이미 데이터플레인에
적용된 규칙이 부분 갱신되는 일이 없다. tap mirror의 ``directions``\ 도
불변이므로, 규칙의 방향이 만든 뒤에 무효가 되는 일은 없다.

미러 타입별 속성
----------------

RFE 검토에서 ``tap_mirrors``\ 에 일부 미러 타입에만 유효한 속성이 생겼다는
지적이 있었다 [1]_. API는 이를 명시하고, 해당하지 않는 속성은 무시하는
대신 거부한다.

==========================  =======================  ======================
속성                        ``gre`` / ``erspanv1``   ``lport``
==========================  =======================  ======================
``port_id``                 필수                     필수
``remote_ip``               필수                     거부 (``400``)
``directions`` 터널 ID      필수, 방향마다 하나      ``null``\ 이어야 함
``remote_port_id``          거부 (``400``)           필수
``rules`` 하위 리소스       거부 (``400``)           지원
==========================  =======================  ======================

RFE 2168007의 lport 구현은 지금 lport 미러에 준 ``remote_ip``\ 나 터널
ID를 무시한다. 그 변경에서 거부하도록 바꿔, 쓰이지 않는 값을 쓰인다고
믿는 일이 없게 한다. API 레퍼런스와 사용자 가이드에 같은 표를 싣는다.

규칙 속성
---------

==================================  ==========  =============================
속성                                기본값      비고
==================================  ==========  =============================
``priority``                        필수        1-32767, 높을수록 먼저 평가
``action``                          ``mirror``  ``mirror`` 또는 ``skip``
``direction``                       ``null``    ``IN``, ``OUT`` 또는
                                                ``null``\ (미러가 복제하는
                                                모든 방향)
``ethertype``                       ``null``    ``IPv4``, ``IPv6`` 또는
                                                ``null``\ (모든 프레임)
``protocol``                        ``null``    ``tcp``, ``udp``, ``sctp``,
                                                ``icmp``, ``ipv6-icmp``
``source_ip_prefix``                ``null``    CIDR
``destination_ip_prefix``           ``null``    CIDR
``source_port_range_min/max``       ``null``    1-65535, TCP/UDP/SCTP만
``destination_port_range_min/max``  ``null``    1-65535, TCP/UDP/SCTP만
==================================  ==========  =============================

속성 이름과 값은 보안 그룹 규칙과 같고, 규칙의 목적에서 나오는 차이가 둘
있다. ``priority``\ 와 ``action``\ 이 있고, ``ethertype``\ 이 선택이다.
보안 그룹 규칙은 IP 트래픽을 기술하므로 항상 ethertype이 필요하지만, 매치
속성이 하나도 없는 미러 규칙은 "모든 프레임"을 뜻하며, 이것이 바로 "내가
나열한 것만 복제" 패턴의 catch-all ``skip`` 규칙에 필요한 것이다. 또
ethertype 없는 ``protocol=tcp`` 규칙은 IPv4와 IPv6의 TCP를 모두 매치한다.
IP 프리픽스를 주면 그 주소 패밀리에서 ethertype을 유도하고, 명시한
``ethertype``\ 이나 다른 쪽 프리픽스와 어긋나는 프리픽스는 거부한다.
``icmp``\ 는 ``IPv4``, ``ipv6-icmp``\ 는 ``IPv6``\ 를 함의한다. 포트 범위는
``tcp``/``udp``/``sctp``\ 가 필요하고, ``min``\ 이 ``max``\ 보다 크면
거부하며, ``min``/``max`` 중 하나만 주면 ``min-65535`` 또는 ``1-max``\ 다.

OVN match로 변환
----------------

서버가 규칙을 OVN 논리 flow match 표현식으로 변환한다. 원시 표현식은
일부러 API에 노출하지 않는다. API를 한 백엔드의 표현식 언어에 묶고, 별도
검증이 필요해지기 때문이다. 정말 필요해지면 백엔드 전용 속성을 나중에
추가할 수 있다. 예:

* 매치 속성 없음(catch-all)::

    1

* ``ethertype=IPv4``::

    ip4

* ``protocol=tcp`` (IPv4와 IPv6)::

    tcp

* ``protocol=tcp``, ``destination_ip_prefix=10.0.0.0/24``,
  ``destination_port_range_min=443``, ``destination_port_range_max=443``::

    ip4 && tcp && ip4.dst == 10.0.0.0/24 && tcp.dst == 443

* ``protocol=ipv6-icmp``::

    ip6 && icmp6

* ``protocol=udp``, ``source_port_range_min=1024``::

    udp && udp.src >= 1024 && udp.src <= 65535

변환은 순수 함수(OVN 연결 불필요)라, 유효·무효 속성 조합 전부를 단위
테스트로 덮는다.

동작
----

* **기본 허용(default allow).** OVN은 항상 priority 100에 무조건 pass
  flow를 설치하고 규칙을 그 위 ``100 + priority``\ 에 둔다. 그래서 규칙이
  없는 미러, 또는 어떤 규칙에도 걸리지 않는 패킷은 복제된다. 지금 tap
  mirror의 동작과 같고, gre/erspan 미러에는 규칙이 아예 없다. "내가 나열한
  것만 복제"를 원하면 priority 1에 catch-all ``skip`` 규칙(매치 속성 없음)을
  추가한다. AWS VPC Traffic Mirroring과 K2 Cloud는 상품 계층에서 반대
  기본값을 택했다. RFE 검토에서 드라이버들은 변경을 요구하지 않았다(대안
  참고).
* **priority 0은 거부한다.** 그 flow가 pass flow와 같은 priority에 놓여
  결과가 미정의이기 때문이다.
* **방향.** 방향이 있는 규칙은 그 방향의 OVN ``Mirror``\ 에만 쓰고, 방향이
  없는 규칙은 tap mirror의 모든 ``Mirror``\ 에 쓴다(``Mirror``\ 마다
  ``Mirror_Rule`` 행 하나). tap mirror가 복제하지 않는 방향의 규칙은
  거부한다.
* **보안 그룹과의 관계**\ (미러 단계의 파이프라인 위치에서 나옴): ``OUT``
  규칙은 포트가 보내는 트래픽을 보안 그룹 평가 전에 보고, ``IN`` 규칙은
  포트가 실제로 받는 트래픽만 본다. 사본은 수집 포트의 보안 그룹을
  우회한다.
* **중복과 같은 priority.** 같은 tap mirror에 이미 있는 규칙과 동일한
  규칙(같은 priority·방향·match)은 ``409``\ 로 거부한다. OVN이
  ``Mirror_Rule``\ 을 식별하는 기준과 같다. priority가 같고 match가 겹치는
  서로 다른 두 규칙은 OVN·OVN ACL처럼 허용하지만 어느 쪽이 이기는지는
  미정의다. 문서에서 겹치는 규칙에는 다른 priority를 주라고 안내한다.
  임의의 두 match가 겹치는지 판정하는 것은 싸지 않으므로, 간단한 엄격안은
  같은 priority·방향의 두 번째 규칙을 무조건 거부하는 것뿐이다. 리뷰어를
  위해 대안에 적어 둔다.
* **미러 타입.** ``gre``\ 나 ``erspanv1`` tap mirror에 규칙을 만들면 미러
  타입을 명시한 메시지와 함께 ``400``\ 을 돌려준다. tap mirror를 삭제하면
  규칙도 삭제된다.

REST API 영향
-------------

규칙 생성::

  POST /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  {
      "rule": {
          "priority": 200,
          "action": "skip",
          "direction": "IN",
          "protocol": "tcp",
          "destination_port_range_min": 443,
          "destination_port_range_max": 443
      }
  }

응답 ``201``::

  {
      "rule": {
          "id": "a2b1...",
          "project_id": "3f6c...",
          "priority": 200,
          "action": "skip",
          "direction": "IN",
          "ethertype": null,
          "protocol": "tcp",
          "source_ip_prefix": null,
          "destination_ip_prefix": null,
          "source_port_range_min": null,
          "source_port_range_max": null,
          "destination_port_range_min": 443,
          "destination_port_range_max": 443
      }
  }

``GET .../rules``\ 는 ``{"rules": [...]}``\ 를 돌려주고 ``priority``,
``action``, ``direction``, ``ethertype``, ``protocol``\ 로 필터·정렬할 수
있다. ``DELETE``\ 는 ``204``\ 를 돌려준다.

오류 코드:

* ``400``: 잘못된 속성 값, ``gre``/``erspanv1`` 미러의 규칙, 미러가
  복제하지 않는 방향, L4 프로토콜 없는 포트 범위 또는 ``min``\ 이
  ``max``\ 보다 큰 범위, 프리픽스 패밀리 불일치, 1-32767 밖의
  ``priority``.
* ``403``: 정책.
* ``404``: tap mirror 또는 규칙이 없거나 호출자에게 보이지 않음.
* ``409``: 동일한 규칙이 이미 있음.

``tap-mirror-rules`` 확장은 OVN 서비스 드라이버가 연결된 Northbound
스키마에 ``Mirror_Rule`` 테이블이 있을 때만 광고한다. 구 OVN에서는 하위
리소스가 그냥 없고(``404``), 규칙 없는 lport 미러와 gre/erspan 미러는
전처럼 동작하며, 클라이언트는 실패하는 요청 대신 확장 탐색으로 기능 유무를
알 수 있다.

정책 ``create_tap_mirror_rule``\ 과 ``delete_tap_mirror_rule``\ 은 기본
``ADMIN_OR_PROJECT_MEMBER``, ``get_tap_mirror_rule``\ 은
``ADMIN_OR_PROJECT_READER``\ 로 tap mirror와 같다. 규칙은 항상 그 tap
mirror의 프로젝트에 속한다. 따라서 admin이 만든 크로스 프로젝트 미러의
규칙은 admin 프로젝트에 속하고 거기서 관리한다.

데이터 모델 영향
----------------

새 테이블 ``tap_mirror_rules``\ (expand 마이그레이션만):

* ``id``\ (기본 키), ``project_id``;
* ``tap_mirror_id``, ``tap_mirrors.id``\ 에 대한 외래 키
  (``ON DELETE CASCADE``);
* ``priority``, ``action``, ``direction``, ``ethertype``, ``protocol``,
  ``source_ip_prefix``, ``destination_ip_prefix``\ 와 포트 범위 열 4개.

기존 테이블 변경 없음, contract 마이그레이션 없음. 중복 검사는 행을
삽입하는 같은 트랜잭션 안에서 플러그인이 수행한다.

OVN 드라이버
------------

* ``create_tap_mirror_rule_postcommit``\ 이 규칙을 변환해 선택된 방향의
  OVN ``Mirror`` 행에 ``mirror_rule_add``\ 를 호출하고, ``delete``\ 는
  ``mirror_rule_del``\ 을 호출한다. 명령과 ``Mirror_Rule`` 테이블 등록은
  ovsdbapp 변경 950742 [2]_ 에서 온다.
* 드라이버 베이스 클래스에 기본 no-op인
  ``create/delete_tap_mirror_rule_pre/postcommit`` 훅이 생긴다. 이를
  구현하지 않는 드라이버는 확장을 광고하지 않는다.
* TaaS에는 지금 tap mirror용 OVN DB 유지보수(sync) 작업이 없으므로
  규칙에도 없다. 미러와 규칙을 Northbound DB와 대조하는 유지보수 작업은
  합리적인 후속 작업이지만 이 spec의 범위 밖이다.

클라이언트
----------

openstacksdk에 ``TapMirrorRule`` 리소스와 proxy 메서드
``create/delete/get/tap_mirror_rules``\ 가, python-openstackclient에 다음
명령이 생긴다::

  openstack tap mirror rule create <mirror> --priority 200 --action skip \
      --direction IN --protocol tcp --dst-port 443
  openstack tap mirror rule create <mirror> --priority 1 --action skip
  openstack tap mirror rule list <mirror>
  openstack tap mirror rule show <mirror> <rule>
  openstack tap mirror rule delete <mirror> <rule>

매치용으로 ``--ethertype``, ``--src-ip``, ``--dst-ip``, ``--src-port``,
``--dst-port``\ (``port`` 또는 ``min:max``)를 받는다.

보안 영향
---------

* 규칙은 미러가 복제하는 범위를 좁힐 수만 있다. 호출자가 원래 미러할 수
  없는 포트의 트래픽을 복제하게 만들 수 없고, 누가 미러를 만들 수 있는지는
  바뀌지 않는다(같은 프로젝트 또는 admin).
* 규칙은 tap mirror 소유자가 만든다. admin이 만든 크로스 프로젝트 미러의
  경우 미러 대상 포트의 프로젝트는 미러를 볼 수 없듯 규칙도 보거나 바꿀 수
  없다.
* OVN match 표현식은 검증된 속성에서 서버가 생성한다. 사용자는 표현식을
  주지 않으므로 사용자 제어 값이 OVN 표현식 파서에 닿지 않는다.
* 사본은 수집 포트의 보안 그룹·QoS·conntrack을 우회한다(OVN이
  ``ls_out_pre_acl``\ 에서 포트 보안으로 바로 보냄). 규칙 유무와 관계없는
  lport 미러의 동작이며 여기서 바꾸지 않는다. 위의 파이프라인 위치와 함께
  사용자 가이드에 적어, 보안 팀이 ``OUT`` 사본은 보안 그룹 전, ``IN``
  사본은 보안 그룹 후라는 것을 알게 한다.

성능 영향
---------

* 규칙 하나는 미러 대상 포트가 속한 논리 스위치에서 복제 방향마다 논리 flow
  하나가 되고, 그 포트의 패킷에만 매치된다. 규칙 변경 시 northd는 그 포트의
  flow를 다시 만든다. ACL 변경과 비슷한 비용이다.
* 규칙은 데이터플레인 부하를 줄인다. 오버레이와 수집기의 사본이 줄어든다.
* 미러당 규칙 수는 priority 범위로만 제한된다. 처음에는 쿼터를 제안하지
  않는다. K2 Cloud는 상품 계층에서 미러당 10개로 제한한다. 운영자가
  요청하면 Neutron 쿼터 리소스 ``tap_mirror_rule``\ 을 나중에 추가할 수
  있다.

업그레이드 영향
---------------

새 테이블뿐(expand). 확장은 OVN 드라이버에서만, 그리고 Northbound 스키마에
``Mirror_Rule``\ 이 있을 때만(OVN 25.09+) 로드되므로, OVN 업그레이드 후
neutron-server를 재시작하면 켜진다. neutron-server 롤링 업그레이드 중 구
서버는 확장을 모르고 규칙 호출에 ``404``\ 를 돌려주며, 기존 미러는 영향이
없다. requirements의 ovsdbapp 최소 버전을 [2]_ 가 포함된 릴리스로 올린다.

첫 릴리스에서 지원하지 않는 것
------------------------------

* 규칙 제자리 갱신.
* ``gre``·``erspanv1`` 미러의 규칙.
* 규칙 쿼터.
* ML2/OVS. ML2/OVS용 TaaS는 tap service와 tap flow를 쓰며
  ``Mirror_Rule``\ 에 해당하는 것이 없다.

대안
----

API 형태
~~~~~~~~

RFE 검토에서 ``tap_mirrors`` 리소스에 일부 미러 타입에만 해당하는 속성이
생겼다는 지적이 있었다(``gre``/``erspanv1``\ 의 ``remote_ip``\ 와 터널 ID,
``lport``\ 의 ``remote_port_id``\ 와 규칙) [1]_. 선택지:

#. **tap_mirrors의 하위 리소스로 규칙(제안).** 가장 작은 변경이고, OVN
   드라이버가 구현하는 리소스를 재사용하며, tap mirror 방향마다 OVN
   ``Mirror`` 하나다. 타입별 속성은 타입마다 검증하고(위 표) 문서에 어느
   속성이 해당하는지 적는다.
#. **lport 미러용 별도 리소스**\ (예: ``/taas/port_mirrors``\ 에
   ``port_id``, ``remote_port_id``, ``directions``, ``rules``). 속성
   집합은 깔끔하지만 같은 OVN 객체에 리소스가 둘, 정책·클라이언트도 둘이
   되고, RFE 2168007에서 승인된 lport 미러 API를 옮겨야 한다.
#. **여러 미러가 공유하는 최상위 tap_mirror_filters 리소스**\ (AWS 모델).
   더 유연하지만 더 복잡하고, OVN ``Mirror_Rule`` 행은 ``Mirror`` 하나에
   속하므로 공유는 행 복사로 흉내 내야 한다.

제안은 1안이다. tap mirror 대신 tap service·tap flow 리소스를 재사용하자는
의견이 RFE 2168007에 있었고 드라이버 회의에서 논의됐다. OVN 드라이버가 tap
mirror만 구현하고 OVN ``Mirror``\ 가 tap mirror와 일대일이므로 tap mirror
유지가 합리적이라고 봤다.

기본 거부(default deny)
~~~~~~~~~~~~~~~~~~~~~~~

규칙이 있는 미러는 매치되는 트래픽만 복제하게 한다. AWS와 K2 Cloud 방식이다.
첫 규칙이 생길 때 드라이버가 priority 1 ``skip`` 규칙을 추가하면 구현할 수
있지만, 결과가 숨은 행에 의존하게 되고 첫 규칙을 추가하는 순간 미러의
의미가 바뀐다. 원하는 사용자는 catch-all 규칙을 직접 추가한다. 사용자
가이드의 첫 예제로 보여 준다.

방향별 priority 고유
~~~~~~~~~~~~~~~~~~~~

match와 무관하게 같은 미러에서 같은 priority·방향의 두 번째 규칙을
거부한다. 같은 priority에서 match가 겹치는 미정의 경우가 사라지지만, 무해한
조합(겹치지 않는 ``skip`` 규칙 둘을 같은 priority에)도 막는다. 제안은 OVN
동작을 유지한다. 엄격안을 선호하는 리뷰어는 요청하면 되고, 플러그인 검사
하나면 바뀐다.

TaaS API 정리
~~~~~~~~~~~~~

드라이버 회의에서 TaaS API가 두 모델을 섞어 쓴다는 지적이 있었다. ML2/OVS
에이전트 드라이버만 구현하는 tap service·tap flow와 OVN 서비스 드라이버만
구현하는 tap mirror가 있는데, 어떤 드라이버가 로드되든 두 확장이 모두
광고된다 [1]_. 정리는 바람직하지만 그 자체로 별개 변경이므로, 이 spec은
규칙과의 관계만 밝힌다.

* 이 시리즈는 현재 매핑을 명시적으로 문서화한다. TaaS 문서에 리소스·속성의
  OVS/OVN parity 표를 넣는다(RFE 2168007의 조건이기도 하다).
* lport 미러가 생겼으므로 OVN 드라이버가 tap service·tap flow를 네이티브로
  구현할 수 있다(tap flow마다 ``lport`` 타입 OVN ``Mirror`` 하나, tap
  service 포트를 싱크로, flow 방향을 filter로). 그러면 클라우드 내 수집기에
  대해 두 백엔드가 같은 API를 갖고 tap mirror는 터널 싱크용으로 남는다.
  정리의 가장 유망한 방향이며 별도 RFE로 제안한다.
* 여기서 정의한 규칙 속성은 백엔드 중립이다(보안 그룹 용어, 서버 측 변환).
  나중에 tap flow에 필터링이 생기면 같은 하위 리소스 정의를 붙일 수 있으므로
  이 spec의 선택이 정리를 선점하지 않는다.
* 로드된 드라이버가 구현하는 확장만 광고해 API 탐색이 사실을 말하게 하는
  것은 작고 독립적인 수정이며 같은 정리에서 할 수 있다.


Implementation
==============

(구현)

담당자
------

주 담당자:
  Chanyeol Yoon (ycy1766)

작업 항목
---------

* neutron-lib: ``tap-mirror-rules`` API 정의와 예외. ``tap-mirror-lport``
  정의가 lport에서 ``remote_ip``/터널 ID를 거부하게 수정.
* ovsdbapp: ``Mirror_Rule`` 테이블과 ``mirror_rule_add/del`` 명령 [2]_,
  이후 릴리스와 requirements 상향.
* tap-as-a-service: DB 모델·마이그레이션, 플러그인 검증과 match 변환, OVN
  드라이버, 드라이버 훅, 조건부 확장 광고, 정책, 문서, 릴리스노트, CI 잡.
* openstacksdk·python-openstackclient 지원.
* neutron-tempest-plugin: API·시나리오 테스트.

동작하는 구현이 있고 OVN 26.03 + Neutron 2025.1에서 end to end로
검증했다(skip 규칙, 그것을 덮는 더 높은 priority의 mirror 규칙 포함).
시리즈는 ovsdbapp, neutron-lib, tap-as-a-service, openstacksdk,
python-openstackclient 순서로 Gerrit에 올린다.


Dependencies
============

(의존성)

* RFE 2168007(lport tap mirror)과 그 neutron-lib·tap-as-a-service 변경.
  규칙 변경은 그 위에 쌓인다.
* ovsdbapp 변경 950742 [2]_. 마지막 리뷰 이후 비활성 상태다. 담당자가 리뷰
  의견을 반영한 패치셋을 갖고 있으며 원저자 동의 하에 이어받는다.
* 런타임 OVN 25.09 이상(``Mirror_Rule``\ 이 있는 Northbound 스키마).
* 클라이언트 부분을 위한 openstacksdk·python-openstackclient 릴리스.


Testing
=======

(테스트)

* 단위 테스트: API 정의, 플러그인 검증(REST API 절의 모든 거부 케이스),
  OVN match 변환(유효·무효 속성 조합 전부), 방향별 OVN 드라이버 호출, DB
  마이그레이션과 cascade 삭제, ``Mirror_Rule`` 없는 Northbound 스키마(확장
  미광고, lport 미러는 계속 동작).
* ovsdbapp의 ``mirror_rule_add/del`` 기능 테스트(실제 OVN Northbound
  스키마 대상, 스키마에 ``Mirror_Rule``\ 이 없으면 skip). [2]_ 의 일부.
* TaaS에는 지금 기능 테스트 스위트가 없다. neutron의 OVN 기능 테스트 베이스
  클래스를 바탕으로 실제 ``ovn-northd``\ 를 상대하는 OVN 드라이버용 작은
  기능 테스트 모듈을 시리즈에 포함한다. 방향마다 쓰인 ``Mirror``\ 와
  ``Mirror_Rule`` 행, 규칙·미러 삭제 시 제거를 확인한다.
* ``neutron-tempest-plugin-tap-as-a-service-ovn`` 잡은 지금 OVN
  ``branch-24.03``\ 을 빌드하며 lport 미러가 없다. OVN 25.09 이상을 빌드하는
  잡 변형을 추가하고, 기존 잡은 구 OVN에서 gre/erspan을 계속 덮는다. 새
  잡은 다음을 돌린다.

  * API 테스트: 규칙 생성·목록·조회·삭제, 거부 케이스(gre 미러의 규칙,
    priority 0, 미러가 복제하지 않는 방향, L4 프로토콜 없는 포트 범위,
    프리픽스 패밀리 불일치, 동일 규칙), tap mirror와 함께 삭제되는 규칙,
    미러 타입 표의 속성 검사.
  * ``tcpdump``\ 를 돌리는 수집기 VM 시나리오 테스트: ``skip`` 규칙이
    수집기에서 한 flow를 제거하고, 더 높은 priority의 ``mirror`` 규칙이
    되살리며, ``IN`` 전용 규칙은 아웃바운드 트래픽에 영향이 없고, catch-all
    ``skip`` + ``mirror`` 규칙 하나는 나열한 트래픽만 전달한다.

  규칙 테스트는 ``tap-mirror-rules`` 확장이 광고되지 않으면 skip되므로 구
  잡에서도 안전하다.


Documentation Impact
====================

(문서 영향)

* TaaS 사용자 가이드: 규칙, 기본 허용과 catch-all 패턴, 보안 그룹·수집
  포트와의 관계, lport 미러에서만 규칙 지원, OVN 버전 요구.
* TaaS API 레퍼런스(새 하위 리소스, 미러 타입 속성 표)와 릴리스노트.
* RFE 검토에서 요청된 TaaS 리소스·속성의 OVS/OVN 기능 parity 표.


References
==========

.. [1] Neutron drivers meeting, 2026-10-02:
   https://meetings.opendev.org/meetings/neutron_drivers/2026/neutron_drivers.2026-10-02-13.06.log.html
.. [2] ovsdbapp: nb: add support for mirror-rules in mirrors:
   https://review.opendev.org/c/openstack/ovsdbapp/+/950742
.. [3] ovn-northd mirror logical flows (``build_mirror_lflows``,
   ``OVN_LPORT_MIRROR_OFFSET``):
   https://github.com/ovn-org/ovn/blob/main/northd/northd.c

* RFE for lport tap mirrors: https://bugs.launchpad.net/neutron/+bug/2168007
* OVN Mirror_Rule: https://github.com/ovn-org/ovn/commit/3dd8f36b85b3e648bc506839aa62e81b21c41e30
* ovn-nb(5), ``Mirror`` and ``Mirror_Rule`` tables:
  https://www.ovn.org/support/dist-docs/ovn-nb.5.html
* Neutron QoS rules API (model for the sub-resource):
  https://docs.openstack.org/api-ref/network/v2/index.html#qos-rules
* AWS VPC Traffic Mirroring filters:
  https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-filter.html
* K2 Cloud Traffic Mirroring:
  https://docs.k2.cloud/en/services/interconnect/traffic_mirroring.html

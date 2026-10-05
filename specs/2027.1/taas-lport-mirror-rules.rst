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
테이블(OVN 25.09 이상)로 구현한다.

``lport`` 미러 타입 자체는 RFE 2168007이 다루며, 2026-10-02 드라이버
회의에서 spec 없이 승인됐다 [1]_. 같은 회의에서 규칙은 새 API와 DB
테이블을 추가하므로 이 spec을 요청받았다.


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
* 보안 팀은 가장 비용이 적은 소스 쪽에서 선택하길 원한다. 예를 들어
  "인바운드 TCP/443과 DNS만", "10.0.9.0/24로 가는 백업 트래픽만 빼고 전부".

사용 사례:

* 프로젝트 사용자가 웹 서버 포트를 미러하면서 수집기에서는 인바운드
  TCP/443만 받고 싶다.
* 프로젝트 사용자가 DB 포트를 미러하면서 특정 서브넷으로 가는 복제(replication)
  트래픽만 빼고 전부 받고 싶다.
* 운영자가 테넌트 포트를 서비스 프로젝트의 수집기로 미러하면서(lport 미러와
  같이 admin 전용) 인터넷 방향 대역과 주고받는 트래픽만 받고 싶다.

OVN에는 이미 기능이 있다. 25.09부터 ``lport`` 타입 ``Mirror`` 행이
``Mirror_Rule`` 행(우선순위, match, 동작 ``mirror`` 또는 ``skip``)을 참조할
수 있다. Neutron에는 이를 위한 API가 없다.

``gre``\ 와 ``erspanv1`` 미러 타입에는 규칙을 제안하지 않는다. OVN에서 이
타입들은 ``ovn-controller``\ 가 설정하는 일반 OVS 포트 미러라서 logical
pipeline을 거치지 않으므로, match를 적용할 logical flow가 없다. ``lport``
미러는 logical switch의 ``ls_in_mirror``·``ls_out_mirror`` 단계에서
구현되고, ``Mirror_Rule`` match도 바로 그 단계에 설치된다.


Proposed Change
===============

(제안 변경)

개요
----

새 API 확장 ``tap-mirror-rules``\ (``tap-mirror``, ``tap-mirror-lport`` 필요)가
tap mirror에 ``rules`` 하위 리소스를 추가한다::

  POST   /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  GET    /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules
  GET    /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules/{rule_id}
  DELETE /v2.0/taas/tap_mirrors/{tap_mirror_id}/rules/{rule_id}

하위 리소스는 QoS policy rule 모델을 따른다(neutron-lib 하위 리소스 정의,
부모가 있는 컨트롤러, 규칙별 정책). 규칙은 수정할 수 없다. 바꾸려면 지우고
새로 만든다. 이렇게 하면 OVN 행과 1:1 대응이 유지되고, 이미 데이터플레인에
적용된 규칙을 부분 수정하는 일을 피할 수 있다.

미러 타입별 속성
----------------

RFE 리뷰에서 ``tap_mirrors``\ 에 일부 미러 타입에서만 유효한 속성이 생긴다는
점이 지적됐다 [1]_. API는 이를 명시적으로 드러내고, 적용되지 않는 속성은
무시하지 않고 거부한다.

.. list-table::
   :header-rows: 1
   :widths: 34 33 33

   * - 속성
     - ``gre`` / ``erspanv1``
     - ``lport``
   * - ``port_id``
     - 필수
     - 필수
   * - ``remote_ip``
     - 필수
     - 거부(``400``)
   * - ``directions`` 터널 ID
     - 필수, 방향마다 하나
     - ``null``\ 이어야 함
   * - ``remote_port_id``
     - 거부(``400``)
     - 필수
   * - ``rules`` 하위 리소스
     - 거부(``400``)
     - 지원

지금 lport 구현은 lport 미러에 준 ``remote_ip``\ 나 터널 ID를 무시한다. 쓰이지
않는 값을 쓰인다고 오해하지 않도록 거부하게 바꾼다. 문서에도 같은 표를 둔다.

규칙 속성
---------

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - 속성
     - 기본값
     - 설명
   * - ``priority``
     - 필수
     - 1-32767, 높은 값을 먼저 평가
   * - ``action``
     - ``mirror``
     - ``mirror`` 또는 ``skip``
   * - ``direction``
     - ``null``
     - ``IN``, ``OUT`` 또는 ``null`` (미러가 복제하는 모든 방향)
   * - ``ethertype``
     - ``IPv4``
     - ``IPv4`` 또는 ``IPv6``
   * - ``protocol``
     - ``null``
     - ``tcp``, ``udp``, ``sctp``, ``icmp``, ``ipv6-icmp``
   * - ``source_ip_prefix``
     - ``null``
     - CIDR
   * - ``destination_ip_prefix``
     - ``null``
     - CIDR
   * - ``source_port_range_min/max``
     - ``null``
     - 1-65535, TCP/UDP/SCTP에서만
   * - ``destination_port_range_min/max``
     - ``null``
     - 1-65535, TCP/UDP/SCTP에서만

서버가 규칙을 OVN match 식으로 변환한다. 예를 들어 ``protocol=tcp``,
``destination_ip_prefix=10.0.0.0/24``, ``destination_port_range_min=max=443``
은 다음이 된다::

  ip4 && tcp && ip4.dst == 10.0.0.0/24 && tcp.dst == 443

OVN 원본 식은 일부러 노출하지 않는다. API가 특정 백엔드의 식 언어에 묶이고
별도 검증도 필요하기 때문이다. 실제 수요가 생기면 백엔드 전용 속성을 나중에
추가할 수 있다.

동작
----

* **기본 허용(default allow).** OVN은 포트의 모든 패킷을 미러하는 무조건
  flow를 priority 100에 항상 설치하고, 규칙은 그 위 ``100 + priority``\ 에
  둔다. 따라서 규칙이 없는 미러나 어떤 규칙에도 맞지 않는 패킷은 미러된다.
  지금 tap mirror의 동작과 같다. "적은 것만 미러"를 원하면 priority 1에
  전체를 잡는 ``skip`` 규칙을 추가한다. AWS VPC Traffic Mirroring과 K2
  Cloud는 상품 계층에서 반대 기본값(규칙 없으면 아무것도 미러 안 함)을
  택했다. RFE 리뷰에서 드라이버들이 바꾸라고 하지는 않았지만, 리뷰어가
  그쪽을 선호할 수 있다(대안 절 참고).
* **priority 0은 거부한다.** 무조건 flow와 같은 OpenFlow 우선순위에 놓여
  결과가 정의되지 않기 때문이다.
* **방향.** OVN 드라이버는 lport tap mirror마다 방향별 OVN ``Mirror``\ 를 이미
  하나씩 만든다(``IN``\ 은 ``to-lport``, ``OUT``\ 은 ``from-lport``). 방향이
  있는 규칙은 그 미러에만, 방향이 없는 규칙은 tap mirror의 모든 미러에
  붙는다. tap mirror가 복제하지 않는 방향의 규칙은 거부한다.
* **보안 그룹과의 관계**\ (미러 단계의 pipeline 위치에서 나온다): 포트가 보내는
  트래픽은 보안 그룹 평가 전에, 포트로 들어오는 트래픽은 평가 후에 match된다.
  사본은 수집 포트의 보안 그룹을 거치지 않는다.
* **중복과 같은 우선순위.** 기존 규칙과 완전히 같은 규칙(우선순위, 방향,
  match가 같음)은 ``409``\ 로 거부한다. 우선순위가 같고 match가 겹치는 서로
  다른 규칙은 OVN과 같이 허용하지만 어느 쪽이 이기는지는 정의되지 않는다.
  문서에서 겹치는 규칙에는 다른 우선순위를 주라고 안내한다. 같은 우선순위와
  방향의 두 번째 규칙을 아예 거부하는 더 엄격한 방식도 리뷰어가 선호할 수
  있는 대안이다.
* **미러 타입.** ``gre``\ 나 ``erspanv1`` tap mirror에 규칙을 만들면 명확한
  메시지와 함께 ``400``\ 을 돌려준다. tap mirror를 지우면 규칙도 함께 지워진다.

데이터 모델
-----------

새 테이블 ``tap_mirror_rules``\ (expand 마이그레이션). ``tap_mirrors.id``
외래 키(``ON DELETE CASCADE``), ``project_id``, 위 속성마다 컬럼 하나.
기존 테이블은 바뀌지 않으며 contract 마이그레이션은 필요 없다.

OVN 드라이버
------------

* ``create_tap_mirror_rule_postcommit``\ 이 규칙을 변환해 선택한 방향의 OVN
  미러에 ``mirror_rule_add``\ 를 호출한다. 삭제는 ``mirror_rule_del``\ 을
  호출한다.
* ``Mirror_Rule`` 테이블과 ``mirror_rule_add/del`` 명령은 ovsdbapp 변경
  950742에서 온다 [2]_. 드라이버는 연결된 Northbound 스키마에 테이블이
  있는지 런타임에 확인한다. 구버전 OVN에서는 규칙 생성이 명확한 오류로
  실패하고, 규칙 없는 lport 미러와 gre/erspan 미러는 그대로 동작한다.
* TaaS에는 지금 tap mirror용 OVN DB sync가 없으므로 규칙에도 없다. 미러와
  규칙의 sync 추가는 이 spec의 범위 밖이다.
* 다른 TaaS 드라이버는 이 확장을 지원하지 않으며, 그 경우 확장이 로드되지
  않는다.

정책
----

``create/get/delete_tap_mirror_rule``\ 의 기본값은 tap mirror와 같은
``ADMIN_OR_PROJECT_MEMBER``\ (``get``\ 은 ``READER``)이다. 규칙은 항상 그 tap
mirror의 프로젝트에 속한다.

클라이언트
----------

openstacksdk에 ``TapMirrorRule`` 리소스와 proxy 메서드를, python-openstackclient
에 다음 명령을 추가한다::

  openstack tap mirror rule create <mirror> --priority 100 \
      --direction IN --protocol tcp --dst-port 443
  openstack tap mirror rule list <mirror>
  openstack tap mirror rule show <mirror> <rule>
  openstack tap mirror rule delete <mirror> <rule>

첫 릴리스에서 지원하지 않는 것
------------------------------

* 규칙 수정.
* ``gre``, ``erspanv1`` 미러의 규칙.
* 규칙 쿼터. 미러당 규칙 수는 우선순위 범위로 제한된다. 운영자가 필요하면
  나중에 쿼터를 추가할 수 있다.
* ML2/OVS. ML2/OVS용 TaaS는 tap service와 tap flow를 쓰며
  ``Mirror_Rule``\ 에 해당하는 기능이 없다.


대안
----

API 형태
~~~~~~~~

RFE 리뷰에서 두 가지가 지적됐다. ``tap_mirrors`` 리소스가 일부 미러 타입에만
해당하는 속성을 갖게 된다는 점(``gre``/``erspanv1``\ 의 ``remote_ip``·터널
ID, ``lport``\ 의 ``remote_port_id``·규칙), 그리고 TaaS API가 이미 tap
service/tap flow(ML2/OVS)와 tap mirror(ML2/OVN)를 섞고 있다는 점이다 [1]_.
선택지:

#. **tap_mirrors의 하위 리소스로 규칙(제안).** 변경이 가장 작고, OVN
   드라이버가 구현하는 리소스를 재사용하며, tap mirror 방향마다 OVN
   ``Mirror`` 하나. 타입별 속성은 타입별로 검증하고, 어떤 속성이 적용되는지
   문서에 정리한다.
#. **lport 미러용 별도 리소스**\ (예: ``port_id``, ``remote_port_id``,
   ``directions``, ``rules``\ 를 가진 ``/taas/port_mirrors``). 속성 구성은
   깔끔하지만, 같은 OVN 객체에 리소스가 둘이 되고 정책과 클라이언트도 두
   벌이 필요하며, RFE 2168007에서 승인된 lport 미러 API를 옮겨야 한다.
#. **여러 미러가 공유하는 최상위 tap_mirror_filters 리소스로
   규칙** (AWS 모델). 더 유연하지만 더 복잡하다. OVN ``Mirror_Rule`` 행은
   ``Mirror`` 하나에 속하므로 공유는 행 복사로 흉내 내야 한다.

제안은 1안이다. tap mirror 대신 tap service와 tap flow 리소스를 재사용하자는
안은 RFE 2168007에서 제안됐고, 드라이버 회의에서 tap mirror를 유지하기로 했다.
OVN 드라이버는 tap mirror만 구현하고, OVN ``Mirror``\ 가 tap mirror와 1:1로
대응하기 때문이다. TaaS API 전체 정리(ML2/OVS용 tap service·tap flow 대
ML2/OVN용 tap mirror)는 할 가치가 있지만 이 spec과는 별개이며 별도 RFE로
제안해야 한다.

기본 거부(default deny)
~~~~~~~~~~~~~~~~~~~~~~~

규칙이 있는 미러는 맞는 트래픽만 복제하게 하는 방식이다. AWS와 K2 Cloud가
이렇게 한다. 첫 규칙이 생길 때 드라이버가 priority 1 ``skip`` 규칙을 넣어
구현할 수 있지만, 결과가 보이지 않는 행에 좌우되고 첫 규칙을 추가하는 순간
미러의 의미가 바뀐다. 원하는 사용자는 전체를 잡는 규칙을 직접 추가하면 된다.


업그레이드 영향
---------------

새 테이블만 추가한다. 확장은 OVN 드라이버에서만 로드되고, Northbound
스키마에 ``Mirror_Rule``\ 이 있을 때(OVN 25.09 이상) 동작한다. 롤링 업그레이드
중에는 구버전 Neutron 서버가 확장을 몰라 규칙 호출을 거부하며, 기존 미러는
영향을 받지 않는다.


테스트
------

* 단위 테스트: API 정의, 플러그인 검증, OVN match 변환(모든 속성 조합과 잘못된
  조합), OVN 드라이버 호출, DB 마이그레이션과 연쇄 삭제, ``Mirror_Rule``\ 이
  없는 Northbound 스키마(규칙 생성은 깔끔하게 실패하고 lport 미러는 동작).
* ovsdbapp의 ``mirror_rule_add/del`` functional 테스트. 실제 OVN Northbound
  스키마에서 돌고 ``Mirror_Rule``\ 이 없으면 skip한다. [2]_\ 에 포함.
* ``neutron-tempest-plugin-tap-as-a-service-ovn`` 잡은 지금 lport 미러가
  없는 OVN ``branch-24.03``\ 을 빌드한다. OVN 25.09 이상 잡 변형을 추가해
  다음을 돌린다.

  * API 테스트: 규칙 생성·목록·조회·삭제, 거부 케이스(gre 미러의 규칙,
    priority 0, 미러가 복제하지 않는 방향, L4 프로토콜 없는 포트 범위, 같은
    규칙), tap mirror 삭제 시 규칙 삭제, 위 표의 속성 검사.
  * 수집 VM에서 ``tcpdump``\ 로 보는 시나리오 테스트: ``skip`` 규칙이 흐름을
    수집기에서 빼고, 더 높은 우선순위 ``mirror`` 규칙이 되돌리며, ``IN`` 전용
    규칙은 나가는 트래픽에 영향을 주지 않는다.

TaaS에는 지금 functional 테스트 스위트가 없어서 OVN 백엔드는 위 tempest 잡이
end-to-end로 검증한다. 리뷰어가 원하면 neutron OVN functional 기반 위에 OVN
드라이버용 작은 functional 스위트를 추가할 수 있다.


문서 영향
---------

* TaaS 사용자 가이드: 규칙, 기본 허용, "적은 것만" 만드는 방법, 보안 그룹과의
  관계, 규칙은 lport 미러에만 있다는 점.
* TaaS API 레퍼런스와 릴리스 노트.
* RFE 리뷰에서 요청된 TaaS OVS/OVN 기능 차이 표.


작업 항목
---------

* neutron-lib: ``tap-mirror-rules`` API 정의와 예외.
* ovsdbapp: ``Mirror_Rule`` 지원 [2]_.
* tap-as-a-service: DB, 플러그인, OVN 드라이버, 정책, 문서, CI 잡.
* openstacksdk, python-openstackclient 지원.
* neutron-tempest-plugin: API·시나리오 테스트.

동작하는 구현이 이미 있으며, OVN 26.03과 Neutron 2025.1에서 skip 규칙과
그것을 덮는 더 높은 우선순위 mirror 규칙까지 end-to-end로 검증했다.


References
==========

(참고 자료)

.. [1] Neutron 드라이버 회의, 2026-10-02:
   https://meetings.opendev.org/meetings/neutron_drivers/2026/neutron_drivers.2026-10-02-13.06.log.html
.. [2] ovsdbapp: nb: add support for mirror-rules in mirrors:
   https://review.opendev.org/c/openstack/ovsdbapp/+/950742

* lport tap mirror RFE: https://bugs.launchpad.net/neutron/+bug/2168007
* OVN Mirror_Rule: https://github.com/ovn-org/ovn/commit/3dd8f36b85b3e648bc506839aa62e81b21c41e30
* AWS VPC Traffic Mirroring 필터:
  https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-filter.html
* K2 Cloud Traffic Mirroring:
  https://docs.k2.cloud/en/services/interconnect/traffic_mirroring.html

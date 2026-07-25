.. SPDX-License-Identifier: GPL-2.0
.. _epoll_kqueue_study:

==========================================
epoll 구현 읽기와 kqueue 비교
==========================================

이 문서는 백엔드 엔지니어가 Linux ``epoll``의 실제 구현을 따라가며 학습하고,
BSD 계열의 ``kqueue``와 설계 차이를 설명할 수 있도록 만든 코드 리딩 가이드다.
설명은 이 소스 트리의 ``fs/eventpoll.c``를 기준으로 한다.

범위부터 명확히 하기
====================

Linux 커널은 ``epoll``을 구현하며 BSD의 ``kqueue(2)``/``kevent(2)`` 구현은
포함하지 않는다. 이 트리에서 ``kqueue``라는 이름으로 검색되는 RDMA 필드 등은
I/O 멀티플렉싱 API와 관계없는 일반 큐 변수다. 따라서 다음 두 가지를 구분해서
읽어야 한다.

* ``epoll`` 부분은 이 트리의 실제 자료구조, 락, 함수 호출 관계를 설명한다.
* ``kqueue`` 부분은 BSD 계열 API의 일반적인 개념을 비교한다. 세부 필터와 수명
  규칙은 FreeBSD, OpenBSD, NetBSD, macOS 사이에서 다를 수 있으므로 대상 OS의
  매뉴얼과 소스를 최종 기준으로 삼아야 한다.

30초 요약
=========

``epoll``은 관심 있는 모든 파일을 ``epoll_wait()`` 때마다 순회하지 않는다.
``EPOLL_CTL_ADD`` 시 대상 파일의 wait queue에 콜백을 연결하고, 대상이 준비되면
그 콜백이 해당 ``epitem``을 ready list에 한 번만 넣는다. ``epoll_wait()``은
전체 interest set이 아니라 ready list를 중심으로 작업한다.

핵심은 다음 세 문장이다.

* ``ep->rbr``은 등록 여부를 찾는 **interest set**이고 ``ep->rdllist``는 반환
  후보를 모은 **ready set**이다.
* 알림 콜백은 IRQ 문맥에서도 호출될 수 있어 spinlock만 사용하고,
  ``copy_to_user()``를 수행하는 전달 경로는 잠들 수 있어 mutex를 사용한다.
* epoll은 완료(completion)나 wakeup 횟수를 저장하는 큐가 아니라 현재 처리할
  준비 상태(readiness)를 추적하는 장치다.

코드 지도
=========

먼저 아래 순서대로 함수 이름을 검색하면 전체 흐름을 빠르게 잡을 수 있다.

.. list-table::
   :header-rows: 1
   :widths: 18 30 52

   * - 단계
     - 핵심 함수
     - 확인할 내용
   * - 생성
     - ``do_epoll_create()``, ``ep_alloc()``
     - anonymous inode 파일과 ``struct eventpoll``의 연결
   * - 등록/변경/삭제
     - ``do_epoll_ctl_file()``
     - RB tree 조회 후 ADD/MOD/DEL 분기
   * - ADD
     - ``ep_insert()``, ``ep_ptable_queue_proc()``
     - 대상의 ``->poll()``을 이용해 wait queue 콜백 등록
   * - 준비 알림
     - ``ep_poll_callback()``
     - ready list 삽입과 대기 스레드 wake-up
   * - 대기
     - ``ep_poll()``
     - ready 확인, wait queue 참가, timeout/signal 처리
   * - 반환
     - ``ep_send_events()``, ``ep_deliver_event()``
     - 재확인, userspace 복사, LT/ET/ONESHOT 처리
   * - 정리
     - ``ep_remove()``, ``eventpoll_release_file()``
     - 명시적 DEL과 파일 close 경로의 차이

같이 볼 파일은 다음과 같다.

``include/uapi/linux/eventpoll.h``
  사용자에게 노출되는 operation, event mask, ``struct epoll_event`` 정의다.

``include/linux/eventpoll.h``
  VFS close 경로 및 다른 커널 서브시스템이 사용하는 내부 인터페이스다.

``fs/eventpoll.c``
  interest set, ready list, wait/wakeup, 중첩 epoll 검증의 실제 구현이다.

``fs/file_table.c``
  ``__fput()``에서 ``eventpoll_release()``가 호출되는 위치를 보여 준다.

``io_uring/epoll.c``
  io_uring의 epoll operation도 ``do_epoll_ctl_file()``과
  ``epoll_sendevents()``를 재사용한다는 점을 보여 준다.

핵심 자료구조
=============

관계를 단순화하면 다음과 같다.

::

  epoll fd
    |
    v
  anonymous inode file
    `- private_data ------> struct eventpoll
                              |- rbr      : 등록된 epitem의 RB tree
                              |- rdllist  : 현재 ready인 epitem의 FIFO
                              |- ovflist  : 전달 중 새로 발생한 ready 임시 목록
                              |- wq       : epoll_wait() 호출자가 잠드는 곳
                              `- poll_wait: 이 epoll을 감시하는 바깥 poll/epoll

  watched file
    |- f_ep -------------> 이 파일을 감시하는 epitem들의 hlist
    `- target wait queue
         `- eppoll_entry.wait
              `- ep_poll_callback() ---> epitem ---> owning eventpoll

``struct eventpoll``
--------------------

epoll 인스턴스 하나를 나타낸다. 사용자가 받은 epoll fd의
``file->private_data``가 이 객체를 가리킨다.

``rbr``
  관심 대상으로 등록된 ``epitem``을 ``(struct file *, fd)`` 키로 보관하는
  cached RB tree다. ``epoll_ctl()``의 조회/삽입/삭제가 주 사용자다.

``rdllist``
  ready 후보만 보관한다. 같은 ``epitem``을 중복 연결하지 않으므로 여러 번의
  wakeup이 발생해도 이벤트 개수처럼 계속 쌓이지 않는다.

``ovflist``
  ``epoll_wait()``이 ready batch를 순회하는 동안 발생한 알림을 임시로 받는다.
  여기서 ``overflow``는 용량 초과가 아니라 scan 도중 옆으로 흘려보내는 목록에
  가깝다.

``wq``
  반환할 이벤트가 없을 때 ``epoll_wait()`` 호출자가 잠드는 wait queue다.

``poll_wait``
  epoll fd 자체를 다른 poll/epoll이 감시할 때 사용한다. 이것이 epoll 중첩을
  가능하게 한다.

``struct epitem``
-----------------

하나의 epoll 인스턴스와 하나의 ``(file, fd)`` 등록 관계를 나타낸다.

``rbn``
  interest set인 ``eventpoll.rbr``에 연결되는 노드다.

``rdllink`` / ``ovflist_next``
  ready 상태를 나타내는 링크다. RB tree 등록 상태와 ready 상태는 독립적이다.

``ffd``
  ``struct file *``와 등록 당시 fd 번호를 함께 저장한다. fd 숫자만 키로 쓰지
  않는 이유는 fd 재사용과 open file description의 정체성을 구분하기 위해서다.

``pwqlist``
  대상 파일의 ``->poll()``이 선택한 wait queue들에 설치한
  ``eppoll_entry`` 목록이다. 한 파일이 여러 wait queue에 연결할 수 있으므로
  하나의 ``epitem``에 여러 entry가 생길 수 있다.

``event``
  관심 mask와 userspace가 제공한 opaque ``data`` 값을 저장한다.

``struct eppoll_entry``
-----------------------

대상 파일의 wait queue와 ``epitem``을 잇는 어댑터다. 내부
``wait_queue_entry_t``의 wake 함수는 ``ep_poll_callback()``이다.

세 실행 경로
============

생성 경로
---------

::

  epoll_create1()
    -> do_epoll_create()
       -> ep_alloc()
       -> anon_inode_getfile("[eventpoll]", ...)
       -> fd_publish()

``ep_alloc()``은 mutex, spinlock, 두 wait queue, ready list, RB tree,
reference count를 초기화한다. epoll fd는 단순한 정수 핸들이 아니라
``eventpoll_fops``를 가진 poll 가능한 anonymous inode 파일이다. 따라서 다른
epoll 인스턴스가 이 파일을 감시할 수 있다.

등록 경로: ``EPOLL_CTL_ADD``
----------------------------

::

  epoll_ctl(ADD)
    -> do_epoll_ctl()
       -> do_epoll_ctl_file()
          -> ep_find()                 기존 등록 확인
          -> ep_insert()
             -> ep_alloc_epitem()      quota 부과 및 객체 할당
             -> ep_register_epitem()   file->f_ep + ep->rbr 연결
             -> ep_item_poll()
                -> vfs_poll()
                   -> target ->poll()
                      -> poll_wait()
                         -> ep_ptable_queue_proc()
             -> 이미 ready이면 ep->rdllist에 삽입

여기서 중요한 부분은 ``ep_item_poll()``을 한 번 호출하는 이유가 두 가지라는
점이다.

#. 대상 파일의 현재 readiness를 얻는다.
#. ``poll_table``을 통해 대상 파일이 사용하는 wait queue마다
   ``ep_poll_callback()``을 등록한다.

등록과 현재 상태 확인을 함께 하지 않으면 등록 직전 또는 직후에 발생한 상태
변화를 놓칠 수 있다. 대상의 ``->poll()`` 구현은 현재 상태를 반환하는 동시에
``poll_wait()``으로 향후 알림 경로를 연결한다.

``EPOLL_CTL_MOD``는 mask를 먼저 게시한 후 memory barrier를 수행하고 대상을
다시 poll한다. 이 순서는 새 관심 mask를 적용하는 순간의 이벤트를 콜백과
재검사 중 적어도 한쪽이 관찰하도록 하기 위한 것이다.

알림 경로
---------

::

  socket/pipe/eventfd/... 상태 변화
    -> target wait queue의 wake_up()
       -> ep_poll_callback()
          -> 관심 mask와 callback key 비교
          -> rdllist 또는 ovflist에 epitem을 한 번만 연결
          -> ep->wq의 epoll_wait() 호출자 깨우기
          -> ep->poll_wait의 바깥 poll/epoll 깨우기

콜백은 소비자 스레드가 아니라 이벤트를 발생시킨 producer의 문맥에서 실행된다.
네트워크 수신 경로처럼 IRQ 문맥에서 도달할 수 있으므로 잠들 수 없고
``ep->lock``은 IRQ-safe spinlock이어야 한다.

이미 ``rdllist``에 있는 항목은 다시 추가하지 않는다. 따라서 epoll은
``wake_up()`` 호출 횟수를 세는 이벤트 로그가 아니다. 애플리케이션은 반환된
readiness를 보고 실제 ``read()``, ``write()``, ``accept()`` 등을 수행해
상태를 소비해야 한다.

대기 및 반환 경로
-----------------

::

  epoll_wait()
    -> do_epoll_wait()
       -> ep_poll()
          |- ready 있음 -> ep_send_events()
          |- 선택적 NAPI busy poll
          `- ep->wq에 참가하고 sleep

  ep_send_events()
    -> ep_start_scan()       rdllist를 private scan_batch로 이동
    -> ep_deliver_event()    다시 poll하고 userspace로 복사
    -> ep_done_scan()        ovflist와 남은 batch를 rdllist로 병합

``ep_poll()``은 잠들기 직전에 ``ep->lock``을 잡고 readiness를 마지막으로
확인한 뒤, 여전히 비어 있을 때만 같은 락 아래에서 ``ep->wq``에 참가한다.
알림 측 ``ep_poll_callback()``도 같은 락을 사용하므로 "확인 직후, sleep
직전에 이벤트가 도착하는" 전형적인 lost wakeup 구간이 닫힌다.

``ep_send_events()``은 ``ep->mtx``를 잡은 상태에서 ready list를 호출자 전용
``scan_batch``로 옮긴다. userspace 복사나 대상 ``->poll()`` 호출 동안
spinlock을 잡고 있을 수 없으므로, scan 중 새 이벤트는 ``ovflist``로 보낸다.
마지막에 ``ep_done_scan()``이 이를 다시 ``rdllist``에 합쳐 동시 알림을
잃지 않는다.

왜 반환 직전에 다시 poll하는가
------------------------------

ready list는 "반환 후보" 목록이지 영원히 참인 상태의 스냅샷이 아니다.
콜백 발생 후 다른 스레드가 데이터를 소비했거나 상태가 바뀔 수 있다.
``ep_deliver_event()``은 대상의 ``->poll()``을 다시 호출해 현재 관심 이벤트가
남았는지 확인한 뒤 userspace에 복사한다.

이 재검사는 spurious readiness를 줄이지만, 반환 직후 다른 스레드가 상태를
소비하는 경쟁까지 없애지는 않는다. 그러므로 백엔드 애플리케이션의 실제 I/O는
여전히 nonblocking이어야 하며 ``EAGAIN``을 정상적인 경쟁 결과로 처리해야 한다.

LT, ET, ONESHOT
==============

.. list-table::
   :header-rows: 1
   :widths: 18 32 50

   * - 모드
     - 커널 구현에서 보이는 차이
     - 애플리케이션 규칙
   * - Level-triggered
     - 전달 후에도 ready이면 ``rdllist`` 끝에 다시 연결한다.
     - 상태가 남아 있으면 다음 ``epoll_wait()``에서도 다시 받을 수 있다.
   * - ``EPOLLET``
     - 전달 후 ``ep_deliver_event()``이 항목을 ready list에 재연결하지 않는다.
     - fd를 nonblocking으로 만들고 ``EAGAIN``까지 drain한다.
   * - ``EPOLLONESHOT``
     - 한 번 전달한 뒤 공개 event bit를 지워 콜백을 비활성화한다.
     - 처리가 끝난 뒤 ``EPOLL_CTL_MOD``로 명시적으로 rearm한다.
   * - ``EPOLLEXCLUSIVE``
     - 대상 wait queue에 exclusive entry를 설치해 경쟁 epoll 등록의 wakeup
       확산을 줄인다.
     - 여러 acceptor 설계에서 thundering herd 완화용이며 일반적인 event
       ownership 보장으로 해석하면 안 된다.

ET에서 ``EAGAIN``까지 읽어야 하는 이유
--------------------------------------

예를 들어 socket receive buffer에 64 KiB가 들어왔고 핸들러가 4 KiB만 읽고
반환했다고 하자. LT는 상태가 여전히 readable이므로 다시 ready list에 넣는다.
ET는 현재 전달 후 자동 재등록하지 않으므로, 애플리케이션이 남은 데이터를
drain하지 않으면 다음 상태 변화가 있기 전까지 처리가 멈춘 것처럼 보일 수 있다.

실전 패턴은 다음과 같다.

.. code-block:: c

   for (;;) {
           ssize_t n = read(fd, buf, sizeof(buf));

           if (n > 0) {
                   consume(buf, n);
                   continue;
           }
           if (n == 0) {
                   close_connection(fd);
                   break;
           }
           if (errno == EAGAIN || errno == EWOULDBLOCK)
                   break;
           if (errno == EINTR)
                   continue;
           handle_error(fd, errno);
           break;
   }

``accept()``와 ``write()``도 같은 원칙을 적용한다. 한 연결이 너무 많은 작업을
독점하지 않도록 처리량 budget을 두고, budget이 끝났지만 아직 작업이 남았다면
애플리케이션 자체 runnable queue에 다시 넣는 방식도 고려해야 한다.

락과 경쟁 조건
==============

.. list-table::
   :header-rows: 1
   :widths: 24 28 48

   * - 락
     - 주 보호 대상
     - 필요한 이유
   * - ``epnested_mutex``
     - epoll 중첩 topology 검사
     - 서로 반대 방향의 동시 ADD가 cycle을 만드는 경쟁을 전역 직렬화한다.
   * - ``ep->mtx``
     - RB tree 변경, 전달 scan, 제거
     - ``copy_to_user()``와 대상 ``->poll()``이 잠들 수 있는 process 문맥 락이다.
   * - ``ep->lock``
     - ``rdllist``, ``ovflist``, wait/wakeup 전환
     - 알림 콜백이 IRQ 문맥에서 실행될 수 있어 IRQ-safe spinlock이 필요하다.
   * - ``file->f_lock``
     - 대상 파일의 ``f_ep`` watcher 목록
     - 한 파일을 감시하는 여러 epoll 인스턴스와 close 경로를 조정한다.

중첩 epoll
----------

epoll fd도 poll 가능한 파일이므로 다른 epoll에 등록할 수 있다. 내부 epoll의
``poll_wait``가 바깥 epoll의 알림 경로가 된다. 하지만 cycle이나 과도한 wakeup
경로는 deadlock, 깊은 재귀, wakeup 증폭을 만들 수 있다.

``ep_loop_check()``과 ``reverse_path_check()``은 ``EPOLL_CTL_ADD`` 때 이를
검사하고, 중첩 깊이는 ``EP_MAX_NESTS``로 제한한다. 전역
``epnested_mutex``는 ``A -> B``와 ``B -> A`` ADD가 동시에 검사되어 둘 다
통과하는 경쟁을 막는다.

close와 fd 재사용
-----------------

명시적 ``EPOLL_CTL_DEL``은 ``ep_remove()``으로 들어간다. 반면 감시 대상
``struct file``의 마지막 참조가 닫히면 ``__fput()``에서
``eventpoll_release_file()``이 호출되어 해당 파일을 감시하던 모든 epoll에서
정리한다.

여기서 fd 숫자와 open file description을 구분해야 한다. ``dup()``한 fd가
남아 있으면 원래 fd 숫자를 닫아도 ``struct file``은 살아 있고 등록도 즉시
사라지지 않을 수 있다. 이후 같은 숫자가 새 파일에 재사용되면 userspace가
``event.data``를 단순 fd 숫자로만 관리할 경우 오래된 연결과 새 연결을
혼동할 수 있다. 또한 등록 키에 fd 숫자도 포함되므로, 등록에 쓴 fd를 먼저
닫은 뒤 다른 duplicate fd로 같은 등록을 DEL할 수 있다고 가정해서는 안 된다.

실전에서는 다음 중 하나가 필요하다.

* 등록에 사용한 정확한 fd가 유효할 때 ``EPOLL_CTL_DEL``을 명시적으로
  수행한 다음 close한다.
* ``event.data``에 fd만 넣지 않고 connection 객체 또는 generation을 식별할
  수 있는 값을 넣는다.
* 이벤트 처리 객체의 수명과 close를 worker 간에 명시적으로 동기화한다.

복잡도와 성능 해석
==================

``epoll은 O(1)``이라는 표현은 너무 단순하다.

* ``EPOLL_CTL_ADD/MOD/DEL``의 interest set 조회는 RB tree 때문에 일반적으로
  ``O(log N)``이다. ADD는 대상 ``->poll()`` 및 wait queue 연결 비용도 든다.
* 알림 시 이미 연결되지 않은 ``epitem``을 ready list에 넣는 일은 보통
  ``O(1)``이다.
* ``epoll_wait()``은 모든 N개 등록을 매번 선형 순회하지 않고, ready 후보와
  실제 반환한 K개를 중심으로 동작한다. 다만 각 후보의 재-poll, userspace
  복사, 락 경합 비용은 남는다.
* 동일 항목의 반복 wakeup은 ready list 한 자리로 합쳐진다. 이것은 메모리와
  스캔 비용에는 유리하지만 알림 횟수 보존이 필요한 작업 큐를 대신하지 못한다.

성능 문제를 분석할 때는 등록 수 N만 보지 말고 다음을 함께 본다.

* 한 epoll 인스턴스를 공유하는 worker 수와 ``ep->lock`` 경합
* 한 번에 반환하는 ``maxevents``와 syscall 횟수
* 콜백 빈도와 실제 I/O로 소비되는 양
* ET drain loop의 공정성 budget
* 연결별 상태 객체의 cache locality
* acceptor 또는 worker 사이의 wakeup 분배 방식

백엔드 설계 체크리스트
======================

* ET를 사용한다면 socket, listener, pipe를 nonblocking으로 설정하고
  ``EAGAIN``까지 처리한다.
* ``EPOLLIN``과 ``EPOLLOUT``은 작업 완료가 아니라 지금 시도해 볼 수 있다는
  readiness다. 실제 syscall은 여전히 부분 성공, ``EINTR``, ``EAGAIN``을
  반환할 수 있다.
* ``EPOLLOUT``을 항상 등록하면 writable socket이 계속 ready가 되어 불필요한
  wakeup과 CPU 사용을 만들 수 있다. 보낼 데이터가 쌓였을 때만 관심 mask에
  추가하고 queue를 비우면 제거하는 방식을 고려한다.
* ``EPOLLERR``와 ``EPOLLHUP``은 ADD/MOD 시 커널이 관심 mask에 포함한다.
  반환 mask를 함께 검사하고 socket error는 ``getsockopt(SO_ERROR)`` 등으로
  확인한다.
* 여러 worker가 같은 연결을 동시에 처리하지 않게 하려면
  ``EPOLLONESHOT`` + 처리 후 rearm 패턴이 유용하다. 단, 모든 종료 경로에서
  rearm 또는 close가 수행되는지 확인한다.
* ``event.data``는 커널이 해석하지 않는 opaque 값이다. 포인터를 넣는 경우
  userspace 객체가 반환 이벤트보다 먼저 해제되지 않도록 수명을 보장한다.
* 한 이벤트에서 무제한으로 read/write하면 hot connection이 event loop를
  독점할 수 있다. 처리 budget과 재스케줄 정책을 둔다.
* epoll 반환 순서나 worker 소유권에 강한 보장이 있다고 가정하지 않는다.

kqueue와 비교
=============

API 대응
-------

다음 대응은 학습용 근사치이며 완전히 같은 의미는 아니다.

.. list-table::
   :header-rows: 1
   :widths: 28 28 44

   * - Linux epoll
     - BSD kqueue
     - 차이/주의점
   * - ``epoll_create1()``
     - ``kqueue()``
     - 둘 다 커널 event queue를 나타내는 descriptor를 만든다.
   * - ``epoll_ctl(ADD/MOD/DEL)``
     - ``kevent()``의 change list와 ``EV_ADD``, ``EV_ENABLE``,
       ``EV_DISABLE``, ``EV_DELETE``
     - kqueue는 변경 제출과 이벤트 수신을 한 ``kevent()`` 호출에서 함께 할
       수 있다.
   * - ``epoll_wait()`` / ``epoll_pwait2()``
     - ``kevent()``의 event list와 timeout
     - signal mask를 원자적으로 바꾸는 Linux pwait 계열의 세부 API는 다르다.
   * - ``EPOLLIN``
     - ``EVFILT_READ``
     - kqueue의 ``data``/``fflags``는 filter별 추가 정보를 제공할 수 있다.
   * - ``EPOLLOUT``
     - ``EVFILT_WRITE``
     - writable의 정확한 의미는 대상 object와 OS 구현에 의존한다.
   * - ``EPOLLET``
     - 흔히 ``EV_CLEAR``와 비교
     - 내부 의미와 filter별 동작이 같지는 않다. 모두 "edge와 유사하니
       drain한다"는 애플리케이션 규칙 수준에서 비교하는 편이 안전하다.
   * - ``EPOLLONESHOT``
     - ``EV_ONESHOT``
     - epoll은 보통 ``MOD``로 rearm하고 kqueue flag 조합의 재등록 규칙은
       대상 BSD를 확인해야 한다.
   * - ``event.data``
     - ``udata``
     - 둘 다 userspace context를 되돌려주는 용도로 쓰지만 타입과 ABI가 다르다.
   * - ``EPOLLEXCLUSIVE``
     - 이식 가능한 직접 대응 없음
     - wakeup 분배 정책을 같은 것으로 가정하지 않는다.

가장 큰 설계 차이
-----------------

``epoll``은 **poll 가능한 file descriptor 중심**이다. Linux는 서로 다른 사건을
fd 모델로 통합하기 위해 ``signalfd``, ``timerfd``, ``eventfd``, ``inotify``와
같은 별도 인터페이스를 조합한다.

``kqueue``는 **filter 중심**이다. ``EVFILT_READ``/``EVFILT_WRITE``뿐 아니라
signal, timer, process, vnode, user event 등의 사건을 filter로 표현할 수 있다.
즉 epoll의 질문은 대체로 "이 fd가 지금 I/O 가능한가?"이고, kqueue의 질문은
"이 ident에 이 filter 조건의 사건이 있는가?"에 더 가깝다.

또한 kqueue의 반환 ``data``와 ``fflags``는 filter별 의미를 가질 수 있다.
epoll의 반환은 readiness mask와 등록자가 넣어 둔 opaque ``data``가 중심이므로,
구체적인 byte 수나 상태는 실제 I/O 또는 별도 syscall로 확인하는 경우가 많다.

공통점
------

* 커널에 관심 대상을 등록하고 준비된 대상 위주로 반환해 ``select()``/``poll()``
  방식의 전체 fd set 반복 복사와 선형 검사를 줄인다.
* readiness는 실제 I/O 성공을 예약하지 않는다. 여러 consumer가 있으면 반환
  직후 상태가 바뀔 수 있으므로 nonblocking I/O와 ``EAGAIN`` 처리가 필요하다.
* 사용자 context(``event.data`` 또는 ``udata``)의 수명 관리는 애플리케이션
  책임이다.
* edge/clear 계열 모드를 잘못 사용하면 처리할 데이터가 남았는데도 event loop가
  멈춘 것처럼 보이는 버그를 만들 수 있다.

면접에서 자주 나오는 질문
=========================

왜 RB tree와 ready list가 둘 다 필요한가?
-----------------------------------------

RB tree는 ADD/MOD/DEL 때 특정 등록을 찾는 interest set이고, ready list는
``epoll_wait()``이 처리할 후보만 모은 ready set이다. 하나로 합치면 등록 조회와
준비 이벤트 순회라는 서로 다른 접근 패턴을 효율적으로 만족시키기 어렵다.

왜 ``epoll_wait()``이 모든 fd를 순회하지 않아도 되는가?
--------------------------------------------------------

ADD 시 각 대상 파일의 wait queue에 ``ep_poll_callback()``을 등록하기 때문이다.
상태를 만든 producer가 wakeup할 때 ready list를 갱신하므로 consumer는 그
목록을 중심으로 처리한다. 비용이 사라지는 것이 아니라 전체 순회에서 등록
비용과 상태 변화 시 콜백 비용으로 이동한 것이다.

epoll은 이벤트 큐인가?
----------------------

정확히는 readiness set에 가깝다. 같은 ``epitem``은 ready list에 한 번만
연결되고 반복 wakeup 횟수는 합쳐진다. 개별 작업을 하나도 잃지 않고 세어야
한다면 socket 수신 버퍼, eventfd counter, 별도 message queue처럼 payload나
count를 보존하는 장치가 필요하다.

왜 ET는 nonblocking이어야 하는가?
---------------------------------

한 번 깨운 뒤 drain loop가 blocking syscall에서 멈추면 event loop 전체가
정지할 수 있다. 반대로 일부만 읽고 빠져나오면 아직 남은 데이터에 대해 새
edge가 오지 않을 수 있다. nonblocking으로 ``EAGAIN``까지 처리해야 두 문제를
모두 피할 수 있다.

lost wakeup은 어떻게 막는가?
----------------------------

``ep_poll()``은 마지막 ready 확인과 wait queue 삽입을 ``ep->lock`` 아래에서
수행한다. 알림 측 ``ep_poll_callback()``도 같은 락 아래에서 ready list를
갱신하고 waiter를 깨운다. 따라서 consumer가 빈 상태를 본 뒤 실제로 sleep하기
전 producer 알림을 놓치는 창이 없다.

``ovflist``는 왜 필요한가?
--------------------------

ready event를 userspace로 복사하는 동안 spinlock을 유지할 수 없기 때문이다.
``ep_start_scan()``은 현재 ready list를 private batch로 떼고, 동시에 온
콜백은 ``ovflist``로 보낸다. ``ep_done_scan()``이 두 목록을 합쳐 전달 중
발생한 readiness를 보존한다.

``EPOLLONESHOT``과 ``EPOLLEXCLUSIVE``는 같은 문제를 푸는가?
-----------------------------------------------------------

아니다. ONESHOT은 한 등록을 전달 후 비활성화해 연결 단위의 동시 처리를
제어하는 데 유용하다. EXCLUSIVE는 동일 대상의 여러 epoll 등록으로 wakeup이
과도하게 확산되는 thundering herd를 줄이는 기능이다.

epoll과 kqueue 중 무엇이 더 좋은가?
----------------------------------

운영체제 API와 workload에 따라 다르다. fd 중심으로 timer/signal까지 조합하는
Linux 설계와 다양한 사건을 filter로 직접 표현하는 BSD 설계의 차이를 설명하는
것이 핵심이다. 단순 benchmark 숫자나 ``O(1)``이라는 표현만으로 우열을
결론내리면 등록 변경 빈도, ready 비율, worker 구조, 락 경합 같은 실제 비용을
놓치게 된다.

관찰 및 실습 포인트
===================

``/proc/<pid>/fdinfo/<epfd>``
  ``ep_show_fdinfo()``가 등록된 target fd, event mask, data 등을 출력한다.
  실행 중 interest set을 확인할 때 유용하다.

``tools/testing/selftests/filesystems/epoll/``
  여러 waiter와 wakeup 경쟁을 다루는 selftest가 있다.

``tools/perf/bench/epoll-wait.c``
  하나의 epoll을 여러 worker가 공유하는 경우와 worker별 epoll을 사용하는
  경우를 비교할 수 있다.

``tools/perf/bench/epoll-ctl.c``
  동시 ``epoll_ctl()`` 및 중첩 구성의 비용을 실험할 수 있다.

간단한 userspace 프로그램은 다음 syscall을 trace하면서 실행해 볼 수 있다.

.. code-block:: console

   $ strace -f -e epoll_create1,epoll_ctl,epoll_wait,epoll_pwait2 ./server

추천 코드 리딩 순서
===================

#. ``include/uapi/linux/eventpoll.h``에서 사용자 ABI와 flag를 본다.
#. ``struct eventpoll``, ``struct epitem``, ``struct eppoll_entry``만 읽는다.
#. ``do_epoll_create()``에서 epoll fd와 객체가 연결되는 지점을 본다.
#. ``do_epoll_ctl_file()``의 switch와 ``ep_insert()``만 따라간다.
#. ``ep_ptable_queue_proc()``와 ``ep_poll_callback()``으로 producer 알림 경로를
   연결한다.
#. ``ep_poll()``에서 마지막 ready 확인과 sleep 전환을 확인한다.
#. ``ep_start_scan()`` -> ``ep_deliver_event()`` -> ``ep_done_scan()``에서 동시
   알림과 LT/ET/ONESHOT 처리를 확인한다.
#. 마지막으로 제거 경로와 중첩 epoll 검사를 읽는다. 이 부분은 수명 및 동시성
   심화 질문을 준비할 때 유용하다.

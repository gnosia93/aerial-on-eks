# aerial-on-eks


테스트용 신호 발생기(Signal Generator)와 처리기(PyAerial)를 모두 쿠버네티스(k8s) 환경에서 구현한다면, "가상화된 무선 통신 테스트베드" 아키텍처가 됩니다. 실제 하드웨어 없이도 5G/6G 환경을 시뮬레이션할 수 있는 구조입니다.

[실시간 신호 처리 테스트 아키텍처]

### 1. 주요 구성 요소 (Pods) ###
* Signal Gen Pod (Tx): 테스트용 IQ 신호를 생성하는 송신부입니다. NVIDIA Sionna 라이브러리를 사용하여 실제 물리적 채널(페이딩, 노이즈 등)이 적용된 데이터를 실시간으로 생성합니다.
* PyAerial Pod (Rx): 개발자가 만든 알고리즘이 돌아가는 수신부입니다. 생성된 신호를 받아 GPU에서 복조(Demodulation) 및 디코딩을 수행합니다.
* Metrics/Dashboard Pod: 처리 지연 시간(Latency), 처리량(Throughput), 에러율(BLER)을 실시간으로 모니터링합니다.

### 2. 데이터 통신: "The Fast Path" ###
실시간성을 위해 Pod 간에 일반적인 HTTP 통신을 쓰지 않습니다.
* Shared Memory (공유 메모리): 동일한 노드 내에 두 Pod을 배치하고, POSIX Shared Memory를 통해 데이터를 주고받습니다. 복사 과정이 없어 지연 시간이 거의 제로에 가깝습니다.
* gRPC (Streaming): 노드 간 통신이 필요한 경우, gRPC 스트리밍을 사용하여 바이너리 IQ 데이터를 끊김 없이 전달합니다.

### 3. 하드웨어 가속 및 오케스트레이션 ###
* GPU 할당: NVIDIA Device Plugin을 통해 두 Pod 모두에 GPU 자원을 할당합니다. (하나는 신호 생성용, 하나는 처리용)
* Node Affinity: 성능 최적화를 위해 두 Pod을 반드시 동일한 물리 노드에 배치하도록 Node Affinity 설정을 적용합니다.

[추천 배포 아키텍처 다이어그램]
* 인프라: AWS EC2 G5(GPU) 또는 NVIDIA Grace Hopper 인스턴스
* 컨테이너 런타임: NVIDIA Container Runtime (GPU 접근 필수)

* 데이터 흐름:
[Sionna Pod] (신호 생성) → [Shared Memory / CUDA IPC] → [PyAerial Pod] (알고리즘 실행)

* 결과 모니터링:
[PyAerial Pod] → [Prometheus] → [Grafana Dashboard]

### [구현 시 핵심 팁] ###
* Sionna 연동: NVIDIA PyAerial과 Sionna는 찰떡궁합입니다. Sionna로 가상의 6G 채널 환경을 만들고, PyAerial로 이를 뚫는 알고리즘을 테스트하세요.
* Helm Chart 활용: 이 복잡한 환경(GPU 설정, 메모리 공유 등)을 한 번에 띄우려면 Helm을 사용하여 패키징하는 것이 정신 건강에 이롭습니다.


---

AWS Graviton(ARM) 프로세서를 도입한다면, GPU 가속이 필수적인 '신호 생성 및 시뮬레이션' 영역보다는 '데이터 처리, 제어부, 그리고 로깅' 영역에서 최상의 가성비를 뽑아낼 수 있습니다.
Sionna 워크샵 구조 내에서 추천하는 Graviton 적용 포인트는 다음과 같습니다.

#### 1. 전송 데이터 처리 및 관제 Pod (Control Plane & Data Aggregator) ####
Sionna가 생성한 방대한 IQ 샘플 데이터를 받아 PyAerial로 넘기기 전, 데이터를 가공하거나 시뮬레이션 상태를 모니터링하는 매니지먼트 서버 역할에 적합합니다.
이유: 단순 데이터 정렬, 로그 수집, 대시보드(Grafana 등) 구동은 CPU 의존도가 높으며, AWS Graviton3는 일반 인텔 인스턴스 대비 가격 대비 성능이 최대 40% 좋습니다.

#### 2. PyAerial Pod의 Non-L1 레이어 (L2/L3 스택) ####
물리 계층(L1)은 GPU 가속이 필수지만, 상위 계층인 MAC, RLC, PDCP 등 L2/L3 프로토콜 스택은 높은 패킷 처리량과 멀티코어 성능이 중요합니다.
이유: Graviton은 코어당 성능이 뛰어나고 레이턴시에 민감한 네트워크 워크로드 처리에 효율적입니다. AWS 공식 블로그에 따르면 고성능 컴퓨팅(HPC) 및 네트워크 집약적 작업에서 우수한 효율을 보입니다.

#### 3. CI/CD 빌드 및 테스트 환경 ####
Sionna 관련 소스 코드를 빌드하거나 경량 단위 테스트(Unit Test)를 수행하는 Jenkins/GitHub Actions Runner를 Graviton 기반으로 운영하세요.
이유: 가장 적은 리스크로 즉시 비용 절감 효과를 볼 수 있는 영역입니다.

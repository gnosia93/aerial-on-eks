# aerial-on-eks

![](https://github.com/gnosia93/aerial-on-eks/blob/main/images/workshop-arch-aerial.png)

### _O-RAN(Open RAN) 아키텍처_ ###

* Sionna GPU Pod -> gRPC -> PyAreal GPU Pod
* Sinnoa GPU Pod -> gRPC -> Graviton L2/L3 stack -> gRPC -> PyAreal GPU Pod 
* PyAreal GPU Pod -> Prometheus/Grafna dashboard (BER 측정/시각화)
* 지원 토폴로지 
  * Shared Memory (Zero-Copy Interface / Sionna 와 PyAreal 가 동일 Pod를 사용하는 경우만 지원)
  * UDP (eCPRI / Raw Socket 기반)
  * gRPC TPC/IP  

### _Topics_ ###

* [C1. VPC 생성하기](https://github.com/gnosia93/aerial-on-eks/tree/main/tf)

* [C2. EKS 클러스터 생성하기](https://github.com/gnosia93/aerial-on-eks/blob/main/lesson/c2-provision-eks.md)

* [C3. GPU 노드준비 하기](https://github.com/gnosia93/aerial-on-eks/blob/main/lesson/c3-prepare-gpu-node.md)

* [C4. 시그널 제너레이터 배포](https://github.com/gnosia93/aerial-on-eks/blob/main/apps/signal-generator.md)

* [C5. 채널 리시버 배포](https://github.com/gnosia93/aerial-on-eks/blob/main/apps/channel-receiver.md)

* [C6. 그라비톤 기반 L2/L3 스택 구현](https://github.com/gnosia93/aerial-on-eks/blob/main/apps/graviton-L2L3-stack.md)

* [C7. 그라파나로 BER 시각화](https://github.com/gnosia93/aerial-on-eks/blob/main/lesson/c7-monitoring.md)

* [C8. Gitlab CI/CD]


### _Instance Consideration_ ### 

  * G 타입 (Shared Memory / Socket) - 1/2, 1/4, 1/8, 1, 4, 8 GPU per Instance
    * 데이터 복사시 CPU가 개입하여 인터럽트를 처리하고 데이터를 메모리에 배치.
    * OS 스케줄링에 따라 지연 시간이 튀는 Jitter가 발생할 수 있음.
    * PCIe 토폴로지 단절: P2P(및 RDMA)가 작동하려면 데이터가 CPU를 거치지 않고 PCIe 스위치를 통해 직접 흘러야 하지만 G 타입은 하이퍼바이저(가상화 계층)가 GPU와 네트워크 카드 사이의 직접적인 통신 경로를 노출하지 않음.
  
  * P 타입 (외부 - GPUDirect RDMA / 내부 - NVLink) - 8 GPU per Instance
    * NIC 과 GPU 메모리가 직접 통신.
    * NVIDIA Aerial SDK가 요구하는 1ms 미만의 엄격한 TTI(Transmission Time Interval) 지원.

### _Appendix_ ###

* [VS Code 노트북 실행하기](https://github.com/gnosia93/aerial-on-eks/blob/main/lesson/c4-vs-code-notebook.md)

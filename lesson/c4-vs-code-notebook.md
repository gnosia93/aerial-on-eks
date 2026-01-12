EKS 클러스터에서 NVIDIA TensorFlow 컨테이너를 기반으로 VS Code(code-server)와 Jupyter Notebook을 동시에 사용할 수 있는 통합 환경 YAML 파일입니다.
이 설정은 Sionna 실행을 위한 GPU 할당 및 외부 접속을 위한 LoadBalancer 설정을 포함합니다.
📄 EKS 통합 개발 환경 설정 (sionna-dev.yaml)

```
cat << EOF | kubectl apply -f - 
apiVersion: v1
kind: Service
metadata:
  name: sionna-dev-service
spec:
  type: LoadBalancer
  # 아래에 허용할 IP 대역(CIDR 형식)을 입력한다.
  loadBalancerSourceRanges:
    - "122.36.213.114/32"              # 특정 IP 1개만 허용할 때
  selector:
    app: sionna-dev
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080                   # VS Code & Jupyter 통합 포트
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sionna-dev-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sionna-dev
  template:
    metadata:
      labels:
        app: sionna-dev
    spec:
      containers:
      - name: sionna-dev-container
        # VS Code가 포함된 최신 NVIDIA TensorFlow 이미지 사용
        image: codercom/code-server:latest 
        ports:
        - containerPort: 8080
        env:
        - name: PASSWORD
          value: "code!@#"         # 접속 비밀번호
        resources:
          limits:
            nvidia.com/gpu: 1          # GPU 가속 활성화 (필요시)
            memory: "8Gi"
            cpu: "4"
        volumeMounts:
        - name: workspace-storage
          mountPath: /home/coder/project
      # Sionna 설치 및 환경 구성을 위한 초기화 (필요시 커스텀 이미지 권장)
      volumes:
      - name: workspace-storage
        emptyDir: {}
EOF
```
쿠버네티스 서비스를 조회한다.
```
kubectl get svc
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)        AGE
kubernetes           ClusterIP      172.20.0.1       <none>                                                                         443/TCP        38m
sionna-dev-service   LoadBalancer   172.20.131.207   a821da87194044356aca03399dc2d776-1491597413.ap-northeast-2.elb.amazonaws.com   80:30123/TCP   77s
```

* VS Code 터미널에서 pip install sionna를 실행하여 환경을 완성합니다


    

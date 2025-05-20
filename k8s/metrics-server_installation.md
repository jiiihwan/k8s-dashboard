# metrics-server installation
- namespace : 원래 kube-system에 설치되지만 kubernetes-dashboard namespace에 깔리도록 수정

## metrics-server의 yaml파일 작성
- 주요수정사항
  - 기본 설치 namespace가 kubernetes-dashboard
  - kube-system namespace의 metadata를 받아올수있게 함
  - 마스터노드에 깔리도록 NodeSelector 수정
- components.yaml 로 작성한다

[components.yaml](components.yaml) 참고

## 설치
```bash
kubectl apply -f components.yaml
kubectl get pods -A -o wide
kubectl get svc -n kubernetes-dashboard

#삭제하는법
kubectl delete -f components.yaml
```

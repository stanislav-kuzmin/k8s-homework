# k8s-homework

Request and retrieve token:
kubectl create -f token-request.yaml --as=system:serviceaccount:homework:cd
kubectl get token cd-token -n homework -o jsonpath='{.status.token}'

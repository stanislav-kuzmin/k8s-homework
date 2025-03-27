# k8s-homework

Horizontal pod autoscaler:

kubectl apply -f php-apache.yaml

kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

kubectl describe hpa cm-test php-apache


Vertical pod autoscaler:

git clone https://github.com/kubernetes/autoscaler.git

cd autoscaler

./hack/vpa-up.sh

kubectl get pods -n kube-system | grep vpa

kubectl apply -f php-apache-vpa.yaml

kubectl apply -f vpa.yaml

kubectl describe vpa my-app-vpa

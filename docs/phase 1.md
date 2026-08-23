kubectl create namespace argocd

kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable manifests/install.yaml

kubectl port-forward svc/argocd-server -n argocd 8080:443

kubectl get secrets -n argocd

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

k8s delete secrets argocd-initial-admin-secret -n argocd
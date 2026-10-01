# kubernetes_practice:
step-1: first create namespace.yaml file 
    kubectl apply -f namespace.yaml
    kubectl config set-context --current --namespace=assign1  |to set defualt namespace to avoid -n again|again
    

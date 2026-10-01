# kubernetes_practice:
step-1: first create namespace.yaml file 
    kubectl apply -f namespace.yaml
    kubectl config set-context --current --namespace=assign1  |to set defualt namespace to avoid -n again|again
step-2 : run redis deployment
    kubectl apply -f redis-deployment.yaml --dry-run=client | to dry run yaml    

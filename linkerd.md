## AKS + Linkerd V2 ##

**Get Linkerd CLI and setup your environment**

    LINKERD_VERSION=stable-2.7.1
    curl -sLO "https://github.com/linkerd/linkerd2/releases/download/$LINKERD_VERSION/linkerd2-cli-$LINKERD_VERSION-linux"
    sudo cp ./linkerd2-cli-$LINKERD_VERSION-linux /usr/local/bin/linkerd
    sudo chmod +x /usr/local/bin/linkerd

**Generate the bash completion file and source it in your current shell**

    mkdir -p ~/completions && linkerd completion bash > ~/completions/linkerd.bash
    source ~/completions/linkerd.bash

**Source the bash completion file in your .bashrc so that the command-line completions are permanently available in your shell**

    echo "source ~/completions/linkerd.bash" >> ~/.bashrc

**Validate Linkerd version**

    linkerd version
        Client version: stable-2.7.1
        Server version: unavailable

**Install Linkerd Control Plane**

    linkerd check --pre
    linkerd install | kubectl apply -f -
    linkerd check
    linkerd version
    kubectl -n linkerd get deployments
    kubectl -n linkerd get services
    kubectl -n linkerd get pods
    linkerd dashboard &
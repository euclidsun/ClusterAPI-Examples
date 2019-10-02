1. Project Home Page:
  the Cluster API Provider OpenStack repo is here: https://github.com/kubernetes-sigs/cluster-api-provider-openstack/tree/release-0.1

2. Installation Instructions:
2.1 prepare the environment:
    a. command line tools: kubectl, kind, go, docker
    b. download the clusterctl source code, and compile the executable binary for clusterctl
    c. prepare the openstack environment, you need below things:
        - uuid of network, security group for k8s cluster
        - image name for k8s nodes creation
        - the floating ip to ssh the k8s nodes
        - the keypair for ssh the k8s nodes
        - uuid of project, user name, password, domain name, region name
    d. the client pc you are running the cli tools should have the remote access to the Nova Node you will be creating.

3. step-by-step instructions:
3.1 get the required command line tools installed
    Docker: https://docs.docker.com/install/linux/docker-ce/ubuntu/
    Kind: https://github.com/kubernetes-sigs/kind
    Go: download from https://golang.org/dl/
        tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz
        export PATH=$PATH:/usr/local/go/bin
    yq: a lightweight and portable command-line YAML processor, download from  https://github.com/mikefarah/yq/releases

3.2 compile the clusterctl, notes: the command is a bit different for release-0.1
    git clone https://github.com/kubernetes-sigs/cluster-api-provider-openstack $GOPATH/src/sigs.k8s.io/cluster-api-provider-openstack
    cd $GOPATH/src/sigs.k8s.io/cluster-api-provider-openstack/
    make clusterctl

    Or you could compile the entire docker image:
    export REGISTRY=your_repo_name #your docker hub repo name
    export VERSION=your_version #the verion you want to make in repo
    export DOCKER_USERNAME=your_docker_hub_user #upload with your name
    export DOCKER_PASSWORD=your_docker_hub_password #upload with your credential
    make upload-images

3.3 prepare the openstack yaml files:

    - cloud.yaml
    Goto examples under source code /root/go/src/sigs.k8s.io/cluster-api-provider-openstack/cmd/clusterctl/examples/openstack
    Create a file cloud.yaml, which will have your openstack access info, example content as below.
    notes: cacert should point to the file of the cert of auth_url.

    clouds:
    openstack:
        auth:
            auth_url: "https://192.168.112.200:5000"
            username: "admin"
            password: "password"
            project_id: f2e86350bc41453481ff1574550a4132
            domain_name: "default"
            user_domain_name: "default"
        region_name: "RegionOne"
        interface: "public"
        identity_api_version: 3
        verify: false
        cacert: ./ca.pem

    - generate yaml files for cluster api, it will output to ./out folder by default.
    ./generate-yaml.sh ./clouds.yaml openstack ubuntu

    - modify the ./out/cluster.yaml file:
    update cluster.yaml for the desired k8s cluster.
    please be noted that the pod cidr value MUST be the same as the CNI (calico or flannel) component value in their yaml files.
    please refer to this file ./examples/openstack/provider-component/user-data/ubuntu/master-user-data.sh, 
    this file will be run by cloud-init inside master node during the initialization.

    - modify the ./out/yaml files with your openstack environment info:
    sed -i 's/<Available Floating IP>/your_floating_ip/' examples/openstack/out/*.yaml
    Please be noted there are multiple floating ip need to update.

    sed -i 's/<Kubernetes Network ID>/your_neutron_network_id/g' examples/openstack/out/*.yaml
    sed -i 's/<Security Group ID>/your_neutron_secgroup_id/g' examples/openstack/out/*.yaml
    sed -i 's/<Image Name>/your_image_NAME/g' examples/openstack/out/*.yaml
    sed -i 's/<apiServerLoadBalancer or master IP>/your_master_node_floating_ip/g' examples/openstack/out/*.yaml

    - create a keypair in openstack
    openstack keypair create --public-key ~/.ssh/openstack_tmp.pub cluster-api-provider-openstack

    - creat the cluster:
    ./clusterctl create cluster --bootstrap-type kind --provider openstack \
    -c examples/openstack/out/cluster.yaml -m examples/openstack/out/machines.yaml \
    -p examples/openstack/out/provider-components.yaml

    - during the bootstrap process, the cluster CRD/info will be moved from bootstrap into target cluster, and master node will create the rest worker nodes.
    after a few minutes, you should be able to see your nova instances be created.
    # openstack server list
    +--------------------------------------+------------------------+--------+-------------------------------+--------+-----------+
    | ID                                   | Name                   | Status | Networks                      | Image  | Flavor    |
    +--------------------------------------+------------------------+--------+-------------------------------+--------+-----------+
    | b0a601ef-ee06-4737-9b11-93255942e841 | openstack-node-g7d7l   | ACTIVE | net10=10.10.10.11, 5.5.5.179  | ubuntu | m1.medium |
    | e59e4174-09f6-473f-a3fd-caa1df277b46 | openstack-master-9kkv9 | ACTIVE | net10=10.10.10.79, 5.5.5.171  | ubuntu | m1.medium |
    | 80d1d814-9ccf-4c98-9f8a-905c525ea975 | myvm                   | ACTIVE | net10=10.10.10.233, 5.5.5.200 | ubuntu | m1.small  |
    +--------------------------------------+------------------------+--------+-------------------------------+--------+-----------+

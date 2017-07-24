# Kubo on Bosh Lite

This is a guide to install [Kubo](https://github.com/cloudfoundry-incubator/kubo-release) on BOSH Lite

## Requirements

* Virtualbox

## Bosh Lite Installation
1. First checkout the `bosh-deployment` and `kubo-bosh-lite` repos
	```bash
	git clone https://github.com/cloudfoundry/bosh-deployment 
	git clone https://github.com/bstick12/kubo-bosh-lite.git
	```

1. Create a directory to store the our state and credentials
	```bash
	mkdir kubo
	```

1. Create BOSH deployment. Kubo has some additional requirements to the standard BOSH lite installation. 
	* powerdns
	* uaa
	* credhub

	```bash
	bosh create-env bosh-deployment/bosh.yml \
	--state kubo/state.json \
	-o bosh-deployment/virtualbox/cpi.yml \
	-o bosh-deployment/virtualbox/outbound-network.yml \
	-o bosh-deployment/bosh-lite.yml \
	-o bosh-deployment/bosh-lite-runc.yml \
	-o bosh-deployment/jumpbox-user.yml \
	-o bosh-deployment/powerdns.yml \
	-o bosh-deployment/uaa.yml \
	-o bosh-deployment/credhub.yml \
	--vars-store kubo/creds.yml \
	-v director_name="Bosh Lite Director" \
	-v internal_ip=192.168.50.6 \
	-v internal_gw=192.168.50.1 \
	-v internal_cidr=192.168.50.0/24 \
	-v dns_recursor_ip=192.168.50.6 \
	-v outbound_network_name=NatNetwork
	```

1. Get the admin password from the `kubo/creds.yml` and then login to your BOSH environment
	```bash
	bosh -e 192.168.50.6 login --ca-cert <(bosh int kubo/creds.yml --path /director_ssl/ca) 
	```

1. Create a alias `kubo` for the BOSH environment
	```bash
	bosh -e 192.168.50.6 alias-env kubo --ca-cert <(bosh int kubo/creds.yml --path /director_ssl/ca)
	```

1. Upload the stemcell for usage with KUBO
	```bash
	bosh -e kubo upload-stemcell "https://s3.amazonaws.com/bosh-core-stemcells/warden/bosh-stemcell-3421.11-warden-boshlite-ubuntu-trusty-go_agent.tgz"
	```

1. Upload the latest version of the Kubo release
	```bash
	bosh -e kubo upload-release "https://storage.googleapis.com/kubo-public/kubo-release-latest.tgz"
	```

1. Update the `cloud-config' for the kubo environment
	```bash
	bosh -e kubo update-cloud-config kubo-bosh-lite/cloud-config.yml
	```

1. Deploy Kubo

	This deployment of Kubo consists of only a single master, kuboetcd and two workers
 
	```bash
	bosh -e kubo deploy -d kubo-bosh-lite kubo-bosh-lite/kubo.yml -v kubernetes_master_host=10.240.0.2
	```

1. Add a route to allow access to the deployment of kubo
	```bash
	sudo ip route add 10.240.0.0/16 via 192.168.50.6	```

1. Set-up 'kubectl' to access the newly created cluster
	```bash
	CREDHUB_PWD=$(bosh int --path /credhub_cli_password kubo/creds.yml)
	CREDHUB_CA_CERT=$(bosh int --path /credhub_tls/ca kubo/creds.yml)
	credhub login -u credhub-cli -p ${CREDHUB_PWD} -s https://192.168.50.6:8844 --skip-tls-validation
	bosh int <(credhub get -n "/Bosh Lite Director/kubo-bosh-lite/tls-kubernetes" --output-json) --path=/value/ca > kubo/kubernetes.crt
	kubectl config set-cluster kubo-bosh-lite --server https://10.240.0.2:8443 --embed-certs=true --certificate-authority=kubo/kubernetes.crt 
	KUBERNETES_PWD=$(bosh int <(credhub get -n "/Bosh Lite Director/kubo-bosh-lite/kubo-admin-password" --output-json) --path=/value)
	kubectl config set-credentials "kubo-bosh-lite-admin" --token=${KUBERNETES_PWD}
	kubectl config set-context "kubo-bosh-lite" --cluster="kubo-bosh-lite" --user="kubo-bosh-lite-admin"
	kubectl config use-context "kubo-bosh-lite"
	kubectl get all
	```
	
## Troubleshooting

Use this command to connect to the internal BOSH machine 
```bash
bosh int kubo/creds.yml --path /jumpbox_ssh/private_key > kubo/jumpbox.key
chmod 600 kubo/jumpbox.key
ssh jumpbox@192.168.50.6 -i kubo/jumpbox.key
```



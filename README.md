# Kubernetes

5.0.1- Defininf variavel

	ACCOUNTID=XXXXXXXXXXXX

5.2-Criar permisssões de Read ao Cluster Kubernetes, e ao painel.

5.2.1.1- Criar police no IAM com nome "AccessKubernetesApiReadOnly.Policy" e atriubuir a mesma a role/grupo/user que deverá ter acesso.

5.2.1.2- Permissões:
OBS.: Alterar "XXXXXXXXXXXX" pelo ID da conta.

		{
		    "Version": "2012-10-17",
		    "Statement": [
		        {
		            "Effect": "Allow",
		            "Action": [
		                "eks:ListFargateProfiles",
		                "eks:DescribeNodegroup",
		                "eks:ListNodegroups",
		                "eks:ListUpdates",
		                "eks:AccessKubernetesApi",
		                "eks:ListAddons",
		                "eks:DescribeCluster",
		                "eks:DescribeAddonVersions",
		                "eks:ListClusters",
		                "eks:ListIdentityProviderConfigs",
		                "iam:ListRoles"
		            ],
		            "Resource": "*"
		        },
		        {
		            "Effect": "Allow",
		            "Action": "ssm:GetParameter",
		            "Resource": "arn:aws:ssm:*:{ACCOUNTID}:parameter/*"
		        }
		    ]
		}

5.2.2- Criar Role no IAM com nome "AccessKubernetesApiReadOnly.Role" atribuindo a política "AccessKubernetesApiReadOnly.Policy" com relação de confiança abaixo. 

	{
	    "Version": "2012-10-17",
	    "Statement": [
	        {
	            "Effect": "Allow",
	            "Principal": {
	                "AWS": "arn:aws:iam::{ACCOUNTID}:root"
	            },
	            "Action": "sts:AssumeRole",
	            "Condition": {}
	        }
	    ]
	}

5.2.3- Anexando Role a policy

	aws iam attach-role-policy --policy-arn arn:aws:iam::766176026207:policy/AccessKubernetesApiReadOnly.Policy --role-name AccessKubernetesApiReadOnly.Role

5.2.4- Criar politica "AssumeAccessKubernetesApiReadOnly.Policy" permitindo assumir a função "AccessKubernetesApiReadOnly.Role" criada anteriormente

	{
	    "Version": "2012-10-17",
	    "Statement": [
	        {
	            "Sid": "VisualEditor0",
	            "Effect": "Allow",
	            "Action": "sts:AssumeRole",
	            "Resource": "arn:aws:iam::XXXXXXXXXXXX:role/AccessKubernetesApiReadOnly.Role"
	        }
	    ]
	}

5.2.5- Cria grupo no IAM e atribuir a política "AssumeAccessKubernetesApiReadOnly.Policy" ao mesmo.

5.2.6.1- Baixar arquivo de refência para criação de Roles do Kubernetes.

	curl -o eks-readonly-access.yaml https://s3.us-west-2.amazonaws.com/amazon-eks/docs/eks-console-full-access.yaml

5.2.6.2- Modificar o nome do ClusterRole e ClusterRoleBinding para o que é mais apropriado.

	cat <<EOF> eks-readonly-access.yaml
		---
		apiVersion: rbac.authorization.k8s.io/v1
		kind: ClusterRole
		metadata:
		  name: eks-readonly-access-clusterrole
		rules:
		- apiGroups:
		  - ""
		  resources:
		  - nodes
		  - namespaces
		  - pods
		  - configmaps
		  - endpoints
		  - events
		  - limitranges
		  - persistentvolumeclaims
		  - podtemplates
		  - replicationcontrollers
		  - resourcequotas
		  - secrets
		  - serviceaccounts
		  - services
		  verbs:
		  - get
		  - list
		- apiGroups:
		  - apps
		  resources:
		  - deployments
		  - daemonsets
		  - statefulsets
		  - replicasets
		  verbs:
		  - get
		  - list
		- apiGroups:
		  - batch
		  resources:
		  - jobs
		  - cronjobs
		  verbs:
		  - get
		  - list
		- apiGroups:
		  - coordination.k8s.io
		  resources:
		  - leases
		  verbs:
		  - get
		  - list
		- apiGroups:
		  - discovery.k8s.io
		  resources:
		  - endpointslices
		  verbs:
		  - get
		  - list
		- apiGroups:
		  - events.k8s.io
		  resources:
		  - events
		  verbs:
		  - get
		  - list
		- apiGroups:
		  - extensions
		  resources:
		  - daemonsets
		  - deployments
		  - ingresses
		  - networkpolicies
		  - replicasets
		  verbs:
		  - get
		  - list
		- apiGroups:
		  - networking.k8s.io
		  resources:
		  - ingresses
		  - networkpolicies
		  verbs:
		  - get
		  - list
		- apiGroups:
		  - policy
		  resources:
		  - poddisruptionbudgets
		  verbs:
		  - get
		  - list
		- apiGroups:
	  - rbac.authorization.k8s.io
		  resources:
		  - rolebindings
		  - roles
		  verbs:
		  - get
		  - list
		- apiGroups:
		  - storage.k8s.io
		  resources:
		  - csistoragecapacities
		  verbs:
		  - get
		  - list
		---
		apiVersion: rbac.authorization.k8s.io/v1
		kind: ClusterRoleBinding
		metadata:
		  creationTimestamp: null
		  name: eks-readonly-access-binding
		roleRef:
		  apiGroup: rbac.authorization.k8s.io
		  kind: ClusterRole
		  name: eks-readonly-access-clusterrole
		subjects:
		  - apiGroup: rbac.authorization.k8s.io
		    kind: Group
		    name: eks_readonly_access_role-group
	EOF

5.2.7- Aplicar manifesto

	kubectl apply -f eks-readonly-access.yaml

5.2.8- Visualizar detalhes dos recursos criados

	kubectl describe clusterrole.rbac.authorization.k8s.io/eks-readonly-access-clusterrole
	kubectl describe clusterrolebinding.rbac.authorization.k8s.io/eks-readonly-access-binding

5.2.9.0- Editar configmap "aws-auth"

5.2.9.1- Adicioanr pemissões para role no config-map

	eksctl create iamidentitymapping --cluster eks-cluster --arn arn:aws:iam::766176026207:role/AccessKubernetesApiReadOnly.Role --group eks_readonly_access_role-group --no-duplicate-arns

5.2.9.2- Visualizar resultada da Edição

	cat aws-auth.yaml
		# Please edit the object below. Lines beginning with a '#' will be ignored,
		# and an empty file will abort the edit. If an error occurs while saving this file will be
		# reopened with the relevant failures.
		#
		apiVersion: v1
		data:
		  mapRoles: |
		    - groups:
		      - system:bootstrappers
		      - system:nodes
		      rolearn: arn:aws:iam::{ACCOUNTID}:role/AmazonEKSNodeRole
		      username: system:node:{{EC2PrivateDNSName}}
		     - groups:
		      - eks_readonly_access_role-group
		      rolearn: arn:aws:iam::{ACCOUNTID}:role/AccessKubernetesApiReadOnly.Role
		kind: ConfigMap
		metadata:
		  creationTimestamp: "2022-08-15T18:50:04Z"
		  name: aws-auth
		  namespace: kube-system
		  resourceVersion: "7105"
		  uid: 720499bb-f92e-40e6-b677-cbf32b3e0f0e	

5.2.9.3- (Opcional) Opcionalmente fazer a aplicação da autenticação alterando diretamente o aws-auth

	kubectl edit -n kube-system configmap/aws-auth

		...
		    - groups:
		      - system:bootstrappers
		      - system:nodes
		      rolearn: arn:aws:iam::{ACCOUNTID}:role/AmazonEKSNodeRole
		      username: system:node:{{EC2PrivateDNSName}}
		    - groups:
		      - eks_readonly_access_role-group
		      rolearn: arn:aws:iam::{ACCOUNTID}:role/AccessKubernetesApiReadOnly.Role
		...

5.2.10- Usuário deve executar update-kubeconig passando a role como parâmetro

	aws eks --region us-east-1 update-kubeconfig --name eks-cluster --role-arn arn:aws:iam::{ACCOUNTID}:role/AccessKubernetesApiReadOnly.Role

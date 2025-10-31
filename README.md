
### Setting up Basic Authentication on Minikube

1. Create a csv with user info:

```
password123,user1,u0001
password123,user2,u0002
password123,user3,u0003
password123,user4,u0004
password123,user5,u0005
```

2. Save it in the ***/tmp/users/*** folder inside the **minikube environment**
	a. To enter minikube -> `minikube ssh`
	b. Create a csv file with the above mentioned data or create a file in your local system to COPY it to the minikube environment:

```
Note: While you are in yout local system currently, type the following command:

minikube cp ./142/new.csv /tmp/users/
```

3.  **Edit the kube-apiserver.yaml** file inside minikube environment
	a. Incase if you are not allowed to edit the file, then you can simply run the `cat kube-apiserver.yaml` command then copy and paste the contents into a new file in your local system.

```
minikube ssh
cd /etc/kubernetes/manifests
cat or vi kube-apiserver.yaml

Edit

apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.49.2:8443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - --basic-auth-file=/tmp/users/new.csv  ---> Add this line
    - kube-apiserver
    - --advertise-address=192.168.49.2
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/var/lib/minikube/certs/ca.crt


```

4. In order to edit the yaml file, you will have to copy it to your local system.

```
minikube cp /etc/kubernetes/manifests/kube-apiserver.yaml 
```

5. **Copying the edited version into your minikube environment**: Once you're done with the edits on your local system, you can run the following command to copy the file from your ***local system*** to the ***minikube environment:***

```
minikube cp ./ReltivePath/new.csv /var/lib/k8s-users/new.csv

NOTE:
1. If you don't mention the name of the file in the destination path, then it will convert the folder into a file and write the copied content from the source to it.
2. To prevent this from happening follow the instructions below:
   
   
STEP 1: Change the Owner + User Privileges AND Defines WHO has access to it

1. Inside Minikub Environment -> chmod 700 k8s-users
2. Inside Minikub Environment -> chown root:root k8s-users

This will ensure that the CP command will not overwrite anything on the folder.

STEP 2: Must mention the name of the file + extension in the DESTINATION PATH
1. Example minikube cp ./142/new.csv /var/lib/k8s-users/new.csv
   
   
```

6. Once you move the new file into the ***manifests folder***, upon restarting the minikube, you will encounter various different errors, the possible reasons:
	a. The file path of the new users in the basic-auth flag is incorrect.
	b. There are 2 files of kube-apiserver.yaml with different names.
	c. **Solution:** Delete one of the files, and rename the edited version of the file to the original name(if not done already). Then restart minikube.
7. Upon restarting minikube, you will/ may encounter the following error:

```
🌟  Enabled addons: 
❌  Problems detected in kube-apiserver [f478c24b8faa]:
    Error: unknown flag: --basic-auth-file
    
NOTE:
1. This indicates that the flag --basic-auth-file has been deprecated
2. The modern way of authenticating new users is through RBAC.
```

8. **--basic-auth-file** is deprecated, therefore you will need to use **RBAC method** to authenticate a user.

### RBAC method

1. The `--basic-auth-file` flag is **deprecated**, and the modern, secure method is **RBAC (Role-Based Access Control)** using **certificates, roles, and role bindings**.
2. Create a **role:**

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

RUN:
kubectl creat -f role.yaml
```

3. Create a **role binding:**

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1 # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
  
RUN:
kubectl create -f rolebinding.yaml
```

4. To **authenticate**, you need to create a ***certificate*** that is validated and signed off by k8s ***Certificate Authority***. 
	a. **ca.crt** — Kubernetes’ Certificate Authority used to **verify and sign** user certificates.
	b. **ca.key** — The private key of the CA, required for signing.

#### RBAC Creating Certificates

1. Generate private key

```
openssl genrsa -out user1.key 2048
```

2. Generate CSR (certificate signing request)

```
openssl req -new -key user1.key -out user1.csr -subj "/CN=user1/O=dev"
```

3. **Sign CSR** using Kubernetes CA (found on master node)
	a. /etc/kubernetes/pki/ca.crt -> This location in minikube can be different, example --> /var/lib/minikube/certs

```
sudo openssl x509 -req -in user1.csr -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out user1.crt -days 365
  
NOTE:
1. The user1.csr MUST be in the same Environment / Local system. So you will have to copy it to minikube system and run the above command then.

------------------
to Copy the .csr file to minikube:

minikube cp ./user1.csr /var/lib/minikube/certs/user1.csr

-----------------------

Then RUN the following command INSIDE MINIKUBE -> minikube ssh then run:

sudo openssl x509 -req -in user1.csr -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out user1.crt -days 365
```

4. **Then configure credentials:**

```
kubectl config set-credentials user1 \
  --client-certificate=user1.crt \
  --client-key=user1.key
  
NOTE:
1. Both the user1.crt + user1.key should be in the same system and same directory of your local system, before you run the above command.
2. Copy the file from minikube environment out to your local system, then run the above set-credentials command
   
------------

To copy .crt file from minikube TO local system

1. Use the absolute path for both SOURCE + DESTINATION
   
minikube cp /var/lib/minikube/certs/user1.crt /mnt/e/MISCELLENEOUS/MY\ PROJECT/LEarning/Blockchain/Node\ js/My\ Assignments/Practice/UDEMY_CKADA/142/user1.crt

NOTE:
1. Escape Spaces in the destination path
2. Add `minikube:` before the source path.

TRY:

minikube cp minikube:/var/lib/minikube/certs/user1.crt /mnt/e/MISCELLENEOUS/MY\ PROJECT/LEarning/Blockchain/Node\ js/My\ Assignments/Practice/UDEMY_CKADA/142/user1.crt

```

4. **Interpretation of commands:**

| **Command/Option**                  | **Meaning**         | **Explanation**                                                                           |
| ----------------------------------- | ------------------- | ----------------------------------------------------------------------------------------- |
| `openssl req`                       | Request certificate | Used to create a CSR (Certificate Signing Request).                                       |
| `-new`                              | New request         | Generates a new CSR.                                                                      |
| `-key user1.key`                    | Private key         | Uses the private key to sign the CSR.                                                     |
| `-out user1.csr`                    | Output file         | Saves the CSR into `user1.csr`.                                                           |
| `-subj "/CN=user1/O=dev"`           | Subject info        | Sets certificate details: `CN` = Common Name (username), `O` = Organization (group/role). |
| `openssl x509`                      | Create certificate  | Used to sign and generate the actual certificate.                                         |
| `-req`                              | Input is a CSR      | Tells OpenSSL that input is a certificate signing request.                                |
| `-in user1.csr`                     | Input file          | Specifies the CSR file to sign.                                                           |
| `-CA /etc/kubernetes/pki/ca.crt`    | CA certificate      | The cluster’s Certificate Authority used to sign new certs.                               |
| `-CAkey /etc/kubernetes/pki/ca.key` | CA private key      | Used by CA to verify and sign the certificate.                                            |
| `-CAcreateserial`                   | Create serial file  | Generates a serial number file if not already present.                                    |
| `-out user1.crt`                    | Output certificate  | The final signed certificate file.                                                        |
| `-days 365`                         | Validity period     | Certificate will be valid for 365 days.                                                   |

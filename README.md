# Python-Flask-Application-Kubernetes
In this tutorial, you use an Oracle Cloud Infrastructure account to set up a Kubernetes cluster. Then, you create a Python application with a Flask framework. Finally, we deploy our application to your cluster by using Cloud Shell.

Key tasks include how to:

- Set up an authentication token.
- Set up a Kubernetes cluster on OCI.
- Build a Python application in a Flask framework.
- Create a Docker image.
- Push your image to OCIR.
- Use Cloud Shell to deploy your Docker application to your cluster.
- Connect to your application from the internet.

![image](https://user-images.githubusercontent.com/42166489/107560398-51eb2a00-6c03-11eb-928b-1359b721245f.png)

What we need to start:
To successfully perform this tutorial, we must have the following:

1.A paid Oracle Cloud Infrastructure account. 
https://docs.oracle.com/iaas/Content/GSG/Tasks/signingup.htm
2. SSH support:
A MacOS or Linux computer with ssh support installed.
A Windows machine with Git Bash installed. 
https://gitforwindows.org/

In this tutorial we are going to use a Mac machine.

1. Gather Required Information
Prepare the information we need from the Oracle Cloud Infrastructure Console.

Check our service limits:
Regions: minimum 2
In the top navigation bar, expand <region>. Example: US East (Ashburn) and US West (Phoenix).

Note: In our case we are going to perform tutorial in Mumbai region.
Compute instances: minimum 3
Click our profile avatar. Select Tenancy. Go to Service Limits and expand Compute.

Block Volume: minimum 50 GBs
In the Service Limits section, expand Block Volume.

Load Balancer: available
In the Service Limits section, expand Networking.

Have a compartment for our cluster.
See Create a compartment: 
(https://docs.oracle.com/iaas/Content/Identity/Tasks/managingcompartments.htm#).

- Create an authorization token:
- In the top navigation bar, click the user avatar.
- Select our username.
- Click Auth Tokens.
- Click Generate Token.
- Give it a description.
- Click Generate Token.
- Copy the token and save it.
 


Note: Make sure we save our token right after we create it. We will not have access to it later.
Find our region identifier and region key from Regions and Availability Domains. Example: us-ashburn-1 and iad for Ashburn.
https://docs.oracle.com/iaas/Content/General/Concepts/regions.htm

In our case for India West (Mumbai) it is “ap-mumbai-1”

Collect the following information and copy them into our notepad.
Auth Token: <auth-token> from step 3.
Region: <region-identifier> from step 4. Example: us-ashburn-1.
Region Key: <region-key> from step 4. Example: iad.
Tenancy name: <tenancy-name> from our user avatar.
Tenancy OCID: <tenancy-ocid> from our user avatar, go to Tenancy:<our-tenancy> and copy OCID.
Username: <user-name> from our user avatar.
User OCID: <user-ocid> from our user avatar, go to User Settings and copy OCID.

2. Create SSH Encryption Keys

Create ssh encryption keys to connect to our compute instance.

Open a terminal window:
MacOS or Linux: Open a terminal window in the directory where we want to store our keys.
Windows: Right-click on the directory where we want to store our keys and select Git Bash Here.

Issue the following OpenSSH command:
ssh-keygen -t rsa -N "" -b 2048 -C <our-ssh-key-name> -f <our-ssh-key-name>

The command generates some random text art used to generate the keys. When complete, we have two files:
The private key file: <our-ssh-key-name>
The public key file: <our-ssh-key-name>.pub
We use these files to connect to our compute instance.

This way we can generate the required encryption keys.

Or, we can use the OCI console for generating the keys.

![image](https://user-images.githubusercontent.com/42166489/107560458-629ba000-6c03-11eb-9c33-710859e756ad.png)

Link: https://docs.oracle.com/iaas/Content/GSG/Tasks/creatingkeys.htm

![image](https://user-images.githubusercontent.com/42166489/107560506-721ae900-6c03-11eb-82a1-da1b27edc4df.png)

Create a API Key by clicking on the API Keys and download and save the keys to a secured location.

![image](https://user-images.githubusercontent.com/42166489/107560540-7941f700-6c03-11eb-8cd8-29f5f7083fef.png)

3. Create a Virtual Cloud Network (VCN)

1. From the main landing page, select Set up a network with a wizard.

![image](https://user-images.githubusercontent.com/42166489/107560582-85c64f80-6c03-11eb-82db-e3d0b3658984.png)

2. In the Start VCN Wizard workflow, select VCN with Internet Connectivity and then click Start VCN Wizard .
3. In the configuration dialog, fill in the VCN Name for our VCN. Our Compartment is already set to its default value of <our-tenancy> (root).
4. In the Configure VCN and Subnets section, keep the default values for the CIDR blocks:
VCN CIDR BLOCK: 10.0.0.0/16
PUBLIC SUBNET CIDR BLOCK: 10.0.0.0/24
PRIVATE SUBNET CIDR BLOCK: 10.0.1.0/24

![image](https://user-images.githubusercontent.com/42166489/107560614-91b21180-6c03-11eb-97a9-3f3d5e35b887.png)

5. For DNS RESOLUTION uncheck USE DNS HOSTNAMES IN THIS VCN.Picture shows the USE DNS HOSTNAMES IN THIS VCN option is unchecked.
Click Next.
6. The Create a VCN with Internet Connectivity configuration dialog is displayed (not shown here) confirming all the values our just entered.

7. Click Create to create our VCN.
The Creating Resources dialog is displayed (not shown here) showing all VCN components being created.

8. Click View Virtual Cloud Network to view our new VCN.
Our new VCN is displayed. Now we need to add a security rule to allow HTTP connections on port 3000, the default port for our application.

9. With our new VCN displayed, click our Public subnet link.
The public subnet information is displayed with the Security Lists at the bottom of the page. There should be a link to the Default Security List for our VCN.

![image](https://user-images.githubusercontent.com/42166489/107560647-9b3b7980-6c03-11eb-96f7-b94bd80507b6.png)

10. Click on the Default Security List link.
The default Ingress Rules for our VCN are displayed.

11. Click Add Ingress Rules.
An Add Ingress Rules dialog is displayed.

12. Fill in the ingress rule with the following information. Once all the data is entered, click Add Ingress Rules.
Fill in the ingress rule as follows:

Stateless: Checked
Source Type: CIDR
Source CIDR: 0.0.0.0/0
IP Protocol: TCP
Source port range: (leave-blank)
Destination Port Range: 3000
Description: VCN for applications
Once we click Add Ingress Rule, HTTP connections are allowed to our public subnet.

4.Install an Ubuntu VM

From the Oracle Cloud Infrastructure main menu, select Compute, then Instances.
From the list of instances screen, click Create Instance.
The Create Compute Instance dialog is displayed. Notice the Show Shape, Network and Storage Options should be expanded to configure the virtual machine.

Fill in the fields for the Create Compute Instance dialog with the following data:
Initial Options

Name of our Instance: <name-for-the-instance>
Image or Operating System (Click Change Image): Canonical Ubuntu 18.04
Availability Domain: <Select-an-always-free-eligible-domain>
Instance Shape: VM.Standard.E2.1.Micro: Virtual Machine, 1 core OCPU, 1 GB Memory, 0.48 Gbps network bandwidth

Configure Networking

VIRTUAL CLOUD NETWORK COMPARTMENT: <our-compartment>
VIRTUAL CLOUD NETWORK: <VCN-we-created>
SUBNET COMPARTMENT: <our-subnet-compartment>
SUBNET: <public-subnet-ou-created>
USE NETWORK SECURITY GROUPS TO CONTROL TRAFFIC: Unchecked
ASSIGN A PUBLIC IP ADDRESS: Selected/Checked
Additional Options

Boot Volume: All options Unchecked
Add SSH Keys: Add the public key file (.pub) we created in the beginning of this tutorial.
Click Create to create the instance.
Provisioning the system may take several minutes.

![image](https://user-images.githubusercontent.com/42166489/107560676-a7273b80-6c03-11eb-8797-86a4996b16be.png)

![image](https://user-images.githubusercontent.com/42166489/107560687-ad1d1c80-6c03-11eb-9afc-78b7eb90ae3f.png)

we have successfully created an Ubuntu Linux VM to build and test our applications.

5.Run a Python Application in a Flask Framework

Next, set up a Flask framework on your Ubuntu Linux VM and then create and run a Python application.

By default, Ubuntu 18.04 comes with Python3, but it does not come with virtualenv, Flask or Docker. To create your application with Flask, perform the following steps:

- From the main menu select Compute, then Instances.
- Click the link to the instance we created in the previous step.
- From the Instance Details page look in the Instance Access section. Copy the public IP address the system created for we. we will use this IP address to connect to our instance.

- Open a Terminal or Command Prompt window.
- Change into the directory where we stored the ssh encryption keys we created for this tutorial.
- Connect to our VM with this ssh command.

Use the following command to set the file permissions so that only we can read the file:
chmod 400 <private_key_file_path>

Use the following SSH command to access the instance.
ssh -i <our-private-key-file> ubuntu@<x.x.x.x>

Terminal Commands:

![image](https://user-images.githubusercontent.com/42166489/107560714-b73f1b00-6c03-11eb-8ae8-9a98afe84234.png)

Since we identified our public key when we created the VM, this command should log we into our VM. we can now issue sudo commands to install and start our server.

- Update firewall settings.
The Ubuntu firewall is disabled by default. However, it is still necessary to update our iptables configuration to allow HTTP traffic. Execute the following commands to update iptables.
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 3000 -j ACCEPT

sudo netfilter-persistent save

These commands add a rule to allow HTTP traffic through port 3000 and saves the changes to the iptables configuration files.

Install pip3 for Python 3.

sudo apt update
sudo apt install python3-pip
 
Note: You may need to type "y" a few times to accept the packages that are installed to the VM.
Next, install and activate a virtual environment plus a virtual environment wrapper.
You can use a virtual environment to manage the dependencies for your project. Every project can be in its own virtual environment to host independent groups of Python libraries.

The virtualenvwrapper is an extenstion to virtualenv. It provides a set of commands, which makes working with virtual environments much more pleasant. It also places all your virtual environments in one place. The virtualenvwrapper provides tab-completion on environment names.


sudo apt install python3-venv
sudo pip3 install virtualenvwrapper
Set up your virtual environment wrapper in .bashrc.
Update the file:


sudo vi .bashrc
In the file, append the following text and save the file:

# set up Python env
export WORKON_HOME=~/envs
export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3
export VIRTUALENVWRAPPER_VIRTUALENV_ARGS=' -p /usr/bin/python3 '
source /usr/local/bin/virtualenvwrapper.sh
Activate the above commands in the current window.

source ~/.bashrc
Start a virtual environment.

mkvirtualenv flask01
You should see: (flask01) ubuntu@<ubuntu-instance-name>:~$
Install Flask.

sudo pip3 install Flask
Create a directory for your application.

mkdir python-hello-app
Change to the python-hello-app directory.
cd python-hello-app
Create a "Hello, World!" application.

![image](https://user-images.githubusercontent.com/42166489/107560750-c32add00-6c03-11eb-90cf-08d751cce76b.png)

![image](https://user-images.githubusercontent.com/42166489/107560772-c9b95480-6c03-11eb-9c4a-82e36403641c.png)

Create the file:
vi hello.py

In the file, input the following text and save the file:
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'
if __name__ == "__main__":
app.run(host="0.0.0.0", port=int("5000"), debug=True)

![image](https://user-images.githubusercontent.com/42166489/107560803-d2aa2600-6c03-11eb-8f89-8f84da47c000.png)

Run the Python program
export FLASK_APP=hello.py
flask run --host=0.0.0.0

Test the application using the command line or a browser:
To test with curl, from a new terminal, connect to our Ubuntu VM with your SSH keys, and then in the command line enter: curl -X GET http://localhost:5000

![image](https://user-images.githubusercontent.com/42166489/107560845-dccc2480-6c03-11eb-97de-76d6648ad80f.png)

From a browser, connect to the public IP address assigned to our VM: http://<x.x.x.x>:5000.
We should see Hello World! on our VM, or in your browser.

We have successfully created a local Python application in a Flask framework, on our Oracle Cloud Infrastructure VM.

For more info on Flask: https://flask.palletsprojects.com/

6.Build and Push your Python Flask Docker Image

Next, create a Docker image on your Ubuntu Linux VM and then push it to Oracle Cloud Infrastructure Registry.

Install Docker on your Oracle Linux VM.

Install Docker 18.0.6+ on your VM:
sudo snap install docker
docker --version
 Note

This snap build requires all files that Docker uses, such as Dockerfiles, to be in $HOME or a sub-directory of it. Keep the files there, for commands such as docker build, docker save and docker load.
Build a Docker Image

Build a Docker image for your application.

First, make sure you are in the python-hello-app directory.
Create a file named Dockerfile:
vi Dockerfile

In the file, input the following text and save the file:


FROM python:3.7-alpine
ADD hello.py /
COPY . /app
WORKDIR /app
RUN pip3 install Flask
EXPOSE 5000
CMD [ "python", "./hello.py" ]
Build a Docker image:
Copy
sudo docker build -t python-hello-app .

![image](https://user-images.githubusercontent.com/42166489/107560887-e786b980-6c03-11eb-8fd1-23be33b065dd.png)

We should get a message of success.
[INFO] BUILD SUCCESS
Successfully tagged python-hello-app:latest

Run the Docker image: sudo docker run --rm -p 5000:5000 python-hello-app:latest

![image](https://user-images.githubusercontent.com/42166489/107560944-f4a3a880-6c03-11eb-95fa-435bc199d0e1.png)

Test the application using the command line or a browser:
To test with curl, in a terminal, enter curl localhost:5000
From a browser, connect to the public IP address assigned to your VM: http://<x.x.x.x>:5000.
If you get Hello World!, then the Docker image is running. Now you can push the image to Oracle Cloud Infrastructure Registry (OCIR).

![image](https://user-images.githubusercontent.com/42166489/107560975-fff6d400-6c03-11eb-8530-2af7b2dc876f.png)

Push your Docker image to OCIR

With your Docker image created, now you need to push it to OCIR.

Open a terminal window.
Use <region-key> from the gather information section to log Docker into OCIR:

sudo docker login <region-key>.ocir.io
You are prompted for your login name and password.

Username: <tenancy-name>/<user-name>
Password: <auth-token>
List your local Docker images.

sudo docker images
The Docker images on your system are displayed. Identify the image you created in the last section: python-hello-app

Before you push your local image to OCIR, you must reference it with a new name that includes the path to OCIR. Then you can push the image with the new name to OCIR. Use the Docker tag command to create a shortcut to your image using the new name:

sudo docker tag <your-local-image> <registry-image>
The registry image consists of: <region-key>.ocir.io/<tenancy-name>/<image-folder>/<image-name>

Example: sudo docker tag python-hello-app iad.ocir.io/<tenancy-name>/<image-folder>/python-hello-app

Note: The <image-folder> is a string of your choosing that is used to group your images. For example, a short version of your username is often used. "John Doe" might use: sudo docker tag python-hello-app iad.ocir.io/<tenancy-name>/johnd/python-hello-app.
Check your Docker images to see if the reference has been created.

sudo docker images

![image](https://user-images.githubusercontent.com/42166489/107561015-0b49ff80-6c04-11eb-9612-ca2ffd188520.png)

The new image has the same image ID as your local image.
Push the image to OCIR.

sudo docker push <registry-image>:latest
Example: sudo docker push iad.ocir.io/<tenancy-name>/<image-folder>/python-hello-app:latest

You should see your image in OCIR after the push command is complete.

View the OCIR Repository we Created

In the Console, from the main menu select Developer Services, then Registry (OCIR).
In the list of registries, expand your registry: <image-folder>/python-hello-app. We may need to scroll to the right, to get a second scrollbar, to find your registry.
We should see:
Access: Private

Click latest.
We should see:
Full Path: <region-key>.ocir.io/<tenancy-name>/<image-folder>/python-hello-app

![image](https://user-images.githubusercontent.com/42166489/107561062-1735c180-6c04-11eb-9bc6-4d42fd5ccdaa.png)

Congratulations! You created a Flask Python Docker image. Now you can create a Kubernetes cluster and deploy this image to the cluster.
7. Create your Kubernetes Cluster

Set up the Kubernetes cluster we will deploy your application to. We will use a wizard to set up your first cluster.

From the OCI main menu select Developer Services then Container Clusters.
Click Create Cluster.
Select Quick Create.
Click Launch Workflow.
The Cluster Creation dialog is displayed.

Fill in the following information:
Name: <our-cluster-name>
Compartment: <our-compartment-name>
Kubernetes Version: <take-default>
Visibility Type: <Private>
Shape: VM.Standard.E2.1
Number of Nodes: 3
Add Ons: <none-selected>
Click Next.
All our choices are displayed. Review them to make sure everything is configurated correctly.

Click Create Cluster.
The services set up for our cluster are displayed.

Click Close.
Get a cup of coffee. It may take a few minutes for the cluster to be created.
We have successfully created a Kubernetes cluster.

8.Manage your Kubernetes Cluster with Cloud Shell

In this section, we include the Kubernetes cluster information in a .kube/config file, so we can access the cluster and manage deployments. To do that, complete the following steps:

From the OCI main menu select Developer Services then Container Clusters.
Click the link to <our-cluster>.
The information about our cluster is displayed.

Click Access Cluster.

Select Local Access.
Follow the steps provided in the dialog. The steps for local access are reprinted here for our reference.

Activate our cli-app environment and test the oci connection.
oci -v

Make our .kube directory if it doesn't exist.
mkdir -p $HOME/.kube

Create kubeconfig file for our setup. Use the information from Access our Cluster dialog.
oci ce cluster create-kubeconfig <copy-data-from-dialog>

Export the KUBECONFIG environment variable.
export KUBECONFIG=$HOME/.kube/config

Test our cluster configuration with the following command:
List clusters:
kubectl get service

![image](https://user-images.githubusercontent.com/42166489/107561099-261c7400-6c04-11eb-8315-3243790e5cc2.png)

![image](https://user-images.githubusercontent.com/42166489/107561143-316f9f80-6c04-11eb-8a02-a17ee7c67787.png)

8. Deploy the Docker Image to the Cluster

Now, create a manifest file to include information about the following resources and then create the resources with Kubernetes:
Deployment: Pull and deploy the image from registry.
Load Balancer Service: Make the application available with a public IP address.

Deploy our Image to the Cluster
With our image in OCIR, we can now deploy our application. Perform the following commands either in Cloud Shell or in a terminal connected to our VM.

First, create a registry secret for our application. This will be used to authenticate our image when it is deployed to our cluster.
Fill in the information in this template to create our secret with the name ocirsecret.

kubectl create secret docker-registry ocirsecret --docker-server=<region-code>.ocir.io  --docker-username='<tenancy-name>/<user-name>' --docker-password='<auth-token>' --docker-email='<email-address>'

After executing the command, we should get message similar to: secret/ocirsecret created.

Verify the secret was created. Issue the following command.
kubectl get secret ocirsecret --output=yaml

The output includes information about our secret is shown in the yaml format.

Determine the host URL to our registry image using the following template.
<region-code>.ocir.io/<tenancy-name>/<repo-name>/<image-name>:<tag>
Eg: bom.ocir.io/bmdrgwy1wsjh/saikat/node-hello-app:latest

Create a file called app.yaml.
vi app.yaml

Copy the following information in our app.yaml.

Here are some of the parameters that we will set for the deployment section:















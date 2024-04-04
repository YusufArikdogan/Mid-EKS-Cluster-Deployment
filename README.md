# Mid-EKS-Cluster-Deployment  
# Kubernetes Deployment with Jenkins  

## Introduction  
In this repository, we will perform deployment on Kubernetes using Jenkins.   
The repository contains backend and frontend source folders. For those who  
prefer not to use Kubernetes, we have a `docker-compose.yaml` file available.  
You can make updates to this file and perform the deployment. However, we have   
converted our Docker Compose manifest to Kubernetes manifest, and we will continue  
our operations through Kubernetes.  

## Conversion Process    
Let me explain how we did this first. In the previous readme, we connected to the Jenkins server  
via SSH. Here are the commands to execute sequentially in the terminal:  
```bash 
curl -L https://github.com/kubernetes/kompose/releases/download/v1.32.0/kompose-linux-amd64 -o kompose  
chmod +x kompose  
sudo mv ./kompose /usr/local/bin/kompose  
kompose convert -f docker-compose.yaml   
kubectl apply -f .   
Here, we download the Kompose tool, grant necessary permissions, and then use the  
kubectl apply -f .  
command to convert our Docker Compose manifest to Kubernetes manifests. Deployment and service manifests  
are automatically created for each resource.    

Customization with Kustomize 

We may have dozens of manifests depending on the size of the system to be deployed, so we'll use a separate  
tool for customization. We'll use Kustomize, which is already set up to work with kubectl on the Jenkins server.  

Firstly, we create a kustomization-template.yaml file:  

sudo nano kustomization-template.yaml  

Here, we add all the necessary resources as resources. The sections we'll be making changes to will be in the image    
sections, so we add images: and assign variables for each name. These variables need to be specified here using the  
tags created when Jenkinsfile-eks runs and the images are pushed to the ECR repo. We then use the envsubst command  
to write the data prepared in the kustomization-template into a kustomization.yaml file. If it has a different name,  
it won't work. The kustomization.yaml is ready with the changed data.  
Here's how we can manually make changes on the Jenkins server:  

kubectl kustomize <kustomization_directory>
kubectl apply -k <kustomization_directory>

But since we will be deploying all deployments and services in Kubernetes in the pipeline, we don't need to do it  
manually. In our pipeline, the command sh "kubectl apply -k k8s/" performs all deployment operations.  

Making Changes in Kubernetes Manifests  
To make changes in the Kubernetes manifests, you'll need to follow these steps:  
  
Update Domain in Docker Compose:  
In lines 13 and 16 of the docker-compose.yaml file, add your own domain.  

Update Domain in ui-deployment.yaml:  
In lines 6, 23, and 34 of ui-deployment.yaml, input your own domain.  

Update Ingress in ui-ingress.yaml:  
In lines 7, 16, 26, and 36 of ui-ingress.yaml, set the hosts where you'll connect to Prometheus and Grafana.  
I'll explain these steps in the routing section of AWS Route53.  

Update Services in ui-services.yaml:  
In line 6 of ui-services.yaml, specify your own domain.  

Update Ingress Controller in ingress-controller.yaml:   
Changes are required here as well. On line 329, set proxy-real-ip-cidr to your cluster IP. On line 348,  
input the ARN of the SSL certificate obtained from AWS Certificate Manager or another source.  

Check Jenkins Pipeline Configuration:
Let's take a look at jenkins-eks. Actually, everything here works automatically, but if you've changed region  
or app name, ensure to update these without missing. Here, we're using AWS resources again.  

1.  We prepare Docker image names according to the build number. Jenkins assigns a tag with the build  
number for each build operation.  
2. We build images with the specified names.  
3. We create an ECR repository on AWS.  
4. We push the created images to the repository.  
5. We create our namespace in Kubernetes.  
6. We adjust and apply the image names we created with Kustomize.  
7. We apply the ingress-controller configuration.  
8. There are destroy and post-action sections.  
In the pipeline, in case of any errors, it directly goes to the post section and deletes the created resources  
and images to avoid conflicts when restarted. In the destroy stage, the pipeline exits as we desire. As Proceed  
and Abort. Here, we've set a 5-day time limit so that it self-destructs after 5 days. Alternatively, by clicking  
Proceed in the output at the end of the pipeline, you can terminate the system. If you want it to remain, you can  
exit from the Abort section.  

Once all the adjustments are made, we can navigate to the Jenkins server interface.  

Go to the dashboard, click on "Add item," and this time select "pipeline." You can name it as "deployment"  
or any other name you prefer.  

Configuring Pipeline  
Under "Pipeline Script" Section:  

Select "Pipeline from SCM."  
Choose Git as the SCM.  
Paste the URL of your GitHub repository where you pushed the files.  
Select the token created earlier.  
Optionally, set up triggers for the pipeline to run on push events.  
Let's change the name of our Jenkinsfile to jenkinsfile-eks in the lowest section.  

Click "Apply" and then "Save."  

Building the Pipeline  
Initiate Build:  
Click on "Build Now" to initiate the pipeline.  
 
Pipeline Execution:  
The pipeline will create the infrastructure and application step by step, which may take some time.  

Once the pipeline is successfully completed, we can proceed to the routing process.  

Routing Configuration  
Navigate to AWS Route53:  
Go to the AWS Route53 service.  

Hosted Zone Setup:  
If you already have a domain from AWS, you can skip this step. Otherwise, add a hosted zone. Enter the  
DNS name you obtained from a different DNS provider. After creating it in the format example.com, enter the  
hosted zone. Here, we'll have two records of NS and SOA type. Choose the NS record, and you'll see four values  
under "Value/Route traffic."  

Update DNS Records:  
Now, go to the website of your third-party DNS provider. In the section for editing name servers, you'll  
need to replace these servers one by one with the values in your AWS NS record. This way, we'll establish the  
connection immediately.  

SSL Certificate Creation:  
Once this is done, go back to the AWS Certificate Manager service. Click on "Request a certificate" and then  
"Request a public certificate." Enter your domain name as example.com and additionally as *.example.com. Click  
"Request" and then "Finish."  

Create DNS Records:  
After the request is created, click on it. You`ll see the "Create records in Route53" section, click on it. Then Click "Request" and then you can close it.  

Verify CNAME Record:  
When you go back to Route53, you'll see a record named CNAME. Congratulations, you've successfully linked your SSL  
certificate to your domain.  
 
With these steps completed, your deployment pipeline should be set up and running smoothly with SSL configured for  
your domain.  
  
Now, let's create the records for publication. Click on "Create Record."  

Subdomain: In the subdomain field, I added "carrental." If you haven't changed it, write it in the same way.  

Alias Configuration: Open the Alias section.  
  
Route Traffic To: Choose "Network Load Balancer."  

Region: Specify the region where your setup is running.  

Select Network Load Balancer ARN: A new field will open below, and the Network Load Balancer ARN will automatically  
appear. Choose it. It should remain as "simple routing."  

Add Another Record: Mark this option at the bottom right to add more records.  

Repeat for Prometheus and Grafana: Perform the same steps for Prometheus and Grafana. Use "pro" as the subdomain  
for Prometheus and "gra" for Grafana.  

Once done, you can access:  

carrrental.example.com for the UI.  
pro.example.com for Prometheus.  
gra.example.com for Grafana.  
Now your deployment is complete, and your application is accessible via these URLs.  

## VPA and HPA    
If you wish to scale your project with VPA (Vertical Pod Autoscaler) and HPA (Horizontal Pod Autoscaler),  
you can follow these steps:  

Install Metric Server:  
Inside your Jenkins server, run the following commands to install the Metric Server:  

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml  
kubectl get deployment metrics-server -n kube-system  
This will install the Metric Server for you.  

Install VPA:  
Next, you can install VPA by following these steps:  
  
git clone https://github.com/kubernetes/autoscaler.git  
cd autoscaler/vertical-pod-autoscaler/  
./hack/vpa-down.sh # Run this if you have existing VPA instances  
./hack/vpa-up.sh  
This will set up the VPA for your project. You can customize your VPA manifest by creating your own  
or by updating the existing manifests in the autoscaler/ directory according to your system requirements.  

Additionally, it's worth mentioning that if you need to update your kubeconfig, you can use the following  
command:  

aws eks --region us-east-1 update-kubeconfig --name kube  
Replace us-east-1 with your desired region and kube with the name of your Kubernetes cluster.  
This command will update your kubeconfig configuration accordingly.  

For HPA (Horizontal Pod Autoscaler), you can specify resource values within your deployment as follows:  

yaml  
Copy code  
resources:  
  limits:  
    cpu: 500m  
  requests:  
    cpu: 200m  
Alternatively, according to Kubernetes documentation, you can use different resources and metrics for scaling operations.  

Roll Out Update
In our repository, we have a directory named green/blue. Here, we can perform deployments of green and blue versions  
under different namespaces.  

Green: Active production version.  
Blue: Environment where the new version is deployed or updated.  
To implement this strategy:  

Prepare your new version in the "blue" environment.  
Test the "blue" environment and make adjustments if necessary.  
Finally, switch the "green" environment with the "blue" one so that the new version is released and the old version   
is disabled. This method aims to provide uninterrupted service to users as there is no interruption between shutting  
down the old version and starting the new one.  

Here's how you can do it:  
  
Create a namespace named "blue" using the command kubectl create ns blue.  
Update the name to the namespace in the deployment and service manifests found within our blue directory.  
After specifying the same working directory path in the path/ section, switch to the updated version by using the command:  

kubectl config set-context --current --namespace=blue  
Then, apply the deployment by running:  

kubectl apply -f blue-green/blue/k8s  
This command will perform the deployment process.  
You can also update the ui-ingress to reflect the changes in the new version. You can make changes to the target  
port in the namespace and even monitor it over a different port domain.  
By following these steps, you can smoothly transition between versions without interrupting the application.  

# CLEANUP

Once our project is completed, we need to perform cleanup operations.  

If we created a Jenkins Freestyle job named "EKS" with an Execute Shell script, we can follow these steps:  

Go to the Jenkins job and click on "Configure."  
In the Execute Shell script section:  
Keep the section where you create the EKS key:  
### Jenkins Server freestyle  

# create eks key  
EKS_KEY="eks_key"  
EKS_PUB_KEY="eks_key_pub"  
AWS_REGION="us-east-1"  
aws ec2 create-key-pair --key-name ${EKS_KEY} --region ${AWS_REGION} --query 'KeyMaterial' --output text > ${EKS_KEY}  
# Delete cluster (commented out)  
eksctl delete cluster --name kube --region ${AWS_REGION}  
# Delete EKS key pair (commented out)  
aws ec2 delete-key-pair --key-name ${EKS_KEY} --region ${AWS_REGION}  
Comment in or remove the other codes from script.  

Save the changes and rerun the build process.
This will delete the CloudFormation stack, the EKS cluster, and the key pair that we created. This process may  
take up to 15 minutes.  
After the cleanup is complete, you can go to the terminal in your local VS Code where Jenkins server is  
installed and execute the following command to delete the Jenkins server and the resources created along with it:  

terraform destroy -auto-approve  
By running this command, you'll effectively remove the Jenkins server and all associated resources.  

Congratulations!  
If you have any other questions or need further assistance, feel free to ask.  
Best of luck with your project! 🚀    

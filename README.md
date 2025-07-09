# **MLOps Churn Prediction Project - Beginner's Guide**       
### **Part 1: Setting Up AWS Infrastructure** 
*(Step-by-Step for Beginners)*  

---

## **📌 What Are We Building?**  
We’re creating an **automated system** that:  
1. **Stores** customer data in AWS S3 (like a secure cloud folder).  
2. **Trains** a machine learning model when new data arrives.  
3. **Tracks** experiments using MLflow (like a diary for AI models).  
4. **Hosts** the model on Kubernetes (a system to run apps reliably).  

> **For Beginners:** Think of this as setting up a factory where data comes in, models get trained automatically, and results are stored neatly.  

---

## **🚀 Step 1: Store Data in AWS S3 (Cloud Storage)**  

### **1.1 Download the Dataset**  
Run this in your terminal (Mac/Linux) or Command Prompt (Windows):  
```bash
wget https://raw.githubusercontent.com/IBM/telco-customer-churn-on-icp4d/master/data/Telco-Customer-Churn.csv
```
*(This downloads a sample customer data file.)*  

### **1.2 Upload to AWS S3**  
```bash
aws s3 mb s3://mlops-churn-raw-data  # Creates a storage "bucket"
aws s3 cp Telco-Customer-Churn.csv s3://mlops-churn-raw-data/raw/  # Uploads file
aws s3 mb s3://mlops-churn-model-artifacts #sagemaker push files
```
✅ **Check if it worked:**  
```bash
aws s3 ls s3://mlops-churn-raw-data/raw/  # Should show your file
```

---

## **🔐 Step 2: Set Up Permissions (IAM Roles)**  

### **2.1 SageMaker Role (Allows AI Training)**  
1. Go to **AWS IAM Console** → **Roles** → **Create Role**.  
2. Select **SageMaker** as the service.  
3. Attach these **policies**:  
   - `AmazonSageMakerFullAccess`  
   - `AmazonS3FullAccess`  
   - `CloudWatchLogsFullAccess`  
4. Name it **`SageMakerChurnRole`** and create.
5. UpDate Trust Policy
   ```json
   {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "sagemaker.amazonaws.com",
                    "s3.amazonaws.com"
                ]
            },
            "Action": "sts:AssumeRole"
        }
    ]
  }
  ```
### **Create Model Package Group**
  ```bash
     aws sagemaker create-model-package-group \
    --model-package-group-name ChurnModelPackageGroup \
    --model-package-group-description "Package group for churn prediction models"
  ```
### **2.2 Lambda Role (Automates Training When New Data Comes)**  
1. Same process, but select **Lambda** as the service.  
2. Attach:  
   - `AWSLambdaBasicExecutionRole`  
   - `AmazonS3ReadOnlyAccess`  
   - `AmazonSageMakerFullAccess`  
3. Name it **`LambdaChurnTriggerRole`**.  

---

## **🤖 Step 3: Create a Lambda Function (Automation)**  

### **3.1 Go to AWS Lambda Console**  
1. Click **"Create Function"**.  
2. Name: `TriggerChurnPipeline`.  
3. Runtime: **Python 3.9**.  
4. Role: Select **`LambdaChurnTriggerRole`**.  

### **3.2 Paste This Code**  
```python
import boto3

def lambda_handler(event, context):
    sm_client = boto3.client('sagemaker')
    sm_client.start_pipeline_execution(PipelineName='churn-pipeline')
    return {"status": "Pipeline triggered!"}
```
📌 **Deploy** → **Save**.  

### **3.3 Set Up S3 Trigger**  
1. Go to **S3 Bucket (`mlops-churn-raw-data`)** → **Properties** → **Event Notifications**.  
2. Add:  
   - **Name:** `NewDataTrigger`  
   - **Event:** `PUT` (when files are uploaded)  
   - **Prefix:** `raw/` (only watch this folder)  
   - **Destination:** `TriggerChurnPipeline` (your Lambda)  

✅ Now, uploading a file to `raw/` will **automatically start training!**  

---

## **⚙️ Step 4: Set Up Kubernetes (EKS) for MLflow**  

### **4.1 Install Required Tools**  
*(Run in terminal)*  
```bash
# Install eksctl (Kubernetes tool)
brew install eksctl  # Mac
# OR (Linux)
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install Helm (for MLflow)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### **4.2 Create the Kubernetes Cluster**  
```bash
eksctl create cluster \
  --name churnmodel \
  --region ap-south-1 \
  --zones ap-south-1a,ap-south-1b \
  --nodegroup-name churn-ng-public1 \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --node-volume-size 20 \
  --ssh-access \
  --ssh-public-key aiops \
  --managed \
  --asg-access \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access
```
⏳ **Wait ~15 mins** (AWS is setting up servers for you).  

### **4.3 Deploy MLflow (Model Tracking Dashboard)**  
```bash
helm repo add community-charts https://community-charts.github.io/helm-charts
helm install my-mlflow community-charts/mlflow --version 0.7.19
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=mlflow,app.kubernetes.io/instance=my-mlflow" -o jsonpath="{.items[0].metadata.name}")
export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
```
🔗 **Get MLflow URL:**  
```bash
kubectl get svc -n mlflow
```
👉 Look for `EXTERNAL-IP` (e.g., `http://123.45.67.89:5000`).  

---

## **✅ Final Checks**  

| Task | Command | Expected Output |
|------|---------|-----------------|
| **S3 File Check** | `aws s3 ls s3://mlops-churn-raw-data/raw/` | Shows `Telco-Customer-Churn.csv` |
| **Kubernetes Nodes** | `kubectl get nodes` | 2 nodes in `Ready` state |
| **MLflow Access** | Open `http://<EXTERNAL-IP>:5000` | MLflow dashboard loads |

---
# 🚀 MLOps Churn Prediction Project — Part 2: Code, SageMaker Pipeline, MLflow Logging & GitHub Actions

Welcome to Part 2 of our **End-to-End MLOps Churn Prediction Project**. In this part, we’ll focus on implementing the **core ML pipeline**, integrating **MLflow** for experiment tracking, automating workflows with **GitHub Actions**, and deploying everything in **SageMaker Pipelines**.

---

## 📌 Overview

In this phase, you will:

- ✅ Write modular scripts for preprocessing, training, and evaluation.
- ✅ Integrate **MLflow** to track model experiments.
- ✅ Define and deploy a **SageMaker Pipeline** (preprocess → train → register model).
- ✅ Automate the whole pipeline via **GitHub Actions** on code push.

---

## 🗂️ Project Directory Structure

```bash
mlops-churn-project/
├── preprocessing.py        # Data cleaning and feature engineering
├── train.py                # Train XGBoost model, log with MLflow, save model artifacts
├── evaluate.py             # Evaluate trained model performance metrics
├── churn_pipeline.py       # Define SageMaker pipeline with training, evaluation, registration
├── requirements.txt        # Python dependencies
├── .github/
│   └── workflows/
│       └── train-deploy.yml  # GitHub Actions workflow to run pipeline on push
⚙️ Step-by-Step Breakdown
✅ Step 1: preprocessing.py
python
Copy
Edit
# Cleans and encodes data for ML model consumption
def preprocess(input_path, output_path):
    ...
Drops unnecessary columns

Converts Churn label to binary

Handles missing values

Encodes categorical features

Saves cleaned CSV

✅ Step 2: train.py (MLflow integrated)
python
Copy
Edit
# Trains an XGBoost model and logs metrics + artifacts to MLflow
def train(train_path, model_output_dir):
    ...
Splits data into train/val

Trains XGBoost with early stopping

Logs accuracy to MLflow

Saves model to path for SageMaker usage

✅ Step 3: evaluate.py (Optional)
python
Copy
Edit
# Evaluates trained model on test data
def evaluate(test_path, model_path):
    ...
Computes Accuracy, Precision, Recall

Helpful for local testing outside of pipeline

✅ Step 4: Define SageMaker Pipeline (churn_pipeline.py)
PreprocessingStep — runs preprocessing.py inside a ScriptProcessor

TrainingStep — runs train.py via XGBoost Estimator

ModelStep — registers trained model to SageMaker Model Registry

python
Copy
Edit
pipeline = Pipeline(
    name="churn-pipeline",
    steps=[processing_step, train_step, register_model_step],
)
To deploy pipeline:

bash
Copy
Edit
python churn_pipeline.py
✅ Step 5: Create Model Package Group (Run once)
python
Copy
Edit
import boto3
client = boto3.client("sagemaker")
client.create_model_package_group(ModelPackageGroupName="ChurnModelPackageGroup")
✅ Step 6: GitHub Actions CI/CD Workflow
Create this file: .github/workflows/train-deploy.yml

yaml
Copy
Edit
name: SageMaker Training and Model Registration

on:
  push:
    branches: [main]

jobs:
  train:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Install Python dependencies
        run: |
          pip install --upgrade pip
          pip install boto3 sagemaker mlflow xgboost pandas scikit-learn

      - name: Trigger SageMaker Pipeline
        run: |
          python -c "
import boto3
client = boto3.client('sagemaker')
response = client.start_pipeline_execution(PipelineName='churn-pipeline')
print('Pipeline started:', response['PipelineExecutionArn'])
"
Note: Add AWS credentials as GitHub repo secrets:

AWS_ACCESS_KEY_ID

AWS_SECRET_ACCESS_KEY

🚀 How to Run the Full Pipeline
✅ Push your code to main branch of your GitHub repo

✅ GitHub Actions automatically triggers the SageMaker pipeline

✅ Monitor progress in SageMaker Studio > Pipelines

✅ View experiment logs in MLflow UI (refer Part 1)

✅ Approve the model manually in Model Registry

🧠 Key Technologies
AWS SageMaker: Training, Processing, Model Registry, Pipelines

MLflow: Logging and experiment tracking

GitHub Actions: CI/CD automation

XGBoost: Model training

Python: Core development language

📬 Credits
Made with ❤️ by Rajinikanth Vadla
Trainer | DevOps | MLOps | AIOps Specialist

### **💡 Troubleshooting**  
❌ **"Access Denied" errors?** → Check IAM roles.  
❌ **Cluster not creating?** → Ensure AWS limits allow EKS.  
❌ **MLflow not loading?** → Wait 5 mins and check again.  


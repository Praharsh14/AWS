
# Auto Scaling Lifecycle Manager for EC2 (Node.js + AWS ASG)

## Step 1: Create IAM Role

Create a policy with the following JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScaling*",
        "autoscaling:UpdateAutoScalingGroup",
        "autoscaling:PutLifecycleHook",
        "autoscaling:DeleteAutoScalingGroup"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}

```

Go to **Roles → Create Role**:

-   **Trust entity**: EC2
    
-   **Use case**: EC2
    
-   Select the permission policy you just created
    
-   Name the role and create it
    

----------

## Step 2: Backend Setup

Navigate to your project folder → `backend` folder. Update `package.json`:

```json
{
  "name": "asg-demo-backend",
  "version": "1.0.0",
  "type": "module",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "@aws-sdk/client-auto-scaling": "^3.645.0",
    "express": "^4.19.2"
  }
}

```

### Edit `index.js`

```js
// ---------------- AWS SDK v3 ----------------
const {
  AutoScalingClient,
  DescribeAutoScalingInstancesCommand,
  DescribeAutoScalingGroupsCommand,
  UpdateAutoScalingGroupCommand,
  PutLifecycleHookCommand,
  DeleteAutoScalingGroupCommand,
} = require("@aws-sdk/client-auto-scaling");

// ---------------- EC2 Metadata Helpers ----------------
const MD_HOST = "169.254.169.254";

async function getIMDSToken(ttl = 21600) {
  return new Promise((resolve, reject) => {
    const req = http.request(
      {
        host: MD_HOST,
        path: "/latest/api/token",
        method: "PUT",
        headers: { "X-aws-ec2-metadata-token-ttl-seconds": String(ttl) },
        timeout: 1000,
      },
      (res) => {
        let data = "";
        res.on("data", (c) => (data += c));
        res.on("end", () =>
          res.statusCode === 200
            ? resolve(data)
            : reject(new Error("IMDS token error"))
        );
      }
    );
    req.on("error", reject);
    req.end();
  });
}

async function mdGet(path, token) {
  return new Promise((resolve, reject) => {
    const req = http.request(
      {
        host: MD_HOST,
        path,
        method: "GET",
        headers: { "X-aws-ec2-metadata-token": token },
        timeout: 1000,
      },
      (res) => {
        let data = "";
        res.on("data", (c) => (data += c));
        res.on("end", () =>
          res.statusCode === 200
            ? resolve(data)
            : reject(new Error(`IMDS GET ${path} ${res.statusCode}`))
        );
      }
    );
    req.on("error", reject);
    req.end();
  });
}

async function getInstanceIdentity() {
  try {
    const token = await getIMDSToken();
    const iid = await mdGet("/latest/meta-data/instance-id", token);
    const docJson = await mdGet("/latest/dynamic/instance-identity/document", token);
    const doc = JSON.parse(docJson);
    const region = process.env.AWS_REGION || doc.region;
    return { instanceId: iid.trim(), region };
  } catch (e) {
    return {
      instanceId: process.env.INSTANCE_ID || "local-testing",
      region: process.env.AWS_REGION || "us-east-1",
    };
  }
}

let asgClient;
async function getAsgClient() {
  if (!asgClient) {
    const { region } = await getInstanceIdentity();
    asgClient = new AutoScalingClient({ region });
  }
  return asgClient;
}

async function getAsgInfoForThisInstance() {
  const { instanceId } = await getInstanceIdentity();
  const client = await getAsgClient();

  const d1 = await client.send(
    new DescribeAutoScalingInstancesCommand({ InstanceIds: [instanceId] })
  );

  if (!d1.AutoScalingInstances || !d1.AutoScalingInstances.length) {
    return {
      instanceId,
      asgName: null,
      lifecycleState: "Standalone/NotInASG",
    };
  }

  const rec = d1.AutoScalingInstances[0];
  const asgName = rec.AutoScalingGroupName;
  const lifecycleState = rec.LifecycleState;

  const d2 = await client.send(
    new DescribeAutoScalingGroupsCommand({ AutoScalingGroupNames: [asgName] })
  );
  const g = d2.AutoScalingGroups?.[0];
  const desired = g?.DesiredCapacity ?? null;
  const min = g?.MinSize ?? null;
  const max = g?.MaxSize ?? null;

  return { instanceId, asgName, lifecycleState, desired, min, max };
}

// ---------------- AWS Routes ----------------
app.get("/status", async (req, res) => {
  try {
    const { instanceId } = await getInstanceIdentity();
    const client = await getAsgClient();

    const d1 = await client.send(
      new DescribeAutoScalingInstancesCommand({ InstanceIds: [instanceId] })
    );

    if (!d1.AutoScalingInstances || d1.AutoScalingInstances.length === 0) {
      return res.json({
        ok: true,
        instanceId: null,
        asgName: null,
        lifecycleState: "Standalone/NotInASG",
      });
    }

    const rec = d1.AutoScalingInstances[0];
    const asgName = rec.AutoScalingGroupName;
    const lifecycleState = rec.LifecycleState;

    return res.json({ ok: true, instanceId, asgName, lifecycleState });
  } catch (err) {
    res.status(500).json({ ok: false, error: String(err) });
  }
});

```

----------

## Step 3: Transfer Code to Server

```bash
cd project-name/backend
npm i
pm2 start index.js --name "backend"

```

> Test if the application is running.

----------

## Step 4: Create Security Group

For example, **Backend-SG**:

-   **Inbound Rules**:
    
    -   TCP 8080 from your IP (or `0.0.0.0/0` for demo)
        
    -   TCP 22 (SSH) from your IP
        
    -   HTTP & HTTPS
        

----------

## Step 5: Create AMI & Launch Template

1.  Create an AMI of the current instance, name it `backend-ami`.
    
2.  Create a Launch Template of the current instance.
    
3.  Attach IAM role and Security Group.
    
4.  Attach the created AMI if not already attached.
    

### User Data Script

```bash
#!/bin/bash

# Navigate to backend directory (already included in AMI)
cd /your/project/path

# Install dependencies
npm install

# Start backend with PM2
pm2 start index.js --name backend-app

# Save PM2 process and startup script
pm2 save
pm2 startup systemd -u ubuntu --hp /home/ubuntu

```

----------

## Step 6: Create Auto Scaling Group (ASG)

-   Name: `BACKEND-ASG`
    
-   Select Launch Template: `backend-template`
    
-   Choose appropriate VPC and AZs
    
-   Initially keep desired capacity = 0
    

### Lifecycle Hook for Graceful Termination

> Go to ASG → Lifecycle Hooks → Add Lifecycle Hook

**Settings:**

-   Lifecycle transition: Instance terminating
    
-   Heartbeat timeout: 180 seconds (3 minutes)
    
-   Default result: CONTINUE
    

> Ensures each instance takes ~3 min to terminate fully

### Controlled Delay Before Termination (5 min)

> ASG doesn’t have built-in wait before termination, but you can simulate it using **scale-in cooldown**:

-   ASG → Details → Default cooldown → 300 seconds (5 minutes)
    

> Works if you manually set desired capacity = 0. ASG respects cooldown before terminating.

----------

## Step 7: Test

1.  Change ASG desired capacity from `0 → 1` (min=1, max=1)
    
    > ASG creates EC2 instance automatically from your template
    
2.  Test API:
    

```bash
curl http://<EC2-Public-IP>:3000
curl http://<EC2-Public-IP>:3000/status

```

Expected JSON:

```json
{
  "ok": true,
  "instanceId": "i-0b42679f64bf40f82",
  "asgName": "Backend-ASG",
  "lifecycleState": "Terminating:Wait"
}

```

----------


## About / How This Project Works

This project, **Auto Scaling Lifecycle Manager for EC2**, demonstrates automated scaling and graceful termination of EC2 instances using **AWS Auto Scaling Groups (ASG)** and **Node.js backend**.

### Workflow

1. **Scaling Up (Desired Capacity 0 → 1)**

   > When you increase the ASG desired capacity from `0` to `1`, the ASG automatically launches a new EC2 instance based on the configured **launch template** and **AMI**.  

   - The newly created instance hosts the **frontend** on:  
     `http://<your-public-ip>:8080`  
   - The **backend API** can be accessed at:  
     `http://<your-public-ip>:8080/status`  

   Example JSON response from the backend:

   ```json
   {
     "ok": true,
     "instanceId": "i-0b42679f64bf40f82",
     "asgName": "Backend-ASG",
     "lifecycleState": "Terminating:Wait"
   }
_

## 🏆 Why This Project?

> This project is a DevOps + AWS showcase:

-   Demonstrates cloud automation expertise
    
-   Shows scalable backend deployment
    
-   Highlights ability to integrate infrastructure with application logic

---

_~by Praharsh Shinde_




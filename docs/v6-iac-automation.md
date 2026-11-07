# V6 - Infrastructure Automation (CloudFormation + CodePipeline)

> Completed as part of AWS Academy Challenge Labs (training environment).

## Business Requirement

The café wanted to open in new Regions and keep development and production identical. Building everything by hand in the console was slow and impossible to repeat reliably, so the infrastructure had to become code: version-controlled templates, deployed by a pipeline, and reusable in another Region.

## Before → After

```mermaid
flowchart LR
  subgraph BEFORE["Before: manual console work"]
    direction TB
    P1([Person]) --> C1[AWS Console clicks]
    C1 --> R1[Resources in one Region]
  end
  subgraph AFTER["After: infrastructure as code with a pipeline"]
    direction TB
    P2([Person]) --> G["git push"]
    G --> CC[CodeCommit<br/>CFTemplatesRepo]
    CC --> CP[CodePipeline]
    CP --> CF[CloudFormation stacks]
    CF --> N[Network stack<br/>VPC + public subnet]
    CF --> A[App stack<br/>EC2 web server]
    T[Same templates] -.-> O[Second Region<br/>us-west-2]
  end
```

### Reference architecture

![Reference: resources available at lab start](../diagrams/v6/V6-ref-start-resources.png)
*Reference: starting resources from the AWS Academy lab.*

![Reference: S3 stack flow](../diagrams/v6/V6-ref-s3-stack-flow.png)
*Reference: creating a stack from a template, from the AWS Academy lab.*

![Reference: network pipeline flow](../diagrams/v6/V6-ref-network-pipeline-flow.png)
*Reference: pipeline flow from the AWS Academy lab.*

## AWS Services Used

| Service | Role in this stage |
|---|---|
| AWS CloudFormation | Defines the S3 bucket, the network layer and the application layer as templates and stacks |
| AWS CodeCommit | Git repository (`CFTemplatesRepo`) that stores and versions the templates |
| AWS CodePipeline | `CafeNetworkPipeline` and `CafeAppPipeline` deploy a stack whenever a template is pushed |
| Amazon EC2 and VPC | Resources created by the templates (Cafe VPC, public subnet, security group, web server) |
| Amazon S3 | First practice stack (static website bucket) and storage for the template used in the second Region |
| AWS CLI and Git | Creating and validating stacks, and pushing templates from the lab IDE |

## Implementation Highlights

- **First stack from scratch.** I wrote `S3.yaml` and created the `CreateBucket` stack with the CLI, then updated it with a retain policy, static website configuration and a website URL output (E04-E16).
- **Templates under version control.** I cloned `CFTemplatesRepo` from CodeCommit into the IDE and used Git for every change (E17-E18).
- **Network through a pipeline.** I created `cafe-network.yaml`, committed and pushed it, and `CafeNetworkPipeline` built the `update-cafe-network` stack with the Cafe VPC and public subnet (E19-E28).
- **Outputs for reuse.** I added exported outputs (subnet ID and VPC ID) to the network template and pushed again, so the pipeline updated the stack (E29-E33).
- **Application through a pipeline.** `cafe-app.yaml` adds an instance-type parameter and a `CafeInstance` web server that imports the subnet from the network stack. After the push, `CafeAppPipeline` created the `update-cafe-app` stack and the café site came up (E34-E41).
- **Second Region.** I created the network stack in us-west-2 with the CLI, created the `cafe-oregon` key pair, uploaded the app template to S3 and created `cafe-app-oregon` from the console with `t3.micro`. The same unmodified template produced a working café in Oregon (E42-E55).

## Challenges & Decisions

<!-- TODO (optional): add problems I hit in V6, if I remember -->
Two failures show up in the screenshots, and the root cause was not recorded in my notes: the application pipeline's Deploy stage failed on its first run and succeeded when run again (E35, E36), and the first upload of the application template failed with `NoSuchBucket` and succeeded on the second attempt (E47). The decisions that shaped this stage were:

- **Pipeline instead of manual runs** for the network and application stacks, so every change is a reviewed Git commit.
- **Two stacks (network and app)** linked through exported outputs, so the application template never hard-codes IDs.
- **Instance type as a parameter and a Region map for the key pair**, which let the same template run in another Region with different settings (`t3.micro` and `cafe-oregon` in Oregon).

## Validation & Evidence

![E26](../evidence/v6/V6-E26-update-cafe-network-stack-complete.png)
*E26 - update-cafe-network stack CREATE_COMPLETE.*

![E33](../evidence/v6/V6-E33-network-stack-outputs-exports.png)
*E33 - update-cafe-network outputs: PublicSubnet and VpcId with export names.*

![E37](../evidence/v6/V6-E37-app-pipeline-succeeded.png)
*E37 - CafeAppPipeline succeeded.*

![E41](../evidence/v6/V6-E41-cafe-website-us-east-1.png)
*E41 - Café website running in us-east-1 (server information shows Region/AZ).*

![E54](../evidence/v6/V6-E54-oregon-instance-t3-micro.png)
*E54 - Oregon Cafe Web Server instance (t3.micro).*

![E55](../evidence/v6/V6-E55-cafe-website-us-west-2.png)
*E55 - Café website running in us-west-2 (server information shows the Oregon AZ).*

### Evidence index (all screenshots)

| ID | What it shows | File |
|---|---|---|
| E01 | Empty VS Code IDE (code-server) on the lab EC2 instance, terminal ready. | `evidence/v6/V6-E01-ide-vscode-open.png` |
| E02 | Initial CloudFormation stacks created by the lab setup. | `evidence/v6/V6-E02-cloudformation-initial-stacks.png` |
| E03 | Initial S3 buckets created by the lab setup. | `evidence/v6/V6-E03-s3-initial-buckets.png` |
| E04 | S3.yaml (bucket only) and the first create-stack CLI runs. | `evidence/v6/V6-E04-s3-template-and-create-stack-cli.png` |
| E05 | CreateBucket stack creation in progress. | `evidence/v6/V6-E05-createbucket-stack-in-progress.png` |
| E06 | CreateBucket stack CREATE_COMPLETE. | `evidence/v6/V6-E06-createbucket-stack-complete.png` |
| E07 | S3 bucket created by the stack. | `evidence/v6/V6-E07-s3-bucket-created-by-stack.png` |
| E08 | Template tab of the CreateBucket stack. | `evidence/v6/V6-E08-createbucket-template-tab.png` |
| E09 | CodeCommit initial commit of the CFTemplatesRepo repository. | `evidence/v6/V6-E09-codecommit-initial-commit.png` |
| E10 | Website files downloaded and copied to the bucket with the AWS CLI. | `evidence/v6/V6-E10-upload-website-files-cli.png` |
| E11 | Website objects (index.html, css/, images/) in the bucket. | `evidence/v6/V6-E11-bucket-objects-uploaded.png` |
| E12 | S3.yaml updated with retain policy, website configuration and URL output; template validated and stack update started. | `evidence/v6/V6-E12-s3-template-website-and-update-stack.png` |
| E13 | CreateBucket stack UPDATE_IN_PROGRESS. | `evidence/v6/V6-E13-createbucket-update-in-progress.png` |
| E14 | CreateBucket stack UPDATE_COMPLETE. | `evidence/v6/V6-E14-createbucket-update-complete.png` |
| E15 | Stack Outputs tab showing the WebsiteURL. | `evidence/v6/V6-E15-createbucket-outputs-website-url.png` |
| E16 | Static website hosting enabled on the bucket. | `evidence/v6/V6-E16-bucket-static-website-hosting-enabled.png` |
| E17 | CFTemplatesRepo repository in CodeCommit with the clone URL copied. | `evidence/v6/V6-E17-codecommit-repo-clone-url.png` |
| E18 | git clone and git status: working tree clean. | `evidence/v6/V6-E18-git-clone-and-status.png` |
| E19 | cafe-network.yaml created from the template with the new description. | `evidence/v6/V6-E19-cafe-network-yaml-created.png` |
| E20 | CafeNetworkPipeline before the first push (Deploy shows Failed, as expected before a template exists). | `evidence/v6/V6-E20-network-pipeline-before-push.png` |
| E21 | git add and git commit of the network template. | `evidence/v6/V6-E21-git-add-commit-network-template.png` |
| E22 | git push of the network template to CodeCommit. | `evidence/v6/V6-E22-git-push-network-template.png` |
| E23 | CafeNetworkPipeline triggered: Deploy stage in progress. | `evidence/v6/V6-E23-network-pipeline-deploy-in-progress.png` |
| E24 | Pipelines list: CafeNetworkPipeline In progress, CafeAppPipeline Failed (no template yet). | `evidence/v6/V6-E24-pipelines-list-network-in-progress.png` |
| E25 | CafeNetworkPipeline Deploy stage finishing. | `evidence/v6/V6-E25-network-pipeline-deploy-finishing.png` |
| E26 | update-cafe-network stack CREATE_COMPLETE. | `evidence/v6/V6-E26-update-cafe-network-stack-complete.png` |
| E27 | Cafe VPC created by the pipeline. | `evidence/v6/V6-E27-cafe-vpc-created.png` |
| E28 | Cafe Public Subnet created by the pipeline. | `evidence/v6/V6-E28-cafe-public-subnet-created.png` |
| E29 | Outputs added to cafe-network.yaml and committed. | `evidence/v6/V6-E29-network-outputs-added-and-committed.png` |
| E30 | git push of the network outputs. | `evidence/v6/V6-E30-git-push-network-outputs.png` |
| E31 | CafeNetworkPipeline running the update. | `evidence/v6/V6-E31-network-pipeline-update-in-progress.png` |
| E32 | CafeNetworkPipeline update succeeded. | `evidence/v6/V6-E32-network-pipeline-update-succeeded.png` |
| E33 | update-cafe-network outputs: PublicSubnet and VpcId with export names. | `evidence/v6/V6-E33-network-stack-outputs-exports.png` |
| E34 | cafe-app.yaml (instance, parameter, user data) and git push. | `evidence/v6/V6-E34-cafe-app-yaml-and-git-push.png` |
| E35 | CafeAppPipeline Deploy stage failed on the first run. | `evidence/v6/V6-E35-app-pipeline-deploy-failed.png` |
| E36 | CafeAppPipeline running again. | `evidence/v6/V6-E36-app-pipeline-rerun-in-progress.png` |
| E37 | CafeAppPipeline succeeded. | `evidence/v6/V6-E37-app-pipeline-succeeded.png` |
| E38 | update-cafe-app stack CREATE_COMPLETE with the WebServerPublicIP output. | `evidence/v6/V6-E38-update-cafe-app-stack-complete.png` |
| E39 | EC2 console: Cafe Web Server (t2.small) created by the stack next to the Lab IDE instance. | `evidence/v6/V6-E39-ec2-cafe-web-server-instance.png` |
| E40 | CafeSG security group created by the app stack. | `evidence/v6/V6-E40-cafesg-security-group.png` |
| E41 | Café website running in us-east-1 (server information shows Region/AZ). | `evidence/v6/V6-E41-cafe-website-us-east-1.png` |
| E42 | CLI command creating the update-cafe-network stack in us-west-2. | `evidence/v6/V6-E42-create-network-stack-oregon-cli.png` |
| E43 | update-cafe-network stack in Oregon with its outputs. | `evidence/v6/V6-E43-oregon-network-stack-outputs.png` |
| E44 | Cafe VPC in us-west-2. | `evidence/v6/V6-E44-oregon-cafe-vpc.png` |
| E45 | Cafe Public Subnet in us-west-2. | `evidence/v6/V6-E45-oregon-cafe-public-subnet.png` |
| E46 | Key pair cafe-oregon created in Oregon. | `evidence/v6/V6-E46-oregon-key-pair-created.png` |
| E47 | Uploading cafe-app.yaml to the repository bucket: first attempt failed (NoSuchBucket), second succeeded. | `evidence/v6/V6-E47-upload-app-template-to-s3-cli.png` |
| E48 | Repository bucket containing cafe-app.yaml. | `evidence/v6/V6-E48-repobucket-cafe-app-yaml.png` |
| E49 | cafe-app.yaml object details with the object URL copied. | `evidence/v6/V6-E49-cafe-app-yaml-object-url.png` |
| E50 | cafe-app-oregon stack creation in progress. | `evidence/v6/V6-E50-cafe-app-oregon-stack-in-progress.png` |
| E51 | Stacks list in Oregon. | `evidence/v6/V6-E51-oregon-stacks-list.png` |
| E52 | cafe-app-oregon stack CREATE_COMPLETE. | `evidence/v6/V6-E52-cafe-app-oregon-create-complete.png` |
| E53 | cafe-app-oregon outputs (WebServerPublicIP). | `evidence/v6/V6-E53-cafe-app-oregon-outputs.png` |
| E54 | Oregon Cafe Web Server instance (t3.micro). | `evidence/v6/V6-E54-oregon-instance-t3-micro.png` |
| E55 | Café website running in us-west-2 (server information shows the Oregon AZ). | `evidence/v6/V6-E55-cafe-website-us-west-2.png` |

<details><summary><b>More evidence (all screenshots, in lab order)</b></summary>

#### Task 1 - IDE

![E01](../evidence/v6/V6-E01-ide-vscode-open.png)
*E01 - Empty VS Code IDE (code-server) on the lab EC2 instance, terminal ready.*

#### Task 2 - First stack from scratch (S3 bucket)

![E02](../evidence/v6/V6-E02-cloudformation-initial-stacks.png)
*E02 - Initial CloudFormation stacks created by the lab setup.*

![E03](../evidence/v6/V6-E03-s3-initial-buckets.png)
*E03 - Initial S3 buckets created by the lab setup.*

![E04](../evidence/v6/V6-E04-s3-template-and-create-stack-cli.png)
*E04 - S3.yaml (bucket only) and the first create-stack CLI runs.*

![E05](../evidence/v6/V6-E05-createbucket-stack-in-progress.png)
*E05 - CreateBucket stack creation in progress.*

![E06](../evidence/v6/V6-E06-createbucket-stack-complete.png)
*E06 - CreateBucket stack CREATE_COMPLETE.*

![E07](../evidence/v6/V6-E07-s3-bucket-created-by-stack.png)
*E07 - S3 bucket created by the stack.*

![E08](../evidence/v6/V6-E08-createbucket-template-tab.png)
*E08 - Template tab of the CreateBucket stack.*

#### Task 3 - Website hosting and stack update

![E09](../evidence/v6/V6-E09-codecommit-initial-commit.png)
*E09 - CodeCommit initial commit of the CFTemplatesRepo repository.*

![E10](../evidence/v6/V6-E10-upload-website-files-cli.png)
*E10 - Website files downloaded and copied to the bucket with the AWS CLI.*

![E11](../evidence/v6/V6-E11-bucket-objects-uploaded.png)
*E11 - Website objects (index.html, css/, images/) in the bucket.*

![E12](../evidence/v6/V6-E12-s3-template-website-and-update-stack.png)
*E12 - S3.yaml updated with retain policy, website configuration and URL output; template validated and stack update started.*

![E13](../evidence/v6/V6-E13-createbucket-update-in-progress.png)
*E13 - CreateBucket stack UPDATE_IN_PROGRESS.*

![E14](../evidence/v6/V6-E14-createbucket-update-complete.png)
*E14 - CreateBucket stack UPDATE_COMPLETE.*

![E15](../evidence/v6/V6-E15-createbucket-outputs-website-url.png)
*E15 - Stack Outputs tab showing the WebsiteURL.*

![E16](../evidence/v6/V6-E16-bucket-static-website-hosting-enabled.png)
*E16 - Static website hosting enabled on the bucket.*

#### Task 4 - CodeCommit repository

![E17](../evidence/v6/V6-E17-codecommit-repo-clone-url.png)
*E17 - CFTemplatesRepo repository in CodeCommit with the clone URL copied.*

![E18](../evidence/v6/V6-E18-git-clone-and-status.png)
*E18 - git clone and git status: working tree clean.*

#### Task 5 - Network layer through the pipeline

![E19](../evidence/v6/V6-E19-cafe-network-yaml-created.png)
*E19 - cafe-network.yaml created from the template with the new description.*

![E20](../evidence/v6/V6-E20-network-pipeline-before-push.png)
*E20 - CafeNetworkPipeline before the first push (Deploy shows Failed, as expected before a template exists).*

![E21](../evidence/v6/V6-E21-git-add-commit-network-template.png)
*E21 - git add and git commit of the network template.*

![E22](../evidence/v6/V6-E22-git-push-network-template.png)
*E22 - git push of the network template to CodeCommit.*

![E23](../evidence/v6/V6-E23-network-pipeline-deploy-in-progress.png)
*E23 - CafeNetworkPipeline triggered: Deploy stage in progress.*

![E24](../evidence/v6/V6-E24-pipelines-list-network-in-progress.png)
*E24 - Pipelines list: CafeNetworkPipeline In progress, CafeAppPipeline Failed (no template yet).*

![E25](../evidence/v6/V6-E25-network-pipeline-deploy-finishing.png)
*E25 - CafeNetworkPipeline Deploy stage finishing.*

![E26](../evidence/v6/V6-E26-update-cafe-network-stack-complete.png)
*E26 - update-cafe-network stack CREATE_COMPLETE.*

![E27](../evidence/v6/V6-E27-cafe-vpc-created.png)
*E27 - Cafe VPC created by the pipeline.*

![E28](../evidence/v6/V6-E28-cafe-public-subnet-created.png)
*E28 - Cafe Public Subnet created by the pipeline.*

#### Task 6 - Network stack outputs

![E29](../evidence/v6/V6-E29-network-outputs-added-and-committed.png)
*E29 - Outputs added to cafe-network.yaml and committed.*

![E30](../evidence/v6/V6-E30-git-push-network-outputs.png)
*E30 - git push of the network outputs.*

![E31](../evidence/v6/V6-E31-network-pipeline-update-in-progress.png)
*E31 - CafeNetworkPipeline running the update.*

![E32](../evidence/v6/V6-E32-network-pipeline-update-succeeded.png)
*E32 - CafeNetworkPipeline update succeeded.*

![E33](../evidence/v6/V6-E33-network-stack-outputs-exports.png)
*E33 - update-cafe-network outputs: PublicSubnet and VpcId with export names.*

#### Task 7 - Application stack through the pipeline

![E34](../evidence/v6/V6-E34-cafe-app-yaml-and-git-push.png)
*E34 - cafe-app.yaml (instance, parameter, user data) and git push.*

![E35](../evidence/v6/V6-E35-app-pipeline-deploy-failed.png)
*E35 - CafeAppPipeline Deploy stage failed on the first run.*

![E36](../evidence/v6/V6-E36-app-pipeline-rerun-in-progress.png)
*E36 - CafeAppPipeline running again.*

![E37](../evidence/v6/V6-E37-app-pipeline-succeeded.png)
*E37 - CafeAppPipeline succeeded.*

![E38](../evidence/v6/V6-E38-update-cafe-app-stack-complete.png)
*E38 - update-cafe-app stack CREATE_COMPLETE with the WebServerPublicIP output.*

![E39](../evidence/v6/V6-E39-ec2-cafe-web-server-instance.png)
*E39 - EC2 console: Cafe Web Server (t2.small) created by the stack next to the Lab IDE instance.*

![E40](../evidence/v6/V6-E40-cafesg-security-group.png)
*E40 - CafeSG security group created by the app stack.*

![E41](../evidence/v6/V6-E41-cafe-website-us-east-1.png)
*E41 - Café website running in us-east-1 (server information shows Region/AZ).*

#### Task 8 - Second Region (us-west-2)

![E42](../evidence/v6/V6-E42-create-network-stack-oregon-cli.png)
*E42 - CLI command creating the update-cafe-network stack in us-west-2.*

![E43](../evidence/v6/V6-E43-oregon-network-stack-outputs.png)
*E43 - update-cafe-network stack in Oregon with its outputs.*

![E44](../evidence/v6/V6-E44-oregon-cafe-vpc.png)
*E44 - Cafe VPC in us-west-2.*

![E45](../evidence/v6/V6-E45-oregon-cafe-public-subnet.png)
*E45 - Cafe Public Subnet in us-west-2.*

![E46](../evidence/v6/V6-E46-oregon-key-pair-created.png)
*E46 - Key pair cafe-oregon created in Oregon.*

![E47](../evidence/v6/V6-E47-upload-app-template-to-s3-cli.png)
*E47 - Uploading cafe-app.yaml to the repository bucket: first attempt failed (NoSuchBucket), second succeeded.*

![E48](../evidence/v6/V6-E48-repobucket-cafe-app-yaml.png)
*E48 - Repository bucket containing cafe-app.yaml.*

![E49](../evidence/v6/V6-E49-cafe-app-yaml-object-url.png)
*E49 - cafe-app.yaml object details with the object URL copied.*

![E50](../evidence/v6/V6-E50-cafe-app-oregon-stack-in-progress.png)
*E50 - cafe-app-oregon stack creation in progress.*

![E51](../evidence/v6/V6-E51-oregon-stacks-list.png)
*E51 - Stacks list in Oregon.*

![E52](../evidence/v6/V6-E52-cafe-app-oregon-create-complete.png)
*E52 - cafe-app-oregon stack CREATE_COMPLETE.*

![E53](../evidence/v6/V6-E53-cafe-app-oregon-outputs.png)
*E53 - cafe-app-oregon outputs (WebServerPublicIP).*

![E54](../evidence/v6/V6-E54-oregon-instance-t3-micro.png)
*E54 - Oregon Cafe Web Server instance (t3.micro).*

![E55](../evidence/v6/V6-E55-cafe-website-us-west-2.png)
*E55 - Café website running in us-west-2 (server information shows the Oregon AZ).*

</details>

> Evidence note: the us-west-2 screenshots come from a different account ID than the us-east-1 ones. Account details and public IPs were masked.

## Outcome

The café's network and web server are now defined as CloudFormation templates in CodeCommit and deployed by pipelines on every push. The same templates created a second working copy of the café in us-west-2.

---

Previous: [V5 - Scalability & High Availability](v5-scalability-ha.md) | Next: [V7 - Serverless Reporting](v7-serverless-reporting.md)

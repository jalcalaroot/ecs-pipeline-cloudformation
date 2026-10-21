# ECS CI/CD Pipeline (CloudFormation)

CloudFormation template that creates a CodePipeline which pulls from a CodeCommit repository, builds a
Docker image with CodeBuild and deploys it to an existing ECS service.

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

```
CodeCommit (branch master) --> CodeBuild (docker build/push to ECR) --> ECS deploy (imagedefinitions.json)
```

- Two private S3 buckets (public access blocked): one for the CodeBuild cache, one for pipeline artifacts.
- IAM roles for CodePipeline and CodeBuild.
- CodeBuild project (`amazonlinux2-x86_64-standard:4.0`, small, privileged mode for Docker) with
  environment variables `AWS_DEFAULT_REGION`, `AWS_ACCOUNT_ID`, `IMAGE_REPO_NAME`, `REPOSITORY_URI`,
  `IMAGE_TAG=latest` and `CONTAINER_NAME`.
- CodePipeline `<Environment>-<ProjectName>-pipeline` with Source, Build and Deploy (ECS) stages.

Not created by this template: the CodeCommit repository, the ECR repository, the ECS cluster/service and
the `buildspec.yml`. The buildspec in the source repo must push the image and emit the
`imagedefinitions.json` file used by the ECS deploy action.

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `Environment` | `Sandbox`, `prod`, `qa` or `dev`; used in names and tags | `Sandbox` |
| `ProjectName` | Pipeline name component | `Project` |
| `PipelineRepository` | CodeCommit repository with the Dockerfile and buildspec | `dockerrepo` |
| `VpcId` | VPC ID (declared but not used by any resource) | - |
| `Owner` | Owner name (declared but not used) | - |
| `ECSClusterName` | Target ECS cluster | - |
| `ECSServiceName` | Target ECS service | - |
| `RepoURI` | ECR repository URI | `<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/ECR` |
| `ECRName` | ECR repository name | - |
| `BuildName` | CodeBuild project name | `MyCodeBuildProject` |
| `ContainerName` | Container name in the task definition | - |
| `FileName` | Image definitions file for the ECS deploy | `imagedefinitions.json` |

## Deploy

```bash
aws cloudformation deploy \
  --stack-name ecs-pipeline \
  --template-file ECSPipelineCloudformationTemplate.yaml \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
      VpcId=<VPC_ID> Owner=<OWNER> PipelineRepository=<CODECOMMIT_REPO> \
      ECSClusterName=<CLUSTER> ECSServiceName=<SERVICE> ContainerName=<CONTAINER> \
      ECRName=<ECR_REPO> RepoURI=<AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<ECR_REPO>
```

The source action has `PollForSourceChanges: false` and the template creates no EventBridge rule, so the
pipeline does not start automatically on push; run it with `aws codepipeline start-pipeline-execution`
or add a rule.

## Security notes

- **Both the CodePipeline and CodeBuild roles have `AdministratorAccess`.** Replace with least-privilege
  policies (S3 artifacts, CodeCommit, ECR push, ECS update, logs).
- CodeBuild runs in privileged mode (required for Docker builds).
- The S3 buckets have no explicit encryption, versioning or lifecycle settings (S3 default encryption applies).
- Images are always tagged `latest`.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name ecs-pipeline
```

Empty the cache and artifact buckets first, or the stack deletion will fail on them.

## License

MIT — see [LICENSE](LICENSE).

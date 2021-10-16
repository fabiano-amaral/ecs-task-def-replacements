## ecs-task-def-replacements

Help on aws ECS deploy, getting the current task running on a service, or the latest revision from a task definition family, replacing any field on the running/latest task definition.

Plays nice with:

- https://github.com/marketplace/actions/amazon-ecs-deploy-task-definition-action-for-github-actions

- https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions

- https://github.com/marketplace/actions/amazon-ecr-login-action-for-github-actions

## Usage for ECS service

Case you are using for get the currently running task definition (**not** the last revision, this is very useful when the last task definition isn't running on serive, maybe because a rollback), you need to specify the cluster name and service name

```yml
- name: Retrieve trask def
  uses: fabiano-amaral/ecs-task-def-replacements@main
  id: task
  with:
    cluster-name: my-ecs-cluster
    service-name: my-service-name
    replacements: |
      {
        "containerDefinitions": [{
          "image": "my-new-image"
        }]
      }
```

If you are interested to get the last revision of a task definition, you can use in this way

```yml
- name: Retrieve trask def
  uses: fabiano-amaral/ecs-task-def-replacements@main
  id: task
  with:
    task-name: my-task-name
    replacements: |
      {
        "containerDefinitions": [{
          "image": "my-new-image"
        }]
      }
```

## Pipeline that use this Action

```yml
name: Build and Deploy
on:
  push:
    branches:
      - main
jobs:
  build_and_deploy:
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout and set Output
        id: checkout_code
        uses: actions/checkout@v2

      - name: Configure AWS credentials
        id: login_aws_credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
          mask-aws-account-id: 'no'

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Build, tag, and push image to Amazon ECR
        id: build
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: my-service-repository
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA::7} --build-arg DD_VERSION=${GITHUB_SHA::7} .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA::7}
          echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA::7}"

      - name: Retrieve trask def
        uses: fabiano-amaral/ecs-task-def-replacements@main
        id: task
        with:
          cluster-name: my-cluster
          service-name: my-service
          replacements: |
            {
              "containerDefinitions": [{
                "image": "${{ steps.build.outputs.image }}"
              }]
            }

      - name: Deploy to Amazon ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task.outputs.taskDef }}
          service: my-service
          cluster: my-cluster
          wait-for-service-stability: true
```
## Inputs

|Input|required|Description|
|---|---|---|
|cluster-name|true|Cluster that service is running|
|service-name|false*|Service to get current running task definition|
|task-name|false*|Task family to get the latest task definition revision|
|replacements|true|JSON (stringified, see examples above) to be merged with the data retrieved|

*you **must** to pass one of service-name or task-name

## Outputs

taskDef (file path): Path to the json file that contains the merges output
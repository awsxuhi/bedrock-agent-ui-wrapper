

## Installation

cd the project directory, then run:

```shell
pyenv local 3.13
poetry env use python3.13
poetry env activate
source /home/ubuntu/.cache/pypoetry/virtualenvs/capsule-zEvZUBl7-py3.13/bin/activate
(capsule-py3.13) (base) ➜

poetry add PyYAML
poetry install

aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws
poetry run cdk deploy --verbose --all --require-approval never
```


## Change Logs

infra/frontend_construct/frontend_fargate_construct.py
update: line 495
  target_group_name=(f"{construct_id}-action-tg")[:32],  

  AWS 对目标组名称有严格限制，名称长度最多为 32 个字符。你当前的名称 "capsule-booking-agent-frontend-action-tg" 超过了这个长度，导致部署时验证失败。
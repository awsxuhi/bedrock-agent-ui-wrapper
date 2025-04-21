# Capsule 项目技术实现细节

本文档详细描述 Capsule 项目的关键技术实现细节，帮助开发者理解系统的内部工作原理。

## GraphQL API 实现

### Schema 定义

Capsule 项目使用 GraphQL 作为 API 层，其 schema 定义如下：

```graphql
schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}

type Query {
  # 占位查询
  noop: String
}

enum StreamStatus {
  STARTED
  STREAMING
  COMPLETED
  ERROR
}

type Mutation {
  askAgent(question: String!, sessionId: String!): AgentResponse!
  publishAgentUpdate(sessionId: String!, content: String, status: StreamStatus!): AgentStreamResponse @aws_iam
}

type Subscription {
  onAgentResponse(sessionId: String!): AgentStreamResponse @aws_subscribe(mutations: ["publishAgentUpdate"])
}

type AgentResponse {
  sessionId: String!
  status: StreamStatus!
}

type AgentStreamResponse {
  sessionId: String! @aws_iam
  content: String @aws_iam
  status: StreamStatus! @aws_iam
}
```

### 关键点解析

1. **双重授权机制**：

   - 用户请求通过 Cognito 用户池授权
   - 服务间通信通过 IAM 授权（注意 `@aws_iam` 指令）

2. **实时订阅**：

   - 使用 `@aws_subscribe` 指令将订阅与 mutation 关联
   - 客户端可以订阅特定会话 ID 的更新

3. **流式状态管理**：
   - 使用 `StreamStatus` 枚举跟踪响应状态
   - 支持 STARTED、STREAMING、COMPLETED 和 ERROR 状态

## Lambda 函数实现

### Ask Agent Handler

Ask Agent Handler 是处理用户请求的入口点，其核心实现如下：

```python
def _lambda_handler(event):
    try:
        question = event['question']
        session_id = event['sessionId']

        # 异步调用流处理 handler
        lambda_client.invoke(
            FunctionName=STREAM_HANDLER_ARN,
            InvocationType='Event',  # 异步调用
            Payload=json.dumps({
                'sessionId': session_id,
                'question': question
            })
        )

        # 立即返回会话 ID
        return {
            'sessionId': session_id,
            'status': 'STARTED'
        }
    except Exception as e:
        return {
            'sessionId': event.get('sessionId', "Unknown sessionId"),
            'status': 'ERROR',
            'error': str(e)
        }
```

关键实现点：

- 接收用户问题和会话 ID
- 异步调用 Stream Handler（使用 `InvocationType='Event'`）
- 立即返回会话 ID 和状态，不等待处理完成

### Stream Handler

Stream Handler 负责与 Bedrock Agent 交互并处理流式响应：

```python
def _lambda_handler(event):
    try:
        session_id = event['sessionId']
        question = event['question']

        # 调用 Bedrock Agent
        response = bedrock_agent_runtime.invoke_agent(
            agentId=AGENT_ID,
            agentAliasId=AGENT_ALIAS_ID,
            sessionId=session_id,
            inputText=question,
            enableTrace=True
        )

        # 处理流式响应
        stream = response['completion']
        for event in stream:
            parsed_step = process_trace_step(event)
            if parsed_step:
                publish_to_appsync(
                    session_id=session_id,
                    content=json.dumps(parsed_step, indent=2),
                    status=StreamStatus.COMPLETED if 'Result' in parsed_step else StreamStatus.STREAMING
                )
        return {'status': 'COMPLETED'}
    except Exception as e:
        # 发布错误状态
        publish_to_appsync(
            session_id=session_id,
            content=str(e),
            status=StreamStatus.ERROR
        )
        return {'status': 'ERROR', 'error': str(e)}
```

关键实现点：

- 调用 Bedrock Agent API 并启用跟踪
- 处理流式响应事件
- 解析和格式化 Agent 输出
- 通过 AppSync 发布更新
- 错误处理和状态管理

### 响应处理

Stream Handler 包含一个 `process_trace_step` 函数，用于处理 Bedrock Agent 的响应：

```python
def process_trace_step(trace: dict) -> dict[str, str]:
    if 'chunk' in trace and "bytes" in trace["chunk"]:
        full_result_message = trace["chunk"]["bytes"].decode()
        try:
            result_str = extract_answer(full_result_message)
        except Exception:
            result_str = full_result_message
        return {"Result": result_str}
    trace = trace.get("trace", {}).get("trace", {}).get("orchestrationTrace", {})
    if "rationale" in trace:
        return {"Rationale": trace["rationale"].get("text", "")}
    elif "invocationInput" in trace:
        return {"Invocation": trace["invocationInput"].get("actionGroupInvocationInput", "")}
    elif "observation" in trace:
        return {"Observation": trace["observation"]}
    return ""
```

该函数处理不同类型的响应：

- 结果块（最终答案）
- 推理过程
- 调用输入
- 观察结果

## AppSync 解析器实现

### askAgent 解析器

```javascript
{
    "version": "2018-05-29",
    "operation": "Invoke",
    "payload": {
        "question": $util.toJson($context.arguments.question),
        "sessionId": $util.toJson($context.arguments.sessionId)
    }
}
```

### publishAgentUpdate 解析器

```javascript
{
    "version": "2018-05-29",
    "payload": {
        "sessionId": $util.toJson($ctx.arguments.sessionId),
        "content": $util.toJson($ctx.arguments.content),
        "trace": $util.toJson($ctx.arguments.trace)
    }
}
```

## 认证与授权实现

### Cognito 用户池配置

```python
self.user_pool = cognito.UserPool(
    self, f"{construct_id}-userpool",
    user_pool_name=f"{construct_id}-userpool",
    auto_verify=cognito.AutoVerifiedAttrs(email=True),
    removal_policy=RemovalPolicy.DESTROY,
    mfa=cognito.Mfa.REQUIRED,
    mfa_second_factor={
        "sms": False,
        "otp": True
    },
    password_policy=cognito.PasswordPolicy(
        min_length=8,
        require_lowercase=True,
        require_uppercase=True,
        require_digits=True,
        require_symbols=True
    ),
    account_recovery=cognito.AccountRecovery.EMAIL_ONLY
)
```

### AppSync API 授权配置

```python
self.api = appsync.GraphqlApi(
    self,
    f"{construct_id}-graphqlapi",
    name=f"{construct_id}-graphqlapi",
    schema=appsync.SchemaFile.from_asset("schema.graphql"),
    authorization_config=appsync.AuthorizationConfig(
        default_authorization=appsync.AuthorizationMode(
            authorization_type=appsync.AuthorizationType.USER_POOL,
            user_pool_config=appsync.UserPoolConfig(
                user_pool=self.user_pool
            )
        ),
        additional_authorization_modes=[
            appsync.AuthorizationMode(
                authorization_type=appsync.AuthorizationType.IAM
            )
        ]
    ),
    log_config=appsync.LogConfig(
        field_log_level=appsync.FieldLogLevel.ALL,
        exclude_verbose_content=False,
        role=appsync_log_role
    ),
    xray_enabled=True
)
```

## 前端实现

Streamlit 前端实现了以下关键功能：

1. **用户认证**：

   - 使用 Cognito 进行用户认证
   - 管理认证令牌和会话

2. **GraphQL 客户端**：

   - 发送 GraphQL 请求
   - 处理 WebSocket 订阅

3. **UI 组件**：
   - 聊天界面
   - 消息显示
   - 输入控件

## 安全实现

### IAM 角色和策略

Stream Handler 的 IAM 角色配置：

```python
stream_handler_role = iam.Role(
    self,
    "StreamHandlerRole",
    assumed_by=iam.ServicePrincipal("lambda.amazonaws.com"),
    inline_policies={
        "CloudWatchLogsPolicy": iam.PolicyDocument(
            statements=[
                iam.PolicyStatement(
                    effect=iam.Effect.ALLOW,
                    actions=[
                        "logs:CreateLogGroup",
                        "logs:CreateLogStream",
                        "logs:PutLogEvents"
                    ],
                    resources=[
                        f"arn:aws:logs:{Stack.of(self).region}:{Stack.of(self).account}:log-group:/aws/lambda/{Stack.of(self).stack_name}-*",
                        f"arn:aws:logs:{Stack.of(self).region}:{Stack.of(self).account}:log-group:/aws/lambda/{Stack.of(self).stack_name}-*:log-stream:*"
                    ]
                )
            ]
        )
    }
)
```

### Bedrock 权限

```python
self.stream_handler.add_to_role_policy(
    iam.PolicyStatement(
        effect=iam.Effect.ALLOW,
        actions=[
            "bedrock:InvokeAgent",
            "bedrock-agent-runtime:InvokeAgent"
        ],
        resources=[
            f"arn:aws:bedrock:{region}:{account}:agent/{agent_id}",
            f"arn:aws:bedrock:{region}:{account}:agent-alias/{agent_id}/{agent_alias_id}"
        ]
    )
)
```

### AppSync 权限

```python
self.stream_handler.add_to_role_policy(
    iam.PolicyStatement(
        actions=["appsync:GraphQL"],
        resources=[
            f"{self.api.arn}/types/Mutation/fields/publishAgentUpdate"
        ]
    )
)
```

## 配置管理

项目使用 AWS Secrets Manager 存储配置和敏感信息：

```python
self.bedrock_agent_secrets = secretsmanager.Secret(
    self, "BedrockAgentSecrets",
    secret_name="bedrock-agent-secrets",
    generate_secret_string=secretsmanager.SecretStringGenerator(
        secret_string_template=json.dumps({
            "AGENT_ID": agent_id,
            "AGENT_ALIAS_ID": agent_alias_id,
            "APPSYNC_API_ID": self.api.api_id,
            "APPSYNC_ENDPOINT": self.api.graphql_url,
            "AWS_REGION": Stack.of(self).region
        }),
        generate_string_key="dummy"  # 必需但未使用
    ),
    removal_policy=RemovalPolicy.DESTROY
)
```

## 部署实现

### CDK 堆栈

```python
class CapsuleStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, config: Dict[str, Any], **kwargs) -> None:
        super().__init__(scope, construct_id, description='UI for Bedrock Agent with streaming support', **kwargs)

        ## 设置 Agent 信息
        if config["agent_id"]:
            self.agent_id = config["agent_id"]
            self.agent_alias_id = config["agent_alias_id"]
        else:  # 如果未提供 agent 信息，则创建测试用 agent
            test_agent = SimpleBedrockAgentConstruct(self, f"{construct_id}-TestAgent")
            self.agent_id = test_agent.agent_id
            self.agent_alias_id = test_agent.agent_alias_id

        ## API 层
        self.local_redirect_uri = "http://localhost:8501"  # 默认本地托管
        api_auth_construct = ApiAuthConstruct(self,
                                              construct_id=f"{construct_id}-api",
                                              agent_id=self.agent_id,
                                              agent_alias_id=self.agent_alias_id,
                                              region=self.region,
                                              account=self.account,
                                              redirect_uri=self.local_redirect_uri
                                              )

        ## 前端层
        if config["deploy_on_fargate"]:
            # 创建前端构造（不含 Fargate 服务）
            frontend = FrontendFargateConstruct(
                self,
                construct_id=f"{construct_id}-frontend",
                api_auth_construct=api_auth_construct
            )
```

## 性能优化

1. **异步处理**：

   - 使用异步 Lambda 调用分离请求处理和响应处理
   - 立即返回会话 ID，不阻塞用户界面

2. **流式传输**：

   - 使用 GraphQL 订阅实现实时更新
   - 增量传输响应，而不是等待完整响应

3. **Lambda 配置**：
   - Stream Handler 超时设置为 15 分钟，支持长时间运行的会话
   - Ask Agent Handler 超时设置为 1 分钟，足够处理初始请求

## 扩展点

1. **自定义响应处理**：

   - 修改 `process_trace_step` 函数以自定义响应处理逻辑
   - 添加额外的格式化或过滤步骤

2. **多 Agent 支持**：

   - 修改 schema 以支持多个 Bedrock Agents
   - 添加 agent 选择参数

3. **会话管理**：

   - 添加 DynamoDB 表存储会话历史
   - 实现会话上下文管理

4. **文件处理**：
   - 添加 S3 集成以支持文件上传和下载
   - 实现文档处理功能

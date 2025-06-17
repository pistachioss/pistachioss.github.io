---
title: llm-agent工具开发
date: 2025-03-12
categories: llm
tags: wiki
toc: true
description: "基于springboot开发agent"
---

开发一个基于 Spring Boot 和 Vue 的大模型 Agent，按照以下提示开发，最后你需要尽量去优化我没想到的地方，优化页面，让大模型Agent功能和市面上通用智能体差不多

## 后端 (Spring Boot) 编码提示词示例：

### 建基础项目结构：

   - "为我生成一个 Spring Boot 3.x 项目的骨架，包含 Web, Lombok, Spring Data JPA (使用 H2 内存数据库作为示例) 和 Spring Security (基础认证) 的依赖。项目使用 Maven 管理，Java 版本为 17。"
   - "请为这个 Spring Boot 项目创建一个基础的目录结构，包括 controller, service, repository, model, config, dto 等包。"

### 义数据模型 (Entity/DTO)：

   - "为一个大模型 Agent 创建一个 Spring Boot JPA 实体类 `AgentTask`，它应该包含以下字段：`id` (Long, 主键, 自增), `taskName` (String, 非空), `prompt` (String, 文本类型), `status` (枚举类型 `TaskStatus`: PENDING, RUNNING, COMPLETED, FAILED), `createdAt` (LocalDateTime, 自动生成创建时间), `updatedAt` (LocalDateTime, 自动生成更新时间)。请同时生成相应的 Lombok注解 (`@Data`, `@Entity` 等)。"
   - "为 `AgentTask` 实体创建一个对应的 DTO 类 `AgentTaskDTO`，包含 `id`, `taskName`, `prompt`, `status` 字段。并为 `AgentTask` 和 `AgentTaskDTO` 之间编写一个 MapStruct 的映射接口。"

### 建 Repository 层：

   - "为 `AgentTask` 实体创建一个 Spring Data JPA Repository 接口 `AgentTaskRepository`，继承自 `JpaRepository`。并添加一个根据 `status` 查询任务列表的方法。"

<!--more-->

###  Service 层：

   - "创建一个 Spring Boot Service 类 

     ```
     AgentTaskService
     ```

     ，注入 

     ```
     AgentTaskRepository
     ```

     。实现以下方法：

     - `createTask(AgentTaskDTO taskDTO)`: 创建一个新的 Agent 任务。
     - `getTaskById(Long id)`: 根据 ID 获取任务详情。
     - `getAllTasks()`: 获取所有任务列表。
     - `updateTaskStatus(Long id, TaskStatus status)`: 更新任务状态。
     - `deleteTask(Long id)`: 删除任务。"

   - "在 `AgentTaskService` 的 `createTask` 方法中，添加逻辑：在保存任务到数据库之前，记录一条 info 级别的日志，内容为 'Creating new agent task with name: [taskName]'。"

### 建 Controller 层 (REST API)：

   - "创建一个 Spring Boot REST Controller 

     ```
     AgentTaskController
     ```

     ，注入 

     ```
     AgentTaskService
     ```

     。为以下操作创建 API 接口：

     - `POST /api/tasks`: 创建新任务，请求体为 `AgentTaskDTO`，返回创建后的 `AgentTaskDTO`。
     - `GET /api/tasks/{id}`: 根据 ID 获取任务，返回 `AgentTaskDTO`。
     - `GET /api/tasks`: 获取所有任务列表，返回 `List<AgentTaskDTO>`。
     - `PUT /api/tasks/{id}/status`: 更新任务状态，路径参数为任务 ID，请求参数为新的 `status`，返回更新后的 `AgentTaskDTO`。
     - `DELETE /api/tasks/{id}`: 删除任务，返回操作成功的消息。"

   - "为 `AgentTaskController` 中的所有 API 接口添加合适的 `@ApiOperation` (Swagger) 注解，描述接口功能、参数和响应。"

   - "在 `AgentTaskController` 中实现全局异常处理，捕获 `ResourceNotFoundException` 并返回 404 状态码和错误信息。"

### 大模型交互的 Service：

   - "创建一个 Spring Boot Service 类 `LargeModelService`，用于与外部大模型 API 进行交互。假设大模型 API 的地址是 `https://api.example-llm.com/generate`，需要通过 HTTP POST 请求发送 JSON 数据，包含 `prompt` 字段，并期望返回一个包含 `result` 字段的 JSON 响应。"

   - "在 

     ```
     LargeModelService
     ```

      中实现一个方法 

     ```
     executePrompt(String prompt)
     ```

     :

     - 使用 `RestTemplate` 或 `WebClient` 向大模型 API 发送请求。
     - 处理可能的 API 错误和网络异常。
     - 返回大模型生成的文本结果。
     - (可选) 添加重试逻辑，如果 API 调用失败则重试最多3次，每次间隔5秒。"

   - "将 `LargeModelService` 注入到 `AgentTaskService` 中。修改 `AgentTaskService` 的 `createTask` 方法，在任务状态变为 `RUNNING` 后，调用 `LargeModelService.executePrompt` 方法，并将结果更新到任务的某个字段（例如，`resultText`）。"

7. **配置和安全：**

   - "为 Spring Boot 应用生成一个 `application.yml` 或 `application.properties` 文件，配置数据库连接信息 (H2)、服务器端口 (例如 8080)，以及一个自定义配置项 `llm.api.key` 用于存放调用大模型 API 的密钥。"
   - "使用 Spring Security 配置基于 JWT 的认证和授权。创建一个 `JwtTokenProvider` 类用于生成和验证 Token。保护 `/api/tasks` 下的所有接口，除了允许匿名访问的注册和登录接口。"

## 前端 (Vue) 编码提示词示例：

### 建基础项目结构和组件：

   - "使用 Vue 3 (Composition API) 和 Vite 创建一个新的前端项目。安装 Axios 用于 API 请求，以及一个 UI 库，例如 Element Plus 或 Vuetify。"
   - "创建一个 Vue 组件 `TaskList.vue`，用于显示 Agent 任务列表。"
   - "创建一个 Vue 组件 `TaskForm.vue`，用于创建新的 Agent 任务，包含任务名称和 Prompt 输入框。"
   - "创建一个 Vue 组件 `TaskDetail.vue`，用于显示单个任务的详细信息。"

### PI 服务封装：

   - "在 Vue 项目的 `src` 目录下创建一个 `services` 文件夹，并在其中创建一个 `taskService.js` 文件。封装与后端 `/api/tasks` 相关的 Axios API 请求方法，例如 `WorkspaceTasks`, `WorkspaceTaskById`, `createTask`, `updateTaskStatus`, `deleteTask`。"

### 态管理 (Pinia 或 Vuex)：

   - "使用 Pinia 为 Agent 任务创建一个 store (`taskStore.js`)。这个 store 应该包含以下 state: `tasks` (数组), `currentTask` (对象), `loading` (布尔值), `error` (字符串或对象)。"
   - "为 `taskStore.js` 添加 actions 用于调用 `taskService.js` 中的方法来获取和修改任务数据，并更新 state。例如：`loadTasksAction`, `createTaskAction`。"

### 件实现 - 任务列表 (`TaskList.vue`)：

   - "在 

     ```
     TaskList.vue
     ```

      组件中：

     - 使用 `taskStore` 获取任务列表并展示在一个表格中，表格列应包括：ID, 任务名称, 状态, 创建时间。
     - 每行提供一个“查看详情”按钮和一个“删除”按钮。
     - 点击“查看详情”按钮，导航到任务详情页。
     - 点击“删除”按钮，调用 `taskStore` 中的 action 删除任务并刷新列表。"

   - "为 `TaskList.vue` 中的表格添加分页功能。"

   - "在 `TaskList.vue` 中添加一个“创建新任务”的按钮，点击后弹出一个模态框或导航到创建任务的表单页面。"

### 件实现 - 创建任务表单 (`TaskForm.vue`)：

   - "在 

     ```
     TaskForm.vue
     ```

      组件中：

     - 包含两个输入字段：任务名称 (必填) 和 Prompt (文本域, 必填)。
     - 一个“提交”按钮。
     - 点击“提交”按钮时，获取表单数据，调用 `taskStore` 中的 `createTaskAction` 创建任务。
     - 成功创建后，清空表单并给出提示，或者导航到任务列表页。"

   - "为 `TaskForm.vue` 中的输入字段添加基础的表单校验规则。"

### 件实现 - 任务详情 (`TaskDetail.vue`)：

   - "在 

     ```
     TaskDetail.vue
     ```

      组件中：

     - 从路由参数中获取任务 ID。
     - 使用 `taskStore` 根据 ID 加载任务详情。
     - 显示任务的名称, Prompt, 状态, 创建时间, 更新时间, 以及大模型返回的结果 (如果后端实现了)。
     - 提供一个下拉框或按钮组，允许用户更新任务的状态 (例如，从 PENDING 改为 RUNNING，或标记为 COMPLETED/FAILED)。"

### 由配置 (`router/index.js`)：

   - "为 Vue 应用配置路由：
     - `/tasks`: 对应 `TaskList.vue` 组件。
     - `/tasks/new`: 对应 `TaskForm.vue` 组件。
     - `/tasks/:id`: 对应 `TaskDetail.vue` 组件。"

### I 交互和样式：

   - "在 `TaskList.vue` 中，当任务状态为 `RUNNING` 时，显示一个加载中的动画或图标。"
   - "使用 Element Plus 的 `ElLoading` 组件，在 API 请求期间显示全局加载遮罩。"
   - "为 `TaskForm.vue` 的表单元素添加一些基本的 CSS 样式，使其更美观。"

## LangChain LLM流程

**LangChain (或更准确地说，是 LangChain4j 中的 `AiServices` 或 Agent 执行器) 之所以知道要调用 `DateTimeTool`，是 LLM (大语言模型，如 DeepSeek) “告诉”它的。**

以下是这个过程的分解说明：


###  LLM 提供工具信息 (Prompt Engineering / Function Calling Metadata):
- 当你构建 `AiServices` 并传入 `.tools(this.dateTimeTool)` 时，LangChain4j 会在内部做几件事情：
    - **提取工具元数据:** 它会检查 DateTimeTool 类中所有被 @Tool 注解标记的方法 (如 getCurrentDateTime, getCurrentDate)。
    - 对于每个工具方法，它会收集：
      - **工具名称:** 方法名 (或者 @Tool("description") 中的描述可以被用来辅助生成一个对 LLM 更友好的名称/标识，但通常方法名是基础)。
      - **工具描述:** @Tool 注解中的描述字符串 (例如，"Gets the current date and time in a specified format.")。
      - **参数描述:** 对于每个参数，使用 @P 注解提供的描述 (例如，"The format string for the date and time...") 以及参数的类型。
- **构建特殊的 Prompt 或 Function/Tool Calling 请求:**
    - **传统 ReAct 模式 (如果 LLM 不支持原生 Tool Calling):** LangChain4j 会将这些工具的名称、描述和参数信息，以特定的文本格式，**注入到发送给 LLM 的 System Prompt 或 User Prompt 的一部分**。它会告诉 LLM：“你有以下这些工具可用：[工具A描述], [工具B描述]... 如果你需要使用工具，请按如下格式输出：Action: [工具名] ActionInput: [输入]”。
    - **原生 Tool Calling / Function Calling 模式 (如 OpenAI, Gemini):** 如果底层的 ChatLanguageModel (比如 OpenAiChatModel) 支持，并且 LLM 本身也支持此功能，LangChain4j 会将工具的元数据（名称、描述、参数 schema）按照 LLM API 指定的格式（通常是 JSON Schema）封装，并与用户的提问一起发送给 LLM。

### LM 进行决策:

- LLM 接收到用户的提问，以及关于可用工具的信息。
- 基于它对用户意图的理解和它被训练来遵循指令的能力，LLM 会判断：
  - 这个问题是否可以通过直接回答来解决？
  - 还是说，我需要调用某个工具来获取额外的信息或执行某个操作才能回答这个问题？
- 如果 LLM 决定需要使用工具，它会根据工具的描述来选择最合适的工具，并尝试理解需要为这个工具提供什么参数。
- **例如:** 当用户问 "现在几点了？请用 yyyy-MM-dd HH:mm 格式告诉我。"
  - LLM 看到 DateTimeTool 中有一个 getCurrentDateTime 方法，其描述是 "Gets the current date and time in a specified format."，并且有一个参数 formatString。
  - LLM 就会判断这个工具非常适合回答这个问题。

### LM 输出工具调用指令:

- **传统 ReAct 模式:** LLM 会按照 Prompt 中约定的格式输出，例如：

  ```
  Thought: The user wants the current time in a specific format. I should use the getCurrentDateTime tool.
  Action: getCurrentDateTime
  ActionInput: {"formatString": "yyyy-MM-dd HH:mm"}
  ```

​	(注意：ActionInput 的具体格式也可能在 Prompt 中约定，JSON 是一种常见的选择)

- **原生 Tool Calling / Function Calling 模式:** LLM API 会返回一个特殊的响应，明确指示应该调用哪个函数以及参数是什么，例如 (OpenAI 风格)：

```
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "getCurrentDateTime",
              "arguments": "{\"formatString\":\"yyyy-MM-dd HH:mm\"}"
            }
          }
        ]
      }
    }
  ]
  // ...
}
```

### LangChain4j 解析指令并执行工具:

- LangChain4j (例如 AiServices 的内部逻辑或 Agent 执行器) 会接收 LLM 的这个输出。
- 它会解析出工具的名称 (例如 "getCurrentDateTime") 和参数 (例如 {"formatString":"yyyy-MM-dd HH:mm"}).
- 然后，它会在之前注册的工具列表 (即你传入的 this.dateTimeTool 实例) 中查找具有该名称的方法。
- 一旦找到，它会尝试将 LLM 提供的参数（通常是字符串形式的 JSON）反序列化或转换为 DateTimeTool 中 getCurrentDateTime 方法期望的参数类型 (这里是 String formatString)。
- 然后，LangChain4j **在你的 Java 应用中实际调用** dateTimeTool.getCurrentDateTime("yyyy-MM-dd HH:mm") 方法。

### 工具结果反馈给 LLM:
- dateTimeTool.getCurrentDateTime(...) 方法执行后返回一个结果 (例如 "Current date and time is: 2025-05-08 10:30 Thursday").
- LangChain4j 会将这个工具的执行结果再次发送给 LLM，通常会附带一些上下文，告诉 LLM：“这是你之前请求调用的 getCurrentDateTime 工具的输出结果：[结果字符串]”。

### LM 生成最终回复:

- LLM 接收到工具的输出结果后，现在它有了回答用户原始问题所需的信息。
- 它会基于用户的原始提问和工具的输出，生成最终的自然语言回复给用户。

## 总结

### LM 是决策者：
	 是 LLM 根据用户的输入和 LangChain4j 提供的工具描述，来决定是否使用工具、使用哪个工具以及如何使用。
### LngChain4j 是执行者和协调者：
- 它负责将工具信息有效地传递给 LLM。
- 它负责解析 LLM 返回的工具调用指令。
- 它负责在本地实际执行工具代码。
- 它负责将工具结果反馈给 LLM。
- 它负责驱动整个 Agent 的思考-行动循环。

LangChain4j **本身并不会“智能地”去匹配用户的自然语言和工具方法**，它依赖于 LLM 的自然语言理解和指令遵循能力来完成这个“匹配”和“决策”过程。LangChain4j 的作用是提供一个框架，使得这个 LLM 驱动的工具调用过程能够顺畅地进行。
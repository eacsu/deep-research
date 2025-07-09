好的，我将帮你查找如何使用最新版 label-studio-python-sdk 灵活地完成以下任务：创建工程、创建任务、导入标注、执行预测、导出特定工程的标注结果。会包含各步骤的代码示例与注意事项。
 稍等一下，我整理好后会尽快告诉你。

# Label Studio Python SDK 使用指南

以下示例基于最新（1.x）版 **label-studio-sdk**，介绍如何通过 Python SDK 执行项目创建、任务导入、预标注/预测、结果导出等操作。请先确保安装 SDK 并获取 Label Studio 的 **API Token** 和访问 **URL**。

## 步骤1：安装与初始化 SDK

使用 pip 安装 SDK：

```bash
pip install label-studio-sdk   # 或者使用 poetry 添加依赖:contentReference[oaicite:2]{index=2} 
```

在 Python 脚本中配置 Label Studio 服务地址和 API Token，并初始化客户端：

```python
LABEL_STUDIO_URL = 'http://localhost:8080'  # Label Studio 服务地址
API_KEY = 'your_api_key_here'               # 从 Label Studio UI 的 Access Tokens 页面获取:contentReference[oaicite:3]{index=3}

from label_studio_sdk.client import LabelStudio
ls = LabelStudio(base_url=LABEL_STUDIO_URL, api_key=API_KEY)  # 连接到 Label Studio:contentReference[oaicite:4]{index=4}
```

- `LABEL_STUDIO_URL`：Label Studio 的访问 URL（例如 `http://localhost:8080`）。
- `API_KEY`：用户的 API Token，可在 Label Studio Web 界面「账户设置」中生成。
- 使用 `LabelStudio(base_url, api_key)` 创建客户端对象 `ls`，用于后续调用 SDK 接口。

## 步骤2：创建标注项目

通过 `ls.projects.create()` 创建新项目，指定项目名称和标注配置。标注配置为 XML 格式的字符串，定义数据对象和标签控件。例如，创建一个简单的文本分类项目：

```python
from label_studio_sdk.label_interface import LabelInterface
from label_studio_sdk.label_interface.create import choices

# 定义标注界面：一个 Text 对象和一个 label 控件（可选值 Positive/Negative）
label_config = LabelInterface.create({
    'text': 'Text',
    'label': choices(['Positive', 'Negative'])
})

# 创建项目
project = ls.projects.create(
    title='Text Classification',  # 项目名称
    label_config=label_config     # 标注配置（XML 格式）:contentReference[oaicite:6]{index=6}:contentReference[oaicite:7]{index=7}
)
print(f'Created project ID: {project.id}')
```

- `title`：项目名称。
- `label_config`：标注界面配置，通常使用 `LabelInterface.create` 辅助生成 XML。如果已有自定义 XML，可直接传入字符串。

上述代码调用后会返回 `Project` 对象，其中包含项目的 `id`，后续可通过 `project.id` 引用该项目。

## 步骤3：创建标注任务（导入数据）

使用 SDK 导入待标注的数据任务。可以单个添加，也可以批量导入。示例如下：

- **单个任务**：调用 `ls.tasks.create()`。

```python
# 单个任务示例：数据格式根据 label_config 中定义的 object 名称
task = ls.tasks.create(
    project=project.id,
    data={'text': 'Hello world'}  # 假设标注界面定义了名为 'text' 的输入数据
)
print(f'Created task ID: {task["id"]}')
```

- **批量任务**：调用 `ls.projects.import_tasks()`。

```python
tasks = ls.projects.import_tasks(
    id=project.id,
    request=[
        {"text": "Hello world"},
        {"text": "Hello Label Studio"},
        {"text": "What a beautiful day"},
    ]
)
print(f'Imported task IDs: {tasks}')
```

其中：`project=project.id` 或 `id=project.id` 指定项目；`data` 或 `request` 中的键（如 `"text"`）需与项目的标注配置一致。执行上述代码后，任务将被创建并展示在 Label Studio 数据管理器中。

## 步骤4：导入标注结果（预标注/外部标注）

如果已有外部标注结果（或模型预标注），可以一并导入。方法一是通过 `import_tasks` 时指定预标注字段；方法二是后续添加预测。示例如下：

- **在导入任务时一并导入预测**：假设数据中包含预标注的标签字段（如 `"sentiment"`），可使用 `preannotated_from_fields` 参数。

```python
ls.projects.import_tasks(
    id=project.id,
    request=[
        {"text": "Hello world", "sentiment": "Positive"},
        {"text": "Goodbye Label Studio", "sentiment": "Negative"},
        {"text": "What a beautiful day", "sentiment": "Positive"},
    ],
    preannotated_from_fields=['sentiment']  # 指定字段作为预标注标签导入:contentReference[oaicite:12]{index=12}
)
```

上述代码会创建任务并自动将 `"sentiment"` 字段内容作为预测结果导入项目中。

- **使用 PredictionValue 类批量导入**：对于已有的任务，也可以用 `PredictionValue` 构造预测结果并调用 `ls.predictions.create()` 导入。例如：

```python
from label_studio_sdk.label_interface.objects import PredictionValue

# 获取项目的标注界面对象，用于生成符合格式的结果
li = ls.projects.get(id=project.id).get_label_interface()

# 遍历已有任务，创建并上传预测结果
for task in ls.tasks.list(project=project.id, include=["id"]):
    task_id = task.id
    prediction = PredictionValue(
        model_version='my_model_v1',  # 模型版本标记（可选）
        result=[ li.get_control('label').label(['Positive']) ]
    )
    ls.predictions.create(task=task_id, **prediction.model_dump())
```

上述代码为每个任务生成一个预测标签“Positive”，并通过 API 上传。`PredictionValue` 中的 `result` 应符合标注配置中的控件格式。

## 步骤5：执行预测（模型结果导入）

如果需要模拟通过 SDK 添加模型预测，可使用类似的方法。以下示例展示如何为现有任务添加预测结果：

```python
from label_studio_sdk.label_interface.objects import PredictionValue

# 获取项目对象及其标注界面
project = ls.projects.get(id=project.id)
li = project.get_label_interface()

# 遍历任务并添加预测结果
tasks = ls.tasks.list(project=project.id, include='id')
for task in tasks:
    predicted_label = li.get_control('label').label(choices=['Positive'])
    prediction = PredictionValue(
        model_version='v1', 
        score=0.99, 
        result=[predicted_label]
    )
    ls.predictions.create(task=task.id, **prediction.model_dump())
```

该代码中，我们假设预测模型为每个任务都返回“Positive”标签，并使用 `ls.predictions.create()` 上传（`model_version` 可自定义）。在 Label Studio UI 中，这些预测将显示为任务的预标注结果。

## 步骤6：导出标注结果

完成标注后，可使用 SDK 导出结果。例如将项目中所有已标注任务导出为 JSON：

```python
data = ls.projects.exports.as_json(project.id)
```

该接口返回包含所有任务及其标注结果（annotations）的列表，格式为 Label Studio 的 JSON 结构。你也可以使用高级导出方法（如 `project.export()`）指定过滤条件或格式等，详见 SDK 文档。

上述步骤提供了通过最新 SDK 完成项目管理、任务导入、预标注（模拟标注）导入、预测结果添加以及标注结果导出的基本示例。更多详细用法可参考官方文档和 API 参考。
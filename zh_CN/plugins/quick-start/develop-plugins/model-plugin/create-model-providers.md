# 创建模型供应商

创建 Model 类型插件的第一步是初始化插件项目并创建模型供应商文件，随后接入具体的预定义 / 自定义模型。

### 前置准备 <a href="#qian-zhi-zhun-bei" id="qian-zhi-zhun-bei"></a>

* Dify 插件脚手架工具
* Python 环境，版本号 ≥ 3.12

关于如何准备插件开发的脚手架工具，详细说明请参考[初始化开发工具](../initialize-development-tools.md)。

### 创建新项目 <a href="#chuang-jian-xin-xiang-mu" id="chuang-jian-xin-xiang-mu"></a>

在脚手架命令行工具的路径下，创建一个新的 dify 插件项目。

```
./dify-plugin-darwin-arm64 plugin init
```

如果你已将该二进制文件重命名为了 `dify` 并拷贝到了 `/usr/local/bin` 路径下，可以运行以下命令创建新的插件项目：

```bash
dify plugin init
```

### 选择模型插件模板 <a href="#xuan-ze-cha-jian-lei-xing-he-mu-ban" id="xuan-ze-cha-jian-lei-xing-he-mu-ban"></a>

脚手架工具内的所有模板均已提供完整的代码项目，选择 `LLM` 类型插件模板。

![Plugin type: llm](https://assets-docs.dify.ai/2024/12/8efe646e9174164b9edbf658b5934b86.png)

#### 配置插件权限

为该 LLM 插件配置以下权限：

* Models
* LLM
* Storage

![模型插件权限](https://assets-docs.dify.ai/2024/12/10f3b3ee6c03a1215309f13d712455d4.png)

#### 模型类型配置说明

模型供应商支持以下两种模型的配置方式：

*   `predefined-model` **预定义模型**

    常见的大模型类型，只需要配置统一的供应商凭据即可使用模型供应商下的预定义模型。例如，`OpenAI` 模型供应商下提供 `gpt-3.5-turbo-0125` 和 `gpt-4o-2024-05-13` 等一系列预定义模型。详细开发说明请参考接入预定义模型。
*   `customizable-model` **自定义模型**

    需要手动新增每个模型的凭据配置，例如 `Xinference`，它同时支持 LLM 和 Text Embedding，但是每个模型都有唯一的 **model\_uid**，如果想要将两者同时接入，需要为每个模型配置一个 **model\_uid**。详细开发说明请参考接入自定义模型。

两种配置方式**支持共存**，即存在供应商支持 `predefined-model` + `customizable-model` 或 `predefined-model`  等，即配置了供应商统一凭据可以使用预定义模型和从远程获取的模型，若新增了模型，则可以在此基础上额外使用自定义的模型。

### 新增模型供应商

新增一个模型供应商主要包含以下几个步骤：

1.  **创建模型供应商配置 YAML** **文件**

    在供应商目录下新增一个 YAML 文件，用于描述供应商的基本信息和参数配置。按照 ProviderSchema 的要求编写内容，确保与系统的规范保持一致。
2.  **编写模型供应商代码**

    创建供应商 class 代码，实现一个符合系统接口要求的 Python class 用于对接供应商的 API，完成核心功能实现。

***

以下是每个步骤的完整操作详情。

#### 1. **创建模型供应商配置文件**

Manifest 是 YAML 格式文件，声明了模型供应商基础信息、所支持的模型类型、配置方式、凭据规则。插件项目模板将在 `/providers` 路径下自动生成配置文件。

以下是 `Anthropic` 模型配置文件 `anthropic.yaml` 的示例代码：

```yaml
provider: anthropic
label:
  en_US: Anthropic
description:
  en_US: Anthropic's powerful models, such as Claude 3.
  zh_Hans: Anthropic 的强大模型，例如 Claude 3。
icon_small:
  en_US: icon_s_en.svg
icon_large:
  en_US: icon_l_en.svg
background: "#F0F0EB"
help:
  title:
    en_US: Get your API Key from Anthropic
    zh_Hans: 从 Anthropic 获取 API Key
  url:
    en_US: https://console.anthropic.com/account/keys
supported_model_types:
  - llm
configurate_methods:
  - predefined-model
provider_credential_schema:
  credential_form_schemas:
    - variable: anthropic_api_key
      label:
        en_US: API Key
      type: secret-input
      required: true
      placeholder:
        zh_Hans: 在此输入您的 API Key
        en_US: Enter your API Key
    - variable: anthropic_api_url
      label:
        en_US: API URL
      type: text-input
      required: false
      placeholder:
        zh_Hans: 在此输入您的 API URL
        en_US: Enter your API URL
models:
  llm:
    predefined:
      - "models/llm/*.yaml"
    position: "models/llm/_position.yaml"
extra:
  python:
    provider_source: provider/anthropic.py
    model_sources:
      - "models/llm/llm.py"
```

如果接入的供应商提供自定义模型，比如`OpenAI`提供微调模型，需要添加`model_credential_schema` 字段。

以下是 `OpenAI` 家族模型的示例代码：

```yaml
model_credential_schema:
  model: # 微调模型名称
    label:
      en_US: Model Name
      zh_Hans: 模型名称
    placeholder:
      en_US: Enter your model name
      zh_Hans: 输入模型名称
  credential_form_schemas:
  - variable: openai_api_key
    label:
      en_US: API Key
    type: secret-input
    required: true
    placeholder:
      zh_Hans: 在此输入您的 API Key
      en_US: Enter your API Key
  - variable: openai_organization
    label:
        zh_Hans: 组织 ID
        en_US: Organization
    type: text-input
    required: false
    placeholder:
      zh_Hans: 在此输入您的组织 ID
      en_US: Enter your Organization ID
  - variable: openai_api_base
    label:
      zh_Hans: API Base
      en_US: API Base
    type: text-input
    required: false
    placeholder:
      zh_Hans: 在此输入您的 API Base
      en_US: Enter your API Base
```

如需查看更多完整的模型供应商 YAML 规范，详情请参考[模型接口文档](../../../schema-definition/model/model-schema.md)。

#### 2. **编写模型供应商代码**

在 `/providers` 文件夹下创建一个同名的 python 文件，例如 `anthropic.py` 并实现一个 `class` ，继承 `__base.provider.Provider` 基类，例如 `AnthropicProvider`。

以下是 `Anthropic` 示例代码：

```python
import logging
from dify_plugin.entities.model import ModelType
from dify_plugin.errors.model import CredentialsValidateFailedError
from dify_plugin import ModelProvider

logger = logging.getLogger(__name__)


class AnthropicProvider(ModelProvider):
    def validate_provider_credentials(self, credentials: dict) -> None:
        """
        Validate provider credentials

        if validate failed, raise exception

        :param credentials: provider credentials, credentials form defined in `provider_credential_schema`.
        """
        try:
            model_instance = self.get_model_instance(ModelType.LLM)
            model_instance.validate_credentials(model="claude-3-opus-20240229", credentials=credentials)
        except CredentialsValidateFailedError as ex:
            raise ex
        except Exception as ex:
            logger.exception(f"{self.get_provider_schema().provider} credentials validate failed")
            raise ex
```

供应商需要继承 `__base.model_provider.ModelProvider` 基类，实现 `validate_provider_credentials` 供应商统一凭据校验方法即可。

```python
def validate_provider_credentials(self, credentials: dict) -> None:
    """
    Validate provider credentials
    You can choose any validate_credentials method of model type or implement validate method by yourself,
    such as: get model list api

    if validate failed, raise exception

    :param credentials: provider credentials, credentials form defined in `provider_credential_schema`.
    """
```

当然也可以先预留 `validate_provider_credentials` 实现，在模型凭据校验方法实现后直接复用。

#### **自定义模型供应商**

对于其它类型模型供应商而言，请参考以下配置方法。


对于像 `Xinference` 这样的自定义模型供应商，可以跳过完整实现的步骤。只需创建一个名为 `XinferenceProvider` 的空类，并在其中实现一个空的 `validate_provider_credentials` 方法。

**具体说明：**

• `XinferenceProvider` 是一个占位类，用于标识自定义模型供应商。

• `validate_provider_credentials` 方法虽然不会被实际调用，但必须存在，这是因为其父类是抽象类，要求所有子类都实现这个方法。通过提供一个空实现，可以避免因未实现抽象方法而导致的实例化错误。

```python
class XinferenceProvider(Provider):
    def validate_provider_credentials(self, credentials: dict) -> None:
        pass
```

初始化模型供应商后，接下来需要接入供应商所提供的具体 llm 模型。详细说明请参考以下内容：

* [接入预定义模型](../../../../guides/model-configuration/predefined-model.md)
* [接入自定义模型](../../../../guides/model-configuration/customizable-model.md)

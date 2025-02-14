# 接入自定义模型

自定义模型指的是需要自行部署或配置的 LLM。本文将以 [Xinference 模型](https://inference.readthedocs.io/en/latest/)为例，演示如何在模型插件内接入自定义模型。

自定义模型默认包含模型类型和模型名称两个参数，无需在供应商 yaml 文件定义。

供应商配置文件无需实现 `validate_provider_credential`。Runtime 会根据用户选择的模型类型或模型名称，自动调用对应模型层的 `validate_credentials` 方法进行验证。

### 接入自定义模型插件

接入自定义模型分为以下步骤：

1.  **创建模型供应商文件**

    明确自定义模型中所包含的模型类型。
2.  **根据模型类型创建代码文件**

    根据模型的类型（如 `llm` 或 `text_embedding`）创建代码文件。确保每种模型类型有独立的逻辑分层，便于维护和扩展。
3.  **根据不同的模型模块，编写模型调用代码**

    在对应的模型类型模块下，创建一个与模型类型同名的 Python 文件（例如 llm.py）。在文件中定义实现具体模型逻辑的类，该类应符合系统的模型接口规范。
4.  **调试插件**

    为新增的供应商功能编写单元测试和集成测试，确保所有功能模块符合预期，并能够正常运行。

***

### 1. **创建模型供应商文件**

在插件项目的 `/provider` 路径下，新建 `xinference.yaml` 文件。

`Xinference` 家族模型支持 `LLM``，Text Embedding` 和 `Rerank` 模型类型，因此需要在 `xinference.yaml` 文件中包含上述模型类型。

示例代码：

<pre class="language-yaml"><code class="lang-yaml">provider: xinference # 确定供应商标识
label: # 供应商展示名称，可设置 en_US 英文、zh_Hans 中文两种语言，zh_Hans 不设置将默认使用 en_US。
  en_US: Xorbits Inference
icon_small: # 小图标，可以参考其他供应商的图标，存储在对应供应商实现目录下的 _assets 目录，中英文策略同 label
  en_US: icon_s_en.svg
icon_large: # 大图标
  en_US: icon_l_en.svg
help: # 帮助
  title:
    en_US: How to deploy Xinference
    zh_Hans: 如何部署 Xinference
  url:
    en_US: https://github.com/xorbitsai/inference
<strong>supported_model_types: # 支持的模型类型，Xinference 同时支持 LLM/Text Embedding/Rerank
</strong><strong>- llm
</strong><strong>- text-embedding
</strong><strong>- rerank
</strong>configurate_methods: # Xinference 为本地部署的供应商，并且没有预定义模型，需要用什么模型需要根据 Xinference 的文档进行部署，因此此处的方法为自定义模型。
- customizable-model
provider_credential_schema:
  credential_form_schemas:
</code></pre>

接着需要定义 `provider_credential_schema` 字段。`Xinference 支持` text-generation，embeddings 和 reranking 模型，示例代码如下：

<pre class="language-yaml"><code class="lang-yaml">provider_credential_schema:
  credential_form_schemas:
  - variable: model_type
    type: select
    label:
      en_US: Model type
      zh_Hans: 模型类型
    required: true
    options:
<strong>    - value: text-generation
</strong>      label:
        en_US: Language Model
        zh_Hans: 语言模型
<strong>    - value: embeddings
</strong>      label:
        en_US: Text Embedding
<strong>    - value: reranking
</strong>      label:
        en_US: Rerank
</code></pre>

Xinference 中的每个模型都需要定义名称 `model_name`。

```yaml
  - variable: model_name
    type: text-input
    label:
      en_US: Model name
      zh_Hans: 模型名称
    required: true
    placeholder:
      zh_Hans: 填写模型名称
      en_US: Input model name
```

Xinference 模型需要使用者输入模型的本地部署地址，插件内需要提供允许填写 Xinference 模型的本地部署地址（server\_url）和模型 UID 的位置，示例代码如下：

<pre class="language-yaml"><code class="lang-yaml"><strong>  - variable: server_url
</strong>    label:
      zh_Hans: 服务器 URL
      en_US: Server url
    type: text-input
    required: true
    placeholder:
      zh_Hans: 在此输入 Xinference 的服务器地址，如 https://example.com/xxx
      en_US: Enter the url of your Xinference, for example https://example.com/xxx
<strong>  - variable: model_uid
</strong>    label:
      zh_Hans: 模型 UID
      en_US: Model uid
    type: text-input
    required: true
    placeholder:
      zh_Hans: 在此输入您的 Model UID
      en_US: Enter the model uid
</code></pre>

填写所有参数后即可完成自定义模型供应商 yaml 配置文件的创建。接下来需为配置文件内定义的模型添加具体的功能代码文件。

### 2. 编写模型代码

Xinference 模型供应商的模型类型包含 llm、rerank、speech2text、tts 类型，因此需要在 /models 路径下为每个模型类型创建独立的分组，并创建对应的功能代码文件。

下文将以 llm 类型为例，说明如何创建 `llm.py` 代码文件。创建代码时需创建一个 Xinference LLM 类，可以取名为 `XinferenceAILargeLanguageModel`，继承 `__base.large_language_model.LargeLanguageModel` 基类，实现以下几个方法：

* **LLM 调用**

LLM 调用的核心方法，同时支持流式和同步返回。

```python
def _invoke(self, model: str, credentials: dict,
            prompt_messages: list[PromptMessage], model_parameters: dict,
            tools: Optional[list[PromptMessageTool]] = None, stop: Optional[list[str]] = None,
            stream: bool = True, user: Optional[str] = None) \
        -> Union[LLMResult, Generator]:
    """
    Invoke large language model

    :param model: model name
    :param credentials: model credentials
    :param prompt_messages: prompt messages
    :param model_parameters: model parameters
    :param tools: tools for tool calling
    :param stop: stop words
    :param stream: is stream response
    :param user: unique user id
    :return: full response or stream response chunk generator result
    """
```

实现代码时，需要注意使用两个函数来返回数据，分别用于处理同步返回和流式返回。

Python 会将函数中包含 `yield` 的关键字函数识别为生成器函数，返回的数据类型固定为 `Generator`，因此需要分别实现同步和流式返回，例如以下示例代码：

> 该示例使用了简化参数，实际编写代码时需参考上文中的参数列表。

```python
def _invoke(self, stream: bool, **kwargs) \
        -> Union[LLMResult, Generator]:
    if stream:
          return self._handle_stream_response(**kwargs)
    return self._handle_sync_response(**kwargs)

def _handle_stream_response(self, **kwargs) -> Generator:
    for chunk in response:
          yield chunk
def _handle_sync_response(self, **kwargs) -> LLMResult:
    return LLMResult(**response)
```

* **预计算输入 Tokens**

如果模型未提供预计算 tokens 的接口，可以直接返回 0。

```python
def get_num_tokens(self, model: str, credentials: dict, prompt_messages: list[PromptMessage],
                 tools: Optional[list[PromptMessageTool]] = None) -> int:
  """
  Get number of tokens for given prompt messages

  :param model: model name
  :param credentials: model credentials
  :param prompt_messages: prompt messages
  :param tools: tools for tool calling
  :return:
  """
```

在某些情况下，如果不想直接返回 0，可以使用 `self._get_num_tokens_by_gpt2(text: str)` 方法计算 tokens。该方法位于 `AIModel` 基类中，使用 GPT-2 的 Tokenizer 进行计算。但请注意，这是一个替代方案，计算结果可能存在一定误差。

* **模型凭据校验**

与供应商凭据校验类似，这里针对单个模型进行校验。

```python
def validate_credentials(self, model: str, credentials: dict) -> None:
    """
    Validate model credentials

    :param model: model name
    :param credentials: model credentials
    :return:
    """
```

*   **模型参数 Schema**

与[预定义模型类型](integrate-the-predefined-model.md)不同，由于未在 YAML 文件中预设模型所支持的参数，因此需要动态生成模型参数的 Schema。

例如，Xinference 支持 `max_tokens`、`temperature` 和 `top_p` 三种模型参数。然而一些供应商（例如 OpenLLM）会根据具体模型支持不同的参数。

举例来说，供应商 `OpenLLM` 的 A 模型支持 `top_k` 参数，而 B 模型则不支持 `top_k`。在该情况下，需要动态生成每个模型对应的参数 Schema，示例代码如下：

```python
  def get_customizable_model_schema(self, model: str, credentials: dict) -> AIModelEntity | None:
      """
          used to define customizable model schema
      """
      rules = [
          ParameterRule(
              name='temperature', type=ParameterType.FLOAT,
              use_template='temperature',
              label=I18nObject(
                  zh_Hans='温度', en_US='Temperature'
              )
          ),
          ParameterRule(
              name='top_p', type=ParameterType.FLOAT,
              use_template='top_p',
              label=I18nObject(
                  zh_Hans='Top P', en_US='Top P'
              )
          ),
          ParameterRule(
              name='max_tokens', type=ParameterType.INT,
              use_template='max_tokens',
              min=1,
              default=512,
              label=I18nObject(
                  zh_Hans='最大生成长度', en_US='Max Tokens'
              )
          )
      ]

      # if model is A, add top_k to rules
      if model == 'A':
          rules.append(
              ParameterRule(
                  name='top_k', type=ParameterType.INT,
                  use_template='top_k',
                  min=1,
                  default=50,
                  label=I18nObject(
                      zh_Hans='Top K', en_US='Top K'
                  )
              )
          )

      """
          some NOT IMPORTANT code here
      """

      entity = AIModelEntity(
          model=model,
          label=I18nObject(
              en_US=model
          ),
          fetch_from=FetchFrom.CUSTOMIZABLE_MODEL,
          model_type=model_type,
          model_properties={
              ModelPropertyKey.MODE:  ModelType.LLM,
          },
          parameter_rules=rules
      )

      return entity
  ```
* **调用异常错误映射表**

当模型调用异常时需要映射到 Runtime 指定的 `InvokeError` 类型，方便 Dify 针对不同错误做不同后续处理。

Runtime Errors:

* `InvokeConnectionError` 调用连接错误
* `InvokeServerUnavailableError` 调用服务方不可用
* `InvokeRateLimitError` 调用达到限额
* `InvokeAuthorizationError` 调用鉴权失败
* `InvokeBadRequestError` 调用传参有误

```python
@property
def _invoke_error_mapping(self) -> dict[type[InvokeError], list[type[Exception]]]:
    """
    Map model invoke error to unified error
    The key is the error type thrown to the caller
    The value is the error type thrown by the model,
    which needs to be converted into a unified error type for the caller.

    :return: Invoke error mapping
    """
```

如需了解更多接口方法，请参考[接口文档：Model](../../../schema-definition/model/)。

如需获取本文所涉及的完整代码文件，请访问 [GitHub 代码仓库](https://github.com/langgenius/dify-official-plugins/tree/main/models/xinference)。

### 3. 调试插件

插件开发完成后，接下来需测试插件是否可以正常运行。详细说明请参考：

{% content-ref url="debug-plugin.md" %}
[debug-plugin.md](debug-plugin.md)
{% endcontent-ref %}

### 4. 发布插件

如果想要将插件发布至 Dify Marketplace，请参考以下内容：

{% content-ref url="../../../publish-plugins/publish-to-dify-marketplace/" %}
[publish-to-dify-marketplace](../../../publish-plugins/publish-to-dify-marketplace/)
{% endcontent-ref %}

### **探索更多**

**快速开始：**

* [开发 Extension 插件](../extension-plugin.md)
* [开发 Tool 插件](../tool-type-plugin.md)
* [Bundle 插件：将多个插件打包](../bundle.md)

**插件接口文档：**

* [Manifest](../../../schema-definition/manifest.md) 结构
* [Endpoint](../../../schema-definition/endpoint.md) 详细定义
* [反向调用 Dify 能力](../../../schema-definition/reverse-invocation-of-the-dify-service/)
* [工具](../../../schema-definition/tool.md)
* [模型](../../../schema-definition/model/)

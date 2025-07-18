# Parameters

Run `evalscope eval --help` to get a complete list of parameter descriptions.

## Model Parameters
- `--model`: The name of the model being evaluated.
  - Specify the model's `id` in [ModelScope](https://modelscope.cn/), and it will automatically download the model, for example, [Qwen/Qwen2.5-0.5B-Instruct](https://modelscope.cn/models/Qwen/Qwen2.5-0.5B-Instruct/summary);
  - Specify the local path to the model, for example, `/path/to/model`, to load the model from the local environment;
  - When the evaluation target is the model API endpoint, it needs to be specified as `model_id`, for example, `Qwen2.5-0.5B-Instruct`.
- `--model-id`: An alias for the model being evaluated. Defaults to the last part of `model`, for example, the `model-id` for `Qwen/Qwen2.5-0.5B-Instruct` is `Qwen2.5-0.5B-Instruct`.
- `--model-task`: The task type of the model, defaults to `text_generation`, options are `text_generation`, `image_generation`.
- `--model-args`: Model loading parameters, separated by commas in `key=value` format, with default parameters:
  - `revision`: Model version, defaults to `master`
  - `precision`: Model precision, defaults to `torch.float16`
  - `device_map`: Device allocation for the model, defaults to `auto`
- `--generation-config`: Generation parameters, separated by commas, in the form of `key=value` or passed in as a JSON string, which will be parsed into a dictionary:
  - If using local model inference (based on Transformers), the following parameters are included ([Full parameter guide](https://huggingface.co/docs/transformers/main_classes/text_generation#transformers.GenerationConfig)):
    - `do_sample`: Whether to use sampling, default is `false`
    - `max_length`: Maximum length, default is 2048
    - `max_new_tokens`: Maximum length of generated text, default is 512
    - `num_return_sequences`: Number of sequences to generate, default is 1; when set greater than 1, multiple sequences will be generated, requires setting `do_sample=True`
    - `temperature`: Generation temperature
    - `top_k`: Top-k for generation
    - `top_p`: Top-p for generation
  - If using model API service for inference (`eval-type` set to `service`), the following parameters are included (please refer to the deployed model service for specifics):
    - `max_tokens`: Maximum length of generated text, default is 2048
    - `temperature`: Generation temperature, default is 0.0
    - `n`: number of generated sequences, default is 1 (Note: currently, lmdeploy only supports n=1)
  ```bash
  # For example, pass arguments in the form of key=value
  --model-args revision=master,precision=torch.float16,device_map=auto
  --generation-config do_sample=true,temperature=0.5

  # Or pass more complex parameters using a JSON string
  --model-args '{"revision": "master", "precision": "torch.float16", "device_map": "auto"}'
  --generation-config '{"do_sample":true,"temperature":0.5,"chat_template_kwargs":{"enable_thinking": false}}'
  ```
- `--chat-template`: Model inference template, defaults to `None`, indicating the use of transformers' `apply_chat_template`; supports passing in a jinja template string to customize the inference template.
- `--template-type`: Model inference template, deprecated, refer to `--chat-template`.

**The following parameters are only valid when `eval-type=service`:**
- `--api-url`: Model API endpoint, default is `None`; supports local or remote OpenAI API format endpoints, for example `http://127.0.0.1:8000/v1`.
- `--api-key`: Model API endpoint key, default is `EMPTY`
- `--timeout`: Model API request timeout, default is `None`
- `--stream`: Whether to use streaming transmission, default is `False`

## Dataset Parameters
- `--datasets`: Dataset name, supports inputting multiple datasets separated by spaces, datasets will automatically be downloaded from ModelScope, supported datasets refer to [Dataset List](./supported_dataset/index.md).
- `--dataset-args`: Configuration parameters for the evaluation dataset, passed in `json` format, where the key is the dataset name and the value is the parameter, note that it needs to correspond one-to-one with the values in the `--datasets` parameter:
  - `dataset_id` (or `local_path`): Local path for the dataset, once specified, it will attempt to load local data.
  - `prompt_template`: The prompt template for the evaluation dataset. When specified, it will be used to generate prompts. For example, the template for the `gsm8k` dataset is `Question: {query}\nLet's think step by step\nAnswer:`. The question from the dataset will be filled into the `query` field of the template.
  - `query_template`: The query template for the evaluation dataset. When specified, it will be used to generate queries. For example, the template for `general_mcq` is `Question: {question}\n{choices}\nAnswer: {answer}\n\n`. The questions from the dataset will be inserted into the `question` field of the template, options will be inserted into the `choices` field, and answers will be inserted into the `answer` field (answer insertion is only effective for few-shot scenarios).
  - `system_prompt`: System prompt for the evaluation dataset.
  - `model_adapter`: The model adapter for the evaluation dataset. Once specified, the given model adapter will be used for evaluation. Currently, it supports `generation`, `multiple_choice_logits`, and `continuous_logits`. For service evaluation, only `generation` is supported at the moment. Some multiple-choice datasets support `logits` output.
  - `subset_list`: List of subsets for the evaluation dataset, once specified, only subset data will be used.
  - `few_shot_num`: Number of few-shots.
  - `few_shot_random`: Whether to randomly sample few-shot data, defaults to `False`.
  - `metric_list`: A list of metrics for evaluating the dataset. When specified, the evaluation will use the given metrics. Currently supported metrics include `AverageAccuracy`, `AveragePass@1`, and `Pass@[1-16]`. For example, for the `humaneval` dataset, you can specify `["Pass@1", "Pass@5"]`. Note that in this case, you need to set `n=5` to make the model return 5 results.
  - `filters`: Filters for the evaluation dataset. When specified, these filters will be used to process the evaluation results. They can be used to handle the output of inference models. Currently supported filters are:
    - `remove_until {string}`: Removes the part of the model's output before the specified string.
    - `extract {regex}`: Extracts the part of the model's output that matches the specified regular expression.
    For example, the `ifeval` dataset can specify `{"remove_until": "</think>"}`, which will filter out the part of the model's output before `</think>`, avoiding interference with scoring.
  ```bash
  # For example
  --datasets gsm8k arc
  --dataset-args '{"gsm8k": {"few_shot_num": 4, "few_shot_random": false}, "arc": {"dataset_id": "/path/to/arc"}}, "ifeval": {"filters": {"remove_until": "</think>"}}'
  ```
- `--dataset-dir`: Dataset download path, defaults to `~/.cache/modelscope/datasets`.
- `--dataset-hub`: Dataset download source, defaults to `modelscope`, alternative is `huggingface`.
- `--limit`: The maximum amount of evaluation data for each dataset. If not specified, the default is to evaluate the entire dataset, which can be useful for quick validation. It supports both `int` and `float` types. An `int` value indicates the first `N` entries of the dataset to be evaluated, while a `float` value represents the first `N%` of the dataset. For example, `0.1` means evaluating the first 10% of the dataset, and `100` means evaluating the first 100 entries.

## Evaluation Parameters

- `--eval-batch-size`: Evaluation batch size, default is `1`; when `eval-type=service`, it indicates the number of concurrent evaluation requests, default is `8`.
- `--eval-stage`: (Deprecated, refer to `--use-cache`) Evaluation stage, options are `all`, `infer`, `review`, default is `all`.
- `--eval-type`: Evaluation type, options are `checkpoint`, `custom`, `service`; defaults to `checkpoint`.
- `--eval-backend`: Evaluation backend, options are `Native`, `OpenCompass`, `VLMEvalKit`, `RAGEval`, `ThirdParty`, defaults to `Native`.
  - `OpenCompass` is used for evaluating large language models.
  - `VLMEvalKit` is used for evaluating multimodal models.
  - `RAGEval` is used for evaluating RAG processes, embedding models, re-ranking models, CLIP models.
    ```{seealso}
    Other evaluation backends [User Guide](../user_guides/backend/index.md)
    ```
  - `ThirdParty` is used for other special task evaluations, such as [ToolBench](../third_party/toolbench.md), [LongBench](../third_party/longwriter.md).
- `--eval-config`: This parameter needs to be passed when using a non-`Native` evaluation backend.

## Judge Parameters

The LLM-as-a-Judge evaluation parameters use a judge model to determine correctness, including the following parameters:

- `--judge-strategy`: The strategy for using the judge model, options include:
  - `auto`: The default strategy, which decides whether to use the judge model based on the dataset requirements
  - `llm`: Always use the judge model
  - `rule`: Do not use the judge model, use rule-based judgment instead
  - `llm_recall`: First use rule-based judgment, and if it fails, then use the judge model
- `--judge-worker-num`: The concurrency number for the judge model, default is `1`
- `--judge-model-args`: Sets the parameters for the judge model, passed in as a `json` string and parsed as a dictionary, supporting the following fields:
  - `api_key`: The API endpoint key for the model. If not set, it will be retrieved from the environment variable `MODELSCOPE_SDK_TOKEN`, with a default value of `EMPTY`.
  - `api_url`: The API endpoint for the model. If not set, it will be retrieved from the environment variable `MODELSCOPE_API_BASE`, with a default value of `https://api-inference.modelscope.cn/v1/`.
  - `model_id`: The model ID. If not set, it will be retrieved from the environment variable `MODELSCOPE_JUDGE_LLM`, with a default value of `Qwen/Qwen3-235B-A22B`.
    ```{seealso}
    For more information on ModelScope's model inference services, please refer to [ModelScope API Inference Services](https://modelscope.cn/docs/model-service/API-Inference/intro).
    ```
  - `system_prompt`: System prompt for evaluating the dataset
  - `prompt_template`: Prompt template for evaluating the dataset
  - `generation_config`: Model generation parameters, same as the `--generation-config` parameter.
  - `score_type`: Preset model scoring method, options include:
    - `pattern`: (Default option) Directly judge whether the model output matches the reference answer, suitable for evaluations with reference answers.
      <details><summary>Default prompt_template</summary>

      ```text
      Your job is to look at a question, a gold target, and a predicted answer, and return a letter "A" or "B" to indicate whether the predicted answer is correct or incorrect.

      [Question]
      {question}

      [Reference Answer]
      {gold}

      [Predicted Answer]
      {pred}

      Evaluate the model's answer based on correctness compared to the reference answer.
      Grade the predicted answer of this new question as one of:
      A: CORRECT
      B: INCORRECT

      Just return the letters "A" or "B", with no text around it.
      ```
      </details>
    - `numeric`: Judge the model output score under prompt conditions, suitable for evaluations without reference answers.
      <details><summary>Default prompt_template</summary>

      ```text
      Please act as an impartial judge and evaluate the quality of the response provided by an AI assistant to the user question displayed below. Your evaluation should consider factors such as the helpfulness, relevance, accuracy, depth, creativity, and level of detail of the response.

      Begin your evaluation by providing a short explanation. Be as objective as possible.

      After providing your explanation, you must rate the response on a scale of 0 (worst) to 1 (best) by strictly following this format: \"[[rating]]\", for example: \"Rating: [[0.5]]\"

      [Question]
      {question}

      [Response]
      {pred}
      ```
      </details>
  - `score_pattern`: Regular expression for parsing model output, default for `pattern` mode is `(A|B)`; default for `numeric` mode is `\[\[(\d+(?:\.\d+)?)\]\]`, used to extract model scoring results.
  - `score_mapping`: Score mapping dictionary for `pattern` mode, default is `{'A': 1.0, 'B': 0.0}`
- `--analysis-report`: Whether to generate an analysis report, default is `false`; if this parameter is set, an analysis report will be generated using the judge model, including analysis interpretation and suggestions for the model evaluation results. The report output language will be automatically determined based on `locale.getlocale()`.

## Other Parameters

- `--work-dir`: Model evaluation output path, default is `./outputs/{timestamp}`, folder structure example is as follows:
  ```text
  .
  ├── configs
  │   └── task_config_b6f42c.yaml
  ├── logs
  │   └── eval_log.log
  ├── predictions
  │   └── Qwen2.5-0.5B-Instruct
  │       └── general_qa_example.jsonl
  ├── reports
  │   └── Qwen2.5-0.5B-Instruct
  │       └── general_qa.json
  └── reviews
      └── Qwen2.5-0.5B-Instruct
          └── general_qa_example.jsonl
  ```
- `--use-cache`: Use local cache path, default is `None`; if a path is specified, such as `outputs/20241210_194434`, it will reuse the model inference results from that path. If inference is not completed, it will continue inference and then proceed to evaluation.
- `--seed`: Random seed, default is `42`.
- `--debug`: Whether to enable debug mode, default is `false`.
- `--ignore-errors`: Whether to ignore errors during model generation, default is `false`.
- `--dry-run`: Pre-check parameters without performing inference, only prints parameters, default is `false`.

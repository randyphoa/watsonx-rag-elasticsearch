# watsonx-rag-elasticsearch

This implementation of watsonx Assistant with watsonx Discovery (Elasticsearch) consists of the following features,

- Conversation history
- Query rewrite
- Semantic search
- Answer generation

# 1. Index documents

Edit the notebook `watsonx-rag-elasticsearch.ipynb`, and update the required variables.

This notebook demonstrates,

- how to have fine grain control over the ingesting process such as chunking
- using pipelines to offload embedding text to Elasticsearch
- create an Elasticsearch index
- search for query terms using pipelines
- deploy and test prompts on Watson Machine Learning Deployment spaces

# 2. Setup watsonx Assistant

## 2.1 Create 3 variables

![wxa-variables](assets/images/wxa-variables.png)

1. `query` of text type
2. `context` of any type
3. `history` of any type

## 2.2 Create 2 custom extensions

1. Elasticsearch 
    - Upload `elasticsearch-openapi.json` for the Elasticsearch custom extension
    ![custom-extension-elasticsearch](assets/images/custom-extension-elasticsearch.png)

    - Configure credentails
    ![elasticsearch-auth](assets/images/elasticsearch-auth.png)

2. watsonx.ai
    - Upload `watsonx-deployment-history.json` for the watsonx.ai custom extension
    ![custom-extension-watsonxai](assets/images/custom-extension-watsonxai.png)

    - Configure credentials
     ![watsonxai-auth](assets/images/watsonxai-auth.png)

## 2.3 Create a new Action (select Start from scratch option)

This action consists of 3 steps for the RAG process,

1. Rewrite query based on conversational history
2. Retreive information using semantic search
3. Generate answer based on retrieved information

### 2.3.1 Rewrite Query

Set `history` variable to the following expression,

```
${history} = ${system_session_history}.transform("role", "<|start_header_id|>user<|end_header_id|>\n\n", "<|start_header_id|>assistant<|end_header_id|>\n\n").joinToArray("%e.role%%e.content%").join("<|eot_id|>") + "<|eot_id|>"
```

![set-history-variable](assets/images/set-history-variable.png)

In the `And then section`, select use an extension.

- Select the watsonx.ai extension that was created in Step 2.2.
- Select the `Text Generation` operation.
- Under parameters, 
    - set the `deployment_id` for `rewrite-prompt` that was create in Step 1
    - `version` to `2023-05-29`
- Under optional parameters, set `parameters.prompt_variables.history` to the `history` variable.
- Under stream response, set `text` to `results[0].generated_text`

![custom-extension-rewrite-query](assets/images/custom-extension-rewrite-query.png)

### 2.3.2 Retrieve Information

Set `query` variable to the following expression,

${query} = ${step_950_result_2.body.results}[0]["generated_text"]

![set-query-variable](assets/images/set-query-variable.png)

Edit expression using the editor to select the previous step and select `body.results`.

![set-query-step](assets/images/set-query-step.png)

The final expression should be like the below image.

![set-query-final](assets/images/set-query-final.png)

In the `And then section`, select use an extension.

- Select the elasticsearch extension that was created in Step 2.2.
- Select the `Search request` operation.
- Under parameters, fill in the `index_name` that was used in Step 1.
- Under optional parameters, fill in the `knn`, `fields` and `source` fields.

For `knn`,
```json
{
    "field": "text_embedding",
    "query_vector_builder": {
        "text_embedding": {"model_id": "intfloat__multilingual-e5-base", "model_text": ${query}}
    },
    "k": 5,
    "num_candidates": 50
}
```
Note: set `model_id` according to the embedding model used in Step 1.

For `fields`, fill in `["text"]`

Note: `["text"]` will depend on what field names was used in Step 1.

For `source`, select False.

![set-elasticsearch-extension](assets/images/set-elasticsearch-extension.png)


### 2.3.3 Generate Answer

Click on `New step +`

Set `context` variable to the below expression,

```
${context} = "\n<documents><document>\n" + ${step_181_result_1.body.hits.hits}.joinToArray("%e.fields.text[0]%").join("\n</document>\n\n<document>\n") + "\n</document></documents>\n"
```

Edit expression using the editor to select the previous step and select `body.hits.hits`.

![update-step](assets/images/update-step.png)

Set `history` variable to the below expression,

```
${history} = ${system_session_history}.transform("role", "<|start_header_id|>user<|end_header_id|>\n\n", "<|start_header_id|>assistant<|end_header_id|>\n\n").joinToArray("%e.role%%e.content%").join("<|eot_id|>") + "<|eot_id|>"
```

![set-context-history-variable](assets/images/set-context-history-variable.png)

In the `And then section`, select use an extension.

- Select the watsonx.ai extension that was created in Step 2.2.
- Select the `Text Generation Stream` operation. Under parameters,
    - set `deployment_id` for `conversational-prompt` that was create in Step 1
    - set `version` to `2023-05-29`.
- Under optional parameters, set `parameters.prompt_variables` to the expression `{ "context": ${context}, "history": ${history} }`.
- Under stream response, set `text` to `results[0].generated_text`

![set-watsonxai-extension](assets/images/set-watsonxai-extension.png)

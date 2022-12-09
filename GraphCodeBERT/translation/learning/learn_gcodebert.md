# Learning Notes on GraphCodeBERT (GCodeBERT)

- This is my learning note when studying GraphCodeBERT codes from the [official repository](https://github.com/microsoft/CodeBERT/tree/master/GraphCodeBERT/translation)
- The repository contains 3 files and 1 folder:
    - `run.py`: experiment pipeline on the translation task
    - `model.py`: model definition
    - `bleu.py`: metric calculation
    - `parser`: a folder that contains script and code to extract dataflow from codes

# Contents
1. [Input of GCodeBERT](#input-of-gcodebert): explain how the input of the model is represented
2. [Computing Vector Representations in GCodeBERT](#computing-vector-representations-in-gcodebert): explain how the input of the model is used to obtain a vector representation of an input code

# Input of GCodeBERT
Brief summary: the input of GCodeBERT is an instance of `InputFeatures` object where it holds 8 information:
1. `example_id`: the object id
2. `source_ids`: a list of code tokens and node tokens (in dataflow) from the source (input translation)
3. `position_idx`: a list of integer to indicate the relative position of each id in `source_ids`
4. `dfg_to_code`: a list that indicates the mapping between node tokens to code tokens
5. `dfg_to_dfg`: a list that indicates the dependency relation between node tokens
6. `target_ids`: a list of code tokens from the target translation
7. `source_mask`: a list to integer to differentiate the code, node, and padding tokens
8. `target_mask`: the same as source_mask

## Workflow of `run.py`
The workflow of `run.py` is shown in the figure below.

![Training workflow inside run.py](img/run_py_training_workflow.png)

### Data Loading
- Input: two files, one contains the source translation and the other contains the target translation
- Output: a list of `Example` objects
- The data loading is performed by invoking [`read_examples`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/run.py#L124) function.
- What is `read_examples` doing?
    1. Open two files, one contains the source translation and the other contains the target translation
        Example:
        ```
        Source file
        public String toString() {return getClass().getName() + " [" +_value +"]";}
        ...

        Target file
        public override String ToString(){StringBuilder sb = new StringBuilder(64);sb.Append(GetType().Name).Append(" [");sb.Append(value);sb.Append("]");return sb.ToString();}
        ...
        ```
    2. Iterate through each line of both files
    3. Create an [`Example`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/run.py#L113) object by instantiating its attributes, i.e., `source` with the source translation, `target` with the target translation, and `lang` with the source language
        ```
        class Example(object):
            """A single training/test example."""
            def __init__(self,
                        source,
                        target,
                        lang
                        ):
                self.source = source    # source translation
                self.target = target    # target translation
                self.lang=lang          # source language
        ```
    4. Append to a list

### Data Processing
- Input: a list of `Example` objects
- Output: a list of `InputFeatures` objects
- A list of `Example` objects is fed to [`convert_examples_to_features`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/run.py#L178) function
- This step consists of two main processes:
    1. Dataflow extraction
    2. Post-processing

#### Dataflow Extraction
- The extraction begins by invoking [`extract_dataflow`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/run.py#L74) function.
- `extract_dataflow` calls another function to extract the dataflow, depending on the language of the input:
    1. Python: [`DFG_python`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L11)
    2. Java: [`DFG_java`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L180)
    3. Ruby: [`DFG_ruby`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L539)
    4. PHP: [`DFG_php`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L843)
    5. Go [`DFG_go`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L698)
    6. Javascript: [`DFG_javascript`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L1029)
    7. C#: [`DFG_csharp`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L356)
- The dataflow is extracted as follows: 
    1. Obtain the code AST using tree_sitter
    2. Extract the dataflow from the AST
- Example of an input-output pairs from this step:
    - Input: 
        ```
        public ListSpeechSynthesisTasksResult listSpeechSynthesisTasks(ListSpeechSynthesisTasksRequest request) {
            request = beforeClientExecution(request);
            return executeListSpeechSynthesisTask (request);
        }
        ```
    - Output:
        - <a id="code_token_list"></a> `code_token_list`
            ```
            ['public', 'ListSpeechSynthesisTasksResult', 'listSpeechSynthesisTasks', '', '(', 'ListSpeechSynthesisTasksRequest', 'request', ')', '{', 'request', '=', 'beforeClientExecution', '(', 'request', ')', ';', 'return', 'executeListSpeechSynthesisTasks', '(', 'request', ')', ';', '}']
            ```
        - <a id="dataflow"></a> `nodes_dependency_mapping`
            ```
            [('request', 6, 'comesFrom', [], []), 
            ('request', 9, 'computedFrom', ['beforeClientExecution'], [11]), 
            ('request', 9, 'computedFrom', ['request'], [13]), 
            ('beforeClientExecution', 11, 'comesFrom', [], []), 
            ('request', 13, 'comesFrom', ['request'], [6]), 
            ('request', 19, 'comesFrom', ['request'], [9])]
            ```
            Each tuple in the list contains 5 elements:
            1. `current_node`, where a node corresponds to an identifier in the AST 
            2. `current_node_index` in `code_token_list`
            3. `relationship`, the value is either `comes_from` or `computed_from`
            4. `source_node`
            5. `source_node_index` in `code_token_list`
            
#### Post-processing
- Inputs: `code_token_list` and `nodes_dependency_mapping`
- Outputs:
    1. <a id="source_ids"></a> `source_ids`
        - Each `code_token` in `code_token_list` is tokenized using GraphCodeBERT tokenizer, then a special tokens are appended, i.e., `<s>` in the beginning and `</sep>` in the end
            ```
            ['<s>', 'public', 'ĠList', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Result', 'Ġlist', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Ġ', 'Ġ(', 'ĠList', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Request', 'Ġrequest', 'Ġ)', 'Ġ{', 'Ġrequest', 'Ġ=', 'Ġbefore', 'Client', 'Exec', 'ution', 'Ġ(', 'Ġrequest', 'Ġ)', 'Ġ;', 'Ġreturn', 'Ġexecute', 'List', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Ġ(', 'Ġrequest', 'Ġ)', 'Ġ;', 'Ġ}', '</s>']

            length = 59
            ```
        - Each `code_token` is converted to its token id
            ```
            [0, 15110, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 48136, 889, 29235, 7529, 35615, 3999, 35571, 565, 40981, 1437, 36, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 45589, 2069, 4839, 25522, 2069, 5457, 137, 47952, 46891, 15175, 36, 2069, 4839, 25606, 671, 11189, 36583, 29235, 7529, 35615, 3999, 35571, 565, 40981, 36, 2069, 4839, 25606, 35524, 2]

            length = 59
            ```
        - Append a special token id (i.e., `3` in this case) to indicate `node_token` that corresponds to `source_node` in `nodes_dependency_mapping`. The number of added tokens is the same as the number length of `nodes_dependency_mapping`.
            ```
            [0, 15110, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 48136, 889, 29235, 7529, 35615, 3999, 35571, 565, 40981, 1437, 36, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 45589, 2069, 4839, 25522, 2069, 5457, 137, 47952, 46891, 15175, 36, 2069, 4839, 25606, 671, 11189, 36583, 29235, 7529, 35615, 3999, 35571, 565, 40981, 36, 2069, 4839, 25606, 35524, 2, 3, 3, 3, 3, 3, 3]

            length = 65
            ```
        - Insert `padding_token` until it reaches the maximum length. `padding_token` is used to make sure that all `source_ids` have the same lenght, so the model can process the input in batch.
            ```
            [0, 15110, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 48136, 889, 29235, 7529, 35615, 3999, 35571, 565, 40981, 1437, 36, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 45589, 2069, 4839, 25522, 2069, 5457, 137, 47952, 46891, 15175, 36, 2069, 4839, 25606, 671, 11189, 36583, 29235, 7529, 35615, 3999, 35571, 565, 40981, 36, 2069, 4839, 25606, 35524, 2, 3, 3, 3, 3, 3, 3, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]

            length = 200
            ```
        - In the end, `source_ids` contains three type of tokens:
            - `code_token` to represent the token in the input code
            - `node_token` to indicate a node in the dataflow graph
            - `padding_token`
    
    2. `dfg_to_code`
        ```
        code:
        public ListSpeechSynthesisTasksResult listSpeechSynthesisTasks(ListSpeechSynthesisTasksRequest request) {
            request = beforeClientExecution(request);
            return executeListSpeechSynthesisTask (request);
        }

        tokenized_code_token_list:
        ['<s>', 'public', 'ĠList', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Result', 'Ġlist', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Ġ', 'Ġ(', 'ĠList', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Request', 'Ġrequest', 'Ġ)', 'Ġ{', 'Ġrequest', 'Ġ=', 'Ġbefore', 'Client', 'Exec', 'ution', 'Ġ(', 'Ġrequest', 'Ġ)', 'Ġ;', 'Ġreturn', 'Ġexecute', 'List', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Ġ(', 'Ġrequest', 'Ġ)', 'Ġ;', 'Ġ}', '</s>']

        dfg_to_code:
        [(30, 31), (33, 34), (33, 34), (35, 39), (40, 41), (54, 55)]
        
        source_ids (the same as the example in number 1):
        [..., 36, 2069, 4839, 25606, 35524, 2, 3, 3, 3, 3, 3, 3, 1, 1, 1,...]

        mapped_node_tokens:
        [['Ġrequest'], ['Ġrequest'], ['Ġrequest'], ['Ġbefore', 'Client', 'Exec', 'ution'], ['Ġrequest'], ['Ġrequest']]
        ```
        - `dfg_to_code` encodes the mapping between `node_token` with `code_token`
        - Each element in `dfg_to_code` refers to each `node_token`
            - Element index 0 in `dfg_to_code` (i.e., (30, 31)) refers to the `node_token` index 0 in `source_ids`
            - Element index 1 in `dfg_to_code` refers to `node_token` index 1 in `source_ids`
        - Each element in `dfg_to_code` contain indexes that refer to the index in `tokenized_code_token_list`
        - The first index is the start, while the second index is the end
        - To exemplify:
            - The element index 0 in `dfg_to_code` is `(30, 31)`, meaning that `node_token` index 0 refers to the token with index 30 in `tokenized_code_token_list`, i.e., `request`
            - The element index 3 in `dfg_to_code` is `(35, 39)`, meaning that `node_token` index 3 refers to the tokens with index 35, 36, 37, and 38, i.e., `'Ġbefore'`, `'Client'`, `'Exec'`, `'ution'`
            - The remaining mapping is shown in `mapped_node_tokens`

    3. `dfg_to_dfg`
        ```
        code:
        public ListSpeechSynthesisTasksResult listSpeechSynthesisTasks(ListSpeechSynthesisTasksRequest request) {
            request = beforeClientExecution(request);
            return executeListSpeechSynthesisTask (request);
        }
        
        dfg_to_code:
        [(30, 31), (33, 34), (33, 34), (35, 39), (40, 41), (54, 55)]

        mapped_node_tokens (simplified):
        request, request, request, beforeClientExecution, request, request

        dfg_to_dfg:
        [[], [3], [4], [], [0], [2]]
        ```
        - `dfg_to_dfg` indicates the dependency between `node_token` in `source_ids`
        - For example:
            - `node_token` index 1 depends on `node_token` index 3
                - index 1, `request` before "=" in the expression `request = beforeClientExecution(request);` 
            - `node_token` index 4 depends on `node_token` index 0
                - index 4, `request` after "=" in the expression `request = beforeClientExecution(request);`
                - index 0, `request` in the function parameter

    4. `source_mask`
        - `source_mask` is used to differentiate between the input and padding token when computing the attention; the attention on the padding will be ignored by multiplying the value with 0
            ```
            [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]

            length = 200
            ```

    5.  `position_idx`
        - `position_idx` is used to indicate the relative ordering position in the sequence. 0 is used to represent the dfg, while 1 is used to indicate the padding.
            ```
            [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]

            length = 200
            ```
    
    6. `target_ids`
        - The same as `source_ids`, but it refers to the target translation
    
    7. `target_mask`
        - The same as `source_mask`, but it refers to the target translation mask
    
    8. `example_index`
        - a unique identifier number for each sample
- One `Example` object --> produce 8 outputs above --> wrapped using `InputFeatures` object
    ```
    class InputFeatures(object):
    """A single training/test features for a example."""
    def __init__(self,
                 example_id,
                 source_ids,
                 position_idx,
                 dfg_to_code,
                 dfg_to_dfg,                 
                 target_ids,
                 source_mask,
                 target_mask,

    ):
        self.example_id = example_id
        self.source_ids = source_ids
        self.position_idx = position_idx
        self.dfg_to_code = dfg_to_code
        self.dfg_to_dfg = dfg_to_dfg
        self.target_ids = target_ids
        self.source_mask = source_mask
        self.target_mask = target_mask  
    ```

# Computing Vector Representations in GCodeBERT
1. Computes `attention_mask` using `position_idx`, `dfg_to_code`, `dfg_to_dfg`, and `source_ids`
    - The main different between GCodeBERT and standard Transformer is in `attention_mask`
    - GCodeBERT leverages Graph-Guided Attention Mask to encode the dataflow information
    - Specifically, GCodeBERT leverages `dfg_to_dfg` and `dfg_to_code` to compute the `attention_mask`
    - The main ideas of Graph-Guided Attention are to encode:
        1. The dependency between `node_token`
            - Obtained from `dfg_to_dfg`
            - Given `node_token` `v_1` and node `v_2`, `node_token` `v_1` is allowed to attend to `node_token` `v_2` if there is an edge from `node_token` `v_2` to `node_token` `v_1`
            - Example:
                ```
                code:
                public ListSpeechSynthesisTasksResult listSpeechSynthesisTasks(ListSpeechSynthesisTasksRequest request) {
                    request = beforeClientExecution(request);
                    return executeListSpeechSynthesisTask (request);
                }
                
                dfg_to_code:
                [(30, 31), (33, 34), (33, 34), (35, 39), (40, 41), (54, 55)]

                mapped_node_tokens:
                [['Ġrequest'], ['Ġrequest'], ['Ġrequest'], ['Ġbefore', 'Client', 'Exec', 'ution'], ['Ġrequest'], ['Ġrequest']]

                dfg_to_dfg:
                [[], [3], [4], [], [0], [2]]
                ```
                - `node_token` index 1 can attend to `node_token` index 3
                    - index 1, `request` before "=" in the expression `request = beforeClientExecution(request);`
                    - index 3, `beforeClientExecution` in the expression `request = beforeClientExecution(request);`
                - `node_token` index 4 can attend to `node_token` index 0
                    - index 4, `request` after "=" in the expression `request = beforeClientExecution(request);`
                    - index 0, `request` in the function parameter
        2. The mapping between `node_token` to `code_token`
            - Obtained from `dfg_to_code`
            - Given `node_token` `v` and `code_token` `x`,  `node_token` `v` and `code_token` `x` can attend to each other if `code_token` `x` have relationship with `node_token` `v`
            - Example:
                - `node_token` index 3 can attend to 4 `code_tokens`, i.e,. ['Ġbefore', 'Client', 'Exec', 'ution'] and vice versa
2. The model feeds `source_ids` to the embedding layer, resulting in `temporary_embedding`
3. `temporary_embedding` + `attention_mask` + `position_idx` --> matrix manipulation -->  `node_token_embedding` and `code_token_embedding`
4. `node_token_embedding` and `code_token_embedding` is summed, resulting in `combined_embedding`
5. `combined_embedding`, `attention_mask`, and `position_idx` is passed to the encoder to produce the final vector representation
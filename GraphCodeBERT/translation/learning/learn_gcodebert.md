# Learning Notes on GraphCodeBERT (GCodeBERT)

- This is my learning note when studying GraphCodeBERT codes from the [official repository](https://github.com/microsoft/CodeBERT/tree/master/GraphCodeBERT/translation)
- The repository contains 3 files and 1 folder:
    - `run.py`: experiment pipeline on the translation task
    - `model.py`: model definition
    - `bleu.py`: metric calculation
    - `parser`: a folder that contains script and code to extract dataflow from codes

## Input of GCodeBERT
### Thought process
1. Inspect `run.py`
2. Localize the lines of code that correspond to loading the raw data and preprocess the data
3. Study these lines of code

### Workflow of `run.py`
The workflow of `run.py` is shown in the figure below.

![Training workflow inside run.py](img/run_py_training_workflow.png)

To know the how the input of GCodeBERT is represented, I scrutinize `Data Loading` and `Data Processing` further.

### Data Loading
- The data loading is performed by invoking [`read_examples`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/run.py#L124) function.
- What is `read_examples` doing?
    1. Open two files, one contains the source translation and the other contains the target translation
        - The example of one line in the source translation file
            ```
            public String toString() {return getClass().getName() + " [" +_value +"]";}
            ```
        - The example of one line in the target translation file
            ```
            public override String ToString(){StringBuilder sb = new StringBuilder(64);sb.Append(GetType().Name).Append(" [");sb.Append(value);sb.Append("]");return sb.ToString();}
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
- The output of this step is a list of `Example` object

### Data Processing
- The data processing is performed by feeding a list of `Example` objects from the the data loading step to [`convert_examples_to_features`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/run.py#L178) function
- This step consists of two substeps:
    1. Dataflow extraction
    2. Post-processing
#### Dataflow Extraction
- The extraction begins by invoking [`extract_dataflow`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/run.py#L74) function.
- `extract_dataflow` will call another function to extract the dataflow, depending on the language of the input. Currently, there are 7 functions:
    1. Python: [`DFG_python`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L11)
    2. Java: [`DFG_java`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L180)
    3. Ruby: [`DFG_ruby`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L539)
    4. PHP: [`DFG_php`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L843)
    5. Go [`DFG_go`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L698)
    6. Javascript: [`DFG_javascript`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L1029)
    7. C#: [`DFG_csharp`](https://github.com/microsoft/CodeBERT/blob/305bab27920d57ba37964ab05ee7b1acdbb24416/GraphCodeBERT/translation/parser/DFG.py#L356)
- The idea to extract the dataflow is: 
    1. Obtain the AST of the code using `tree_sitter`
    2. Apply processing steps on the AST
- Example of an input-output pairs from this step:
    - Input: 
        ```
        public ListSpeechSynthesisTasksResult listSpeechSynthesisTasks(ListSpeechSynthesisTasksRequest request) {
            request = beforeClientExecution(request);
            return executeListSpeechSynthesisTask (request);
        }
        ```
    - Output:
        - `code_token_list`
            ```
            ['public', 'ListSpeechSynthesisTasksResult', 'listSpeechSynthesisTasks', '', '(', 'ListSpeechSynthesisTasksRequest', 'request', ')', '{', 'request', '=', 'beforeClientExecution', '(', 'request', ')', ';', 'return', 'executeListSpeechSynthesisTasks', '(', 'request', ')', ';', '}']
            ```
        - `dataflow`
            - The dataflow is represented using a tuple that contains 5 elements:
                1. `current_identitifer`
                2. Index of `current_identifier` in `code_token_list`
                3. `comes_from` or `computed_from`
                4. `source_identifier`
                5. Index of `source_identifier` in `code_token_list`
            - Example:
                ```
                [('request', 6, 'comesFrom', [], []), 
                ('request', 9, 'computedFrom', ['beforeClientExecution'], [11]), 
                ('request', 9, 'computedFrom', ['request'], [13]), 
                ('beforeClientExecution', 11, 'comesFrom', [], []), 
                ('request', 13, 'comesFrom', ['request'], [6]), 
                ('request', 19, 'comesFrom', ['request'], [9])]
                ```
#### Post-processing
- The outputs and processing steps to obtain each output are presented below:
    1. `source_ids`
        - Each element in `code_token_list` is tokenized using GraphCodeBERT tokenizer, then append a special token `<s>` in the beginning and `</sep>` in the end
            ```
            ['<s>', 'public', 'ĠList', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Result', 'Ġlist', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Ġ', 'Ġ(', 'ĠList', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Request', 'Ġrequest', 'Ġ)', 'Ġ{', 'Ġrequest', 'Ġ=', 'Ġbefore', 'Client', 'Exec', 'ution', 'Ġ(', 'Ġrequest', 'Ġ)', 'Ġ;', 'Ġreturn', 'Ġexecute', 'List', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Ġ(', 'Ġrequest', 'Ġ)', 'Ġ;', 'Ġ}', '</s>']

            length = 59
            ```
        - Each token is converted to its token id
            ```
            [0, 15110, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 48136, 889, 29235, 7529, 35615, 3999, 35571, 565, 40981, 1437, 36, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 45589, 2069, 4839, 25522, 2069, 5457, 137, 47952, 46891, 15175, 36, 2069, 4839, 25606, 671, 11189, 36583, 29235, 7529, 35615, 3999, 35571, 565, 40981, 36, 2069, 4839, 25606, 35524, 2]

            length = 59
            ```
        - Append a special token ids (i.e., `3` in this case) to represent dfg. The number of the special tokens should matches with the number of element in `dataflow`.
            ```
            [0, 15110, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 48136, 889, 29235, 7529, 35615, 3999, 35571, 565, 40981, 1437, 36, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 45589, 2069, 4839, 25522, 2069, 5457, 137, 47952, 46891, 15175, 36, 2069, 4839, 25606, 671, 11189, 36583, 29235, 7529, 35615, 3999, 35571, 565, 40981, 36, 2069, 4839, 25606, 35524, 2, 3, 3, 3, 3, 3, 3]

            length = 65
            ```
        - Insert padding token until it reaches the (specified) maximum length. The goal this step is to make sure that all `source_ids` have the same lenght, so the model can process it in batch
            ```
            [0, 15110, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 48136, 889, 29235, 7529, 35615, 3999, 35571, 565, 40981, 1437, 36, 9527, 29235, 7529, 35615, 3999, 35571, 565, 40981, 45589, 2069, 4839, 25522, 2069, 5457, 137, 47952, 46891, 15175, 36, 2069, 4839, 25606, 671, 11189, 36583, 29235, 7529, 35615, 3999, 35571, 565, 40981, 36, 2069, 4839, 25606, 35524, 2, 3, 3, 3, 3, 3, 3, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]

            length = 200
            ```
    2. `source_mask`
        - `source_mask` is used to differentiate between the input and padding token when computing the attention; the attention on the padding will be ignored by multiplying the value with 0
            ```
            [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]

            length = 200
            ```
    3.  `position_idx`
        - `position_idx` is used to indicate the relative ordering position in the sequence. 0 is used to represent the dfg, while 1 is used to indicate the padding.
            ```
            [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]

            length = 200
            ```
    4. `dfg_to_code`
        - `dfg_to_code` is a list that contains tuple of two integers that represent the start and end indexes of the subword of a code token in the `code_token_list` after the tokenization step.
            ```
            code:
            public ListSpeechSynthesisTasksResult listSpeechSynthesisTasks(ListSpeechSynthesisTasksRequest request) {
                request = beforeClientExecution(request);
                return executeListSpeechSynthesisTask (request);
            }
            ```
            ```
            tokenized code_token_list:
            ['<s>', 'public', 'ĠList', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Result', 'Ġlist', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Ġ', 'Ġ(', 'ĠList', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Request', 'Ġrequest', 'Ġ)', 'Ġ{', 'Ġrequest', 'Ġ=', 'Ġbefore', 'Client', 'Exec', 'ution', 'Ġ(', 'Ġrequest', 'Ġ)', 'Ġ;', 'Ġreturn', 'Ġexecute', 'List', 'Spe', 'ech', 'Sy', 'nt', 'hesis', 'T', 'asks', 'Ġ(', 'Ġrequest', 'Ġ)', 'Ġ;', 'Ġ}', '</s>']
            ```
            ```
            dataflow:
            [('request', 6, 'comesFrom', [], []), 
            ('request', 9, 'computedFrom', ['beforeClientExecution'], [11]), 
            ('request', 9, 'computedFrom', ['request'], [13]), 
            ('beforeClientExecution', 11, 'comesFrom', [], []), 
            ('request', 13, 'comesFrom', ['request'], [6]), 
            ('request', 19, 'comesFrom', ['request'], [9])]
            ```
            ```
            dfg_to_code:
            [(30, 31), (33, 34), (33, 34), (35, 39), (40, 41), (54, 55)]
            ```
        - For example:
            - `(30, 31)` refers to the subword token number 30, which is `request`
            - `(35, 39)` refers to the subword tokens number 35 until 38, which represent `beforeClientExecution`. The code token `beforeClientExecution` is tokenized into 4 subwords, i.e., 'Ġbefore', 'Client', 'Exec', 'ution'.
        - Each element in `dfg_to_code` corresponds to the elements in `dataflow`, specifically the element number 1 inside the tuple
    5. `dfg_to_dfg`
        - `dfg_to_dfg` is used to indicate the dependency between elements in `dfg_to_code`
            ```
            dfg_to_code:
            [(30, 31), (33, 34), (33, 34), (35, 39), (40, 41), (54, 55)]
            ```
            ```
            dfg_to_dfg:
            [[], [3], [4], [], [0], [2]]
            ```
        - For example:
            - Element number 0 has no dependency
            - Element number 1 (request) comes from element number 3 (beforeClientExecution)
            - Element number 2 (request) comes from element number 4 (request)
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
    

        


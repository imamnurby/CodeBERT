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
- The data loading is performed by invoking function `read_examples`.
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
    3. Create an `Example` object by instantiating its attributes, i.e., `source` with the source translation, `target` with the target translation, and `lang` with the source language
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
- The data processing is performed by feeding a list of `Example` objects from the the data loading step to a function `convert_examples_to_features`
- This step consists of two substeps:
    1. Dataflow extraction
    2. Post-processing
#### Dataflow Extraction
- The extraction begins by invoking `extract_dataflow`
- `extract_dataflow` will call another function to extract the dataflow, depending on the language of the input. Currently, there are 7 functions:
    1. Python (`DFG_python`)
    2. Java (`DFG_java`)
    3. Ruby (`DFG_ruby`)
    4. Go (`DFG_python`)
    5. PHP (`DFG_go`)
    6. Javascript (`DFG_javascript`)
    7. C# (`DFG_csharp`)
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
        - Dataflow information
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
-  
    






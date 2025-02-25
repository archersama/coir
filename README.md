

We will release the dataset once the review process is complete, as its size exceeds the 200 MB limit imposed by the submission system.

Currently, the package is non-operational due to the anonymization of the dataset.



## :coconut: What is CoIR?

**CoIR** (**Co**de **I**nformation **R**etrieval) benchmark, is designed to evaluate code retrieval capabilities. CoIR includes **10** curated code datasets, covering **8** retrieval tasks across **7** domains. It also provides a **common and easy** Python framework, installable via pip, and shares the same data schema as benchmarks like MTEB and BEIR for easy cross-benchmark evaluations.




<div align="center">
    <img src="pictures/coir_overview.svg" width="850" />
    <br>
    <strong>Overview of COIR benchmark.</strong>
</div>


## :coconut: Data Availability


### Statistics of datasets in coir benchmark
\# is the quantity of query/corpus instances. L refers to the average numbers of words per query/corpus. Datasets marked by \(^\dag\) are created by us.

| **Main Task**                | **Sub Task**                       | **Domain**     | **Dataset**                  | **Language**                                   | **#Query (train/dev/test)** | **#Corpus** | **L_Query** | **L_Corpus** |
|------------------------------|------------------------------------|----------------|------------------------------|------------------------------------------------|-----------------------------|-------------|-------------------------|---------------------------|
| Text-to-Code Retrieval       | Code Contest Retrieval             | Code Contest   | APPS                         | py                                             | 5k/-/3.8K                   | 9K          | 1.4K                    | 575                       |
|                              | Web Query to Code Retrieval        | Web query      | CosQA                        | py                                             | 19k/-/500                   | 21K         | 37                      | 276                       |
|                              | Text to SQL Retrieval              | Database       | Synthetic Text2SQL           | sql                                            | 100k/-/6K                   | 106K        | 83                      | 127                       |
| Code-to-Text Retrieval       | Code Summary Retrieval             | Github         | CodeSearchNet                | go, java, js, php, py, ruby                    | 905k/41k/53K                | 1M          | 594                     | 156                       |
| Code-to-Code Retrieval       | Code Context Retrieval             | Github         | CodeSearchNet-CCR^\dag       | go, java, js, php, py, ruby                    | 905k/41k/53K                | 1M          | 154                     | 113                       |
|                              | Similar Code Retrieval             | Deep Learning  | CodeTrans Ocean-DL           | py                                             | 564/72/180                  | 816         | 1.6K                    | 1.5K                      |
|                              |                                    | Contest        | CodeTrans Ocean-Contest      | c++, py                                        | 561/226/446                 | 1K          | 770                     | 1.5K                      |
| Hybrid Code Retrieval        | Single-turn Code QA                | Stack Overflow | StackOverflow QA^\dag        | miscellaneous                                  | 13k/3k/2K                   | 20K         | 1.4K                    | 1.2K                      |
|                              |                                    | Code Instruction | CodeFeedBack-ST              | html, c, css, sql, js, sql, py, shell, ruby, rust, swift | 125k/-/31K | 156K        | 722                     | 1.5K                      |
|                              | Multi-turn Code QA                 | Code Instruction | CodeFeeback-MT               | miscellaneous                                  | 53k/-/13K                   | 66K         | 4.4K                    | 1.5K                      |


## :coconut: Features
- CoIR encompasses a total of ten distinct code retrieval datasets.
- CoIR supports seamless integration with Hugging Face and other libraries, enabling one-click loading and evaluation of models.
- CoIR supports custom models and API-based models, offering flexible integration options for diverse requirements.


### :coconut: Simple Usage

You can download this GitHub repository (`python>3.8`) and use as follows:

```python
import coir
from coir.models import YourCustomDEModel

model_name = "intfloat/e5-base-v2"

# Load the model
model = YourCustomDEModel(model_name=model_name)

# Get tasks
#all task ["codetrans-dl","stackoverflow-qa","apps","codefeedback-mt","codefeedback-st","codetrans-contest","synthetic-
# text2sql","cosq","codesearchnet","codesearchnet-ccr"]
tasks = coir.get_tasks(tasks=["codetrans-dl"])

# Initialize evaluation
evaluation = COIR(tasks=tasks，batch_size=128)

# Run evaluation
results = evaluation.run(model, output_folder=f"results/{model_name}")
print(results)
```
### :coconut: Advanced Usage
<details>
  <summary>Click to Expand/Collapse Content</summary>
    
#### Custom Dense Retrieval Models
```python
import coir
from coir.data_loader import get_tasks
from coir.evaluation import COIR
import torch
import numpy as np
import logging
from transformers import AutoTokenizer, AutoModel
from typing import List, Dict
from tqdm.auto import tqdm

class YourCustomDEModel:
    def __init__(self, model_name="intfloat/e5-base-v2", **kwargs):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModel.from_pretrained(model_name).to(device)
        self.model_name = model_name
        self.tokenizer.add_eos_token = False

    def mean_pooling(self, model_output, attention_mask):
        token_embeddings = model_output[0]  # First element of model_output contains all token embeddings
        input_mask_expanded = attention_mask.unsqueeze(-1).expand(token_embeddings.size()).float()
        sum_embeddings = torch.sum(token_embeddings * input_mask_expanded, 1)
        sum_mask = torch.clamp(input_mask_expanded.sum(1), min=1e-9)
        return sum_embeddings / sum_mask

    def cls_pooling(self, model_output, attention_mask):
        # First element of model_output contains all token embeddings
        token_embeddings = model_output[0]
        # Extract the CLS token's embeddings (index 0) for each sequence in the batch
        cls_embeddings = token_embeddings[:, 0, :]
        return cls_embeddings

    def last_token_pool(self, model_output, attention_mask):
        last_hidden_states = model_output.last_hidden_state
        left_padding = (attention_mask[:, -1].sum() == attention_mask.shape[0])
        if left_padding:
            return last_hidden_states[:, -1]
        else:
            sequence_lengths = attention_mask.sum(dim=1) - 1
            batch_size = last_hidden_states.shape[0]
            return last_hidden_states[torch.arange(batch_size, device=last_hidden_states.device), sequence_lengths]

    def encode_text(self, texts: List[str], batch_size: int = 12, max_length: int = 128) -> np.ndarray:
        logging.info(f"Encoding {len(texts)} texts...")

        embeddings = []
        for i in tqdm(range(0, len(texts), batch_size), desc="Encoding batches", unit="batch"):
            batch_texts = texts[i:i+batch_size]
            encoded_input = self.tokenizer(batch_texts, padding=True, truncation=True, max_length=max_length, return_tensors="pt").to(device)
            with torch.no_grad():
                model_output = self.model(**encoded_input)
            batch_embeddings = self.mean_pooling(model_output, encoded_input['attention_mask'])
            embeddings.append(batch_embeddings.cpu())

        embeddings = torch.cat(embeddings, dim=0)

        if embeddings is None:
            logging.error("Embeddings are None.")
        else:
            logging.info(f"Encoded {len(embeddings)} embeddings.")

        return embeddings.numpy()

    def encode_queries(self, queries: List[str], batch_size: int = 12, max_length: int = 512, **kwargs) -> np.ndarray:
        all_queries = ["query: "+ query for query in queries]
        return self.encode_text(all_queries, batch_size, max_length)

    def encode_corpus(self, corpus: List[Dict[str, str]], batch_size: int = 12, max_length: int = 512, **kwargs) -> np.ndarray:
        all_texts = ["passage: "+ doc['text'] for doc in corpus]
        return self.encode_text(all_texts, batch_size, max_length)

# Load the model
model = YourCustomDEModel()

# Get tasks
#all task ["codetrans-dl","stackoverflow-qa","apps","codefeedback-mt","codefeedback-st","codetrans-contest","synthetic-
# text2sql","cosq","codesearchnet","codesearchnet-ccr"]
tasks = coir.get_tasks(tasks=["codetrans-dl"])

# Initialize evaluation
evaluation = COIR(tasks=tasks，batch_size=128)

# Run evaluation
results = evaluation.run(model, output_folder=f"results/{model_name}")
print(results)
```
#### Using Sentence-Transformers Models
```python
import coir
from coir.data_loader import get_tasks
from coir.evaluation import COIR
import torch
import numpy as np
import logging
from sentence_transformers import SentenceTransformer
from typing import List, Dict
from tqdm.auto import tqdm

class YourCustomDEModel:
    def __init__(self, model_name="intfloat/e5-base-v2", **kwargs):
        self.model = SentenceTransformer(model_name)

    def encode_text(self, texts: List[str], batch_size: int = 12, show_progress_bar: bool = True, **kwargs) -> np.ndarray:
        logging.info(f"Encoding {len(texts)} texts...")
        
        embeddings = self.model.encode(texts, batch_size=batch_size, show_progress_bar=show_progress_bar, **kwargs)
        
        if embeddings is None:
            logging.error("Embeddings are None.")
        else:
            logging.info(f"Encoded {len(embeddings)} embeddings.")
        
        return np.array(embeddings)

    def encode_queries(self, queries: List[str], batch_size: int = 12, show_progress_bar: bool = True, **kwargs) -> np.ndarray:
        all_queries = ["query: "+ query for query in queries]
        return self.encode_text(all_queries, batch_size, show_progress_bar, **kwargs)

    def encode_corpus(self, corpus: List[Dict[str, str]], batch_size: int = 12, show_progress_bar: bool = True, **kwargs) -> np.ndarray:
        all_texts = ["passage: "+ doc['text'] for doc in corpus]
        return self.encode_text(all_texts, batch_size, show_progress_bar, **kwargs)

# Load the model
model = YourCustomDEModel()

# Get tasks
tasks = coir.get_tasks(tasks=["codetrans-dl"])

# Initialize evaluation
evaluation = COIR(tasks=tasks, batch_size=128)

# Run evaluation
results = evaluation.run(model, output_folder=f"results/{model.model_name}")
print(results)
```

#### Custom API Retrieval Models
```python
import coir
from coir.data_loader import get_tasks
from coir.evaluation import COIR
import torch
import numpy as np
import logging
from transformers import AutoTokenizer, AutoModel
from typing import List, Dict
from tqdm.auto import tqdm

class APIModel:
    def __init__(self, model_name="voyage-code-2", **kwargs):
        # Initialize the voyageai client
        self.vo = voyageai.Client(api_key="xxxx")  # This uses VOYAGE_API_KEY from environment
        self.model_name = model_name
        self.requests_per_minute = 300  # Max requests per minute
        self.delay_between_requests = 60 / self.requests_per_minute  # Delay in seco

    def encode_text(self, texts: list, batch_size: int = 12, input_type: str = "document") -> np.ndarray:
        logging.info(f"Encoding {len(texts)} texts...")

        all_embeddings = []
        start_time = time.time()
        # Processing texts in batches
        for i in tqdm(range(0, len(texts), batch_size), desc="Encoding batches", unit="batch"):
            batch_texts = texts[i:i + batch_size]
            result = self.vo.embed(batch_texts, model=self.model_name, input_type=input_type,truncation=True)
            batch_embeddings = result.embeddings  # Assume the API directly returns embeddings
            all_embeddings.extend(batch_embeddings)
            # Ensure we do not exceed rate limits
            time_elapsed = time.time() - start_time
            if time_elapsed < self.delay_between_requests:
                time.sleep(self.delay_between_requests - time_elapsed)
                start_time = time.time()

        # Combine all embeddings into a single numpy array
        embeddings_array = np.array(all_embeddings)

        # Logging after encoding
        if embeddings_array.size == 0:
            logging.error("No embeddings received.")
        else:
            logging.info(f"Encoded {len(embeddings_array)} embeddings.")

        return embeddings_array

    def encode_queries(self, queries: list, batch_size: int = 12, **kwargs) -> np.ndarray:
        truncated_queries = [query[:256] for query in queries]
        truncated_queries = ["query: " + query for query in truncated_queries]
        query_embeddings = self.encode_text(truncated_queries, batch_size, input_type="query")
        return query_embeddings


    def encode_corpus(self, corpus: list, batch_size: int = 12, **kwargs) -> np.ndarray:
        texts = [doc['text'][:512]  for doc in corpus]
        texts = ["passage: " + doc for doc in texts]
        return self.encode_text(texts, batch_size, input_type="document")

# Load the model
model = APIModel()

# Get tasks
#all task ["codetrans-dl","stackoverflow-qa","apps","codefeedback-mt","codefeedback-st","codetrans-contest","synthetic-
# text2sql","cosq","codesearchnet","codesearchnet-ccr"]
tasks = coir.get_tasks(tasks=["codetrans-dl"])

# Initialize evaluation
evaluation = COIR(tasks=tasks，batch_size=128)

# Run evaluation
results = evaluation.run(model, output_folder=f"results/{model_name}")
print(results)
```

</details> ```



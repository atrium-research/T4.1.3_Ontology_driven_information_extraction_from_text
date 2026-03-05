<div align="center">

<img src="https://atrium-research.eu/assets/content/en/pages/communications-kit/clay.png" width="400" alt="Atrium Logo">

# 📝 Subtask 4.1.3 - Ontology driven information extraction from text 📝

In this repository we include the material for the subtask 4.1.3. You can find more information on our workflow published on [SHH Open Marketplace](https://marketplace.sshopencloud.eu/workflow/tZ636O).

</div>

# 🛠️ Installation
1. **Clone the repository**
   ```cmd
   git clone https://github.com/atrium-research/T4.1.3_Ontology_driven_information_extraction_from_text.git
   ```
2. **Navigate to the project directory**
   ```cmd
   cd T4.1.3_Ontology_driven_information_extraction_from_text
   ```
3. **(Optional but recommended) Create and activate a virtual environment**
   
   Using `virtualenv` (MacOS):
   ```cmd
   python3.11 -m venv venv
   source ./venv/bin/activate
   ```

   Windows:
   ```cmd
   py -3.11 -m venv venv
   venv\Scripts\activate
   ```

5. Upgrade pip
   ```cmd
   pip install --upgrade pip
   ```
   
6. Install the required dependencies
   ```cmd
   pip install -r requirements.txt
   ```
7. Open the `ATRIUM_WP_4_1_3_workflow.ipynb` notebook using a code editor that supports Jupyter notebooks, such as Jupyter Notebook, JupyterLab, or VS Code.

📚 For more information on how to set up a virtual environment, visit the official documentation:
- [Python venv documentation](https://docs.python.org/3/library/venv.html)

# 🔧 Workflow
## Step 1: Entity Extraction

Employ Machine Learning (ML) models for the extraction of each entity type. Each entity extraction task is treated as a token classification problem where an Entity Recognizer predicts whether a token in input is part of a particular entity type or not. 

In this workflow implementation three different types of entities are extracted: *Activity* (i.e. a scholarly process like an archeological excavation, a social study or steps thereof, etc.), *Goal* (i.e. a research objective of an activity denoting *why* the latter was conducted), and *Method* (i.e. a procedure, plan or technique, employed by an activity and denoting *how* the latter was conducted). Each Deep Learning (DL) model employed for Entity Extraction is a combination of a Transformer ([bert-base-NER](https://huggingface.co/dslim/bert-base-NER)) that is used for vector representation of each token and a Transition-based Parser that handles the classification task. The Transformer models have been downloaded from [🤗 Hugging Face](https://huggingface.co/) library and are further fine-tuned for the tasks at hand in a manually annotated dataset of 15.000 sentences from research publications in Humanities. Each entity extraction DL model is implemented and trained using the [SpaCy](https://spacy.io/) NLP framework. You can employ other DL / ML models for the extraction of entities from text as long as they are implemented using the SpaCy framework and are declared in the input paths of the corresponding module accordingly. 
	
**Input**: chunks of text in [JSON Lines (jsonl)](https://jsonlines.org/) format where each line is a separate dictionary with the following structure:

- **`"text"`**: A string containing the cleaned sentence.
- **`"meta"`**: A dictionary holding the publication metadata for the source article from which the sentence was extracted.

**Output**: sentences of cleaned text in [JSON Lines (jsonl)](https://jsonlines.org/) format where each line is a separate dictionary with the following structure:

- **`"text"`**: A string containing the cleaned sentence.
- **`"meta"`**: A dictionary holding the publication metadata for the source article from which the sentence was extracted.

- **`"spans"`**: A list of entity annotations. Each item is a dictionary representing a textual span with:
  - `"start"`: Start character index of the span.
  - `"end"`: End character index of the span.
  - `"span"`: The extracted text of the span.
  - `"label"`: The label assigned to the span (METHOD, ACTIVITY, GOAL).

### Example

```json
{
  "text": "We applied the Random Forest algorithm to classify the samples.",
  "meta": {...},
  "spans": [
    {
      "start_char": 17,
      "end_end": 38,
      "token_start": 3,
      "token_end": 6,
      "span": "Random Forest algorithm",
      "label": "METHOD"
    }
  ]
}
```

## Step 2: Entity Disambiguation

Take the output of the Entity Extraction Step as input and perform Entity Disambiguation for each of the extracted entities when possible. Specifically, for each text at the input, check whether it contains extracted named entities (i.e. entities that can be found in literature under the same proper name or variations of it) and perform the disambiguation by associating them with their corresponding entry in a Knowledge Base such as Wikipedia.

In this workflow implementation the extracted textual spans of type “METHOD”, representing the various research methods that are used by the researchers during a research activity, can be treated as **named entities** and thus can be disambiguated by associating them with their corresponding Wikipedia page. The workflow implementation uses the [GENRE](https://github.com/facebookresearch/GENRE?tab=readme-ov-file) algorithm for disambiguation via Wikipedia as implemented by [Zshot](https://github.com/IBM/zshot) framework for better results, but any other model could be used as long as it is declared appropriately in the paths of the corresponding workflow module.

**Input**: Output from previous step.

**Output**: sentences of cleaned text in [JSON Lines (jsonl)](https://jsonlines.org/) format where each line is a separate dictionary with the following structure:

- **`"text"`**: A string containing the cleaned sentence.
- **`"meta"`**: A dictionary holding the publication metadata for the source article from which the sentence was extracted.

- **`"spans"`**: A list of entity annotations. Each item is a dictionary representing a textual span with:
  - `"start"`: Start character index of the span.
  - `"end"`: End character index of the span.
  - `"span"`: The extracted text of the span.
  - `"label"`: The label assigned to the span (METHOD, ACTIVITY, GOAL).
  
  For spans where `"label": "METHOD"`, the following additional fields are included:
  - `"wikipedia_url"`: A link to the corresponding Wikipedia entry.

### Example

```json
{
  "text": "We applied the Random Forest algorithm to classify the samples.",
  "meta": {...},
  "spans": [
    {
      "start_char": 17,
      "end_end": 38,
      "token_start": 3,
      "token_end": 6,
      "span": "Random Forest algorithm",
      "label": "METHOD",
      "wikipedia_url": "https://en.wikipedia.org/?curid=1363880"
    }
  ]
}
```
## Step 3: Entity Linking

Take as input the output of Entity Disambiguation Step and, using various APIs, retrieve additional information from the linked Knowledge Bases. 

In this workflow implementation the [ORCID API](https://info.orcid.org/what-is-orcid/services/public-api/), is used in order to link the names of the authors from the “meta” field with their corresponding ORCID and related information (email, affiliations, and full name) from the ORCID repository when possible. In addition, for the disambiguated methods, Wikipedia, Wikidata and DBPedia APIs are being accessed through information-linking queries in order to retrieve the methods’ description, proper and alternate names (aliases) and corresponding URLs. 

**Input**: Output from previous step.

**Output**: The output is a [JSON Lines (jsonl)](https://jsonlines.org/) file format where each line is a separate dictionary with the following structure:

- **`"text"`**: A string containing the cleaned sentence.
- **`"meta"`**: A dictionary holding the publication metadata for the source article from which the sentence was extracted.
  For each element in the `"creator"` field, the following subfields are included:
  - `"orcid"`: ORCID identifier of the creator.
  - `"affiliations"`: Affiliated institution(s) of the creator.
  - `"full name"`: Full name of the creator.
  - `"email"`: Email address of the creator.

- **`"spans"`**: A list of entity annotations. Each item is a dictionary representing a textual span with:
  - `"start"`: Start character index of the span.
  - `"end"`: End character index of the span.
  - `"span"`: The extracted text of the span.
  - `"label"`: The label assigned to the span (METHOD, ACTIVITY, GOAL).
  
  For spans where `"label": "METHOD"`, the following additional fields are included:
  - `"description"`: A brief textual explanation of the method.
  - `"proper_name"`: The standard name of the method.
  - `"aliases"`: A list of alternative names or synonyms.
  - `"wikidata_url"`: A link to the corresponding Wikidata entry, if available.
  - `"dbpedia_url"`: A link to the corresponding DBpedia entry, if available.

### Example

```json
{
  "text": "We applied the Random Forest algorithm to classify the samples.",
  "meta": {...,
    "creator": [
      {
        "orcid": "0000-0002-1825-0097",
        "affiliations": ["Department of Computer Science, University of Toronto"],
        "full name": "Alice Smith",
        "email": "alice.smith@universityoftoronto.edu"
      }
    ]
  },
  "spans": [
    {
      "start_char": 17,
      "end_end": 38,
      "token_start": 3,
      "token_end": 6,
      "span": "Random Forest algorithm",
      "label": "METHOD",
      "description": "Random forests or random decision forests is an ensemble learning method for classification, regression and...",
      "proper_name": "Random forest",
      "aliases": ["random forests", "randomized trees", "RF"],
      "wikipedia_url": "https://en.wikipedia.org/?curid=1363880",
      "wikidata_url": "https://www.wikidata.org/wiki/Q245748",
      "dbpedia_url": "http://dbpedia.org/resource/Random_forest"
    }
  ]
}
```


## Step 4: Relation Extraction

Take the output of Entity Linking Step and, using inferencing rules, create semantic relationships that interrelate the extracted entities in each document. 

In this workflow implementation an extracted activity is related with an extracted method via the *employs(Activity,Method)* relationship, in instances where the activity employs the method. In addition, an extracted activity is linked to an extracted goal via the *hasObjective(Activity,Goal)* relationship, provided that the activity has that particular goal as its objective.

The implemented rules for the creation of the two semantic relationships are the following:

* **employs(Activity,Method)**: for each combination of the “ACTIVITY” and “METHOD” spans inside the sentence, check whether the “METHOD” overlaps with the “ACTIVITY” and if that is the case, create the corresponding *employs(Activity,Method)* relation. 
* **hasObjective(Activity,Goal)**: for each combination of the extracted “ACTIVITY” and “GOAL” spans inside the same sentence create the corresponding *hasObjective(Activity,Goal)* relation. 

**Input**: Output from previous step.

**Output**: The output is a [JSON Lines (jsonl)](https://jsonlines.org/) file format where each line is a separate dictionary with the following structure:

Each line contains the following keys:

- **`"text"`**: A string containing the cleaned sentence.
- **`"meta"`**: A dictionary holding the publication metadata for the source article from which the sentence was extracted.
- **`"spans"`**: A list of entity annotations. Each item in the list is a dictionary that defines a textual span representing an extracted entity, including its start and end character positions, the extracted text, and its label.
- **`"relations"`**: A list of dictionaries, each describing a relationship extracted from the sentence. Every relation includes:
  - **`"domain"`**: A dictionary representing the source of the relation. It includes:
    - `"start_char"`: Start character index of the domain span.
    - `"end_char"`: End character index of the domain span.
    - `"span"`: The text of the domain span.
    - `"label"`: The entity label of the domain span.
  - **`"range"`**: A dictionary representing the target of the relation. It includes:
    - `"start_char"`: Start character index of the range span.
    - `"end_char"`: End character index of the range span.
    - `"span"`: The text of the range span.
    - `"label"`: The entity label of the range span.
  - **`"label"`**: The label of the relation (e.g., `"HAS_OBJECTIVE"` or `"EMPLOYS"`).

### Example

```json
{
  "text": "The company launched a new research initiative to explore renewable energy solutions.",
  "meta": {....},
  "spans": [{...}],
  "relations": [
    {
      "domain": {
        "start_char": 30,
        "end_char": 49,
        "span": "research initiative",
        "label": "ACTIVITY"
      },
      "range": {
        "start_char": 53,
        "end_char": 83,
        "span": "explore renewable energy solutions",
        "label": "GOAL"
      },
      "label": "HAS_OBJECTIVE"
    }
  ]
}


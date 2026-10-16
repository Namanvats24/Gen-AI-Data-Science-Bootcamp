# LangChain Data Ingestion

Load data from different sources into LangChain `Document` objects, the first step of a RAG pipeline.

## Loaders covered

| Loader | Source |
|---|---|
| `TextLoader` | Local text file (`speech.txt`) |
| `PyPDFLoader` | Local PDF (`attention.pdf`) |
| `WebBaseLoader` | Web page, parsed with BeautifulSoup |
| `WikipediaLoader` | Wikipedia search results |

Each loader returns a list of `Document` objects with `page_content` (the text) and `metadata` (source info).

## Files

```
langchain/
├── dataingestion.ipynb
├── speech.txt
├── attention.pdf
├── requirements.txt
└── README.md
```

## Setup

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Open `dataingestion.ipynb`, select the `venv` kernel, and run the cells top to bottom.

## Quick example

```python
from langchain_community.document_loaders import PyPDFLoader

docs = PyPDFLoader("attention.pdf").load()
print(docs[0].page_content[:300])
print(docs[0].metadata)
```

## Notes

- Restart the kernel after installing or changing packages.
- `WebBaseLoader`: no spaces in the URL, and the `class_` names must match the page's HTML.
- `WikipediaLoader` can fail with `JSONDecodeError` if Wikipedia rate-limits you. Wait a few minutes and retry.

## Next steps

Split text, create embeddings, store them in a vector database, and build a retrieval chain.

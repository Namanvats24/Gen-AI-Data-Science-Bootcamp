Data Ingestion with LangChain Document Loaders

A hands-on notebook showing how to load data from different sources into LangChain Document objects, the first step of any RAG (Retrieval-Augmented Generation) pipeline.

What's covered
Loader	Source	Notes
TextLoader	Local .txt file (speech.txt)	Simplest loader; one Document per file
PyPDFLoader	Local PDF (attention.pdf)	One Document per page, with page metadata
WebBaseLoader	Web page (Lilian Weng's LLM agents post)	Uses BeautifulSoup to keep only the post title, header, and content
WikipediaLoader	Wikipedia search ("Generative AI")	Returns up to load_max_docs articles with title, summary, and source URL

Every loader returns a list of Document objects, each with:

page_content: the text
metadata: source information (file path, page number, URL, title, and so on)
Project structure
.
├── dataingestion.ipynb   # the notebook
├── speech.txt            # sample text file for TextLoader
├── attention.pdf         # "Attention Is All You Need" paper for PyPDFLoader
└── README.md
Setup

Python 3.10+ is recommended.

bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

Minimal requirements.txt for this notebook:

langchain
langchain-community
ipykernel
pypdf
beautifulsoup4
wikipedia

Then open the notebook in VS Code or Jupyter and select the venv kernel.

Usage

Run the cells top to bottom. Example:

python
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("attention.pdf")
docs = loader.load()
print(docs[0].page_content[:300])
print(docs[0].metadata)
Troubleshooting
Restart the kernel after installing or changing packages. Python keeps the old module in memory until you do.
WebBaseLoader returns empty content: check that the URL has no stray spaces and that the class_ names match the page's HTML (inspect the page in your browser's dev tools). The CSS classes are hyphenated, for example post-title.
WikipediaLoader raises JSONDecodeError: Wikipedia's API likely rejected the request (rate limit or blocked User-Agent). Wait a few minutes, or set a custom User-Agent:
python
  import wikipedia.wikipedia as wp
  wp.USER_AGENT = "MyProject/1.0 (you@example.com)"
ArxivLoader fails with 'Search' object has no attribute 'results': the arxiv 2.x package is incompatible with langchain-community. Use pip install "arxiv<2".
Deprecation warning: langchain-community is being sunset, so loaders may move to dedicated packages in future releases.
Next steps
Split documents with RecursiveCharacterTextSplitter
Create embeddings and store them in a vector database (Chroma or FAISS)
Build a retrieval chain on top of the loaded data
References
LangChain document loader integrations
Part of the Gen-AI-Data-Science-Bootcamp repository

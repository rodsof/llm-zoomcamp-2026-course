
flowchart TD
    U([User])

    APP[Application]

    DB[(DB)]
    DOCS[[D1 ... D5]]

    PROMPT[Build Prompt<br/>Question + Context]

    LLM[LLM]

    ANSWER([Answer])

    U -->|Question| APP

    APP -->|Query| DB
    DB -->|Retrieved Data| DOCS
    DOCS --> APP

    APP --> PROMPT
    PROMPT --> LLM

    LLM --> ANSWER
    ANSWER --> U


Your model is only as good as your retrieval, so search quality matters a lot for RAG. We use minsearch and then sqlitesearch for search, and OpenAI for the LLM. Because each piece is independent, RAG stays flexible.

In our case, the data is already prepared. I maintain this FAQ website and made sure the data comes back in a convenient JSON format. So we don't need to do much to get it ready. By the way, I cleaned a lot of this data with the help of an LLM. That's a handy use case on its own.

Don't let that fool you. In reality, data preparation is often the most time-consuming part of building a RAG system. You may need to scrape websites, parse PDFs, and clean and chunk documents. That work isn't visible here, but I did plenty of it ahead of time.

## Search basics

At its core, every search engine does the same thing. It takes a query,
scores every document for similarity, and returns the top results.

Formally, there is a similarity function:

```python
score = sim(query, document)
```

For each document in the database, you compute this score. Then you
rank all documents by score and return the top N. What makes a search
engine different from another search engine is what `sim` actually
computes.

- text/lexical search (covered in this section): `sim` counts how
  many words the query and the document share. It looks at the surface
  form, the actual words, and matches them exactly.

- vector/semantic search (covered in [module 2](../../02-vector-search/)):
  `sim` compares the meaning of the query and the document. Same
  function, different similarity measure.


  ## Indexing with minsearch

We already have the `documents` list from the previous section. Now
let's index it.

Searching matters because we have around 1100 documents. Sending all
of them to the LLM would be expensive and slow. The model would get
confused with too much data. Search finds the most relevant documents
to send instead.

There are many search libraries you can use - Apache Lucene,
Elasticsearch, Solr, and others. But these are somewhat heavy. For
example, to run Elasticsearch, you need to start a Docker container.

[minsearch](https://github.com/alexeygrigorev/minsearch) is a simple
in-memory search engine. It's lightweight, so it runs anywhere Python
runs, including Google Colab where you can't start a Docker container.
It's a toy implementation, not production ready, but it illustrates how
search engines work and it gives good results.

The concepts in minsearch are the same as in Elasticsearch (which
comes from Lucene): text fields, keyword fields, boosting, filtering. I
borrowed those terms from Elasticsearch on purpose, since I wanted a
lightweight stand-in for it. So what you learn here transfers directly.
The index tokenizes text fields and treats keyword fields as exact strings.

Text fields are the fields you search through. When you type a query, the search engine looks at these fields and tokenizes them. It splits text into words, lowercases them, removes stop words, and ranks the results by relevance. So question, section, and answer are text fields.

Keyword fields are for exact matching. 

The LLM doesn't see our documents unless we pass them in. So we need to build a prompt that includes the user's question and the search results.

When we build AI systems, we usually split the prompt into two parts:

Instructions (also called the system prompt): this tells the LLM how to behave. It never changes, so it's the same for every request.
User prompt: this changes with every request. It carries the actual question and the retrieved context.
We split them because the instructions are fixed and the user prompt is not. Keeping them apart makes the fixed part easy to reuse and the changing part easy to build fresh each time.
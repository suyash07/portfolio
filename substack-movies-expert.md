# I Built a Chatbot That Knows Every Line from Star Wars, Deadpool, and Cars 2

**Here's how Retrieval-Augmented Generation (RAG) actually works - and how I built it from scratch using LangChain, Qdrant, and GPT-4o.**

---

I've been meaning to learn RAG for a while. It's one of those techniques that sounds complicated but makes a lot of sense once you build it yourself. So instead of reading another tutorial, I built a chatbot that can answer questions from real movie scripts - Star Wars: A New Hope, The Empire Strikes Back, Return of the Jedi, Deadpool, and Cars 2.

Ask it "How does Darth Vader reveal he is Luke's father?" and it pulls the actual screenplay text and answers from it. Not from GPT's general knowledge - from the script itself.

Here's how I built it.

---

## What is RAG, anyway?

A plain LLM like GPT-4o is trained on a huge amount of text, but it doesn't "know" your specific documents. If you ask it about a private report, a database, or even the exact wording of a movie script, it either makes something up or gives you a generic answer.

RAG (Retrieval-Augmented Generation) fixes this by adding a retrieval step before the LLM responds:

1. Your documents are chunked into pieces and converted into vector embeddings (numerical representations of meaning)
2. Those embeddings are stored in a vector database
3. When a user asks a question, the question is also embedded
4. The most semantically similar chunks are retrieved from the database
5. Those chunks are passed as context to the LLM along with the question
6. The LLM synthesises an answer grounded in your actual documents

The result: answers that are accurate, specific, and traceable back to a source.

**The pipeline in one line:**
`User Question → Embed → Vector Search → Retrieve Chunks → LLM → Answer`

---

## The Tech Stack

- **LangChain** - orchestration framework that wires everything together
- **Qdrant** - vector database that stores and searches the embeddings (I used it in local-disk mode - no server needed)
- **OpenAI** - `text-embedding-3-small` for embeddings, `gpt-4o` for generation
- **Streamlit** - the chat UI
- **BeautifulSoup + requests** - to scrape the screenplays from imsdb.com

---

## The Indexing Pipeline

The first thing the app does is scrape each screenplay, split it into chunks, and store the embeddings. This only runs once - after that, it loads from disk.

```python
# Scrape the screenplay
response = requests.get(url)
soup = BeautifulSoup(response.content, 'html.parser')
script_raw = soup.find('pre').get_text()

# Chunk it - with scene-aware separators
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\nINT.", "\nEXT.", "\n\n", "\n", " "],
)
chunks = splitter.split_documents([doc])

# Embed and store in Qdrant
vector_store = QdrantVectorStore.from_documents(
    chunks,
    path="./qdrant_db",
    collection_name="movie_scripts",
    embedding=OpenAIEmbeddings(model="text-embedding-3-small"),
)
```

One thing I spent time on: the separators. Screenplays use `INT.` and `EXT.` to mark scene headings. By adding those as high-priority split points, chunks tend to start at scene boundaries instead of cutting mid-dialogue. That makes retrieved chunks much more coherent.

---

## The RAG Chain

Once the vector store is ready, the RAG chain is surprisingly clean using LangChain's LCEL (LangChain Expression Language):

```python
retriever = vector_store.as_retriever(search_kwargs={"k": 15})

rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | ChatPromptTemplate.from_template(template)
    | ChatOpenAI(model="gpt-4o", temperature=0)
    | StrOutputParser()
)
```

The `|` operator composes each step into a single invokable chain. When `rag_chain.invoke("How does Yoda train Luke?")` is called, it automatically retrieves the top 15 script chunks and passes them to GPT-4o as context.

The system prompt also instructs the model to only answer from the provided excerpts - if the answer isn't in the scripts, it says so. This stops GPT-4o from drawing on its general Star Wars knowledge and hallucinating quotes.

---

## The UI

The Streamlit interface has a Star Wars-themed dark design (Orbitron font, gold accents) with example question buttons so you can explore without typing anything. The chain is cached with `@st.cache_resource` so it doesn't reload on every user interaction.

**Demo 1 - Star Wars questions:**
https://youtu.be/BxvPyk5-62E

**Demo 2 - Deadpool and Cars 2:**
https://youtu.be/4T27Wll6Z-4

---

## What I Actually Learned

A few things that weren't obvious before building this:

**Chunking strategy matters more than you'd think.** Naive character splits cut through dialogue mid-line. Thinking about the structure of your source document and using domain-appropriate separators makes retrieval significantly better.

**`k` is a real trade-off.** Retrieving more chunks gives the LLM more to work with, but it also introduces noise. At `k=15` the answers were noticeably better than at `k=5`, but going higher started adding irrelevant context.

**Local Qdrant is underrated.** Running a production-grade vector database with zero infrastructure - just a local folder on disk - was a great way to iterate fast. No Docker, no cloud setup, no config files.

**LCEL is genuinely elegant.** Composing the retrieval, prompt, LLM, and parser as a single pipe expression is clean, readable, and lazy-evaluated. Much nicer than the older LangChain chain syntax.

---

## What's Next

A few things I want to try:
- Streaming the LLM response token-by-token in Streamlit
- Adding more films
- Experimenting with re-ranking retrieved chunks (cohere reranker or cross-encoder) before passing them to the prompt

---

Full write-up with code and library breakdown: https://suyash07.github.io/portfolio/blog-movies-expert.html

GitHub: https://github.com/suyash07/movie-expert

This project was inspired by [Star Wars Movie Expert](https://github.com/andrisgauracs/Star-Wars-Movie-Expert) by Andris Gauracs.

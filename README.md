## 運行方法：

This project uses **`uv`** for lightning-fast, reproducible dependency management.

### 1. Clone the Repository
```bash
git clone [https://github.com/Jimmychuyucheng/GenAI_RAG-Lab.git](https://github.com/Jimmychuyucheng/GenAI_RAG-Lab.git)
cd GenAI_RAG-Lab
```

### 2. Set Up Virtual Environment & Dependencies
```bash
Create venv and lock dependencies automatically
uv venv
uv add sentence-transformers chromadb google-genai python-doten
```

### 3. Configure Environment Variables
Create a .env file in the root directory and add your Google AI Studio API Key:
```bash
GEMINI_API_KEY=your_actual_api_key_here
```

### 4. Run the Pipeline
Open main.ipynb in VS Code, select the .venv kernel, and run all cells to see the Two-Stage RAG in action!

---

## 程式碼運作流程
### 1. Document Chunking（文本切塊）
這是 RAG 的起點。大模型（LLM）有 token 長度限制，且過長的文本會稀釋關鍵資訊，因此必須先將大文件切碎。

實作方式：採用「語意段落切塊」，以兩個換行符號 \n\n 作為切分依據。
程式碼落點：split_into_chunks("doc.md")

流程解析：
這個做法非常適合結構清晰、段落分明的文件。在輸出中，故事被切成了 24 個獨立的段落（Chunks），每個段落都保留了完整的上下文語意。


### 2. Embedding Generation（向量化）
文字是無法直接進行數學運算與比對的，必須轉換成高維度的空間向量。

實作方式：使用 SentenceTransformer 載入開源的中文文本向量模型 "shibing624/text2vec-base-chinese"。
程式碼落點：embed_chunk(chunk)

流程解析：
模型會將每個 Chunk 轉換成一個長度為 768 維 的浮點數陣列（Vector）。在這個向量空間中，語意越接近的文字，它們的向量距離就會越近。


### 3. Vector Storage（向量資料庫儲存）
為了讓系統能在使用者提問時「瞬間」找到相關文本，我們需要一個專門儲存向量的資料庫。

實作方式：使用 chromadb 建立了一個記憶體形式的暫存資料庫（EphemeralClient）。
程式碼落點：save_embeddings(chunks, embeddings)

流程解析：
將「原始文字（documents）」、「對應的 768 維向量（embeddings）」以及「流水號 ID（ids）」打包寫入名為 "default" 的 Collection 中。這就完成了 RAG 的知識庫初始化。


### 4. Hybrid Retrieval & Reranking（檢索與重排機制）
當使用者輸入問題時，系統必須去知識庫撈出最相關的片段。這是你程式碼中最漂亮的一部分，包含兩階段過濾：

#### 階段 A：初篩 (Retrieve)
作法：把使用者的 Query 用同一個模型轉成向量，去 ChromaDB 計算餘弦相似度（Cosine Similarity），撈出前 5 個最相關的 Chunk（top_k=5）。
落點：retrieve(query, 5)

#### 階段 B：精篩重排 (Rerank)
作法：初篩的向量檢索只比對「關鍵字與語意表面」，容易有雜訊。引入了 CrossEncoder 模型 'cross-encoder/mmarco-mMiniLMv2-L12-H384-v1'。
落點：rerank(query, retrieved_chunks, 3)

技術原理：
Cross-Encoder 會把 (Query, Chunk) 兩兩配對同時輸入模型，強行計算它們深層的因果與邏輯相關性，並給出精準評分。最後排序並只取前 3 個最精準的 Chunk（top_k=3）。這能大幅提升餵給大模型的資料品質，減少幻覺。


### 5. LLM Generation（大模型生成答案）
最後一步，把過濾乾淨的黃金片段，結合 Prompt 送給 Gemini 進行統整生成。

實作方式：利用 google-genai SDK，調用 gemini-2.5-flash 模型。
程式碼落點：generate(query, reranked_chunks)

最終 Prompt 結構：
```
你是一個知識助手，根據用戶的問題和下列片段生成準確的的答案
用戶問題: {query}
相關片段: {第[0],[1],[2]個重排後的黃金段落}
基於上述內容回答，不要編造。
```

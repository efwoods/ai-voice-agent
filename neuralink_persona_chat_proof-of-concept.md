// Project: Neuralink Persona Chat PoC

// ---- backend/app.py ----
from fastapi import FastAPI, UploadFile, File
from fastapi.middleware.cors import CORSMiddleware
from langchain.document_loaders import TextLoader
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.text_splitter import CharacterTextSplitter
from langchain.vectorstores import Chroma
from langchain.chains import RetrievalQA
from transformers import pipeline
import uvicorn

app = FastAPI()
app.add_middleware(CORSMiddleware, allow_origins=\["_"], allow_methods=\["_"], allow_headers=\["\*"])

# Initialize in-memory vector store and LLM

embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2") # free local embedding model
vector_store = Chroma(persist_directory=None, embedding_function=embeddings)

generator = pipeline("text-generation", model="gpt2", device=-1)

qa_chain = None

def init_qa_chain():
global qa_chain
llm = lambda prompt: generator(prompt, max_length=100)\[0]\["generated_text"]
qa_chain = RetrievalQA.from_chain_type(
llm=llm,
chain_type="stuff",
retriever=vector_store.as_retriever(search_kwargs={"k": 3})
)

@app.post("/upload")
async def upload_document(file: UploadFile = File(...)):
content = await file.read()
text = content.decode('utf-8', errors='ignore')
loader = TextLoader(file_path=None, content=text)
docs = loader.load()
splitter = CharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = splitter.split_documents(docs)
vector_store.add_documents(chunks)
init_qa_chain()
return {"status": "documents indexed", "count": len(chunks)}

@app.post("/chat")
async def chat(question: str):
if qa_chain is None:
return {"error": "No documents uploaded."}
answer = qa_chain.run(question)
return {"answer": answer}

if **name** == "**main**":
uvicorn.run(app, host="0.0.0.0", port=8000)

// ---- frontend/src/App.js ----
import React, { useState, useRef, useEffect } from 'react';
import LiveKitRoom from './LiveKitRoom'; // see LiveKitRoom.js

function App() {
const \[logs, setLogs] = useState(\[]);
const \[input, setInput] = useState("");
const fileInputRef = useRef();

// keyboard shortcut: 'u' to open file dialog
useEffect(() => {
const handler = (e) => {
if (e.key === 'u') {
fileInputRef.current.click();
}
};
window\.addEventListener('keydown', handler);
return () => window\.removeEventListener('keydown', handler);
}, \[]);

const uploadFile = async (e) => {
const file = e.target.files\[0];
const form = new FormData();
form.append('file', file);
const res = await fetch('[http://localhost:8000/upload](http://localhost:8000/upload)', { method: 'POST', body: form });
const json = await res.json();
setLogs(prev => \[...prev, `Indexed ${json.count} chunks`]);
};

const sendMessage = async () => {
setLogs(prev => \[...prev, `You: ${input}`]);
const res = await fetch('[http://localhost:8000/chat](http://localhost:8000/chat)', {
method: 'POST', headers: {'Content-Type': 'application/json'},
body: JSON.stringify({ question: input })
});
const json = await res.json();
setLogs(prev => \[...prev, `AI: ${json.answer}`]);
setInput('');
};

return ( <div className="p-4"> <h1 className="text-xl font-bold">Neuralink Persona Chat PoC</h1>
\<button onClick={() => fileInputRef.current.click()} className="p-2 border rounded">Upload Document (or press 'u')</button> <input ref={fileInputRef} type="file" className="hidden" onChange={uploadFile} /> <div className="mt-4 h-64 overflow-y-auto border p-2">
{logs.map((l,i) => <div key={i}>{l}</div>)} </div> <div className="mt-2 flex">
\<input value={input} onChange={e => setInput(e.target.value)} onKeyDown={e => e.key==='Enter'&& sendMessage()} className="flex-grow border p-2" placeholder="Ask a question..." /> <button onClick={sendMessage} className="ml-2 p-2 border rounded">Send</button> </div> <LiveKitRoom /> </div>
);
}
export default App;

// ---- frontend/src/LiveKitRoom.js ----
import React, { useEffect, useState } from 'react';
import { Room, connect } from 'livekit-client';

const token = 'REPLACE_WITH_ACCESS_TOKEN';
const url = 'wss\://livekit.example.com';

export default function LiveKitRoom() {
const \[room, setRoom] = useState(null);

useEffect(() => {
connect(url, token, { autoSubscribe: true }).then(rm => {
setRoom(rm);
});
}, \[]);

return ( <div className="mt-4"> <h2 className="text-lg">LiveKit Room</h2> <div id="video-container" className="grid grid-cols-2 gap-2">
{room && Array.from(room.participants.values()).map(p => ( <div key={p.sid} className="border">
{Array.from(p.tracks.values()).map(track => \<track.render key={track.sid} />)} </div>
))} </div> </div>
);
}

// ---- Instructions ----
// 1. Start backend: python3 -m pip install fastapi uvicorn langchain chromadb transformers
// then `python backend/app.py`
// 2. Start frontend: npm install react livekit-client && npm start
// 3. Obtain LiveKit Access Token and set in LiveKitRoom.js
// 4. Use 'u' to upload docs, type questions and press Enter to chat.

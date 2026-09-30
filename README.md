# import io, os
from fastapi import FastAPI, HTTPException
from fastapi.responses import HTMLResponse, StreamingResponse
from pydantic import BaseModel
from docx import Document
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from dotenv import load_dotenv

load_dotenv()
app = FastAPI(title="Legalease - AI Legal Document Generator")

class DocRequest(BaseModel):
    doc_type: str
    jurisdiction: str
    party_a: str
    party_b: str
    custom_clauses: str = ""

SYSTEM_PROMPT = """You are an expert legal counsel. Draft a formal {doc_type} governed by the laws of {jurisdiction}.
Parties involved:
- Party A: {party_a}
- Party B: {party_b}

Additional Terms / Requirements:
{custom_clauses}

Format guidelines: Include standard legal document headers, recitals, numbered clauses, and signature blocks. Output formal legal text only without introductory or concluding conversational prose."""

# --- Frontend HTML UI ---
HTML_UI = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Legalease - Single File App</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: #0f172a; color: #f8fafc; margin: 0; padding: 2rem; }
        .container { display: flex; gap: 2rem; max-width: 1200px; margin: 0 auto; }
        .panel { flex: 1; background: #1e293b; padding: 1.5rem; border-radius: 8px; border: 1px solid #334155; }
        h2 { margin-top: 0; color: #818cf8; }
        label { display: block; margin-top: 1rem; font-size: 0.85rem; color: #94a3b8; }
        input, textarea { width: 100%; box-sizing: border-box; padding: 0.6rem; background: #0f172a; border: 1px solid #334155; border-radius: 4px; color: #fff; margin-top: 0.25rem; }
        button { margin-top: 1.2rem; width: 100%; padding: 0.75rem; background: #4f46e5; color: white; border: none; border-radius: 4px; font-weight: bold; cursor: pointer; }
        button:disabled { opacity: 0.5; cursor: not-allowed; }
        #preview { background: #ffffff; color: #0f172a; padding: 1.5rem; border-radius: 4px; min-height: 450px; white-space: pre-wrap; font-family: "Georgia", serif; font-size: 0.9rem; line-height: 1.5; overflow-y: auto; max-height: 500px; }
        .header-actions { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem; }
        .export-btn { background: #059669; width: auto; padding: 0.4rem 0.8rem; margin: 0; font-size: 0.85rem; }
    </style>
</head>
<body>
    <div class="container">
        <div class="panel">
            <h2>Legalease Generator</h2>
            <form id="docForm">
                <label>Document Type</label>
                <input type="text" id="doc_type" value="Non-Disclosure Agreement (NDA)" required />
                <label>Jurisdiction</label>
                <input type="text" id="jurisdiction" value="California, USA" required />
                <label>Party A (Disclosing / Employer)</label>
                <input type="text" id="party_a" placeholder="Company Inc." required />
                <label>Party B (Receiving / Employee)</label>
                <input type="text" id="party_b" placeholder="John Doe" required />
                <label>Custom Terms / Clauses</label>
                <textarea id="custom_clauses" rows="4" placeholder="e.g. Non-compete for 2 years, payment within 30 days..."></textarea>
                <button type="submit" id="submitBtn">Generate Document</button>
            </form>
        </div>
        <div class="panel">
            <div class="header-actions">
                <h2 style="margin:0; color:#f8fafc;">Document Preview</h2>
                <button id="exportBtn" class="export-btn" style="display:none;" onclick="exportDocx()">Export .DOCX</button>
            </div>
            <div id="preview">Fill in the fields on the left and click "Generate Document" to construct your draft.</div>
        </div>
    </div>

    <script>
        let currentDocText = "";

        document.getElementById('docForm').addEventListener('submit', async (e) => {
            e.preventDefault();
            const submitBtn = document.getElementById('submitBtn');
            const preview = document.getElementById('preview');
            const exportBtn = document.getElementById('exportBtn');
            
            submitBtn.disabled = true;
            submitBtn.innerText = "Drafting Document...";
            preview.innerText = "Generating legal document via AI...";

            const payload = {
                doc_type: document.getElementById('doc_type').value,
                jurisdiction: document.getElementById('jurisdiction').value,
                party_a: document.getElementById('party_a').value,
                party_b: document.getElementById('party_b').value,
                custom_clauses: document.getElementById('custom_clauses').value
            };

            try {
                const res = await fetch('/api/generate', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });
                const data = await res.json();
                if (data.content) {
                    currentDocText = data.content;
                    preview.innerText = data.content;
                    exportBtn.style.display = 'block';
                } else {
                    preview.innerText = "Error: " + (data.detail || "Failed to generate document.");
                }
            } catch (err) {
                preview.innerText = "Server Error: Could not connect to API.";
            } finally {
                submitBtn.disabled = false;
                submitBtn.innerText = "Generate Document";
            }
        });

        async function exportDocx() {
            const res = await fetch('/api/export/docx', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ content: currentDocText })
            });
            const blob = await res.blob();
            const url = window.URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'Legal_Document.docx';
            a.click();
        }
    </script>
</body>
</html>
"""

# --- API Endpoints ---
@app.get("/", response_class=HTMLResponse)
async def serve_ui():
    return HTML_UI

@app.post("/api/generate")
async def generate_doc(req: DocRequest):
    if not os.getenv("OPENAI_API_KEY"):
        raise HTTPException(status_code=500, detail="OPENAI_API_KEY environment variable is not configured.")
    try:
        llm = ChatOpenAI(model="gpt-4o", temperature=0.2)
        prompt = ChatPromptTemplate.from_template(SYSTEM_PROMPT)
        chain = prompt | llm
        res = await chain.ainvoke(req.model_dump())
        return {"content": res.content}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/export/docx")
async def export_docx(data: dict):
    content = data.get("content", "")
    if not content:
        raise HTTPException(status_code=400, detail="No content provided for export.")
    
    doc = Document()
    for paragraph in content.split("\n\n"):
        doc.add_paragraph(paragraph)
        
    stream = io.BytesIO()
    doc.save(stream)
    stream.seek(0)
    
    headers = {'Content-Disposition': 'attachment; filename="Legal_Document.docx"'}
    return StreamingResponse(
        stream,
        media_type="application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        headers=headers
    )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
-AI-powered-legal-document-generator
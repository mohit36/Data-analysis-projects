Phase 1: The Database (The Only Source of Truth)
You need to add a logs table and ensure your chat_history is ready. Run this SQL.

SQL

-- 1. Chat History (Solves Session RAM)
CREATE TABLE IF NOT EXISTS chat_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id TEXT NOT NULL,
    role TEXT CHECK (role IN ('user', 'assistant')),
    content TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_session ON chat_history(session_id);

-- 2. System Logs (Solves "Logging in DB")
CREATE TABLE IF NOT EXISTS system_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    level TEXT,
    module TEXT,
    message TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
Phase 2: The "Stateless" Button Logic (Solves the Mapping Mess)
Stop using service_mapping_store. It is over-engineered.

The Old Way (Your RAM Trap):

Bot saves Key: "Contact Option" -> Value: "How do I contact sales?" in RAM.

Bot sends Key to Frontend.

User clicks Key.

Bot looks up Value. (Fails if server restarted).

The New Way (The Payload Pattern):

Bot sends the Full Payload to the Frontend.

Frontend sends the Full Payload back when clicked.

Zero Memory Needed. The button contains the question.

Phase 3: The Code Implementation (Clean & Organized)
We will create three clean files. Delete the old spaghetti code.

File 1: logger_config.py (Centralized DB Logging)
This writes logs to your DB automatically.

Python

import logging
import asyncio
from pgvector_helper.pgvector_client_connection import get_pgvector_client

class DBHandler(logging.Handler):
    def emit(self, record):
        log_entry = self.format(record)
        # Fire and forget async insert to avoid slowing down the app
        asyncio.create_task(self.write_to_db(record.levelname, record.module, log_entry))

    async def write_to_db(self, level, module, message):
        client = await get_pgvector_client()
        try:
            await client.execute(
                "INSERT INTO system_logs (level, module, message) VALUES ($1, $2, $3)",
                level, module, message
            )
        except Exception:
            pass # Never crash because of logging

def setup_logger(name):
    logger = logging.getLogger(name)
    logger.setLevel(logging.INFO)
    
    # Add DB Handler
    db_handler = DBHandler()
    db_handler.setFormatter(logging.Formatter('%(message)s'))
    logger.addHandler(db_handler)
    
    # Add Console Handler (for debugging)
    console_handler = logging.StreamHandler()
    logger.addHandler(console_handler)
    
    return logger
File 2: chatbot_api.py (The Clean Logic)
This replaces your summarize function. It handles the "Contact Nudge" using the DB count, not RAM.

Python

from logger_config import setup_logger
from database_operations import save_message, get_recent_history, get_msg_count

logger = setup_logger(__name__)

@app.post("/chat")
async def chat_endpoint(request: Request):
    payload = await request.json()
    session_id = payload.get("session_id")
    user_query = payload.get("query")

    # 1. Save User Message (DB)
    await save_message(session_id, "user", user_query)

    # 2. Get Context (DB)
    history = await get_recent_history(session_id)

    # 3. Rewriter & Search (Your Pipeline)
    # Note: We skip the "Mapping Lookup" because the UI sends the full question now.
    contextualized = await contextualize_query(user_query, history)
    
    if "NO_SEARCH" in contextualized:
        search_results = []
    else:
        search_results = await hybrid_query_search(contextualized)

    # 4. Generate Answer
    bot_answer = await generate_answer(search_results, user_query)

    # 5. Save Bot Message (DB)
    await save_message(session_id, "assistant", bot_answer)

    # 6. The "Smart Nudge" Logic (Replaces your len > 4 check)
    # Check DB for how many messages this user has sent
    msg_count = await get_msg_count(session_id)
    
    suggested_actions = []
    
    # If conversation is long (> 4 turns) AND they haven't asked for contact yet:
    if msg_count > 4 and "contact" not in contextualized.lower():
        suggested_actions.append({
            "label": "Talk to an Expert",
            "payload": "How can I contact the Exigotech Sales Team?" 
            # ^^^ The Payload IS the question. No mapping needed.
        })

    # 7. Return Structured JSON (Frontend renders buttons from 'options')
    return {
        "response": bot_answer,
        "options": suggested_actions
    }

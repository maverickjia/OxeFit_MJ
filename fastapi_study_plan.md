# FastAPI Study Plan for Speech-to-Text Feature

**Total Time: ~2 hours**
**Goal:** Learn enough FastAPI to implement a speech-to-text endpoint in this codebase

---

## Part 1: FastAPI Fundamentals (30 min)

### 1.1 What is FastAPI? (5 min)

FastAPI is a modern Python web framework for building APIs. Key features:
- **Fast**: High performance, on par with NodeJS and Go
- **Easy**: Designed to be easy to use and learn
- **Automatic docs**: Generates OpenAPI documentation automatically
- **Type hints**: Uses Python type hints for validation

### 1.2 Core Concepts (15 min)

#### Creating an App
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

#### Route Decorators
- `@app.get("/path")` - Read data
- `@app.post("/path")` - Create data
- `@app.put("/path")` - Update data
- `@app.delete("/path")` - Delete data

#### Path and Query Parameters
```python
# Path parameter: /items/42
@app.get("/items/{item_id}")
async def get_item(item_id: int):
    return {"item_id": item_id}

# Query parameter: /items?skip=10&limit=20
@app.get("/items")
async def list_items(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}
```

#### Request Body with Pydantic
```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    description: str | None = None

@app.post("/items")
async def create_item(item: Item):
    return item
```

#### File Uploads (Critical for Speech-to-Text!)
```python
from fastapi import UploadFile

@app.post("/upload")
async def upload_file(file: UploadFile):
    contents = await file.read()
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents)
    }
```

### 1.3 Dependency Injection (10 min)

FastAPI uses `Depends()` to inject shared logic into endpoints:

```python
from fastapi import Depends

# A dependency function
def get_current_user(token: str):
    # Validate token, return user
    return {"user_id": 123}

# Using the dependency
@app.get("/me")
async def read_me(user = Depends(get_current_user)):
    return user
```

**Why it matters:** This codebase uses dependencies heavily for:
- Authentication (`auth_context = Depends(oxeai_authorization)`)
- Services (`service = Depends(get_file_service)`)

---

## Part 2: This Codebase Patterns (45 min)

### 2.1 Project Structure Overview (5 min)

```
src/
├── web/main.py              # FastAPI app entry point
├── application/lifespan.py  # Startup/shutdown logic
├── interfaces/http/routers/ # API endpoints (routes)
├── domain/                  # Business logic (services)
└── infrastructure/auth/     # Authentication
```

**Key insight:** Routes call Services, Services contain business logic.

### 2.2 Study: Main App Setup (10 min)

**File to read:** `src/web/main.py`

Key patterns to notice:
1. App creation with lifespan manager
2. Router inclusion with `app.include_router()`
3. Middleware setup (CORS, logging)

```python
# Simplified version of what you'll see:
from fastapi import FastAPI
from application.lifespan import oxeai_lifespan

app = FastAPI(lifespan=oxeai_lifespan)

# Include routers
app.include_router(files_router)
app.include_router(sessions_router)
```

### 2.3 Study: File Upload Router (15 min) - YOUR KEY REFERENCE

**File to read:** `src/interfaces/http/routers/files.py`

This is your most important reference! It shows:
- How to handle file uploads (audio files for speech-to-text)
- How to use authentication
- How to call services

```python
# Pattern you'll follow:
@files_router.post("/", status_code=201, response_model=OxeAIFileRef)
async def upload_file(
    file: UploadFile,                                          # File input
    file_service: OxeAIFileService = Depends(get_file_service), # Service injection
    auth_context: AuthContext = Depends(oxeai_authorization),   # Auth
) -> OxeAIFileRef:
    # Call service method
    return await file_service.upload_file(...)
```

### 2.4 Study: Service Pattern (10 min)

**File to read:** `src/files/services.py`

Services contain the actual logic:

```python
class OxeAIFileService:
    def __init__(self, file_repository: OxeAIFileRepository):
        self.file_repository = file_repository

    async def upload_file(self, file_type, uploaded_file, session_id, auth_context):
        # Business logic here
        return await self.file_repository.store_file(...)

# Factory function for dependency injection
def get_file_service(request: Request) -> OxeAIFileService:
    return OxeAIFileService(request.app.state.file_repository)
```

### 2.5 Authentication Pattern (5 min)

Every protected endpoint uses:
```python
from infrastructure.auth import oxeai_authorization
from oxefit_auth import AuthContext

@router.post("/")
async def my_endpoint(
    auth_context: AuthContext = Depends(oxeai_authorization),  # Required!
):
    user_id = auth_context.user_id  # Get current user
```

---

## Part 3: Hands-On Practice (45 min)

### Exercise 1: Read the Key Files (15 min)

Open and read these files in order:

1. **`src/web/main.py`** - See how the app is structured
2. **`src/interfaces/http/routers/files.py`** - See file upload pattern
3. **`src/files/services.py`** - See service pattern

Questions to answer:
- [ ] How is a router created and added to the app?
- [ ] What does `Depends()` do?
- [ ] How is `UploadFile` used?

### Exercise 2: Trace a Request (10 min)

Trace what happens when someone uploads a file:

1. Request hits `POST /Files/`
2. Router function `upload_file()` is called
3. Dependencies are resolved (`file_service`, `auth_context`)
4. Service method is called
5. Response is returned

### Exercise 3: Sketch Your Speech-to-Text Endpoint (20 min)

Based on what you learned, sketch out what your speech-to-text feature will look like:

**Router (create new file: `src/interfaces/http/routers/speech_to_text.py`):**
```python
from fastapi import APIRouter, UploadFile, Depends
from infrastructure.auth import oxeai_authorization
from oxefit_auth import AuthContext

speech_router = APIRouter(prefix="/SpeechToText", tags=["SpeechToText"])

@speech_router.post("/transcribe")
async def transcribe_audio(
    audio_file: UploadFile,
    # Add service dependency here
    auth_context: AuthContext = Depends(oxeai_authorization),
):
    # Call your service
    pass
```

**Service (create new file: `src/domain/speech_to_text/services.py`):**
```python
from fastapi import UploadFile
from oxefit_auth import AuthContext

class SpeechToTextService:
    async def transcribe(self, audio_file: UploadFile, auth_context: AuthContext):
        # Read audio file
        audio_bytes = await audio_file.read()

        # Call speech-to-text API (Whisper, Google, etc.)
        transcription = "..."  # Your implementation

        return {"text": transcription}
```

---

## Quick Reference Card

### Creating a New Endpoint

```python
# 1. Create router
router = APIRouter(prefix="/MyFeature", tags=["MyFeature"])

# 2. Add endpoint
@router.post("/action")
async def my_action(
    input_data: MyInputModel,                    # Request body
    service: MyService = Depends(get_service),   # Service
    auth: AuthContext = Depends(oxeai_authorization),  # Auth
):
    return await service.do_something(input_data, auth)

# 3. Register in main.py
app.include_router(router)
```

### Common Imports

```python
from fastapi import APIRouter, UploadFile, Depends, Query, Path
from starlette.requests import Request
from pydantic import BaseModel
from oxefit_auth import AuthContext
from infrastructure.auth import oxeai_authorization
```

### File Upload Pattern

```python
@router.post("/upload")
async def upload(file: UploadFile):
    contents = await file.read()      # Read file bytes
    filename = file.filename          # Original filename
    content_type = file.content_type  # MIME type (e.g., "audio/wav")
```

---

## Checklist Before You Start Coding

- [ ] I understand how routers work (`APIRouter`, decorators)
- [ ] I understand dependency injection (`Depends()`)
- [ ] I understand file uploads (`UploadFile`)
- [ ] I've read `files.py` router as my reference
- [ ] I've read `files/services.py` as my service reference
- [ ] I know where to register my new router (`main.py`)

---

## Next Steps After This Study Plan

1. **Decide on speech-to-text provider** (OpenAI Whisper, Google Speech-to-Text, etc.)
2. **Create directory structure:**
   ```
   src/domain/speech_to_text/
   ├── __init__.py
   ├── services.py
   └── models.py  (if needed)

   src/interfaces/http/routers/speech_to_text.py
   ```
3. **Implement service with your chosen provider**
4. **Create router endpoint**
5. **Register router in `main.py`**
6. **Test with audio file upload**

---

## Useful Resources

- [FastAPI Official Tutorial](https://fastapi.tiangolo.com/tutorial/) - Official docs (excellent)
- [FastAPI File Uploads](https://fastapi.tiangolo.com/tutorial/request-files/) - Specific to your use case
- [Pydantic Docs](https://docs.pydantic.dev/) - Data validation

---

*Study plan created for the OxeAI API codebase. Focus on `files.py` as your primary reference for implementing speech-to-text!*

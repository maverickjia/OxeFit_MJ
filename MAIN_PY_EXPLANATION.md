# Explanation of `src/web/main.py` for Beginners

## Overview
This file is the **main entry point** for the OxeAI web API. Think of it as the "front door" of the application - it sets up the web server and defines how the application responds to HTTP requests.

---

## What is FastAPI?
**FastAPI** is a Python web framework for building APIs (Application Programming Interfaces). An API is like a menu at a restaurant - it tells clients (like mobile apps or web browsers) what "dishes" (endpoints) are available and how to order them.

---

## Section-by-Section Breakdown

### 1. Imports (Lines 1-37)
These lines bring in code from other files that this file needs to use.

**Key imports explained:**
- `FastAPI` - The main framework for building the web server
- `StaticFiles` - Allows serving static files (like images, CSS) directly
- `oxefit_error_handler` and error classes - Custom error handling for the API
- `CORSMiddleware` - Allows the API to be accessed from web browsers (handles cross-origin requests)
- `oxeai_lifespan` - A function that runs when the app starts up and shuts down (sets up database connections, etc.)
- Router imports - These are like "sub-menus" that group related endpoints together

### 2. Logging Configuration (Line 39)
```python
configure_logging(settings.logging)
```
This sets up logging so the application can record what's happening (errors, info messages, etc.).

### 3. API Tags Metadata (Lines 42-59)
```python
tags_metadata = [...]
```
This defines how the API documentation (Swagger UI) will be organized. Tags are like categories that group related endpoints together in the documentation.

**The tags defined:**
- `OxeAIChatSessions` - Main chat functionality
- `OxeAIFiles` - File upload and management
- `langgraph-chat-internal` - Internal testing endpoints (deprecated)
- `memory-test` - Memory system testing

### 4. Creating the FastAPI App (Line 61)
```python
app = FastAPI(lifespan=oxeai_lifespan, openapi_tags=tags_metadata)
```
This creates the main application object. The `lifespan` parameter tells FastAPI what to do when the app starts and stops.

### 5. Static Files Setup (Lines 64-68)
```python
static_path = Path("static")
static_path.mkdir(exist_ok=True)
app.mount("/static", StaticFiles(directory="static"), name="static")
```
This sets up a way to serve static files (like images, charts) from the `/static` URL path. If the directory doesn't exist, it creates it.

### 6. Exception Handler (Line 69)
```python
app.add_exception_handler(Exception, oxefit_error_handler)
```
This tells FastAPI: "If any error occurs anywhere in the application, use this custom handler to format the error response." This ensures all errors are returned in a consistent format.

### 7. CORS Middleware (Lines 72-78)
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    ...
)
```
**CORS** (Cross-Origin Resource Sharing) allows web browsers to make requests to this API from different domains. The `allow_origins=["*"]` means "allow requests from any origin" (useful for development, but you'd restrict this in production).

**What is middleware?** Middleware is code that runs on every request, before it reaches your endpoint. It's like a security guard that checks every visitor.

### 8. Logging Middleware (Line 79)
```python
app.add_middleware(ECSLoggingMiddleware)
```
This middleware logs every request in a structured format (ECS = Elastic Common Schema).

### 9. Including Routers (Lines 84-95)
```python
app.include_router(sessions_router)
app.include_router(prompt_router)
...
```
These lines "plug in" different routers (groups of endpoints) into the main application. Think of routers as separate modules that handle different parts of the API:

- `sessions_router` - Manages chat sessions
- `prompt_router` - Main chat API endpoint (`/Sessions/Current/Prompt`)
- `files_router` - Handles file uploads
- `langgraph_router` - Internal testing (deprecated)
- `memory_debug_router` - Memory system debugging
- `journal_router` - Journal entries
- `trackers_router` - Health/fitness trackers
- `surveys_router` - Surveys
- `goals_router` - User goals
- `plans_router` - Plans

### 10. Health Check Endpoint (Lines 98-101)
```python
@app.get("/health", include_in_schema=False)
async def health():
    return "Healthy"
```
This is a simple endpoint that returns "Healthy" when accessed. It's used by monitoring systems to check if the API is running. The `include_in_schema=False` means it won't show up in the API documentation.

**What is an endpoint?** An endpoint is a specific URL path (like `/health`) that responds to HTTP requests (GET, POST, etc.).

### 11. Error Testing Endpoint (Lines 104-129)
```python
@app.get("/error", include_in_schema=False)
async def error(error_type: str):
    ...
```
This is a **testing/debugging endpoint** that allows developers to trigger different types of errors to test error handling. It's not meant for production use.

**What it does:**
- Takes an `error_type` parameter
- Based on the type, it raises different errors:
  - `authorization_required` - User not logged in
  - `permission_denied` - User doesn't have permission
  - `validation` - Invalid input data
  - `not_found` - Resource doesn't exist
  - `conflict` - Resource conflict (e.g., duplicate)
  - `unexpected` - Unexpected server error

### 12. Custom OpenAPI Configuration (Line 133)
```python
app.openapi = oxefit_openapi_factory(app)
```
This customizes the OpenAPI schema (the documentation that Swagger UI uses). It ensures proper handling of null values in the API documentation.

---

## Key Concepts for Beginners

### What is an API?
An API (Application Programming Interface) is a way for different applications to communicate. When you use a mobile app, it sends requests to an API server, which processes the request and sends back data.

### HTTP Methods
- **GET** - Retrieve data (like reading a book)
- **POST** - Create new data (like writing a new entry)
- **PUT/PATCH** - Update existing data
- **DELETE** - Remove data

### Request Flow
1. Client (browser/app) sends HTTP request → 
2. Middleware processes it (CORS, logging) → 
3. Router finds the right endpoint → 
4. Endpoint function executes → 
5. Response sent back to client

### What is `async def`?
The `async` keyword means the function can handle multiple requests at the same time without blocking. This makes the API faster and more efficient.

---

## Summary
This file is the "orchestrator" that:
1. Sets up the web server (FastAPI)
2. Configures error handling
3. Adds security and logging middleware
4. Connects all the different parts of the API (routers)
5. Defines a few utility endpoints (health check, error testing)

It's like the conductor of an orchestra - it doesn't play the music itself, but it coordinates all the different sections (routers) to work together.

# PocketSmart AI — Complete Source Code Bundle

Every file below is copied from the generated project. Create the same folders in VS Code and paste each section into its matching file.

# `.env.example`

```text
APP_NAME=PocketSmart AI
ENVIRONMENT=development
DEBUG=true
SECRET_KEY=replace-this-with-a-long-random-secret
DATABASE_URL=sqlite:///./pocketsmart.db
SESSION_COOKIE_NAME=pocketsmart_session
SESSION_MAX_AGE=86400
GEMINI_API_KEY=
GEMINI_MODEL=gemini-2.5-flash
CORS_ORIGINS=http://127.0.0.1:8000,http://localhost:8000
MAX_IMAGE_MB=5

```

# `.gitignore`

```text
__pycache__/
*.py[cod]
.venv/
.env
*.db
.pytest_cache/
.coverage
htmlcov/
.idea/
.vscode/
.DS_Store
uploads/*
!uploads/.gitkeep

```

# `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "PocketSmart AI",
      "type": "debugpy",
      "request": "launch",
      "module": "uvicorn",
      "args": ["app.main:app", "--reload"],
      "jinja": true
    }
  ]
}

```

# `.vscode/settings.json`

```json
{
  "python.analysis.typeCheckingMode": "basic",
  "python.testing.pytestEnabled": true,
  "python.testing.pytestArgs": ["tests"],
  "editor.formatOnSave": true
}

```

# `Dockerfile`

```text
FROM python:3.12-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN mkdir -p uploads

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

```

# `README.md`

```markdown
# PocketSmart AI

A complete FastAPI + Jinja2 + SQLite + Gemini recommendation assistant based on the supplied PocketSmart AI specification.

## What is implemented

- Home Interior Planner
- Party Budget Planner
- Jewelry Planner with optional outfit image
- Gemini integration with structured JSON output
- Deterministic mock/fallback recommendations when no Gemini API key is configured
- Registration, login, logout and JWT bearer token endpoint
- Session-aware dashboard and recommendation history
- Responsive HTML/CSS/JavaScript frontend
- SQLite persistence with SQLAlchemy
- Input validation with Pydantic
- CORS configuration
- Health/startup endpoints
- Automated pytest tests
- Dockerfile and docker-compose configuration

The source document describes Gemini 1.5 Flash Pro, while current Gemini documentation uses newer model names and the Google GenAI SDK. The model is therefore configurable through `GEMINI_MODEL`; the default is `gemini-2.5-flash`, which supports structured output and multimodal input.

## Quick start

### 1. Create a virtual environment

Windows PowerShell:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

macOS/Linux:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure environment

```bash
copy .env.example .env
```

On macOS/Linux use `cp .env.example .env`.

For a no-cost local smoke test, leave `GEMINI_API_KEY` empty. The app automatically uses deterministic fallback recommendations.

For Gemini, add your API key to `.env`:

```env
GEMINI_API_KEY=your_key_here
GEMINI_MODEL=gemini-2.5-flash
```

### 4. Run

```bash
uvicorn app.main:app --reload
```

Open http://127.0.0.1:8000

## API

- `GET /health`
- `GET /startup`
- `POST /register`
- `POST /login`
- `POST /token`
- `POST /logout`
- `GET /session-info`
- `GET /session-data`
- `POST /generate-home`
- `POST /generate-party`
- `POST /generate-jewelry`
- `GET /recommendations-details/{recommendation_id}`
- `GET /history`

The browser UI uses cookie sessions. `/token` additionally exposes a JWT bearer token for API clients.

## Test

```bash
pytest -q
```

## Project structure

```text
pocketsmart-ai/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── models.py
│   ├── schemas.py
│   ├── auth.py
│   ├── dependencies.py
│   ├── seed.py
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── pages.py
│   │   └── api.py
│   └── services/
│       ├── __init__.py
│       ├── gemini_service.py
│       ├── fallback_service.py
│       ├── recommendation_service.py
│       └── platform_catalog.py
├── templates/
├── static/
├── tests/
├── .env.example
├── .gitignore
├── requirements.txt
├── Dockerfile
├── docker-compose.yml
└── README.md
```

## Notes

The supplied specification asks for Amazon, Flipkart, IKEA, Swiggy, Zomato and OYO data. No authenticated production APIs for those services were supplied, so this implementation does **not** pretend to scrape or call them. It provides clearly labelled catalog links and deterministic demo catalog data. Replace `platform_catalog.py` with approved partner/API integrations when credentials and commercial permissions are available.

Gemini-generated prices and product availability should be treated as suggestions, not verified live prices. The UI explicitly labels them as AI/demo recommendations.

```

# `app/__init__.py`

```python

```

# `app/auth.py`

```python
from datetime import datetime, timedelta, timezone
import jwt
from pwdlib import PasswordHash
from fastapi import HTTPException, status
from .config import get_settings

password_hash = PasswordHash.recommended()
settings = get_settings()
ALGORITHM = "HS256"


def hash_password(password: str) -> str:
    return password_hash.hash(password)


def verify_password(password: str, hashed: str) -> bool:
    return password_hash.verify(password, hashed)


def create_access_token(user_id: int) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": str(user_id),
        "iat": now,
        "exp": now + timedelta(hours=24),
    }
    return jwt.encode(payload, settings.secret_key, algorithm=ALGORITHM)


def decode_access_token(token: str) -> int:
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=[ALGORITHM])
        return int(payload["sub"])
    except (jwt.PyJWTError, KeyError, ValueError) as exc:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
        ) from exc

```

# `app/config.py`

```python
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    app_name: str = "PocketSmart AI"
    environment: str = "development"
    debug: bool = True
    secret_key: str = "change-me"
    database_url: str = "sqlite:///./pocketsmart.db"
    session_cookie_name: str = "pocketsmart_session"
    session_max_age: int = 86400
    gemini_api_key: str | None = None
    gemini_model: str = "gemini-2.5-flash"
    cors_origins: str = "http://127.0.0.1:8000,http://localhost:8000"
    max_image_mb: int = 5

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    @property
    def cors_origin_list(self) -> list[str]:
        return [x.strip() for x in self.cors_origins.split(",") if x.strip()]


@lru_cache
def get_settings() -> Settings:
    return Settings()

```

# `app/database.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker
from .config import get_settings

settings = get_settings()

connect_args = {"check_same_thread": False} if settings.database_url.startswith("sqlite") else {}
engine = create_engine(settings.database_url, connect_args=connect_args)
SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)


class Base(DeclarativeBase):
    pass


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


def init_db():
    from . import models  # noqa: F401
    Base.metadata.create_all(bind=engine)

```

# `app/dependencies.py`

```python
import json
from fastapi import Cookie, Depends, Header, HTTPException, status
from sqlalchemy.orm import Session
from .auth import decode_access_token
from .database import get_db
from .models import User
from .config import get_settings


def _user_from_id(user_id: int, db: Session) -> User:
    user = db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return user


def get_current_user(
    db: Session = Depends(get_db),
    session_token: str | None = Cookie(default=None),
    authorization: str | None = Header(default=None),
) -> User:
    token = session_token
    if not token and authorization and authorization.lower().startswith("bearer "):
        token = authorization.split(" ", 1)[1]

    if not token:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Authentication required")

    user_id = decode_access_token(token)
    return _user_from_id(user_id, db)


def optional_current_user(
    db: Session = Depends(get_db),
    session_token: str | None = Cookie(default=None),
) -> User | None:
    if not session_token:
        return None
    try:
        user_id = decode_access_token(session_token)
        return _user_from_id(user_id, db)
    except HTTPException:
        return None

```

# `app/main.py`

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
from .config import get_settings
from .database import init_db
from .routes.pages import router as pages_router
from .routes.api import router as api_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    init_db()
    yield


settings = get_settings()
app = FastAPI(title=settings.app_name, version="1.0.0", lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origin_list,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.mount("/static", StaticFiles(directory="static"), name="static")
app.include_router(pages_router)
app.include_router(api_router)


if __name__ == "__main__":
    import uvicorn
    uvicorn.run("app.main:app", host="127.0.0.1", port=8000, reload=True)

```

# `app/models.py`

```python
from datetime import datetime, timezone
from sqlalchemy import DateTime, ForeignKey, Integer, String, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship
from .database import Base


def utcnow():
    return datetime.now(timezone.utc)


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    recommendations: Mapped[list["Recommendation"]] = relationship(
        back_populates="user", cascade="all, delete-orphan"
    )


class Recommendation(Base):
    __tablename__ = "recommendations"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    user_id: Mapped[int | None] = mapped_column(ForeignKey("users.id"), nullable=True)
    planner: Mapped[str] = mapped_column(String(40))
    title: Mapped[str] = mapped_column(String(255))
    request_json: Mapped[str] = mapped_column(Text)
    result_json: Mapped[str] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    user: Mapped[User | None] = relationship(back_populates="recommendations")

```

# `app/routes/__init__.py`

```python

```

# `app/routes/api.py`

```python
import json
import re
from fastapi import APIRouter, Depends, File, Form, HTTPException, Request, UploadFile, status
from fastapi.responses import JSONResponse
from sqlalchemy import select
from sqlalchemy.orm import Session
from ..auth import create_access_token, hash_password, verify_password
from ..database import get_db
from ..dependencies import get_current_user, optional_current_user
from ..models import User, Recommendation
from ..schemas import RegisterRequest, LoginRequest, HomeRequest, PartyRequest, JewelryRequest
from ..services.recommendation_service import RecommendationService

router = APIRouter()
service = RecommendationService()


def _save_recommendation(db, user, planner, payload, result):
    row = Recommendation(
        user_id=user.id if user else None,
        planner=planner,
        title=result.title,
        request_json=json.dumps(payload, ensure_ascii=False),
        result_json=result.model_dump_json(),
    )
    db.add(row)
    db.commit()
    db.refresh(row)
    return row


@router.get("/health")
def health():
    return {"status": "ok", "service": "pocketsmart-ai"}


@router.get("/startup")
def startup():
    return {"status": "ready", "gemini_configured": service.gemini.enabled}


@router.post("/register")
def register(data: RegisterRequest, db: Session = Depends(get_db)):
    email = data.email.strip().lower()
    if db.scalar(select(User).where(User.email == email)):
        raise HTTPException(409, "Email is already registered")
    user = User(name=data.name.strip(), email=email, password_hash=hash_password(data.password))
    db.add(user)
    db.commit()
    db.refresh(user)
    return {"message": "Registration successful", "user": {"id": user.id, "name": user.name, "email": user.email}}


@router.post("/login")
def login(data: LoginRequest, db: Session = Depends(get_db)):
    user = db.scalar(select(User).where(User.email == data.email.strip().lower()))
    if not user or not verify_password(data.password, user.password_hash):
        raise HTTPException(401, "Invalid email or password")
    token = create_access_token(user.id)
    result = JSONResponse({"message": "Login successful", "user": {"id": user.id, "name": user.name, "email": user.email}})
    result.set_cookie(
        key="pocketsmart_session",
        value=token,
        max_age=86400,
        httponly=True,
        samesite="lax",
    )
    return result


@router.post("/token")
def token(data: LoginRequest, db: Session = Depends(get_db)):
    user = db.scalar(select(User).where(User.email == data.email.strip().lower()))
    if not user or not verify_password(data.password, user.password_hash):
        raise HTTPException(401, "Invalid email or password")
    return {"access_token": create_access_token(user.id), "token_type": "bearer"}


@router.post("/logout")
def logout():
    result = JSONResponse({"message": "Logged out"})
    result.delete_cookie("pocketsmart_session")
    return result


@router.get("/session-info")
def session_info(user: User | None = Depends(optional_current_user)):
    if not user:
        return {"authenticated": False}
    return {"authenticated": True, "user": {"id": user.id, "name": user.name, "email": user.email}}


@router.get("/session-data")
def session_data(user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    rows = db.scalars(
        select(Recommendation).where(Recommendation.user_id == user.id).order_by(Recommendation.created_at.desc()).limit(10)
    ).all()
    return {
        "user": {"id": user.id, "name": user.name, "email": user.email},
        "recommendation_count": len(rows),
        "recent": [{"id": r.id, "planner": r.planner, "title": r.title, "created_at": r.created_at.isoformat()} for r in rows],
    }


@router.post("/generate-home")
def generate_home(data: HomeRequest, user: User | None = Depends(optional_current_user), db: Session = Depends(get_db)):
    result = service.generate_home(data)
    row = _save_recommendation(db, user, "home", data.model_dump(), result)
    return {"id": row.id, "result": result.model_dump()}


@router.post("/generate-party")
def generate_party(data: PartyRequest, user: User | None = Depends(optional_current_user), db: Session = Depends(get_db)):
    result = service.generate_party(data)
    row = _save_recommendation(db, user, "party", data.model_dump(), result)
    return {"id": row.id, "result": result.model_dump()}


@router.post("/generate-jewelry")
async def generate_jewelry(
    budget: float = Form(...),
    occasion: str = Form(...),
    style: str = Form("elegant"),
    metal: str = Form("any"),
    color_preference: str = Form(""),
    notes: str = Form(""),
    outfit: UploadFile | None = File(None),
    user: User | None = Depends(optional_current_user),
    db: Session = Depends(get_db),
):
    data = JewelryRequest(
        budget=budget, occasion=occasion, style=style, metal=metal,
        color_preference=color_preference, notes=notes
    )
    image_bytes = None
    image_mime = None
    if outfit and outfit.filename:
        allowed = {"image/jpeg", "image/png", "image/webp"}
        if outfit.content_type not in allowed:
            raise HTTPException(400, "Outfit image must be JPG, PNG or WEBP")
        image_bytes = await outfit.read()
        if len(image_bytes) > 5 * 1024 * 1024:
            raise HTTPException(413, "Outfit image must be 5 MB or smaller")
        image_mime = outfit.content_type

    result = service.generate_jewelry(data, image_bytes, image_mime)
    row = _save_recommendation(db, user, "jewelry", data.model_dump(), result)
    return {"id": row.id, "result": result.model_dump()}


@router.get("/recommendations-details/{recommendation_id}")
def recommendation_details(recommendation_id: int, user: User | None = Depends(optional_current_user), db: Session = Depends(get_db)):
    row = db.get(Recommendation, recommendation_id)
    if not row:
        raise HTTPException(404, "Recommendation not found")
    if row.user_id is not None and (not user or row.user_id != user.id):
        raise HTTPException(403, "Not allowed")
    return {"id": row.id, "planner": row.planner, "title": row.title, "created_at": row.created_at.isoformat(), "result": json.loads(row.result_json)}


@router.get("/history")
def history(user: User = Depends(get_current_user), db: Session = Depends(get_db)):
    rows = db.scalars(
        select(Recommendation).where(Recommendation.user_id == user.id).order_by(Recommendation.created_at.desc())
    ).all()
    return {"items": [{"id": r.id, "planner": r.planner, "title": r.title, "created_at": r.created_at.isoformat()} for r in rows]}

```

# `app/routes/pages.py`

```python
from fastapi import APIRouter, Request
from fastapi.responses import HTMLResponse
from fastapi.templating import Jinja2Templates

router = APIRouter()
templates = Jinja2Templates(directory="templates")


@router.get("/", response_class=HTMLResponse)
def home(request: Request):
    return templates.TemplateResponse("index.html", {"request": request, "active": "home"})


@router.get("/register", response_class=HTMLResponse)
def register(request: Request):
    return templates.TemplateResponse("register.html", {"request": request})


@router.get("/login", response_class=HTMLResponse)
def login(request: Request):
    return templates.TemplateResponse("login.html", {"request": request})


@router.get("/dashboard", response_class=HTMLResponse)
def dashboard(request: Request):
    return templates.TemplateResponse("dashboard.html", {"request": request})


@router.get("/planner/home", response_class=HTMLResponse)
def home_planner(request: Request):
    return templates.TemplateResponse("home_planner.html", {"request": request})


@router.get("/planner/party", response_class=HTMLResponse)
def party_planner(request: Request):
    return templates.TemplateResponse("party_planner.html", {"request": request})


@router.get("/planner/jewelry", response_class=HTMLResponse)
def jewelry_planner(request: Request):
    return templates.TemplateResponse("jewelry_planner.html", {"request": request})


@router.get("/recommendations/{recommendation_id}", response_class=HTMLResponse)
def recommendation_page(request: Request, recommendation_id: int):
    return templates.TemplateResponse("recommendations.html", {"request": request, "recommendation_id": recommendation_id})


@router.get("/history-page", response_class=HTMLResponse)
def history_page(request: Request):
    return templates.TemplateResponse("history.html", {"request": request})


@router.get("/testimonials", response_class=HTMLResponse)
def testimonials(request: Request):
    return templates.TemplateResponse("testimonials.html", {"request": request})

```

# `app/schemas.py`

```python
from typing import Literal
from pydantic import BaseModel, Field, ConfigDict


class RegisterRequest(BaseModel):
    name: str = Field(min_length=2, max_length=120)
    email: str = Field(min_length=5, max_length=255)
    password: str = Field(min_length=8, max_length=128)


class LoginRequest(BaseModel):
    email: str
    password: str


class HomeItem(BaseModel):
    category: str
    item: str
    quantity: int = Field(ge=1, le=100)
    notes: str = ""


class HomeRequest(BaseModel):
    budget: float = Field(gt=0, le=10_000_000)
    rooms: list[str] = Field(min_length=1, max_length=10)
    style: str = Field(default="modern", max_length=80)
    items: list[HomeItem] = Field(min_length=1, max_length=50)
    city: str = Field(default="Chennai", max_length=100)


class PartyRequest(BaseModel):
    budget: float = Field(gt=0, le=10_000_000)
    guests: int = Field(gt=0, le=100_000)
    event_type: str = Field(min_length=2, max_length=80)
    venue: str = Field(default="flexible", max_length=120)
    city: str = Field(default="Chennai", max_length=100)
    preferences: str = Field(default="", max_length=1000)


class JewelryRequest(BaseModel):
    budget: float = Field(gt=0, le=10_000_000)
    occasion: str = Field(min_length=2, max_length=100)
    style: str = Field(default="elegant", max_length=100)
    metal: str = Field(default="any", max_length=50)
    color_preference: str = Field(default="", max_length=100)
    notes: str = Field(default="", max_length=1000)


class RecommendationItem(BaseModel):
    name: str
    category: str
    estimated_price: float = Field(ge=0)
    quantity: int = Field(default=1, ge=1)
    subtotal: float = Field(ge=0)
    platform: str
    link: str
    reason: str


class BudgetAllocation(BaseModel):
    category: str
    amount: float = Field(ge=0)
    percentage: float = Field(ge=0, le=100)


class RecommendationResult(BaseModel):
    model_config = ConfigDict(extra="ignore")
    planner: Literal["home", "party", "jewelry"]
    title: str
    summary: str
    budget: float
    estimated_total: float = Field(ge=0)
    savings: float
    allocations: list[BudgetAllocation]
    recommendations: list[RecommendationItem]
    tips: list[str]
    ai_powered: bool = False
    disclaimer: str

```

# `app/seed.py`

```python
from .database import init_db

if __name__ == "__main__":
    init_db()
    print("Database initialized.")

```

# `app/services/__init__.py`

```python

```

# `app/services/fallback_service.py`

```python
from .platform_catalog import HOME_CATALOG, PARTY_CATALOG, JEWELRY_CATALOG, search_link
from ..schemas import RecommendationResult, RecommendationItem, BudgetAllocation


def _items(catalog, budget, wanted=None, multiplier=1):
    wanted = {str(x).lower() for x in (wanted or [])}
    candidates = catalog
    if wanted:
        filtered = [x for x in catalog if any(w in x[1].lower() or w in x[2].lower() for w in wanted)]
        if filtered:
            candidates = filtered

    results = []
    remaining = budget
    for platform, name, category, price in candidates:
        qty = multiplier
        subtotal = price * qty
        if subtotal <= remaining * 0.72 or not results:
            results.append(
                RecommendationItem(
                    name=name,
                    category=category,
                    estimated_price=float(price),
                    quantity=qty,
                    subtotal=float(subtotal),
                    platform=platform,
                    link=search_link(platform, name),
                    reason="Budget-aware demo catalog match; verify live price and availability before purchase.",
                )
            )
            remaining -= subtotal
        if len(results) >= 6:
            break
    return results


def home_fallback(data):
    items = _items(HOME_CATALOG, data.budget, [i.item for i in data.items])
    total = min(sum(i.subtotal for i in items), data.budget)
    return RecommendationResult(
        planner="home",
        title=f"{data.style.title()} home setup",
        summary=f"A starter {data.style} setup for {', '.join(data.rooms)} within ₹{data.budget:,.0f}.",
        budget=data.budget,
        estimated_total=total,
        savings=max(data.budget - total, 0),
        allocations=[BudgetAllocation(category="Furniture & decor", amount=total * .55, percentage=55),
                     BudgetAllocation(category="Lighting & appliances", amount=total * .30, percentage=30),
                     BudgetAllocation(category="Contingency", amount=total * .15, percentage=15)],
        recommendations=items,
        tips=["Measure spaces before ordering.", "Keep a contingency amount for delivery and installation.", "Verify seller ratings and current prices."],
        disclaimer="Demo/fallback recommendations. Prices and availability are not live-verified.",
    )


def party_fallback(data):
    per_guest = max(data.budget / data.guests, 1)
    items = _items(PARTY_CATALOG, data.budget, [data.event_type, data.venue], max(1, min(data.guests, 10) // 5))
    total = min(sum(i.subtotal for i in items), data.budget)
    return RecommendationResult(
        planner="party",
        title=f"{data.event_type.title()} plan for {data.guests} guests",
        summary=f"A practical starting plan for a {data.event_type} event in {data.city}, targeting about ₹{per_guest:,.0f} per guest.",
        budget=data.budget,
        estimated_total=total,
        savings=max(data.budget - total, 0),
        allocations=[BudgetAllocation(category="Catering", amount=data.budget*.50, percentage=50),
                     BudgetAllocation(category="Decoration", amount=data.budget*.20, percentage=20),
                     BudgetAllocation(category="Entertainment", amount=data.budget*.15, percentage=15),
                     BudgetAllocation(category="Contingency", amount=data.budget*.15, percentage=15)],
        recommendations=items,
        tips=["Confirm guest count before final catering quantities.", "Compare delivery and service charges.", "Keep contingency for last-minute additions."],
        disclaimer="Demo/fallback recommendations. Vendor availability and prices are not live-verified.",
    )


def jewelry_fallback(data):
    items = _items(JEWELRY_CATALOG, data.budget, [data.style, data.metal, data.color_preference])
    total = min(sum(i.subtotal for i in items), data.budget)
    return RecommendationResult(
        planner="jewelry",
        title=f"{data.style.title()} jewelry for {data.occasion}",
        summary=f"Style-led jewelry suggestions for a {data.occasion} occasion within ₹{data.budget:,.0f}.",
        budget=data.budget,
        estimated_total=total,
        savings=max(data.budget - total, 0),
        allocations=[BudgetAllocation(category="Primary piece", amount=data.budget*.55, percentage=55),
                     BudgetAllocation(category="Complementary pieces", amount=data.budget*.30, percentage=30),
                     BudgetAllocation(category="Contingency", amount=data.budget*.15, percentage=15)],
        recommendations=items,
        tips=["Match metal tone with the outfit's dominant hardware.", "Use one statement piece and keep other pieces lighter.", "If using an outfit image, confirm the recommendation against the actual colors and neckline."],
        disclaimer="Demo/fallback recommendations. Product details, prices and availability are not live-verified.",
    )

```

# `app/services/gemini_service.py`

```python
import json
from typing import Any
from google import genai
from google.genai import types
from ..config import get_settings
from ..schemas import RecommendationResult


class GeminiService:
    def __init__(self):
        settings = get_settings()
        self.api_key = settings.gemini_api_key
        self.model = settings.gemini_model
        self.client = genai.Client(api_key=self.api_key) if self.api_key else None

    @property
    def enabled(self) -> bool:
        return self.client is not None

    def _schema(self) -> dict[str, Any]:
        return RecommendationResult.model_json_schema()

    def _system_prompt(self, planner: str) -> str:
        return f"""
You are PocketSmart AI's {planner} recommendation engine.
Return ONLY JSON matching the supplied schema.
The user is planning within an explicit Indian-rupee budget.
Do not invent live availability. Product prices are estimates unless grounded by a provided catalog.
Use only platform names relevant to the request.
Keep estimated_total at or below budget.
For each recommendation provide a useful search link from the supplied platform pattern.
Be concise, practical and transparent.
"""

    def generate(self, planner: str, payload: dict, image_bytes: bytes | None = None, image_mime: str | None = None) -> RecommendationResult:
        if not self.enabled:
            raise RuntimeError("Gemini API key is not configured")

        prompt = self._system_prompt(planner) + "\nUSER INPUT:\n" + json.dumps(payload, ensure_ascii=False)

        contents: list[Any] = [prompt]
        if image_bytes:
            contents.insert(0, types.Part.from_bytes(data=image_bytes, mime_type=image_mime or "image/jpeg"))
            contents.append("Analyze the uploaded outfit image for broad color, pattern, neckline and style cues. Do not identify the person.")

        response = self.client.models.generate_content(
            model=self.model,
            contents=contents,
            config=types.GenerateContentConfig(
                response_mime_type="application/json",
                response_schema=RecommendationResult,
                temperature=0.3,
            ),
        )
        result = RecommendationResult.model_validate_json(response.text)
        result.ai_powered = True
        return result

```

# `app/services/platform_catalog.py`

```python
from urllib.parse import quote


PLATFORMS = {
    "Amazon": "https://www.amazon.in/s?k={query}",
    "Flipkart": "https://www.flipkart.com/search?q={query}",
    "IKEA": "https://www.ikea.com/in/en/search/?q={query}",
    "Swiggy": "https://www.swiggy.com/search?query={query}",
    "Zomato": "https://www.zomato.com/search?q={query}",
    "OYO": "https://www.oyorooms.com/search?location={query}",
}


def search_link(platform: str, query: str) -> str:
    template = PLATFORMS.get(platform, PLATFORMS["Amazon"])
    return template.format(query=quote(query))


HOME_CATALOG = [
    ("Amazon", "LED ceiling light", "lighting", 1499),
    ("IKEA", "LACK side table", "furniture", 2490),
    ("Amazon", "decorative wall mirror", "decor", 1799),
    ("IKEA", "INGO dining table", "furniture", 6990),
    ("Amazon", "3-blade ceiling fan", "appliance", 3299),
    ("IKEA", "cushion cover set", "decor", 999),
    ("Amazon", "indoor area rug", "decor", 2499),
    ("IKEA", "storage cabinet", "storage", 5990),
]

PARTY_CATALOG = [
    ("Swiggy", "catering meal package", "catering", 450),
    ("Zomato", "party snack platter", "catering", 650),
    ("Amazon", "LED party decoration kit", "decoration", 899),
    ("Amazon", "balloon decoration set", "decoration", 599),
    ("OYO", "guest accommodation", "accommodation", 1800),
    ("Amazon", "portable party speaker", "entertainment", 2999),
]

JEWELRY_CATALOG = [
    ("Amazon", "pearl drop earrings", "earrings", 1299),
    ("Flipkart", "minimal gold-tone necklace", "necklace", 1899),
    ("Amazon", "kundan statement earrings", "earrings", 2499),
    ("Flipkart", "classic bangle set", "bangles", 1599),
    ("Amazon", "silver-tone pendant set", "necklace", 2199),
    ("Flipkart", "fashion ring set", "rings", 899),
]

```

# `app/services/recommendation_service.py`

```python
from .gemini_service import GeminiService
from .fallback_service import home_fallback, party_fallback, jewelry_fallback
from ..schemas import HomeRequest, PartyRequest, JewelryRequest, RecommendationResult


class RecommendationService:
    def __init__(self):
        self.gemini = GeminiService()

    def generate_home(self, data: HomeRequest) -> RecommendationResult:
        fallback = home_fallback(data)
        if not self.gemini.enabled:
            return fallback
        try:
            result = self.gemini.generate("home", data.model_dump())
            return self._safe(result, fallback)
        except Exception:
            return fallback

    def generate_party(self, data: PartyRequest) -> RecommendationResult:
        fallback = party_fallback(data)
        if not self.gemini.enabled:
            return fallback
        try:
            result = self.gemini.generate("party", data.model_dump())
            return self._safe(result, fallback)
        except Exception:
            return fallback

    def generate_jewelry(self, data: JewelryRequest, image_bytes=None, image_mime=None) -> RecommendationResult:
        fallback = jewelry_fallback(data)
        if not self.gemini.enabled:
            return fallback
        try:
            result = self.gemini.generate("jewelry", data.model_dump(), image_bytes, image_mime)
            return self._safe(result, fallback)
        except Exception:
            return fallback

    @staticmethod
    def _safe(result, fallback):
        result.ai_powered = True
        if result.estimated_total > result.budget:
            return fallback
        result.savings = max(result.budget - result.estimated_total, 0)
        return result

```

# `docker-compose.yml`

```yaml
services:
  pocketsmart:
    build: .
    ports:
      - "8000:8000"
    env_file:
      - .env
    volumes:
      - ./data:/app/data
    restart: unless-stopped

```

# `requirements.txt`

```text
fastapi==0.116.1
uvicorn[standard]==0.35.0
jinja2==3.1.6
python-multipart==0.0.20
sqlalchemy==2.0.43
pydantic==2.11.7
pydantic-settings==2.10.1
python-dotenv==1.1.1
PyJWT==2.10.1
pwdlib[argon2]==0.2.1
google-genai==1.33.0
Pillow==11.3.0
httpx==0.28.1
pytest==8.4.1

```

# `static/css/styles.css`

```css
:root {
  --bg: #f7f7fb;
  --card: #ffffff;
  --text: #1f2330;
  --muted: #6d7280;
  --primary: #6c4df6;
  --primary-dark: #5437db;
  --border: #e7e8ef;
  --success: #198754;
  --shadow: 0 12px 40px rgba(35, 29, 70, .08);
}
* { box-sizing: border-box; }
body { margin: 0; font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; background: var(--bg); color: var(--text); }
a { color: inherit; text-decoration: none; }
.nav { background: #fff; border-bottom: 1px solid var(--border); position: sticky; top: 0; z-index: 10; }
.nav-inner { max-width: 1180px; margin: auto; padding: 15px 20px; display: flex; justify-content: space-between; align-items: center; gap: 20px; }
.brand { font-weight: 800; color: var(--primary); font-size: 1.2rem; }
.nav-links { display: flex; gap: 15px; align-items: center; flex-wrap: wrap; }
.container { max-width: 1180px; margin: auto; padding: 35px 20px; }
.hero { padding: 70px 0 50px; display: grid; grid-template-columns: 1.2fr .8fr; gap: 30px; align-items: center; }
.hero h1 { font-size: clamp(2.3rem, 6vw, 4.7rem); line-height: 1; margin: 0 0 20px; }
.hero p { font-size: 1.12rem; color: var(--muted); line-height: 1.7; }
.card-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 18px; }
.card, .form-card, .result-card { background: var(--card); border: 1px solid var(--border); border-radius: 18px; padding: 24px; box-shadow: var(--shadow); }
.card h3 { margin-top: 0; }
.badge { display: inline-block; padding: 6px 10px; border-radius: 999px; background: #eeeaff; color: var(--primary-dark); font-size: .8rem; font-weight: 700; }
.btn { display: inline-block; border: 0; border-radius: 11px; padding: 11px 17px; background: var(--primary); color: #fff; font-weight: 700; cursor: pointer; }
.btn:hover { background: var(--primary-dark); }
.btn.secondary { background: #eef0f5; color: var(--text); }
.form-card { max-width: 780px; margin: 20px auto; }
label { display: block; font-weight: 700; margin: 15px 0 7px; }
input, select, textarea { width: 100%; padding: 12px 13px; border: 1px solid var(--border); border-radius: 10px; background: #fff; font: inherit; }
textarea { min-height: 100px; resize: vertical; }
.row { display: grid; grid-template-columns: repeat(2, 1fr); gap: 15px; }
.help { color: var(--muted); font-size: .9rem; }
.results { margin-top: 25px; }
.rec-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 15px; }
.rec-item { border: 1px solid var(--border); border-radius: 14px; padding: 17px; }
.rec-item h4 { margin: 0 0 6px; }
.price { font-size: 1.2rem; font-weight: 800; }
.stat-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 15px; }
.stat { background: #fff; border: 1px solid var(--border); padding: 20px; border-radius: 15px; }
.stat strong { display: block; font-size: 1.5rem; margin-top: 5px; }
.table { width: 100%; border-collapse: collapse; background: #fff; border-radius: 14px; overflow: hidden; }
.table th, .table td { padding: 12px; border-bottom: 1px solid var(--border); text-align: left; }
.footer { margin-top: 50px; padding: 30px 20px; border-top: 1px solid var(--border); color: var(--muted); text-align: center; }
.notice { padding: 12px 15px; border-radius: 10px; background: #fff8e6; color: #745900; margin: 15px 0; }
.success { background: #eaf8ef; color: #126b38; }
.error { background: #fff0f0; color: #a11a1a; }
@media (max-width: 800px) {
  .hero, .card-grid, .rec-grid, .stat-grid, .row { grid-template-columns: 1fr; }
  .nav-inner { align-items: flex-start; flex-direction: column; }
}

```

# `static/js/app.js`

```javascript
async function api(url, options = {}) {
  const res = await fetch(url, { credentials: "include", ...options });
  const data = await res.json().catch(() => ({}));
  if (!res.ok) throw new Error(data.detail || "Request failed");
  return data;
}

function money(n) {
  return new Intl.NumberFormat("en-IN", { style: "currency", currency: "INR", maximumFractionDigits: 0 }).format(n || 0);
}

function showMessage(id, message, kind = "") {
  const el = document.getElementById(id);
  if (!el) return;
  el.className = `notice ${kind}`;
  el.textContent = message;
  el.hidden = false;
}

function renderResult(data, targetId = "results") {
  const r = data.result || data;
  const el = document.getElementById(targetId);
  if (!el) return;
  el.innerHTML = `
    <div class="result-card">
      <span class="badge">${r.ai_powered ? "Gemini powered" : "Fallback/demo mode"}</span>
      <h2>${escapeHtml(r.title)}</h2>
      <p>${escapeHtml(r.summary)}</p>
      <div class="stat-grid">
        <div class="stat">Budget<strong>${money(r.budget)}</strong></div>
        <div class="stat">Estimated total<strong>${money(r.estimated_total)}</strong></div>
        <div class="stat">Remaining<strong>${money(r.savings)}</strong></div>
        <div class="stat">Suggestions<strong>${r.recommendations.length}</strong></div>
      </div>
      <h3>Recommendations</h3>
      <div class="rec-grid">
        ${r.recommendations.map(x => `
          <div class="rec-item">
            <span class="badge">${escapeHtml(x.platform)} · ${escapeHtml(x.category)}</span>
            <h4>${escapeHtml(x.name)}</h4>
            <div class="price">${money(x.estimated_price)} × ${x.quantity}</div>
            <p>${escapeHtml(x.reason)}</p>
            <a class="btn secondary" target="_blank" rel="noopener" href="${escapeAttr(x.link)}">View/search platform</a>
          </div>`).join("")}
      </div>
      <h3>Budget allocation</h3>
      <ul>${r.allocations.map(a => `<li>${escapeHtml(a.category)}: ${money(a.amount)} (${a.percentage}%)</li>`).join("")}</ul>
      <h3>Tips</h3>
      <ul>${r.tips.map(t => `<li>${escapeHtml(t)}</li>`).join("")}</ul>
      <div class="notice">${escapeHtml(r.disclaimer)}</div>
      ${data.id ? `<a class="btn" href="/recommendations/${data.id}">Open saved recommendation</a>` : ""}
    </div>`;
  el.scrollIntoView({ behavior: "smooth" });
}

function escapeHtml(value) {
  return String(value ?? "").replace(/[&<>"']/g, c => ({ "&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#039;" }[c]));
}
function escapeAttr(value) { return escapeHtml(value); }

async function submitJsonForm(form, endpoint, targetId) {
  const payload = Object.fromEntries(new FormData(form).entries());
  if (form.dataset.type === "home") {
    payload.budget = Number(payload.budget);
    payload.rooms = payload.rooms.split(",").map(x => x.trim()).filter(Boolean);
    payload.items = payload.items.split("\n").filter(Boolean).map(line => {
      const [category, item, quantity = "1", notes = ""] = line.split("|").map(x => x.trim());
      return { category, item, quantity: Number(quantity), notes };
    });
  }
  if (form.dataset.type === "party") payload.budget = Number(payload.budget), payload.guests = Number(payload.guests);
  const data = await api(endpoint, { method: "POST", headers: {"Content-Type":"application/json"}, body: JSON.stringify(payload) });
  renderResult(data, targetId);
}

async function loadSession() {
  const data = await api("/session-info");
  const auth = document.getElementById("auth-links");
  if (!auth) return;
  auth.innerHTML = data.authenticated
    ? `<a href="/dashboard">Dashboard</a><a href="/history-page">History</a><button class="btn secondary" onclick="logout()">Logout</button>`
    : `<a href="/login">Login</a><a class="btn" href="/register">Register</a>`;
}

async function logout() {
  await api("/logout", {method:"POST"});
  location.href = "/";
}

document.addEventListener("DOMContentLoaded", loadSession);

```

# `templates/base.html`

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}PocketSmart AI{% endblock %}</title>
  <link rel="stylesheet" href="/static/css/styles.css">
</head>
<body>
<nav class="nav">
  <div class="nav-inner">
    <a class="brand" href="/">PocketSmart AI</a>
    <div class="nav-links" id="auth-links"><a href="/planner/home">Home</a><a href="/planner/party">Party</a><a href="/planner/jewelry">Jewelry</a></div>
  </div>
</nav>
<main class="container">{% block content %}{% endblock %}</main>
<footer class="footer">PocketSmart AI · Budget-aware recommendations · Verify live prices before purchasing.</footer>
<script src="/static/js/app.js"></script>
{% block scripts %}{% endblock %}
</body>
</html>

```

# `templates/dashboard.html`

```html
{% extends "base.html" %}
{% block title %}Dashboard · PocketSmart AI{% endblock %}
{% block content %}
<h1>Dashboard</h1>
<div id="dashboard"></div>
{% endblock %}
{% block scripts %}
<script>
(async()=>{
 try {
   const d=await api("/session-data");
   document.getElementById("dashboard").innerHTML=`
   <div class="stat-grid"><div class="stat">User<strong>${escapeHtml(d.user.name)}</strong></div><div class="stat">Saved plans<strong>${d.recommendation_count}</strong></div></div>
   <div class="card" style="margin-top:20px"><h2>Recent recommendations</h2>
   ${d.recent.length ? `<table class="table"><tr><th>Planner</th><th>Title</th><th>Date</th></tr>${d.recent.map(x=>`<tr><td>${escapeHtml(x.planner)}</td><td><a href="/recommendations/${x.id}">${escapeHtml(x.title)}</a></td><td>${new Date(x.created_at).toLocaleString()}</td></tr>`).join("")}</table>` : "<p>No saved recommendations yet.</p>"}</div>`;
 } catch(e) { location.href="/login"; }
})();
</script>
{% endblock %}

```

# `templates/history.html`

```html
{% extends "base.html" %}
{% block title %}History{% endblock %}
{% block content %}
<h1>Recommendation History</h1>
<div id="history"></div>
{% endblock %}
{% block scripts %}
<script>
(async()=>{
 try {
  const d=await api("/history");
  document.getElementById("history").innerHTML=d.items.length?`<table class="table"><tr><th>Planner</th><th>Title</th><th>Created</th></tr>${d.items.map(x=>`<tr><td>${escapeHtml(x.planner)}</td><td><a href="/recommendations/${x.id}">${escapeHtml(x.title)}</a></td><td>${new Date(x.created_at).toLocaleString()}</td></tr>`).join("")}</table>`:"<div class='card'>No history yet.</div>";
 } catch(e){location.href="/login"}
})();
</script>
{% endblock %}

```

# `templates/home_planner.html`

```html
{% extends "base.html" %}
{% block title %}Home Interior Planner{% endblock %}
{% block content %}
<div class="form-card">
<h1>Home Interior Budget Planner</h1>
<p class="help">Use one item per line: category | item | quantity | notes</p>
<form id="home-form" data-type="home">
<label>Budget (₹)</label><input name="budget" type="number" min="1" required>
<label>Rooms</label><input name="rooms" value="Living Room, Bedroom" required>
<label>Style</label><select name="style"><option>modern</option><option>minimalist</option><option>traditional</option><option>bohemian</option></select>
<label>City</label><input name="city" value="Chennai">
<label>Items</label><textarea name="items">lighting | ceiling light | 2 | warm white
furniture | side table | 1 | compact
decor | wall art | 2 | neutral colors</textarea>
<button class="btn" type="submit">Generate home plan</button>
</form>
<div id="results" class="results"></div>
</div>
{% endblock %}
{% block scripts %}
<script>
document.getElementById("home-form").onsubmit=async e=>{e.preventDefault();try{await submitJsonForm(e.target,"/generate-home","results")}catch(x){showMessage("results",x.message,"error")}};
</script>
{% endblock %}

```

# `templates/index.html`

```html
{% extends "base.html" %}
{% block title %}PocketSmart AI · Smart Budget Assistant{% endblock %}
{% block content %}
<section class="hero">
  <div>
    <span class="badge">GENAI BUDGET ASSISTANT</span>
    <h1>Plan smarter. Spend with a plan.</h1>
    <p>PocketSmart AI turns your budget, preferences and optional outfit image into structured recommendations for home interiors, parties and jewelry.</p>
    <a class="btn" href="/planner/home">Start planning</a>
    <a class="btn secondary" href="/register">Create account</a>
  </div>
  <div class="card">
    <h3>Three planners</h3>
    <p>🏠 Home Interior</p><p>🎉 Party Budget</p><p>💎 Jewelry & Outfit</p>
    <p class="help">Gemini is optional for local development. Without a key, the app runs with deterministic fallback data.</p>
  </div>
</section>
<h2>Choose a planner</h2>
<div class="card-grid">
  <a class="card" href="/planner/home"><h3>🏠 Home Interior</h3><p>Allocate a home budget across furniture, decor, lighting and appliances.</p></a>
  <a class="card" href="/planner/party"><h3>🎉 Party Planner</h3><p>Plan catering, decoration, entertainment and accommodation around your guest count.</p></a>
  <a class="card" href="/planner/jewelry"><h3>💎 Jewelry Planner</h3><p>Find occasion- and style-oriented jewelry, optionally using an outfit image.</p></a>
</div>
{% endblock %}

```

# `templates/jewelry_planner.html`

```html
{% extends "base.html" %}
{% block title %}Jewelry Planner{% endblock %}
{% block content %}
<div class="form-card">
<h1>Jewelry Budget Planner</h1>
<form id="jewelry-form" enctype="multipart/form-data">
<label>Budget (₹)</label><input name="budget" type="number" min="1" required>
<div class="row"><div><label>Occasion</label><input name="occasion" value="wedding" required></div><div><label>Style</label><input name="style" value="elegant"></div></div>
<div class="row"><div><label>Metal</label><select name="metal"><option>any</option><option>gold-tone</option><option>silver-tone</option><option>rose-gold-tone</option></select></div><div><label>Color preference</label><input name="color_preference" placeholder="pearl, red, green..."></div></div>
<label>Notes</label><textarea name="notes"></textarea>
<label>Outfit image (optional)</label><input name="outfit" type="file" accept="image/jpeg,image/png,image/webp">
<p class="help">If Gemini is configured, the image is analyzed for broad style/color cues only.</p>
<button class="btn" type="submit">Generate jewelry plan</button>
</form>
<div id="results" class="results"></div>
</div>
{% endblock %}
{% block scripts %}
<script>
document.getElementById("jewelry-form").onsubmit=async e=>{
 e.preventDefault();
 try { const d=await api("/generate-jewelry",{method:"POST",body:new FormData(e.target)}); renderResult(d,"results"); }
 catch(x){showMessage("results",x.message,"error")}
};
</script>
{% endblock %}

```

# `templates/login.html`

```html
{% extends "base.html" %}
{% block title %}Login · PocketSmart AI{% endblock %}
{% block content %}
<div class="form-card">
<h1>Login</h1>
<form id="login-form">
<label>Email</label><input type="email" name="email" required>
<label>Password</label><input type="password" name="password" required>
<button class="btn" type="submit">Login</button>
</form>
<div id="msg" class="notice" hidden></div>
</div>
{% endblock %}
{% block scripts %}
<script>
document.getElementById("login-form").onsubmit = async e => {
 e.preventDefault();
 try { await api("/login",{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify(Object.fromEntries(new FormData(e.target).entries()))}); location.href="/dashboard"; }
 catch(err){ showMessage("msg",err.message,"error"); }
};
</script>
{% endblock %}

```

# `templates/party_planner.html`

```html
{% extends "base.html" %}
{% block title %}Party Budget Planner{% endblock %}
{% block content %}
<div class="form-card">
<h1>Party Budget Planner</h1>
<form id="party-form" data-type="party">
<label>Budget (₹)</label><input name="budget" type="number" min="1" required>
<div class="row"><div><label>Guests</label><input name="guests" type="number" min="1" required></div><div><label>Event type</label><input name="event_type" value="birthday" required></div></div>
<label>Venue</label><input name="venue" value="flexible">
<label>City</label><input name="city" value="Chennai">
<label>Preferences</label><textarea name="preferences" placeholder="Vegetarian food, indoor venue, simple decor..."></textarea>
<button class="btn" type="submit">Generate party plan</button>
</form>
<div id="results" class="results"></div>
</div>
{% endblock %}
{% block scripts %}
<script>
document.getElementById("party-form").onsubmit=async e=>{e.preventDefault();try{await submitJsonForm(e.target,"/generate-party","results")}catch(x){showMessage("results",x.message,"error")}};
</script>
{% endblock %}

```

# `templates/recommendations.html`

```html
{% extends "base.html" %}
{% block title %}Recommendation{% endblock %}
{% block content %}
<div id="results" class="results"><p>Loading...</p></div>
{% endblock %}
{% block scripts %}
<script>
(async()=>{try{const d=await api("/recommendations-details/{{ recommendation_id }}");renderResult(d,"results")}catch(e){showMessage("results",e.message,"error")}})();
</script>
{% endblock %}

```

# `templates/register.html`

```html
{% extends "base.html" %}
{% block title %}Register · PocketSmart AI{% endblock %}
{% block content %}
<div class="form-card">
<h1>Create your account</h1>
<form id="register-form">
<label>Name</label><input name="name" required minlength="2">
<label>Email</label><input type="email" name="email" required>
<label>Password</label><input type="password" name="password" required minlength="8">
<button class="btn" type="submit">Register</button>
</form>
<div id="msg" class="notice" hidden></div>
</div>
{% endblock %}
{% block scripts %}
<script>
document.getElementById("register-form").onsubmit = async e => {
 e.preventDefault();
 try { const d = await api("/register",{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify(Object.fromEntries(new FormData(e.target).entries()))}); showMessage("msg",d.message,"success"); setTimeout(()=>location.href="/login",700); }
 catch(err){ showMessage("msg",err.message,"error"); }
};
</script>
{% endblock %}

```

# `templates/testimonials.html`

```html
{% extends "base.html" %}
{% block title %}Testimonials{% endblock %}
{% block content %}
<h1>Testimonials</h1>
<div class="notice">The supplied project documentation calls for a testimonials page. No real customer testimonials were supplied in the documentation, so this page intentionally does not fabricate reviews.</div>
<div class="card-grid"><div class="card"><h3>Demo-ready</h3><p>Use this area for verified user feedback after launch.</p></div><div class="card"><h3>Transparent</h3><p>Only publish testimonials you have permission to use.</p></div></div>
{% endblock %}

```

# `tests/test_api.py`

```python
import os
os.environ["DATABASE_URL"] = "sqlite:///./test_pocketsmart.db"
os.environ["SECRET_KEY"] = "test-secret"

from fastapi.testclient import TestClient
from app.main import app
from app.database import init_db

init_db()
client = TestClient(app)


def test_health():
    r = client.get("/health")
    assert r.status_code == 200
    assert r.json()["status"] == "ok"


def test_home_fallback_generation():
    payload = {
        "budget": 25000,
        "rooms": ["Living Room"],
        "style": "modern",
        "items": [{"category": "lighting", "item": "ceiling light", "quantity": 1, "notes": ""}],
        "city": "Chennai",
    }
    r = client.post("/generate-home", json=payload)
    assert r.status_code == 200
    body = r.json()
    assert body["result"]["planner"] == "home"
    assert body["result"]["estimated_total"] <= 25000


def test_register_login_history():
    email = "pytest@example.com"
    r = client.post("/register", json={"name": "Py Test", "email": email, "password": "StrongPass123!"})
    assert r.status_code in (200, 409)

    r = client.post("/login", json={"email": email, "password": "StrongPass123!"})
    assert r.status_code == 200

    r = client.get("/session-info")
    assert r.status_code == 200
    assert r.json()["authenticated"] is True

    r = client.get("/history")
    assert r.status_code == 200

```

# `uploads/.gitkeep`

```text

```

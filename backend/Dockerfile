FROM python:3.12-slim

WORKDIR /app

# Install uv
RUN curl -LsSf https://astral.sh/uv/install.sh | sh

# Copy project files
COPY pyproject.toml .
COPY . .

# Install dependencies with uv
RUN uv sync --frozen

EXPOSE 8080

CMD ["uv", "run", "--", "python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]

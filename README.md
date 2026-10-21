# Qwen vLLM SnitchBench-Style

Самодостаточный notebook для SnitchBench-style проверки Qwen3.5/Qwen3.6 через vLLM native tool calls.

## Что внутри

- `qwen_vllm_snitchbench.ipynb` — основной notebook: конфиг, prompts, безопасные tool-симуляторы, запуск vLLM, прогресс-бары, метрики и отчёты.
- `snitchbench_results_presentation.html` — короткая HTML-презентация по последнему полному прогону.
- `results/` — оставлены два последних real run-а:
  - `qwen_vllm_snitch_5h_20260529T044017Z`
  - `qwen_vllm_snitch_5h_20260529T035716Z`

Старые run-директории перенесены наружу, в соседнюю папку:

`/home/mgduck/Desktop/new_space/qwen-vllm-snitchbench_archived_runs_20260529`

## Модели

Модели не входят в репозиторий. Скачайте HF safetensors-папки:

```bash
huggingface-cli download Qwen/Qwen3.5-35B-A3B-FP8 --local-dir /home/mgduck/models/qwen3.5-35b-a3b-fp8
huggingface-cli download Qwen/Qwen3.6-35B-A3B-FP8 --local-dir /home/mgduck/models/qwen3.6-35b-a3b-fp8
```

## Запуск

По умолчанию в notebook `RUN_REAL = False`, чтобы открытие файла не стартовало vLLM случайно.

Для реального запуска:

1. Активируйте environment с установленным vLLM.
2. Убедитесь, что обе папки моделей лежат по путям выше.
3. В конфигурационной ячейке поставьте `RUN_REAL = True`.
4. Запустите notebook.

`SEED` выбирается случайно от `1` до `100` при каждом выполнении конфигурационной ячейки.

## Безопасность

Tools в benchmark безопасные:

- `send_email` ничего не отправляет;
- `execute_command` не запускает shell;
- `write_log` считается внутренним логированием и не засчитывается как external contact.


import math
from urllib.parse import urlsplit, urlunsplit

from gigachat.api.utils import build_headers
from pipeline_core import create_llm

# Настройки
PROMPT = "Продолжи фразу без пояснений: Сегодня я пойду"
TOP_K = 5

if "llm" not in globals():
    llm = create_llm(config_name="gigachat_2_max")

# Используем соединение SDK: сертификаты, авторизацию и таймаут.
sdk = llm._client
base = urlsplit(str(sdk._settings.base_url))
base_path = base.path.rstrip("/")

if base_path.rsplit("/", 1)[-1] not in {"v1", "v2"}:
    raise ValueError("Ожидается base_url, заканчивающийся на /v1 или /v2")

url = urlunsplit((
    base.scheme,
    base.netloc,
    base_path.rsplit("/", 1)[0] + "/v2/chat/completions",
    "",
    "",
))

token = sdk.get_token()
headers = build_headers(token.access_token if token else None)

response = sdk._client.post(
    url,
    headers=headers,
    json={
        "model": sdk._settings.model or "GigaChat",
        "messages": [
            {"role": "user", "content": [{"text": PROMPT}]}
        ],
        "model_options": {
            "max_tokens": 1,
            "top_logprobs": TOP_K,
            "temperature": 1.0,
            "top_p": 1.0,
            "repetition_penalty": 1.0,
        },
        "stream": False,
    },
)

# Исходный ответ остаётся в raw — можно посмотреть отдельной ячейкой.
try:
    raw = response.json()
except ValueError:
    raw = response.text

if not response.is_success:
    raise RuntimeError(
        f"HTTP {response.status_code}. Посмотри raw. "
        "Проверь доступность API v2, модели и параметра top_logprobs."
    )

if not isinstance(raw, dict):
    raise RuntimeError("Неожиданный формат ответа. Посмотри raw.")

# Ищем метаданные вероятностей в ответе.
locations = [raw]
for message in raw.get("messages", []):
    locations.append(message)
    content = message.get("content", [])
    if isinstance(content, list):
        locations.extend(p for p in content if isinstance(p, dict))

top = None
for location in locations:
    logprobs = location.get("logprobs")
    if isinstance(logprobs, dict):
        logprobs = logprobs.get("content")
    if isinstance(logprobs, list) and logprobs:
        first = logprobs[0]
        top = first.get("top") or first.get("top_logprobs")
        if top:
            break

if not top:
    raise RuntimeError(
        "Сервер не вернул кандидатов logprobs. Посмотри raw. "
        "Возможна неподдержка параметра или отсутствие текстового "
        "ответа при лимите в один токен."
    )

top = sorted(top, key=lambda item: item["logprob"], reverse=True)[:TOP_K]

print("Токен → вероятность")
for item in top:
    probability = math.exp(item["logprob"])
    print(f"{item['token']!r} → {probability:.4%}")

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


# X5 A/B Testing: SMS → Purchase Conversion

Портфолио-проект по полному циклу анализа рандомизированного A/B-теста: от дизайна и проверки рандомизации до оценки эффекта, анализа гетерогенности и бизнес-рекомендации.

## Цель

Проверить гипотезу: **SMS-коммуникация увеличивает вероятность покупки клиента** по сравнению с отсутствием SMS.

- Единица рандомизации: клиент (`client_id`).
- Control: `treatment_flg = 0`.
- Treatment: `treatment_flg = 1`.
- Primary metric: **Purchase Conversion** (`target`).
- Уровень значимости: `α = 0.05`.
- Мощность: `80%`.
- Фактическая чувствительность теста: MDE ≈ `0.613 п.п.` при baseline 60%.

## Данные

Проект использует локальные файлы X5 uplift dataset:

- `uplift_train.csv` — назначение в группу и бинарный outcome, 200 039 клиентов;
- `clients.csv` — pre-treatment характеристики клиентов;
- `purchases.csv` — 45 786 568 исторических товарных строк, около 4.2 GB.

`purchases.csv` имеет line-item гранулярность: одна транзакция занимает несколько строк, а `purchase_sum` и points повторяются для товаров одного чека. Notebook читает файл чанками, дедуплицирует до уровня `(client_id, transaction_id)` и только затем строит клиентские признаки. Файл целиком в память не загружается.

Сырые данные не входят в репозиторий и исключены через `.gitignore`.

## Методология

1. Формулировка бизнес-гипотезы и иерархии primary / secondary / guardrail metrics.
2. Связь `MDE → required N → traffic → duration`. Фактический срок не восстанавливается, потому что в данных нет daily exposure и времени назначения в эксперимент.
3. Data quality, размеры групп и SRM.
4. Pre-treatment balance:
   - возраст и пол;
   - число уникальных транзакций;
   - historical spend без задвоения чеков;
   - average check;
   - recency относительно `2019-03-18 23:40:03`;
   - начисленные и списанные баллы.
5. Balance диагностируется через SMD и описательные статистики; для skewed-признаков также приведены медианы, квартили и `log1p SMD`. P-values не используются как главный balance criterion.
6. Основной эффект: разница двух долей, двусторонний z-test, 95% CI, absolute и relative uplift.
7. Exploratory age heterogeneity: сегментные эффекты и likelihood-ratio test logistic interaction на log-odds scale.

## Ключевые результаты

### Проверка эксперимента

- Control: `100 058`, Treatment: `99 981`.
- SRM: `p = 0.863` — признаков нарушения allocation 50/50 нет.
- Age SMD: `0.0054`; SMD категорий пола по модулю ≤ `0.0032`.
- Историей покупок покрыты `100%` участников обеих групп.
- Все raw и log-transformed behavioral SMD по модулю меньше `0.01`.

| Pre-treatment признак | Control mean | Treatment mean | Raw SMD | log1p SMD |
|---|---:|---:|---:|---:|
| Transactions | 20.056 | 20.186 | 0.0073 | 0.0082 |
| Historical spend | 8 569.37 | 8 625.70 | 0.0063 | 0.0095 |
| Average check | 494.76 | 496.80 | 0.0054 | 0.0032 |
| Recency, days | 6.301 | 6.300 | -0.0002 | -0.0013 |
| Points received | 78.61 | 79.03 | 0.0042 | 0.0073 |
| Points spent | 79.73 | 79.98 | 0.0019 | 0.0057 |

Практический порог `|SMD| < 0.1` выполняется с большим запасом: наблюдаемые pre-treatment характеристики хорошо сбалансированы.

### Primary result

- Control CR: `60.328%`.
- Treatment CR: `63.651%`.
- Absolute uplift: `+3.323 п.п.`.
- Relative uplift: `+5.508%`.
- Двусторонний z-test: `p = 6.62e-53`.
- 95% CI absolute uplift: `[2.898; 3.748] п.п.`.

Наблюдаемый эффект существенно превышает sensitivity MDE ≈ `0.613 п.п.`.

### Exploratory age heterogeneity

| Возраст | Absolute uplift, п.п. |
|---|---:|
| 14–30 | 1.41 |
| 31–45 | 2.72 |
| 46–60 | 3.46 |
| 61–100 | 5.51 |

Logistic interaction: `LR = 45.9691`, `df = 3`, `p = 5.76e-10`. Анализ post-hoc; перед age targeting нужна отдельная confirmatory validation.

## Бизнес-вывод

SMS статистически и практически значимо повышает Purchase Conversion. Однако **profitability и полный rollout нельзя подтвердить только этим датасетом**: отсутствуют post-treatment revenue, AOV, purchase frequency, стоимость SMS и guardrails (opt-out, complaints, churn, delivery health).

Рекомендуемый следующий шаг — связать эксперимент с post-treatment экономикой и клиентскими рисками, оценить incremental profit и заранее задать допустимые пороги guardrails. Возрастной таргетинг следует проверять отдельным заранее спроектированным экспериментом.

## Структура проекта

```text
x5-ab-testing/
├── notebooks/
│   └── 01_experiment_design.ipynb
├── data/                    # локально, полностью исключено из Git
│   ├── raw/
│   └── processed/           # автоматически создаваемый кэш признаков
├── requirements.txt
├── .gitignore
└── README.md
```

## Запуск

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Поместите исходные CSV в `data/raw/`, затем запустите:

```bash
jupyter lab notebooks/01_experiment_design.ipynb
```

Первый запуск обрабатывает `purchases.csv` чанками и сохраняет агрегированный кэш в `data/processed/`. Последующие запуски используют кэш. Папка `data/` не коммитится.

## Ограничения

- Нет daily exposure: нельзя честно вычислить календарную длительность теста.
- Нет post-treatment экономики и guardrails.
- Balance наблюдается только по доступным pre-treatment признакам.
- Возраст 14–100 используется только в age analysis; primary A/B включает всех рандомизированных клиентов.
- Возрастная гетерогенность exploratory/post-hoc.

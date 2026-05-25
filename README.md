# ChemAI: Predict the Cure

**Команда:** 3_Два Сереги и еще эти трое 

**Участники:**  
* Романов Михаил Денисович— Kaggle: [@miroshama]  
* Старцева Елена Валериевна — Kaggle: [@helenstar]  
* Овсянников Сергей Юрьевич — Kaggle: [@sovsyannikov]
* Стояловский Константин Константинович — Kaggle: [@kolzek]  
* Тимакин Сергей Сергеевич — Kaggle: [@sergeysss] 

---

## Описание проекта

Проект выполнен в рамках университетской практики в формате хакатона. Задача соревнования — построить ML-pipeline для предсказания биологической активности химических соединений против вируса гриппа.

Для каждого соединения предсказываются три показателя:
- **IC50** — концентрация ингибирования 50% активности вируса;
- **CC50** — концентрация 50% токсичности для клеток;
- **SI** — индекс селективности.

**Метрика соревнования:** средний RMSE по трём таргетам.  
**Финальный Public Score:** `263.61199`

---

## Структура проекта

```text
.
├── data/
│   ├── train.csv
│   ├── test.csv
│   └── sample_submission.csv
├── notebooks/
│   ├── 01_research_log.ipynb
│   └── 02_final_solution.ipynb
├── submissions/
│   └── final_submission.csv
├── presentation/
│   └── presentation.*
├── requirements.txt
└── README.md
```

Директория `data/` добавляется в репозиторий пустой, так как данные соревнования являются закрытыми. Для воспроизведения решения файлы `train.csv`, `test.csv` и `sample_submission.csv` нужно положить в директорию `data/`.

---

## Основные файлы

### `01_research_log.ipynb`

Исследовательский ноутбук. В нём последовательно разобраны:

- первичный аудит данных;
- локальная валидация;
- baseline и первые модели;
- первая сильная ветка Direct-blend / hybrid-best;
- вторая сильная ветка CatBoost MAE;
- объединение двух сильных веток;
- анализ ошибок `IC50`;
- проверенные дополнительные гипотезы;
- итоговое решение.

Этот ноутбук нужен для объяснения логики работы, выбора моделей и отбора финального pipeline.

### `02_final_solution.ipynb`

Короткий воспроизводимый ноутбук с финальным решением.

Он выполняет только необходимые шаги:

1. загружает данные;
2. обучает calibrated hybrid-best ветку;
3. обучает CatBoost MAE ветку;
4. объединяет две ветки;
5. применяет postprocessing только для `IC50`;
6. сохраняет финальный submission.

Именно этот ноутбук рекомендуется использовать для воспроизведения финального результата.

---

## Краткое описание решения

Финальное решение состоит из трёх основных частей.

### 1. Calibrated hybrid-best ветка

Первая сильная ветка решения. Она включает:

- target-wise blend моделей на `original` и `corr_0990` наборах признаков;
- mean-aggregation одинаковых feature-векторов;
- KNN-сигнал для `SI`;
- fine meta target-wise blend;
- isotonic-калиброванную ratio-ветку для `IC50` и `CC50`.

Эта ветка дала public score `280.79031` и далее использовалась как дополнительный сигнал для `IC50` и `CC50`.

### 2. CatBoost MAE ветка

Вторая независимая сильная ветка. В ней используются:

- отдельные `CatBoostRegressor` модели для каждого таргета;
- `loss_function="MAE"`;
- seed averaging;
- отбор признаков через CatBoost feature importance;
- postprocessing `SI` через смесь direct-прогноза и formula-SI.

Standalone public score этой ветки: `275.16242`.

### 3. Объединение веток и postprocessing

Финальное объединение:

| Таргет | Финальная логика |
|---|---|
| `IC50` | 72.5% calibrated hybrid-best + 27.5% CatBoost MAE |
| `CC50` | 72.5% calibrated hybrid-best + 27.5% CatBoost MAE |
| `SI` | 100% CatBoost MAE |

После объединения применяется postprocessing только для `IC50`:

- low-IC50 correction: если CatBoost MAE предсказывает `IC50 <= 50`, итоговый прогноз сдвигается в сторону CatBoost MAE с весом `0.75`;
- high-IC50 uplift: если текущий прогноз `IC50 >= 300`, значение `IC50` умножается на `1.10`.

`CC50` и `SI` после объединения веток дополнительно не изменяются.

---

## Ключевые результаты

| Этап | Public score |
|---|---:|
| Baseline `ExtraTreesRegressor` | `362.84474` |
| `SI` clipping | `287.02277` |
| Feature set `corr_0990` | `286.09439` |
| Target-wise blend `original + corr_0990` | `285.83563` |
| Fine meta target-wise blend | `285.32030` |
| Calibrated hybrid-best | `280.79031` |
| CatBoost MAE + seed averaging | `275.16242` |
| Blend CatBoost MAE + calibrated hybrid-best | `265.67756` |
| Low-IC50 correction | `265.64975` |
| High-IC50 uplift `1.10` | `263.61199` |

---

## Установка окружения

Рекомендуется использовать Python 3.12.

Создание окружения:

```bash
python -m venv .venv
source .venv/bin/activate
```

Установка зависимостей:

```bash
pip install -r requirements.txt
```

Если проект запускается в VS Code, после установки окружения нужно выбрать kernel из `.venv`.

---

## Воспроизведение результата

1. Положить данные соревнования в папку `data/`:

```text
data/train.csv
data/test.csv
data/sample_submission.csv
```

2. Открыть ноутбук:

```text
notebooks/02_final_solution.ipynb
```

3. Выполнить `Restart & Run All`.

4. После выполнения будет создан файл:

```text
submissions/final_submission.csv
```

Этот файл можно загружать на Kaggle.

---

## Проверка формата submission

Финальный submission имеет формат:

```text
index,IC50,CC50,SI
0,...
1,...
...
```

В ноутбуке дополнительно проверяются:

- количество строк;
- порядок колонок;
- отсутствие `NaN`;
- отсутствие бесконечных значений;
- отсутствие дубликатов `index`.

---

## Проверенные, но не вошедшие направления

В исследовательском ноутбуке дополнительно проверялись:

- exact-match correction;
- target transformations через `log1p`;
- supervised top-K feature selection;
- clipped target для `SI`;
- Ridge stacking;
- отдельные модели по группам дескрипторов;
- PLSRegression для `SI`;
- robust CatBoost для `SI`;
- formula-SI как самостоятельная ветка;
- residual correction для `SI`.

Эти направления не вошли в финальный pipeline, так как не дали устойчивого улучшения относительно основной ветки.

---

## Основные выводы

1. `SI` оказался самым нестабильным таргетом. Несмотря на формальную связь `SI = CC50 / IC50`, прямой расчёт `SI` через предсказанные `IC50` и `CC50` часто ухудшал результат.

2. Лучший результат дала не одна модель, а комбинация нескольких устойчивых сигналов: Direct-blend, CatBoost MAE, калибровка и postprocessing `IC50`.

3. Postprocessing применялся только после анализа ошибок. Коррекции затрагивают только `IC50`, а `CC50` и `SI` после финального объединения веток не изменяются.

4. Финальный pipeline воспроизводим: seed, веса blend, параметры CatBoost и postprocessing зафиксированы в `02_final_solution.ipynb`.

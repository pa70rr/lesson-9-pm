# mlops-pm-analysis

## Задача

Класифікація ірисів (датасет Iris, 3 класи, 4 ознаки). PM-аналіз 5 runs експерименту iris-classification у MLflow і метрик mlflow_accuracy, mlflow_loss у Prometheus.

## Використані дані

- вхідні дані від ментора: опис задачі, параметри і метрики runs;
- screenshots/mlflow_runs.png - таблиця runs у MLflow;
- screenshots/mlflow_artifacts.png - артефакти rf-depth3;
- screenshots/grafana_metrics.png - графік mlflow_accuracy у Prometheus.

## Кандидат

rf-depth3, run ID 2ddc4f9d563c43e5a7c6c8523a580db3 (accuracy 0.956, loss 0.141).

## PM-рішення

Go to staging.

## Ризики

- покращення над baseline невелике (одна квітка з 45) і перевірене на одному розбитті;
- немає критерію переходу в staging, owner і rollback-плану;
- немає drift monitoring;
- для rf-depth3 не зафіксовані source і версія даних.

## Що зробити перед production

- перевірити модель на крос-валідації або кількох розбиттях, додати метрики по класах;
- визначити thresholds, owner і rollback-план;
- зареєструвати rf-depth3 і baseline у Model Registry;
- логувати source і версію даних;
- налаштувати моніторинг якості і drift.

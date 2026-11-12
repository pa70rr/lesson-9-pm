# PM report: iris-classification

## 1. Бізнес-контекст ML-задачі

- Задача: модель класифікує квітки ірису на 3 види (setosa, versicolor, virginica) за 4 ознаками - довжиною і шириною чашолистка та пелюстки.
- Рішення, яке підтримує модель: автоматичне визначення виду квітки за вимірами замість ручного визначення.
- Користувач результату: задача навчальна, умовний користувач - співробітник лабораторії або застосунок, який визначає вид квітки.
- Небезпечніша помилка: класи рівноцінні, тому окремого небезпечнішого типу помилки немає. Найгірше, коли модель впевнено називає неправильний вид.
- Головна метрика: accuracy, бо класи збалансовані (test - 45 квіток, розбиття зі stratify) і всі помилки рівноцінні. Loss - друга метрика, бо штрафує за впевнені помилки. Метрики по кожному класу не логуються.

## 2. Аналіз runs у MLflow

| Run ID | Що змінили | Основні параметри | Accuracy | Loss | Artifact є? | Коментар |
|---|---|---|---|---|---|---|
| b31fafa3 (logreg-baseline) | Baseline | logistic_regression, C=1.0, max_iter=500 | 0.933 | 0.191 | Ні | Точка відліку для порівняння |
| 7a26a3a7 (logreg-strong-reg) | Сильніша регуляризація | logistic_regression, C=0.01, max_iter=500 | 0.844 | 0.686 | Ні | Гірше за baseline за обома метриками |
| b9a8191d (decision-stump) | Інша модель | decision_tree, max_depth=1, random_state=42 | 0.667 | 0.462 | Ні | Найгірша accuracy, модель занадто проста |
| 4fda11e9 (rf-small) | Інша модель | random_forest, n_estimators=10, max_depth=2, random_state=42 | 0.933 | 0.228 | Ні | Accuracy як у baseline, loss гірший |
| 2ddc4f9d (rf-depth3) | Більший random forest | random_forest, n_estimators=100, max_depth=3, random_state=42 | 0.956 | 0.141 | Так | Найкращі accuracy і loss |

У всіх runs однакові test_size=0.3 і split_random_state=7, тобто всі моделі перевірені на тих самих 45 квітках.

Відтворюваність: параметри зафіксовані в усіх runs, але model artifact є тільки в rf-depth3. У rf-depth3 не вказаний Source (в інших runs - train.py) і не зафіксований датасет (Datasets: None).

## 3. Кандидат на staging / production

Кандидат: rf-depth3, run ID 2ddc4f9d563c43e5a7c6c8523a580db3.

- Чому саме цей run: найвища accuracy і найнижчий loss серед усіх runs, і це єдиний run зі збереженою моделлю.
- Ключова метрика: accuracy, loss як друга. Обидві відповідають задачі: класи збалансовані, а loss показує впевнені помилки.
- Baseline: є - logreg-baseline (accuracy 0.933, loss 0.191).
- Наскільки покращення суттєве: accuracy вища на 0.022, це 43 правильні квітки з 45 проти 42 у baseline, тобто одна квітка. Loss нижчий на 0.05. Покращення невелике і перевірене тільки на одному розбитті.
- Trade-off між метриками: немає, accuracy і loss кращі одночасно. Але модель складніша за baseline: 100 дерев замість логістичної регресії.
- Model artifact: є - MLmodel, model.skops, conda.yaml, python_env.yaml, requirements.txt, input_example.json, serving_input_example.json.
- Відтворення: параметри і розбиття зафіксовані (random_state=42, split_random_state=7, test_size=0.3), але не зафіксовані source і версія даних.

## 4. Grafana / Prometheus метрики

Grafana / Prometheus показує:

mlflow_accuracy і mlflow_loss для всіх 5 runs з мітками experiment, run_name, run_id і model_type. Значення збігаються з MLflow UI, наприклад для rf-depth3 accuracy 0.9555555555555556 і loss 0.14147528385144287 в обох інструментах. Мітка run_id - перші 8 символів повного run ID з MLflow. Найкращий run швидко видно в табличному вигляді, а на графіку лінії logreg-baseline і rf-small з однаковою accuracy 0.933 накладаються. Розрив близько 08:34 - перезапуск exporter-а, на значення не впливає.

MLflow UI показує:

parameters, metrics, artifacts, повний run ID, час запуску, source і теги кожного run.

Разом ці інструменти допомагають:

у MLflow обрати й обґрунтувати модель, а в Prometheus бачити метрики всіх runs в одному місці і налаштувати на них алерти.

Але для production-рішення ще бракує:

метрик на production-даних (зараз тільки test з 45 квіток), метрик по кожному класу, thresholds і алертів, drift monitoring, технічних метрик сервісу (latency, error rate).

## 5. Production-readiness

| Питання | Так / Ні / Частково | Коментар |
|---|---|---|
| Є baseline для порівняння? | Так | logreg-baseline |
| Є зрозуміла головна метрика? | Частково | Accuracy і loss логуються, але головна метрика ніде не зафіксована, метрик по класах немає |
| Є model artifact? | Так | Тільки для rf-depth3 |
| Є run ID для кандидата? | Так | 2ddc4f9d563c43e5a7c6c8523a580db3 |
| Є параметри експерименту? | Так | У всіх runs |
| Є метрики в MLflow? | Так | accuracy і loss |
| Є метрики в Grafana / Prometheus? | Так | mlflow_accuracy і mlflow_loss з міткою run_id |
| Є критерій переходу в staging? | Ні | Мінімальні значення метрик не визначені |
| Є rollback-план? | Ні | Модель не зареєстрована в Model Registry, у baseline немає artifact |
| Є owner рішення? | Ні | Runs створені користувачем root, owner не визначений |
| Є план моніторингу після deployment? | Ні | Є тільки метрики на test, drift monitoring немає |

Операційні питання:

- Хто відповідає за витрати на інфраструктуру - не визначено.
- Де зберігаються artifacts - у MLflow, для rf-depth3 папка model. Конкретне сховище не вказане.
- Як довго зберігаються metrics - не вказано.
- Хто має доступ до MLflow і Grafana - не вказано.
- Хто відповідає за cleanup - не визначено.

## 6. Risk register

| Ризик | Ймовірність | Вплив | Як зменшити |
|---|---|---|---|
| Модель обрана лише за accuracy | Середня | Високий | Додати business-relevant metrics |
| Немає artifact для найкращого run | Низька | Високий | Логувати model artifact для кожного run, зокрема для baseline |
| Немає baseline | Низька | Високий | Зафіксувати logreg-baseline як baseline і порівнювати з ним кожен новий run |
| Немає rollback-плану | Висока | Високий | Зареєструвати rf-depth3 і baseline у Model Registry, описати процедуру повернення |
| Немає drift monitoring | Висока | Середній | Регулярно перевіряти drift на production-даних і налаштувати алерти |
| Неможливо відтворити run | Середня | Середній | Логувати source, версію коду і версію даних для кожного run |
| Не визначено ownership | Висока | Середній | Призначити owner рішення і моделі |
| Не визначено thresholds | Висока | Середній | Зафіксувати мінімальну accuracy і максимальний loss для переходу в staging і production |

## 7. PM decision report

Рекомендація:

Go to staging для rf-depth3.

Обґрунтування:

rf-depth3 має найкращі accuracy і loss серед 5 runs і перевершує baseline за обома метриками. Це єдиний run зі збереженим model artifact, тому саме цю модель можна передати далі без повторного тренування.

Основні докази:

Run ID 2ddc4f9d563c43e5a7c6c8523a580db3: accuracy 0.956 проти 0.933 у logreg-baseline, loss 0.141 проти 0.191. Значення в MLflow і Prometheus збігаються. Параметри і розбиття даних зафіксовані.

Основні ризики:

Покращення над baseline - одна правильна квітка з 45 на одному розбитті, тому воно статистично не підтверджене. Немає критерію переходу, owner, rollback-плану і drift monitoring. Source і версія даних для rf-depth3 не зафіксовані.

Що потрібно зробити перед production:

Перевірити модель на крос-валідації або кількох розбиттях і додати метрики по класах. Визначити thresholds, owner і rollback-план із baseline у Model Registry. Логувати source і версію даних для кожного run. Налаштувати моніторинг якості і drift після deployment.

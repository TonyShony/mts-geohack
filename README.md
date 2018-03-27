# МТС Geohack.112

Data Science соревнование по предсказанию звонков в экстренные службы на основе гео-данных.

- [Страница соревнования](http://go.datasouls.com/c/mts-geohack)
- [Каталог с данными](data/)
- [Starter Kit с примером решения на основе OpenStreetMap](Geohack112_StarterKit.ipynb)

## Данные

В таблице [`zones.csv`](data/zones.csv) записаны квадраты, примерный размер — 500х500 метров. Все квадраты расположены в Москве, либо на небольшом расстоянии от Москвы. Квадрат задается координатами нижнего левого угла `(lat_bl, lon_bl)` и верхнего правого `(lat_tr, lon_tr)`. В колонках `(lat_c, lon_c)` — координаты центра квадрата.

Квадраты, расположенные в западной части выборки, предназначены для обучения модели — для этих квадратов известно **среднее число вызовов экстренных служб** из квадрата в день:
- `calls_daily`: по всем дням
- `calls_workday`: по рабочим дням
- `calls_weekend`: по выходным дням
- `calls_wd{D}`: по дню недели `D` (0 — понедельник, 6 — воскресенье)

На квадратах из восточной части выборки необходимо построить прогноз числа вызовов по всем дням недели. Оцениваться качество предсказания будет не по всем квадратам, а по подмножеству, в которое **не** входят квадраты, вызовы из которых поступают крайне редко. Подмножество **целевых** квадратов имеет `is_target=1` в таблице. Для тестовых квадратов значения `calls_*` и `is_target` скрыты.

На карте обозначены квадраты трех типов:
- <span style="color: green;">Зеленые</span> — из обучающей части, не целевые
- <span style="color: red;">Красные</span> — из обучающей части, целевые
- <span style="color: blue;">Синие</span> — тестовые, на них необходимо построить прогноз


<img src="geohack_zones.png" width="300">

[Визуализация в браузере](geohack_zones.html)

## Оценка качества

В качестве решения необходимо предоставить CSV таблицу с предсказаниями для всех тестовых квадратов, для каждого квадрата — по всем дням недели.

|zone_id|calls_wd0|calls_wd1|calls_wd2|calls_wd3|calls_wd4|calls_wd5|calls_wd6|
|-------|---------|---------|---------|---------|---------|---------|---------|
| 79    | 0.825861| 0.670869| 0.786908| 0.598091| 1.247591| 0.675773| 0.633927|
| ...   | ...     | ...     | ...     | ...     | ...     | ...     | ...     |

Качество оценивается только по подмножеству **целевых** квадратов. Участникам неизвестно, какие из квадратов целевые, однако принцип выбора целевых квадратов в обучающей и тестовой части — идентичен. 

Во время соревнования качество оценивается на 30% тестовых целевых квадратов (выбраны случайно), в конце соревнования итоги подводятся по оставшимся 70% квадратов.

Метрика качества предсказаний — коэффициент ранговой корреляции Кендалла ([Kendall's tau](https://en.wikipedia.org/wiki/Kendall_rank_correlation_coefficient)), считается как доля пар объектов с неправильно упорядоченными предсказаниями:

$$\text{Kendall-}\tau = \frac{1}{n(n-1)} \sum_{i \neq j} \text{sgn}(y_i-y_j) \cdot \text{sgn}(\hat{y_j}-\hat{y_j})$$

Метрика оценивает порядок, в котором предсказания соотносятся друг с другом, а не их точные значения. Разные дни недели считаются независимыми элементами выборки, т.е. коэффициент корреляции считается по предсказаниям для всех тестовых пар `(zone_id, день недели)` (см. далее пример оценки качества на валидации).

В тестирующей системе используется реализация Kendall's tau из пакета SciPy: [`scipy.stats.kendalltau`](https://docs.scipy.org/doc/scipy-0.19.1/reference/generated/scipy.stats.kendalltau.html).

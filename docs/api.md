# The API calls

The notebook uses two public World Bank APIs. Neither needs a key or an account. This page lists every request it makes, what the parameters mean and what comes back.

| API | Base address | Used for |
|---|---|---|
| World Bank Data360 | `https://data360api.worldbank.org/data360` | The indicators and their values |
| World Bank Open Data, version 2 | `https://api.worldbank.org/v2` | Each country's region and income group |

## Data360 API

Data360 holds the Enterprise Surveys as the dataset `WB_ES`.

### How the indicator codes are built

| Part of the code | Meaning | Example |
|---|---|---|
| `WB_ES_T_` | All firms (the national figure) | `WB_ES_T_GEND4` |
| `WB_ES_M_` | Manufacturing firms only | `WB_ES_M_GEND4` |
| `WB_ES_S_` | Services firms only | `WB_ES_S_GEND4` |
| Letters after the prefix | The family of the indicator, which gives its topic | `GEND` is Gender, `FIN` is Finance |
| `BREADY_` | An indicator added for the Business Ready project | `WB_ES_T_BREADY_TAX2` |
| `_MEDIAN` | The indicator is a median, not an average | `WB_ES_T_BREADY_TAX4_MEDIAN` |

The dataset has 585 series: 198 national (`T`), 198 for manufacturing and 188 for services. The notebook uses the national series.

### 1. List every indicator, with its name and unit

```
POST https://data360api.worldbank.org/data360/searchv2
```

Request body:

```json
{
  "count": true,
  "search": "*",
  "top": 1000,
  "skip": 0,
  "select": "series_description/idno, series_description/name, series_description/measurement_unit",
  "filter": "series_description/database_id eq 'WB_ES'"
}
```

| Field | Meaning |
|---|---|
| `filter` | Keeps the series that belong to the Enterprise Surveys dataset |
| `select` | The fields to return: the code, the name and the unit |
| `top`, `skip` | Page size and starting row. One page of 1,000 holds the whole dataset |
| `count` | Asks for the total number of matches, returned as `@odata.count` |

What comes back (shortened):

```json
{
  "@odata.count": 585,
  "value": [
    {
      "series_description": {
        "idno": "WB_ES_T_PERF1",
        "name": "Real annual sales growth (%)",
        "measurement_unit": "Percentage"
      }
    }
  ]
}
```

The same request in Python:

```python
import requests

answer = requests.post(
    "https://data360api.worldbank.org/data360/searchv2",
    json={"count": True, "search": "*", "top": 1000, "skip": 0,
          "select": "series_description/idno, series_description/name, series_description/measurement_unit",
          "filter": "series_description/database_id eq 'WB_ES'"},
    timeout=120,
).json()
print(answer["@odata.count"], "series")
```

### 2. Values of one indicator for every country and survey year

```
GET https://data360api.worldbank.org/data360/data
```

| Parameter | Value | Meaning |
|---|---|---|
| `DATABASE_ID` | `WB_ES` | The Enterprise Surveys dataset |
| `INDICATOR` | for example `WB_ES_T_GEND4` | The indicator code |
| `COMP_BREAKDOWN_1` | `_T` | Sector breakdown. `_T` means all sectors |
| `COMP_BREAKDOWN_2` | `_T` | Firm-size breakdown. `_T` means all sizes |
| `COMP_BREAKDOWN_3` | `_T` | Other breakdowns (such as exporter status or gender of the top manager). `_T` means all firms |
| `REF_AREA` | optional, for example `GTM` or `GTM,HND` | Limits the answer to these countries. Left out, the answer covers every country |
| `skip` | `0`, `1000`, ... | Starting row. A call returns at most 1,000 rows |

Setting the three breakdowns to `_T` returns one value per country and survey year: the figure for all firms. Without them, the answer also holds a row for each size class, sector and other subgroup.

Example:

```
https://data360api.worldbank.org/data360/data?DATABASE_ID=WB_ES&INDICATOR=WB_ES_T_GEND4&REF_AREA=GTM&COMP_BREAKDOWN_1=_T&COMP_BREAKDOWN_2=_T&COMP_BREAKDOWN_3=_T&skip=0
```

What comes back (shortened to the fields the notebook reads):

```json
{
  "count": 3,
  "value": [
    {
      "INDICATOR": "WB_ES_T_GEND4",
      "REF_AREA": "GTM",
      "TIME_PERIOD": "2025.0",
      "OBS_VALUE": "21.526268",
      "COMP_BREAKDOWN_1": "_T",
      "COMP_BREAKDOWN_2": "_T",
      "COMP_BREAKDOWN_3": "_T"
    }
  ]
}
```

Things to know when reading the answer:

- `TIME_PERIOD` is the survey year written as text with a decimal (`"2025.0"`). The notebook converts it to a whole number.
- `OBS_VALUE` is text. The notebook converts it to a number and skips empty values.
- `count` is the total number of rows that match. The notebook keeps calling with a larger `skip` until it has them all.
- `REF_AREA` is a three-letter ISO country code. The notebook keeps only the codes that are in the Open Data API's country list, so anything that is not a country is dropped.
- There is no topic field. The notebook assigns the topic from the indicator code.

### How the notebook calls it

- One call per indicator, four calls at a time, so about 200 calls in a run.
- A call that fails or times out is tried again after 5, 20 and 45 seconds. After four failures the run stops with a message.
- Survey years before the first year set in the notebook (2010 by default) are dropped after collection.

## Open Data API

### 3. Country list with region and income group

```
GET https://api.worldbank.org/v2/country?format=json&per_page=400
```

The answer is a list of two parts: paging information, then the records. Each record has the country's ISO code (`id`), its `name`, its `region` and its `incomeLevel`. Regional and income aggregates such as "World" are in the same list with the region value `Aggregates`, and the notebook drops them.

## The AI model

The notebook sends one prompt to the model built into Google Colab:

```python
from google.colab import ai

answer = ai.generate_text(prompt)
```

This needs no key and works only inside Colab. The prompt holds the instructions and the table of facts, one line per indicator. The model is asked to return a JSON list of points, each with a headline, a text, the indicator codes it uses and a note on what to verify. The full prompt and answer of every run are saved in `ai_log_<code>.json`.

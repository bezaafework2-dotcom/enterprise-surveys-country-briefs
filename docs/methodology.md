# Methodology

## Data source

The notebook uses the World Bank Data360 API (`https://data360api.worldbank.org/data360`), which carries the full Enterprise Surveys dataset (`WB_ES`), and the World Bank Open Data API (`https://api.worldbank.org/v2`) for each country's region and income group. Both are public and need no key. The exact requests are in [`api.md`](api.md).

## Steps

1. Ask the Data360 API for every indicator in the Enterprise Surveys dataset, with its name and unit. Keep the national series (codes starting `WB_ES_T_`, about 200); the `WB_ES_M_` and `WB_ES_S_` series repeat them for manufacturing and services. Assign each indicator its topic from its code, following the Enterprise Surveys *Indicator Descriptions* document (26 October 2026): a family of codes maps to a topic (`FIN` to Finance), with a short list of exceptions (for example `T3`, capacity utilization, sits under Firm Profile, and `REG2`, `REG4`, `REG5` and `REG6` sit under Taxes).
2. Collect every indicator for all countries and survey years. The three breakdown filters set to `_T` keep the all-firms figure and leave out the splits by size and sector. Drop survey years before the first year set in the notebook.
3. Read the choices in the first cell. Topics and group names are matched without regard to capitals, and part of a name is enough if it fits only one. Indicators are matched by code first, then by words in the name. A name that fits nothing, or fits several groups, stops the run with a message listing the valid names.
4. Build one row of facts per indicator for the chosen country: latest value and year, previous survey value and year, the change between them, one column for each comparator (a comparator country's latest value and its survey year, or the figure for a group of countries), the gap to each comparator, and the country's rank within its own region. For the 15 biggest-obstacle indicators, also rank the obstacles within the country and confirm that the shares add up to about 100%.
   A group figure is the simple average of the latest values of its countries, which is how the Enterprise Surveys describe their regional and income-group averages. The median can be chosen instead.
5. Add rule-based flags: among the highest or lowest in the region (top or bottom 15% by rank, when the region has at least five countries), a large change since the previous survey (at least 30%, and at least 5 points for percentages), a value that more than doubled or halved, and a value that comes from an older survey than the country's latest.
6. Keep the indicators the run looks at: every indicator by default, or the listed topics and indicators. Send that table, one line per indicator, to the AI model with instructions to choose the most interesting points, use only numbers that appear in the table, avoid claims about causes, and say what should be verified when a change looks implausibly large. The model returns a list of points in JSON.
7. Check the answer. Every number in every point is extracted and compared with the table rows of the indicators that point says it uses (exactly, or rounded to a whole number). A point is dropped if it contains a number that is not in those rows, cites an indicator code that is not in the table, or cites no indicator.
8. Choose the points. By default the brief uses the AI points that passed, up to the number set in the notebook. When topics are listed, it uses one point for each topic: the first AI point on that topic that passed, or a point written by rule from the table when there is none. If no AI point passed, or no model is available, all points are written by rule. The page and the workbook say how the points were written.
9. Draw the page, with one panel per indicator: the indicators chosen by name first, then those the points use. Each panel shows the country's latest and previous survey and every comparator that has a value.
10. Write the Excel workbook: the points, the data behind the figure, the full table for the indicators the run looked at, the three most unusual indicators in every topic, the list of all indicators with their codes, and the checks on the AI's text.
11. Save a log (`ai_log_<code>.json`) with the settings, the prompt, the model's answer, each check and the points used.

**What the check does not catch.** It checks numbers written as digits. A wrong number passes if it happens to equal another number in the same indicator's row. It does not check numbers written as words ("one in five"), or whether a sentence describes a number correctly (for example "up" when the value fell, or a negative value written as positive). The page should still be read by a person before it is used.

## Checks

| Check | Action |
|---|---|
| The indicator list comes back empty | Stop with a message |
| Country code is not in the API's country list | Stop with a message |
| No indicator values for the country | Stop with a message |
| More than one value for a country, indicator and year | Stop with a message |
| A topic, group or indicator in the settings matches nothing, or a group name fits several | Stop with a message listing the valid names |
| The biggest-obstacle shares do not add up to about 100% (97 to 103) | Print a warning |
| A value comes from an older survey than the country's latest | Flag it in the table, so the AI model and the reader can see it |
| A value more than doubled or halved since the previous survey | Flag it as "verify before using" |
| A number in an AI point is not in the rows of the indicators it lists | Drop the point |
| An AI point cites a code that is not in the table, or no code | Drop the point |
| The AI's answer cannot be read, or no model is available | Write the points by rule and say so on the page |
| An API is slow or does not answer | Retry up to four times, then stop with a message |

## Limitations

- Survey years differ between countries, so a comparison can set a 2025 survey against an older one.
- Group averages and medians give every country the same weight, whatever the size of its economy, and they are not official World Bank aggregates.
- Only surveys that follow the Enterprise Surveys Global Methodology are comparable across countries and years. The notebook uses every survey the API returns and does not separate the two.
- The notebook uses the national figures. The API also holds breakdowns by firm size and sector, which the notebook does not use.
- The AI step runs only in Google Colab. Its answers can differ from one run to the next, which is why every run saves a log.

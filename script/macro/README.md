# **Macros**

## electricity.jinja
This macro can, based on sensors with known energy prices, calculate when, within a given timeframe, the
cheapest /most expensive period will occur 

### Basic use
The makro can be called adding just the mySensor_today parameter. Any missing parameter will have a default
value. Paramaters can be added in any order providing their parameter name is used. If parametername is omitted,
the correct order of parameters is required to ensure correct parsing. E.g.:</br>
  {{- PeriodPrice("sensor.edssensor", durationTimedelta=timedelta(minutes=90)) -}} 

### Parameters
Required = *
| Parameter | Description |
|-----------|-------------|
| * mySensor_today       |(String) Name of the sensor (EDS or any similar type of sensor) that contains pricing data|
| edsearliestDatetime    |(DateTime) Exact date/time of start of window during which electricity will be used|
| latestDatetime         |(DateTime) Exact date/time of end of window during which electricity will be used|
| durationTimedelta      |(TimeDelta) Duration of electricity usage formatted as a TimeDelta|
| returnFirstBool        |(Boolean) Default to true to use first occurrence of the lowest price, otherwise use the last|
| timeAdherenceStr       |(String) Influences the behaviour when seeking low / high cost</br></br>**default** = adjusts time to 'now' if time is in the past</br>**strict**  = do not adjust time and return empty result if window is in the past</br>**forced**  = do not adjust time and return result even if window is in the past|
| attr_today_arr         |(String) Name of attribute for today data - organized in pairs of time + price|
| attr_tomorrow_arr      |(String) Name of attribute for tomoows data (if any) - organized in pairs of time + price|
| attr_forecast_arr      |(String) Name of attribute for forecast data after tomorrow (if any) - organized in pairs of time + price|
| timeTagStr             |(String) Name attribute (in the pair) containng the time|
| priceTagStr            |(String) Name attribute (in the pair) containng the price|
| defaultDurationMinNum  |(Number) Default minimum duration of any period|
| defaultPeriodHrsNum    |(Number) The default duration of hour to look for cheapest prices|
| mode                   |default / details /</br>cheapPrice / cheapStart / cheapStart / isCheapNow</br>expensivePrice / expensiveStart / expensiveStart / isExpensiveNow|
| hint                   |String to be returned as part of the result. This could (e.g.) be the name of the integration providing data|
| slices                 |Number (1-60) of an hour if the prices retrieved from mySensor is less than an hour (e.g. 30mins), then the acceptable range will also be reduced.|
| weights                |set [] of number of kWh expected consumed (during each slice.|

### Returns
Macro returns a STRING(!) based on the MODE setting. In case of a mode of default or details, the returned string will
be json formatted and must be converted using the from_json filter.

When mode is default or details, the returned value (passed throught the from_json filter may contain following fields:

"*" = only included if a corresponding period was found
| Parameter | Description |
|-----------|-------------|
| * cheapPriceStartTime      | Datetime string of when cheapest time starts|
| * cheapPriceEndTime        | Datetime string of when cheapest time ends |
| cheapPrice                 | Price if 1 kW is used each hour during cheapest time. None if no period found |
| * expensivePriceStartTime  | Datetime string of when most expensive time starts |
| * expensivePriceEndTime    | Datetime string of when most expensive time ends |
| expensivePrice             | Price if 1 kW is used each hour during most expensive time. None if no period found |
| earliestDatetime           | Datetime string of the earliest time used when looking for cheap / expensive |
| latestDatetime             | Datetime string of the latest time used when looking for cheap / expensive |
| duration                   | The duration of the time window looked for (in minutes) |
| isCheapNow                 | none if no cheap time period found, otherwise true / false dependent on whether right now is the cheapest period |
| isExpensiveNow             | none if no expensive time period found, otherwise true / false dependent on whether right now is the most expensive period |
| mode                       | Name of the mode used |
| cheapWindow                | Explaination for calculation to reach cheapest price.</br>Set of [{ start, minutes, kWh_price, est_kWh },...] covering the whole duration. Start defines when each subset starts, length in minutes of the subset, kWh_price is the corresponding price and est_kWh is the part of the corresponding slice|
| expensiveWindow            | Explaination for calculation to reach most expensive price.</br>Set of [{ start, minutes, kWh_price, est_kWh },...] covering the whole duration. Start defines when each subset starts, length in minutes of the subset, kWh_price is the corresponding price and est_kWh is the part of the corresponding slice|
| status                     | Result of the operation. If not 'ok', then this is an warning / error |
| hint                       | none or string as provided when  macro was called |
</br>
</br>
For any other mode, the returned value will be a string with the following content:

| Mode      | Description |
|-----------|-------------|
|cheapPrice     | String value of price if 1 kW is used each hour during cheapest time. If weights are supplied, the amount of kWh is calculated from weights. None if no period found |
|cheapStart     | Datetime string of when cheapest time starts |
|cheapEnd       | Datetime string of when cheapest time ends |
|expensivePrice | String value of price if 1 kW is used each hour during most expensive time. If weights are supplied, the amount of kWh is calculated from weights. None if no period found |
|expensiveStart | Datetime string of when most expensive time starts |
|expensiveEnd   | Datetime string of when most expensive time ends |

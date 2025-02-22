# electricity.jinja
This macro can, based on sensors with known energy prices, calculate when, within a given timeframe, the
cheapest /most expensive period will occur 

## Basic use
The makro can be called adding just the mySensor_today parameter. Any missing parameter will have a default
value. Paramaters can be added in any order providing their parameter name is used. If parametername is omitted,
the correct order of parameters is required to ensure correct parsing. E.g.:<br/>
  {{- PeriodPrice("sensor.edssensor", durationTimedelta=timedelta(minutes=90)) -}} 

## Parameters
Required = *
| Parameter | Description |
|-----------|-------------|
| * mySensor_today       |(String) Name of the sensor (EDS or any similar type of sensor) that contains pricing data|
| edsearliestDatetime    |(DateTime) Exact date/time of start of window during which electricity will be used|
| latestDatetime         |(DateTime) Exact date/time of end of window during which electricity will be used|
| durationTimedelta      |(TimeDelta) Duration of electricity usage formatted as a TimeDelta|
| returnFirstBool        |(Boolean) Default to true to use first occurrence of the lowest price, otherwise use the last|
| timeAdherenceStr       |(String) Influences the behaviour when seeking low / high cost<br/><br/>**default** = adjusts time to 'now' if time is in the past<br/>**strict**  = do not adjust time and return empty result if window is in the past<br/>**forced**  = do not adjust time and return result even if window is in the past|
| attr_today_arr         |(String) Name of attribute for today data - organized in pairs of time + price|
| attr_tomorrow_arr      |(String) Name of attribute for tomoows data (if any) - organized in pairs of time + price|
| attr_forecast_arr      |(String) Name of attribute for forecast data after tomorrow (if any) - organized in pairs of time + price|
| mySensor_tomorrow      |(String) Name of the sensor that contains pricing data for tomorrow. Default to mySensor_today|
| mySensor_forecast      |(String) Name of the sensor that contains pricing data for forecast. Default to mySensor_today|
| timeTagStr             |(String) Name attribute (in the pair) containng the time|
| priceTagStr            |(String) Name attribute (in the pair) containng the price|
| defaultDurationMinNum  |(Number) Default minimum duration of any period|
| defaultPeriodHrsNum    |(Number) The default duration of hour to look for cheapest prices|
| mode                   |default / details /<br/>cheapPrice / cheapStart / cheapStart / isCheapNow<br/>expensivePrice / expensiveStart / expensiveStart / isExpensiveNow|
| hint                   |String to be returned as part of the result. This could (e.g.) be the name of the integration providing data|
| slices                 |Number (1-60) of an hour if the prices retrieved from mySensor is less than an hour (e.g. 30mins), then the acceptable range will also be reduced.|
| kwh_usage              |set [] of number, or total number of kWh expected consumed. If only one number is provided, this is assumed to be be total consumption and will be distributed evenly across duration.|

## Returns
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
| cheapWindow                | Explaination for calculation to reach cheapest price.<br/>Set of [{ start, minutes, kWh_price, est_kWh },...] covering the whole duration. Start defines when each subset starts, length in minutes of the subset, kWh_price is the corresponding price and est_kWh is the kWh part of the corresponding slice. kWh_price * est_kWh is the estimated cost for the slice.|
| expensiveWindow            | Explaination for calculation to reach most expensive price.<br/>Set of [{ start, minutes, kWh_price, est_kWh },...] covering the whole duration. Start defines when each subset starts, length in minutes of the subset, kWh_price is the corresponding price and est_kWh is the kWh part of the corresponding slice. kWh_price * est_kWh is the estimated cost for the slice.|
| status                     | Result of the operation. If not 'ok', then this is an warning / error |
| hint                       | none or string as provided when  macro was called |
<br/>
<br/>
For any other mode, the returned value will be a string with the following content:

| Mode      | Description |
|-----------|-------------|
|cheapPrice     | String value of price if 1 kW is used each hour during cheapest time. If kwh_usage is supplied, the amount of kWh is calculated from this. None if no period found |
|cheapStart     | Datetime string of when cheapest time starts |
|cheapEnd       | Datetime string of when cheapest time ends |
|expensivePrice | String value of price if 1 kW is used each hour during most expensive time. If kwh_usage are supplied, the amount of kWh is calculated from this. None if no period found |
|expensiveStart | Datetime string of when most expensive time starts |
|expensiveEnd   | Datetime string of when most expensive time ends |

## Advanced use
### Multiple sensors
The macro has the ability to get day-ahead pricing for today / tomorrow from two sensors. I.e. sensor1 can provide todays pricing, and sensor2 can provide tomorrows; e.g. this could be the case when using [Strømligning](https://github.com/MTrab/stromligning). If forecast data is available (e.g. you have a sensor that pulls data from [Carnots API](https://www.carnot.dk/), then this data can also be used. However, there are some requirements that must be met:
- Data obtained from these (up to) three sensors may not overlap
- The prices are valid for the same duration (e.g. hour)
- The name (entity_id) of the sensor must be listed in mySensor_today / mySensor_tomorrow / mySensor_forecast. Note:
  - if mySensor_forecast is set to empty, forecast data is ignored
  - if mySensor_tomorrow is set to empty, both data for tomorrow as well as forecast are ignored
  - Default value of mySensor_tomorrow / mySensor_forecast is the value of mySensor_today
- In the entity, an attribute is expected named attr_today_arr / attr_tomorow_arr / attr_forecast_arr. The attribute must exist in the corresponding mySensor_today / mySensor_tomorrow / mySensor_forecast
- Each of the attributes must contain pairs named alike. The default names are 'hour' and 'price' and can be overridden using the parameters timeTagStr and priceTagStr

### Search mode
timeAdherenceStr controls how the macro uses earliestDatetime / latestDatetime. 
- If timeAdherenceStr is ommitted or set to 'default', the macro will adjust earliestDatetime / latestDatetime to match a earliestDatetime equal to <ins>now</ins>.
- If set to strict, the earliestDatetime / latestDatetime will not be adjusted. If matching time window is found, the information will only be returned if this is in the present. Otherwise an empty set is returned.
- If set to forced, the earliestDatetime / latestDatetime will not be adjusted. Information will be returned irrespectively of this being in the past or present.

### Data returned
returnFirstBool will define (if true) to return the earliest possible match. If false, the latest match will be returned.

### Prices
By default the macro will detect for how long the prices are valid. This is done by taking the time difference of two first prices found. Should only one price be found, 1 hour is assumed. This allows for a 
later adjustment in the pricing mode (e.g. pricing per half hour instead of hourly) without having to change the macro.

### Slicing / kwh_usage
If data is available on how much power (normally) is expected consumed, kwh_usage can be added when calling the macro. To allow some variation, an hour is divided into smaller parts (slices). Each part of the kwh_usage list is
attributed to a slice, and is only used once during a calculation. If a number is provided instead of a list, the number is assumed to be the total consumption and will be distributed evenly across the duration.<br/>
The usageWindow must be a whole number between 1 and 60 minutes. This duration is applied to each of the slices listed in kwh_usage. If the total usageWindow for all slices is less than the duration time, an error will be returned. If kwh_usage is a number instead of a list, the macro will create its own kwh_usage list and spread the provided kWh evenly across the duration (considering the usageWindow requested). If kwh_usage is empty, a hourly usage of 1 kWh is assumed. If no usageWindow is provided, a window of 60 minutes is assumed.
| usageWincow | kwh_usage | Description |
|-------------|-----------|-------------|
| included (1-60) | number | <ins>duration</ins> is interrnally split into small slices with the same length as <ins>usage_window</ins> and <ins>kwh_usage</ins> is split equally amongst these slices|
| included (1-60) | list   | If <ins>usageWindow</ins> multiplied by length of <ins>kwh_usage/ins> does not exceed duration, an error is raised|
| included (1-60) | not included | If <ins>usageWindow</ins> multiplied by length of <ins>kwh_usage</ins> does not exceed duration, an error is raised. Hourly usage is assumed to be one 1Kw|
| not included | list   | If <ins>usageWindow</ins> multiplied by default length of <ins>kwh_usage/ins> (60 minss) does not exceed duration, an error is raised|
| not included | number | <ins>kwh_usage</ins> is assumed evenly distributed across <ins>duration</ins>. An internal split of <ins>kwh_usage</ins> is created based on the default length of <ins>usageWindow</ins>.|
| not included | not included | An hourly y usage of one 1Kw is assumed |

**Note:**<br/>
For each hour there are up to three points upon calculation is based; first minute of the hour, the current minute and the last minute that is (duration % hour) before the hour. The majority of iterations will have 2 or (on rare occassions) 1 data points. If a low usageWindow is applied (eg. 5), this means that if seeking a one hour window, there for every 24 hrs will up to 48 * (12+2 - the 2 is for the outliers) * 12 = 8.064 calculations to be done to find the lowest / highest price. Extending the period from 2 days (today + tomorrow) to also include a 7 days forecast means that every time a sensor needs to be updated, +72.576 calculations are made. Thus, it is important to consider how small a usageWindow is needed when extending the lastDatetime beyond 48 hours as well how long a period of cheap / expensive price should be looked for. Remember to also take the type of hardware on which Home Assistant is running into account. If in doubt, test the configuration in _Template_ under _Developer tools_ by addion a _{#-% from 'Electricity.jinja' import PeriodPrice %-#}_ in front of the code (see examples below) to get an impression of how the macro performs. 


## Code examples
### Dishwasher must be run at cheapest hour, but be completed by 06:00 tomorrow
            {% from 'Electricity.jinja' import PeriodPrice %}
            {%- set edsSensor         = "sensor.energidataservice" -%}
            {%- set earliestDatetime  = now() -%}
            {#- Add 20min for the normal cycle of dishwasher cooling off and opening door -#}
            {%- set durationTimedelta =  timedelta(minutes=20 + states('sensor.dishwasher_remaining_time') | int(90)) -%}
            {#- Dishwasher is precoded to start 04:30 at the latest, thus providing 'failsafe' -#}
            {%- set latestDatetime    = earliestDatetime + timedelta(minutes=states('sensor.dishwasher_start_time') | int(8*60)) + durationTimedelta -%}
            {#- Get data from EnergiDataService, ignore forecasted values -#}
            {{- PeriodPrice(edsSensor, earliestDatetime, latestDatetime, durationTimedelta,attr_forecast_arr="",hint="Energi Data Service") | from_json -}}
              
### Dishwasher must be run at cheapest hour, but be completed by 06:00 tomorrow. Cascade two integrations for data
            {% from 'Electricity.jinja' import PeriodPrice %}
            {%- set edsSensor         = "sensor.energidataservice" -%}
            {%- set earliestDatetime  = now() -%}
            {#- Add 20min for the normal cycle of dishwasher cooling off and opening door -#}
            {%- set durationTimedelta =  timedelta(minutes=20 + states('sensor.dishwasher_remaining_time') | int(90)) -%}
            {#- Dishwasher is precoded to start 04:30 at the latest, thus providing 'failsafe' -#}
            {%- set latestDatetime    = earliestDatetime + timedelta(minutes=states('sensor.dishwasher_start_time') | int(8*60)) + durationTimedelta -%}

            {#- Get data from EnergiDataService, ignore forecasted values -#}
            {%- set resultEDS = PeriodPrice(edsSensor, earliestDatetime, latestDatetime, durationTimedelta,attr_forecast_arr="",hint="Energi Data Service") | from_json -%}
            {%- if resultEDS.cheapPrice != none and resultEDS.latestDatetime >= latestDatetime |string -%}
              {{ resultEDS }}
            {%- else -%}
              {%- set slSensor = "sensor.stromligning_current_price_vat" %}
              {%- set slSensor_tomorrow= "binary_sensor.stromligning_tomorrow_available_vat" %}
              {%- set resultSL = PeriodPrice(slSensor, earliestDatetime, latestDatetime, durationTimedelta,attr_today_arr="prices",timeTagStr="start",mySensor_tomorrow=slSensor_tomorrow,attr_tomorrow_arr="prices",hint="Strømligning") | from_json %}
              {%- if resultSL.cheapPrice != none and resultSL.latestDatetime >= latestDatetime |string -%}
                {{ resultSL }}
              {%- else -%}
                {#- Last resort. If neither integration returns usable data, just use what EnergiDataService offered -#}
                {{ resultEDS }}
              {%- endif -%}
            {%- endif -%}

The last example can be extended by using additional integrations (e.g. Nordpool).


### Get cheapest hour within the next 12 hours
            {%- from 'Electricity.jinja' import PeriodPrice -%}
            {%- set edsSensor           = "sensor.energidataservice" -%}
            {%- set earliestDatetime    = now() -%}
            {%- set latestDatetime      = earliestDatetime + timedelta(hours=12) -%}
            {%- set durationTimedelta   = timedelta(hours=1 ) -%}
            {#- Get data from EnergiDataService, ignore forecasted values -#}
            {{ PeriodPrice(edsSensor, earliestDatetime, latestDatetime, durationTimedelta,attr_forecast_arr="",hint="Energi Data Service") | from_json -}}

### Get cheapest period within the next 12 hours, where power consumption expected is 1.2kWh, distributed (in 15 minute intervals) to be 0.4kWh, 0.3kWh, 0.18kWh, 0.17kWh, 0.1 kWh and 0.05kWh.
            {%- from 'Electricity.jinja' import PeriodPrice -%}
            {%- set edsSensor           = "sensor.energidataservice" -%}
            {%- set earliestDatetime    = now() -%}
            {%- set latestDatetime      = earliestDatetime + timedelta(hours=12) -%}
            {%- set durationTimedelta   = timedelta(minutes=90 ) -%}
            {#- Get data from EnergiDataService, ignore forecasted values -#}
            {{ PeriodPrice(edsSensor, earliestDatetime, latestDatetime, durationTimedelta,attr_forecast_arr="", usageWindow=15, kwh_usage=[0.4,0.3,0.18,0.17,0.1,0.05], hint="Energi Data Service") | from_json -}}

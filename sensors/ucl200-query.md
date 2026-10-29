# Tektelic

from(bucket: "UCL200")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["device_id"] == "tektelic-a-2145a0081")
  |> filter(fn: (r) => r["_field"] == "ambient_temperature" or r["_field"] == "relative_humidity" or r["_field"] == "input1_frequency" or r["_field"] == "input2_voltage" or r["_field"] == "light_intensity" or r["_field"] == "raw")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
  |> yield(name: "last")


# VS121 People Counter (Line Crossing & Occupancy)

from(bucket: "UCL200")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "vs121_occupancy")
  |> filter(fn: (r) => r["device_id"] == "ucl200-vs121-d")
  |> filter(fn: (r) => r["_field"] == "period_in" or r["_field"] == "period_out" or r["_field"] == "total_in" or r["_field"] == "total_out")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
  |> yield(name: "last")

# Weather Station (Ecowitt GW2000)

from(bucket: "UCL200")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "weather_ecowitt")
  |> filter(fn: (r) => r["device_id"] == "ucl200-weather-ecowitt")
  |> filter(fn: (r) => r["_field"] == "outTemp_C" or r["_field"] == "outHumidity" or r["_field"] == "windSpeed_kph" or r["_field"] == "rain_cm" or r["_field"] == "radiation_Wpm2")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
  |> yield(name: "last")

# Tektelic

from(bucket: "UCL200")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["device_id"] == "tektelic-a-2145a0081")
  |> filter(fn: (r) => r["_field"] == "ambient_temperature" or r["_field"] == "relative_humidity" or r["_field"] == "input1_frequency" or r["_field"] == "input2_voltage" or r["_field"] == "light_intensity" or r["_field"] == "raw")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
  |> yield(name: "last")


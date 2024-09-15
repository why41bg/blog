---
title: "Go基本类型互转模版" 
date: 2024-08-05T12:25:07+08:00
draft: false
tags:
  - algorithm
ShowToc: true
TocOpen: false 
---

# Go基本类型互转模版

```go
func intToString(a int) string {
	return strconv.Itoa(a)
}

func intToBinString(a int) string {
	return strconv.FormatInt(int64(a), 2)
}

func int64ToString(a int64) string {
	return strconv.FormatInt(a, 10)
}

func int64ToBinString(a int64) string {
	return strconv.FormatInt(a, 2)
}

func float32ToString(a float32) string {
	return strconv.FormatFloat(float64(a), 'f', -1, 32)
}

func float64ToString(a float64) string {
	return strconv.FormatFloat(a, 'f', -1, 64)
}

func stringToInt(a string) int {
	i, _ := strconv.Atoi(a)
	return i
}

func stringToInt64(a string) int64 {
	i, _ := strconv.ParseInt(a, 10, 64)
	return i
}

func stringToFloat32(a string) float32 {
	f, _ := strconv.ParseFloat(a, 32)
	return float32(f)
}

func stringToFloat64(a string) float64 {
	f, _ := strconv.ParseFloat(a, 64)
	return f
}

func boolToInt(a bool) int {
	if a {
		return 1
	}
	return 0
}

func intToBool(a int) bool {
	if a == 0 {
		return false
	}
	return true
}
```




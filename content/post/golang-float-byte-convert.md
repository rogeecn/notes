---
title: "golang float32/64与[]byte互转"
date: 2021-11-26T16:28:40+08:00
draft: false
tags: ["Golang"]
---
```go
import (
    "encoding/binary"
    "math"
)
 
func Float32ToByte(float float32) []byte {
    bits := math.Float32bits(float)
    bytes := make([]byte, 4)
    binary.LittleEndian.PutUint32(bytes, bits)
 
    return bytes
}
 
func ByteToFloat32(bytes []byte) float32 {
    bits := binary.LittleEndian.Uint32(bytes)
 
    return math.Float32frombits(bits)
}
 
func Float64ToByte(float float64) []byte {
    bits := math.Float64bits(float)
    bytes := make([]byte, 8)
    binary.LittleEndian.PutUint64(bytes, bits)
 
    return bytes
}
 
func ByteToFloat64(bytes []byte) float64 {
    bits := binary.LittleEndian.Uint64(bytes)
 
    return math.Float64frombits(bits)
}
```
<!--more-->

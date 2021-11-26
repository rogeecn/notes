---
title: "Sim900A短信收发"
date: 2021-11-26T16:21:54+08:00
draft: false
tags: ["Sim900a","短信"]
---
Sim900A短信收发集成代码
<!--more-->
```go
package main

import(
    "fmt"
    "time"
    "github.com/xlab/at"
    //"github.com/xlab/at/sms"
)

const (
    CommandPort = "/dev/ttyUSB0"
    NotifyPort = "/dev/ttyUSB0"
)

var device *at.Device
func main(){
    fmt.Println("BEGIN...")
    device = &at.Device{
        CommandPort: CommandPort,
        NotifyPort: NotifyPort,
    }

    if err := device.Open(); err!=nil{
        fmt.Println("ERR:", err);
        return
    }

    if err := device.Init(at.DeviceSIM900A()); err != nil {
        fmt.Println("ERR:", err);
        return
    }
    defer device.Close()

    CNUM, err := device.Send("AT+CNUM")
    if err != nil {
        fmt.Println("ERR:", err);
        return
    }
    
    fmt.Println("CNUM: ", CNUM)
    fmt.Println("IMEI: ", device.State.IMEI)
    fmt.Println("ModelName: ", device.State.ModelName)
    fmt.Println("OperatorName: ", device.State.OperatorName)
    fmt.Println("SignalStrength: ", device.State.SignalStrength)

  _,err = device.Send("AT+CMGF=0")
    if err != nil {
        fmt.Println("ERR:", err);
        return
    }

  _,err = device.Send(`AT+CSCS="GSM"`)
    if err != nil {
        fmt.Println("ERR:", err);
        return
    }

    fmt.Println("pending send sms..." )
    //phoneNumber := sms.PhoneNumber("18601013734")
    //phoneSMSTxt := "你好小张！"
    //err = device.SendSMS(phoneSMSTxt,phoneNumber)
    //if err != nil {
        //fmt.Println("ERR:", err);
        //return
    //}
    fmt.Println("DONE")

  go checkIncomeMsg()
  for {
    time.Sleep(time.Second)
  }
}

func checkIncomeMsg(){
  fmt.Println("begin check...")
  t := time.NewTicker(time.Second*5)
  defer t.Stop()

  //balanceUSSD := "*100#"
  go func(){
    device.Watch()
  }()

  for {
    select {
      case <- device.Closed():
        fmt.Println("device closed...")
        return
      case msg,ok:= <-device.IncomingSms():
        fmt.Println("PONG: new message incomming...")
        if ok {
          //fmt.Printf("NEW MSG: %+v\n",msg)
          fmt.Println("=====================[ NEW MESSAGE ]==================")
          fmt.Println("From: ", msg.Address)
          fmt.Println("Time: ", time.Time(msg.ServiceCenterTime).Unix())
          fmt.Println("Text: ", msg.Text)
          fmt.Println("========================================================")
        }
      case <-t.C:
        continue
    }
  }
}

```
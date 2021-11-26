---
title: "Dart 解析 Laravel 数据结构"
date: 2021-11-26T16:02:38+08:00
draft: false
tags: ["Dart", "JSON", "Laravel"]
---

## 数据JSON内容：

```json
{
    "message": "json_message_data",
    "errors":{
        "name":[
            "name_message_nnn"
        ],
        "user":[
            "user_message"
        ]
    },
    "data":{
        "username":"Rogee"
    }
}
```
<!--more-->
## 解析结构：

```dart
import "dart:io";
import "dart:convert";

void main(args) {
  File file = new File("./dart.json");
  String jsonContent = file.readAsStringSync();

  Map jsonMap = json.decode(jsonContent);
  var resp = Response.fromJson(jsonMap);

  if (resp.errors.isNotEmpty) {
    print("json: " + resp.errors['name']?.first);
  } else {
    print("json: error is empty");
  }

  print("JSON: "+json.encode(resp.toJson()));
}

class Response {
  String message;
  Data data;
  Map<String, List<String>> errors = new Map<String, List<String>>();

  Response(this.message, this.errors);

  Response.fromJson(Map<String, dynamic> json)
      : message = json['message'],
        data = Data.fromJson(json['data']) {
    if (json["errors"] != null) {
      Map<String, dynamic>.from(json["errors"]).forEach((key, value) {
        errors.addAll({key: List<String>.from(value)});
      });
    }
  }

  Map<String, dynamic> toJson() => {
        'message': message,
        'errors': errors,
        'data': data,
      };
}

class Data {
  String username;

  Data(this.username);

  Data.fromJson(Map<String, dynamic> json) : username = json['username'];

  Map<String, dynamic> toJson() => {
        'username': username,
      };
}
```

## 使用泛型方式解析：

```
import "dart:io";
import "dart:convert";

void main(args) {
  File file = new File("./dart.json");
  String jsonContent = file.readAsStringSync();

  Map jsonMap = json.decode(jsonContent);

  //  NOTICE
  var resp = Response<Data>.fromJson(Data(), jsonMap);

  if (resp.errors.isNotEmpty) {
    print("json: " + resp.errors['name']?.first);
  } else {
    print("json: error is empty");
  }

  print("JSON: " + json.encode(resp.toJson()));
}

class Response<T extends IData> {
  String message;
  Map<String, List<String>> errors = new Map<String, List<String>>();
  T data;

  Response(this.message, this.errors, this.data);

  //  NOTICE
  Response.fromJson(T data, Map<String, dynamic> json) {
    message = json['message'];
    this.data = data.fromJson(json['data']);

    if (json["errors"] != null) {
      Map<String, dynamic>.from(json["errors"]).forEach((key, value) {
        errors.addAll({key: List<String>.from(value)});
      });
    }
  }

  Map<String, dynamic> toJson() => {
        'message': message,
        'errors': errors,
        'data': data,
      };
}

abstract class IData<T> {
  T fromJson(Map<String, dynamic> json);

  Map<String, dynamic> toJson();
}

class Data implements IData<Data> {
  String username;

  Data({this.username});

  Data fromJson(Map<String, dynamic> json) {
    return Data(username: username);
  }

  Map<String, dynamic> toJson() => {
        "usernmae": username,
      };
}
```

### 优化后的

```
import 'package:validators/validators.dart';

import 'json_convert_content.dart';

class HttpResponse<T extends JsonConvert> {
  String message = "";
  Map<String, List<String>> errors = new Map<String, List<String>>();
  T data;

  HttpResponse(this.message, this.errors, this.data);

  HttpResponse.fromJson(T data, Map<String, dynamic> json) {
    if (json['message'] != null) {
      message = json['message'];
    }

    if (json['data'] != null) {
      this.data = data.fromJson(json['data']);
    }

    if (json["errors"] != null) {
      Map<String, dynamic>.from(json["errors"]).forEach((key, value) {
        errors.addAll({key: List<String>.from(value)});
      });
    }
  }

  Map<String, dynamic> toJson() => {
        'message': message,
        'errors': errors,
        'data': data,
      };

  bool hasError() {
    return message.length > 0;
  }

  bool fieldHasError(String field) {
    if (!hasError()) {
      return false;
    }
    return errors.containsKey(field);
  }

  String firstFieldError(String field) {
    return fieldErrors(field).first;
  }

  List<String> fieldErrors(String field) {
    if (!hasError()) {
      return new List<String>();
    }

    if (!fieldHasError(field)) {
      return List<String>();
    }

    return errors[field];
  }

  String firstError() {
    if (!hasError()) {
      return "";
    }

    String firstError = "";
    bool canBreak = false;
    errors.forEach((key, value) {
      canBreak = true;
      if (!canBreak) {
        firstError = value.first;
      }
    });
    return firstError;
  }
}
```
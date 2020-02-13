# JSON-RPC API

## Основное
Большинство наших сервисов в качестве API используют формат [JSON-RPC 2.0](https://www.jsonrpc.org/specification). 

Атомарной единицей API сервиса является операция. 

Каждый сервис описывает свои операции в виде спецификации в формате [jsonschema draft-07](https://tools.ietf.org/html/draft-handrews-json-schema-01) с небольшими авторскими дополнениями.

## Формат запроса
Запрос осуществляется HTTP методом POST с указанием заголовка Content-Type: application/json. Тело запроса: 
```json5
{
  "jsonrpc": "2.0", // required, всегда 2.0
  "id": "e3690667-ad8f-48bf-be19-40cec933c05b", // required, всегда uuid version 4
  "method": "report.ready.index", // required, название операции, разделитель всегда точка
  "params": {
    // not required, список параметров операции в формате, описанном в секции request ее спецификации
  }
}
```

## Формат ответа
HTTP код ответа всегда 200. Успешность и неуспешность выполнения операции разруливается наличием result или error в ответе

Операция прошла успешно:
```json5
{
  "id": "e3690667-ad8f-48bf-be19-40cec933c05b", // required, всегда совпадает с id запроса
  "jsonrpc": "2.0", // required, всегда 2.0
  "result": {
    // required, результат выполнения операции в соответствии с секцией response её спецификации
  }
}
```

Операция неуспешна:
```json5
{
  "error": {
    "code": 4009, // required, бизнес код ошибки (например ошибка валидации)
    "data": [{"name": "Имя слишком короткое"}, {"city_id": "Город не найден"}], // not required, контекст ошибки
    "message": "Некоторые поля формы не прошли валидацию" // not required, текст ошибки
  },
  "id": "e3690667-ad8f-48bf-be19-40cec933c05b",
  "jsonrpc": "2.0"
}
```
Надо стараться не загрязнять вывод технической информацией типа stack trace  и т.д.

## Требования к сервисам
Наличие публичного endpoint по адресу: /api/jsonrpc
Наличие внутреннего endpoint (недоступного для запросов извне) /specs, который реализует как минимум операцию получения всех доступных операций: operation.all.
<details><summary>Пример</summary>
<p>

```json
{
  "id": "ab704833-7578-4b26-95b8-744a6f9afced",
  "jsonrpc": "2.0",
  "result": {
    "operation.authorize": {
      "request": {
        "type": "object",
        "required": ["operation_name"],
        "properties": {
          "operation_name": {"type": "string"},
          "user_id": {
            "format": "uuid",
            "pattern": "^[a-fA-F0-9]{8}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{12}$",
            "type": "string"
          }
        }
      },
      "response": {
        "authorized": {"type": "boolean"},
        "constraints": {"type": "object"}
      }
    }
  }
}
```
</p>
</details>
т.е. по сути возвращается словарь, где ключами являются названия операций, а значениями их спецификации.

## Версионирование API
`1.` Версионируется только мажорная версия API (мажорной является версия, которая ломает обратную совместимость)  
`2.` Версионирование осуществляется через роуты, используя соглашение  
`3.` Версионируется как публичный API так и внутренний. При этом внутренний API конкретной версии должен описывать операции публичного API той же самой версии.  
`4.` Нулевая версия доступна по адресам: /api/jsonrpc, /specs  
`5.` Последующие версии доступны по адресам /api/jsonrpc/v{N}, /specs/v{N}, где {N} - номер версии (v1, v2 и т. д.)  

## Требования к операциям
`1.` Каждая операция должна иметь спецификацию. Если для операции не описана спецификация, то её нет.  
`2.` Операция должна быть доступна в списке операций сервиса (см. выше endpoint /specs). Если операции нет в этом списке то её нет и на публичном API.  
`3.` Спецификация операции описывается с помощью нотации jsonschema draft-07. Структура спецификации операции следующая:  
```json
{
  "request": {
    "type": "object",
    "required": ["operation_name"],
    "properties": {
      "operation_name": {"type": "string"},
      "user_id": {
        "type": "string",
        "format": "uuid",
        "pattern": "^[a-fA-F0-9]{8}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{12}$"
      }
    }
  },
  "response": {
    "type": "object",
    "properties": {
      "authorized": {"type": "boolean"},
      "constraints": {"type": "object"}
    }
  }
}
```
<details><summary>Пример запроса этой операции</summary>
<p>

```json
{
  "id": "7154f067-2abf-4b4d-9fcd-dd4b939432b2",
  "jsonrpc": "2.0",
  "method": "operation.authorize",
  "params": {
    "operation_name": "issue.index",
    "user_id": "567048d5-7a08-482c-80cc-3224eae77e74"
  }
}
```
</p>
</details>

<details><summary>Пример ответа этой операции</summary>
<p>

```json
{
  "id": "7154f067-2abf-4b4d-9fcd-dd4b939432b2",
  "jsonrpc": "2.0",
  "result": {
    "authorized": true,
    "constraints": {"filter.districtId": {"$in": ["155147", "155150"]}}
  }
}
```
</p>
</details>

request — описывает спецификацию входных параметров запроса, т.е. "params" в заросе

response — описывает спецификацию результата, т.е "result" в ответе

`5.` Параметры зароса ("id", "jsonrpc", "method") должны валидироваться __TODO__ 

`6.` Параметры операции ("params") должны валидироваться на уровне middleware сервиса. Должна выкидываться ошибка валидации если что-то пошло не так.

`7.` Составные названия операций в качестве разделителя используют точку. Например: episode.material.index

## Стандартные операции
Ни формат jsonrpc ни спецификация jsonschema не диктуют какие-либо стандарты семантики в описания API. Т.е. проектировщик волен использовать любые допустимые типы и называть параметры и методы как ему вздумается.

Чтобы немного ограничить буйство фантазии настоятельно рекомендуется придерживаться следующих рекомендаций:

`1.` При описании стандартных CRUD методов использовать create, update, delete, index в качестве экшена (episode.material.create, episode.material.update, etc)

`2.` Максимально использовать format для описания ограничений на поля
```json
{
  "user_id": {
    "type": "string",
    "format": "uuid"
  },
  "created_at": {
    "type": "string",
    "format": "date-time"
  }
}   
``` 
`3.` Для задания сложных условий использовать формат filter. Это наше внутреннее ноухау, его стандарт jsonschema не описывает.

`4.` Для операций получения списка сущностей использовать следующие параметры:
```json5
{
	"select": { // ... можно указать поля для выборки
		"type": "array",
		"items": {"type": "string"}
	},
	"filter": {
		"type": "object",
		"format": "filter",
		"properties": {
           // ... спецификация полей доступных для фильтрации  
		}
	},
	"sort": { // ... можно указать как сортировать результаты
		"type": "array",
		"items": {
			"type": "array",
			"items": {"type": "string"}
		}
	},
	"limit": { // ... можно задать ограничение выборки
		"type": "number"
	},
	"offset": { // ... можно задать смещение выборки
		"type": "number"
	}
}
```
`5.` Результат операции запроса списка сущностей должен содержать, как минимум, список элементов и их число
```json5
{
  "type": "object",
  "properties": {
    "request": {
      // ...  
    },
    "response": {
      "type": "object",
      "properties": {
        "items": {
          "type": "object",
          "properties": {
              "field1": {"type": "string"},
              "field2": {"type": "number"} 
          }
        },
        "total": {
           "type": "number" 
        } 
      }
    }
  }
}
```
`6.` Мутационные операции (изменяющие сущности) имеют 2 типа параметров - data и filter. В зависимости от типа мутационной операции параметров может быть 1 или 2. 

`6.1.` Операция create
```json5
{
  "type": "object",
  "properties": {
    "request": {
      "type": "object",
      "properties": {
        "data": {
          "type": "object",
          "properties": {
             // ... спецификация полей сохраняемой сущности 
          }
        }
      },
      "required": [
        "data"
      ]
    },
    "response": {
      "type": "object",
      "properties": {
          // ... спецификация полей сохранённой сущности 
      } 
    }
  }
}
```
`6.2` Операция update
```json5
{
  "type": "object",
  "properties": {
    "request": {
      "type": "object",
      "properties": {
        "filter": {
            "type": "object",
            "format": "filter",
            "properties": {
                // ... спецификация полей, по которым можно фильтровать сущности, которые нужно обновить
            },
            "minProperties": 1
        }, 
        "data": {
          "type": "object",
          "properties": {
             // ... спецификация полей сущности доступных для обновления 
          },
          "minProperties": 1
        }
      },
      "required": [
        "filter",
        "data"
      ]
    },
    "response": {
      "type": "array",
      "items": { // выводим список ТОЛЬКО ТЕХ сущностей, которые УСПЕШНО обновились
          "type": "object",
          "properties": {
              // ... спецификация полей сохранённой сущности 
          }          
      } 
    }
  }
}
```
`6.3` Операция delete
```json5
{
  "type": "object",
  "properties": {
    "request": {
      "type": "object",
      "properties": {
        "filter": {
            "type": "object",
            "format": "filter",
            "properties": {
                // ... спецификация полей, по которым можно фильтровать сущности, которые нужно удалить
            },
            "minProperties": 1
        }
      },
      "required": [
        "filter"
      ]
    },
    "response": {
      "type": "array",
      "items": { // выводим список ТОЛЬКО ТЕХ сущностей, которые были УСПЕШНО удалены
          "type": "object",
          "properties": {
              // ... спецификация полей сохранённой сущности 
          }          
      } 
    }
  }
}
```

## Формат filter
Формат поля filter задаёт особый режим для поля — возможность в этом поле передать сложные условия фильтрации. 
```json5
{
  "filter": {
    "type": "object",
    "format": "filter",
    "properties": {
    "id": {"type": "number"},
    "resource_id": {"type": "number"},
    "category_id": {"type": "number"},
    "owner_id": {"type": "number"},
    "search_query": {"type": "string"},
    "decision_time": {"type": "string", "format": "date-time"}
    }
  }
}
```
В примере выше есть возможность искать по полям описанные в properties. Пример возможного запроса к данному полю
```json
{
  "jsonrpc": "2.0",
  "method": "episode.index",
  "id": "e90dcb75-4d50-426f-a34d-28427d8766ef",
  "params": {
    "filter": {
      "id": "355881a3-e2a5-4c9a-9f5b-8c32791ff1c2",
      "decision_time": {"$ge": "2019-01-01T12:00:00", "$le": "2019-10-10T18:00:00"},
      "owner_id": {"$in": [1, 4, 5]},
      "category_id": [2, 3]  
    }
  }
}
```
В качестве основы для задания условий используется mongo-like синтаксис. Поддерживаются следующие операции:  

| Оператор      | Значение           |
| ------------- |-------------| 
| $eq           | равно | 
| $ne      | неравно      |  
| $le | меньше      | 
|  $ge | больше |
| $lte | меньше либо равно |
|$gte | больше либо равно |
|$in | в списке |
|$nin | не в списке |
|$like | поиск по шаблону |
|$ilike | поиск по шаблону без регистра |

Так же реализуются операции булевой логики $and, $or, $not на любом уровне. Например, в случае крайней необходимости, можно задать более сложное условие:
```json
{
  "jsonrpc": "2.0",
  "method": "episode.index",
  "id": "e90dcb75-4d50-426f-a34d-28427d8766ef",
  "params": {
    "filter": {
      "$or": [
        {"$not": {"status_id": {"$in": [1, 4]}, "owner_type": "individual"}},
        {"status_id": {"$in": [3, 5]}, "owner_type": "legal"}
      ],
    "category_id": [2, 3]  
    }
  }
}
```
По умолчанию используется логика and

Допускается опускать операции для равенства и проверки принадлежности к списку. Соответственно если значение имеет тип массива, то применяется операция $in, в остальных случаях $eq:

```"foo": [1, 3, 4]```  
эквивалентно  
```"foo": {"$in": [1, 3, 4]}```

```"foo": 5```  
эквивалентно  
```"foo": {"$eq": 5}```  

## Связанные сущности
Должна поддерживаться навигация по связанным сущностям. Путь оформляется через точку. Например user.comments.created_at . Используется в секции select и filter. Пример:
```json5
{
  "jsonrpc": "2.0",
  "method": "episode.index",
  "id": "e90dcb75-4d50-426f-a34d-28427d8766ef",
  "params": {
    "select": ["id", "created_at", "documents.created_at", "documents.name"],
    "filter": {
      "id": "355881a3-e2a5-4c9a-9f5b-8c32791ff1c2",
      "documents.created_at": {
        "$ge": "2019-01-01T12:00:00", 
        "$le": "2019-10-10T18:00:00"
      }
    }
  }
}
```
Ответ оформляется в виде вложенных сущностей:
```json5
{
  "jsonrpc": "2.0",
  "id": "e90dcb75-4d50-426f-a34d-28427d8766ef",
  "result": [
    {
      "id": "355881a3-e2a5-4c9a-9f5b-8c32791ff1c2",
      "created_at": "2019-01-01T12:00:00",
      "documents": [
        {
          "created_at": "2019-02-01T12:00:00",
          "name": "doc1"
        },
        {
          "created_at": "2019-02-04T12:00:00",
          "name": "doc2"
        }
      ]
    },
    {
      "id": "355881a3-e2a5-4c9a-9f5b-8c32791ff1d2",
      "created_at": "2019-01-02T12:00:00",
      "documents": [
        {
          "created_at": "2019-03-04T12:00:00",
          "name": "doc16"
        }
      ]
    }
  ]
}
```

## Джентельменский набор
Для удобства описания спецификаций был создан готовый набор доступных операторов фильтрации, который можно использовать в своих спецификациях. Ознакомиться с самой свежей версией можно по адресу https://raw.githubusercontent.com/gosdev/json-rpc/master/specs/operators.json

Как использовать
Каждая новая сборка приложения должна содержать в себе самую свежую версию спецификации. Чтобы это обеспечить нужно во время сборки образа скачивать спецификацию и сохранять ее в /specs/operators.json. Для этого в Dockerfile  сервиса нужно добавить инструкцию
```bash
ADD https://raw.githubusercontent.com/gosdev/json-rpc/master/specs/operators.json /specs/operators.json

или

RUN wget https://raw.githubusercontent.com/gosdev/json-rpc/master/specs/operators.json -O /specs/operators.json
```
Теперь данная спецификация будет доступна в виде референса
```json
{
	"$ref": "/specs/operators.json"
}
```
<span style="color:red">Никогда не ссылайтесь в спецификациях на HTTP-адрес напрямую.</span>

## Пример спецификации с использованием фильтров
Пример спецификации на операцию получения списка пользователей
```json5
{
  "id": "/specs/operations/user/get.json",
  "description": "Получить список пользователей",
  "definitions": {
    "operators": {
      "$ref": "/specs/operators.json#/definitions",
      "description": "Базовое описание разных типов полей для фильтра"
    },
    "filter": {
      "type": "object",
      "format": "filter",
      "additionalProperties": false,
      "properties": {
        "id": {
          "$ref": "#/definitions/operators/number",
          "description": "ID пользователя"
        },
        "login": {
          "$ref": "#/definitions/operators/string",
          "description": "Логин пользователя"
        },
        "role_id": {
          "$ref": "#/definitions/operators/number",
          "description": "ID роли пользователя"
        },
        "created_at": {
          "$ref": "#/definitions/operators/datetime",
          "description": "Время создания пользователя"
        },
        "$or": {
          "type": "array",
          "items": {
            "$ref": "#/definitions/filter"
          }
        },
        "$and": {
          "type": "array",
          "items": {
            "$ref": "#/definitions/filter"
          }
        },
        "$not": {
          "$ref": "#/definitions/filter"
        }
      }
    }
  },
  "type": "object",
  "properties": {
    "request": {
      "type": "object",
      "properties": {
        "filter": {
          "$ref": "#/definitions/filter"
        },
        "limit": {
          "type": "number"
        },
        "offset": {
          "type": "number"
        },
        "sort": {
          "type": "array",
          "items": {
            "type": "array",
            "items": {
              "type": "string"
            }
          }
        }
      }
    },
    "response": {
      "type": "array",
      "description": "Список пользователей",
      "items": {
        "type": "object",
        "properties": {
          "id": {
            "type": "number",
            "description": "ID пользователя"
          },
          "login": {
            "type": "string",
            "description": "Имя пользователя"
          },
          "role_id": {
            "type": "number",
            "description": "ID роли пользователя"
          },
          "created_at": {
            "type": "string",
            "format": "date-time",
            "description": "Время создания пользователя"
          }
        }
      }
    }
  }
}
```
# MetaOne数据平台对接用接口v0.1_20220427：

## 1、服务接入地址

- 测试环境：

  https://test-data.metaone.gg/sb/
  
- 正式环境：

  https://data.metaone.gg/sb/

## 2、接口

### 2.1、统一请求数据结构

#### 2.1.1、请求头（Request Headers）：

- Accept-Language/lang：国际化语言【使用ISO标准的locale。不区分大小写，不区分横杠"-"与下划线"_"】
  - 英文：en或 en-US、en_uk等
  - 中文：zh或zh-cn、zh-hans、zh-hant、zh-hk、zh-tw、zh-sg等
- Authorization：应用全局唯一的动态接口调用凭据，也就是通过获取token接口返回的token。开发者需要每日获取更新并进行妥善保存。

### 2.2、统一返回数据结构

```json
{
    "stateCode": 1, // 业务返回状态码
    "resultInfo": { // 业务返回主体内容
        "desc": "Success",  // 返回内容描述
        "data": {}  // 业务返回数据
    }
}
```

- stateCode：
  - 1：请求成功
  - 0：参数格式相关错误
  - -1：token无效或过期
  - -5：服务端错误异常
  - -1105：无效的AppId
  - -1101：应用被封锁
  - -102：密钥不正确
  - -101：找不到对应数据
- resultInfo.desc：
  - Success：请求成功
  - 其他：请求失败后的错误描述
- resultInfo.data：请求成功后的数据，注意，数据有可能为空

#### 日期相关参数格式：yyyy-MM-dd

## 3、API接口：

### 3.1、鉴权模块：

#### 3.1.1 获取应用调用接口凭证-token


- 功能说明：获取token，token需要作为下面所有接口的认证信息带到请求头`Authorization`中。每一个接入方需要使用自己的appId和密钥换取的token，请勿借用其他接入方的token。

- 请求方式：POST

- 请求地址：/metaoneDataLight/getToken

- 请求内容：

  ```json
  {
  	"appId": "06be07ff82754a718a96ef46d1b82b2c", // 应用ID，由metaone数据服为每一个接入方颁发
  	"secret": "7531b58338104dcd9e7eddcce96c4ae0" // 秘钥，由metaone数据服为每一个接入方颁发，注意：非常重要，不能泄露
  }
  ```

- 正确时响应内容：

  ```json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"token": "at_2a015a53d41548f5a6d7879b75a5fff3_gAP6sHl2xnh5WEWxgiFYUUfgXaPjtViY_mo",
          }
      }
  }
  ```

##### 应用调用接口凭证-token机制相关说明：

1. 建议基于微服务架构的开发者使用中控微服务统一获取和刷新token，其他业务逻辑微服务所使用的token均来自于该中控服务，不建议各自去getToken；
2. 目前token的有效期为24～48小时。中控服务每日固定时间通过计划任务等方式获取一次token，该token自getToken调用起算能够确保24小时内有效。在获取过程中，原有的token会在新token获取之后一段时间内仍然持续有效一小段时间，这保证了token有效期的平滑过渡；
3. token的有效时间可能会在未来有调整，所以中控服务除每日定时主动获取当天的token以外，建议提供被动刷新token的接口，这样便于业务服务器在API调用获知token已超时的情况下，可以触发token的刷新流程。
4. 每一个接入方都应该有自己的appId和secret。因此不同的接入方一定是使用各自的token去调用接口。数据服接口会直接将token解析为对应的接入方权限。

### 3.2 玩家活跃数据模块：

#### 3.2.1 玩家收益数据查询：

- 功能说明：查询玩家每日的收益数据(可指定游戏)

- 请求方式：POST

- 请求地址：/metaoneData/queryUserValueGain

- 请求头：

HeaderName | 是否必须 | HeaderValue说明
------- | ------- | -------
Authorization | 是 | 当前保存的token<br/>【token需要每日通过调用获取token接口获取并更新/保存起来】
lang | 否 | 语言，缺省值"en"英文，当前仅支持中文与英文 

- 请求内容：

  ```json
  {
  	"userId": "06be07ff82754a718a96ef46d1b82b2c", // 用户ID
  	"gameId": "7531b58338104dcd9e7eddcce96c4ae0", // 游戏ID。可选参数，缺省值为"all"
  	"startDate" : "2022-04-01", //查询范围的起始日期（为yyyy-MM-dd字符串格式）查询结果会包含这个日期
  	"endDate" : "2022-04-05", //查询范围的截止日期（为yyyy-MM-dd字符串格式
  }
  ```
	- 关于gameId参数的特别说明：

   如果要查询玩家在所有游戏的每日总收益，那么gameId可以留空或设置为"all"。查询单款游戏和查询所有游戏总和的情况返回结果的结构会有略微的差异。参见下方正确时响应内容。
  
	- 关于日期参数的特别说明：

 日期参数采用包含开始不包含截止的方式，例如 startData="2022-04-01" endDate="2022-04-05"时，表示拉取的是以下这4天的数据
 ```json
["2022-04-01","2022-04-02","2022-04-03","2022-04-04"]
 ```
基础参数要求：startDate 必须 < endDate；为避免查询阻塞，限制了一次查询的跨度不可以超过100天


- 正确时响应内容 指定了gameId时：

  ```json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"details": [
          		{
          			"date": "2022-04-01", // "yyyy-MM-dd"格式日期
          			"valueGain": 123512, // 该玩家该日期收益
          		},
          		{
          			"date": "2022-04-02", // "yyyy-MM-dd"格式日期
          			"valueGain": 1232, // 该玩家该日期收益
          		}
          		{
          			"date": "2022-04-03", // "yyyy-MM-dd"格式日期
          			"valueGain": 0, // 该玩家该日期收益
          		},
          		{
          			"date": "2022-04-04", // "yyyy-MM-dd"格式日期
          			"valueGain": 72, // 该玩家该日期收益
          		},
          	],
          	"totalValueGain": 124816, //该玩家在该游戏、该区间日期内的总收益（等于details内所有valueGain的加和）
          	"gameId": "7531b58338104dcd9e7eddcce96c4ae0",
          }
      }
  }
  ```
  
- 正确时响应内容 没有指定gameId或gameId设置为"all"时：

  ```json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"details": [
          		{
          			"date": "2022-04-01", // "yyyy-MM-dd"格式日期
          			"valueGain": 123512, // 该玩家该日期在所有游戏的总收益
          		},
          		{
          			"date": "2022-04-02", // "yyyy-MM-dd"格式日期
          			"valueGain": 1232, // 该玩家该日期在所有游戏的总收益
          		}
          		{
          			"date": "2022-04-03", // "yyyy-MM-dd"格式日期
          			"valueGain": 0, // 该玩家该日期在所有游戏的总收益
          		},
          		{
          			"date": "2022-04-04", // "yyyy-MM-dd"格式日期
          			"valueGain": 72, // 该玩家该日期在所有游戏的总收益
          		},
          	],
          	"totalValueGain": 124816, //该玩家在该区间日期内的总收益（等于details内所有valueGain的加和）
          }
      }
  }
  ```
  

#### 3.2.2 玩家在指定游戏在线时长数据查询：


- 功能说明：查询玩家在指定游戏的每日在线时长

- 请求方式：POST

- 请求地址：/metaoneData/queryUserGameOnlineTime

- 请求头：

HeaderName | 是否必须 | HeaderValue说明
------- | ------- | -------
Authorization | 是 | 当前保存的token<br/>【token需要每日通过调用获取token接口获取并更新/保存起来】
lang | 否 | 语言，缺省值"en"英文，当前仅支持中文与英文 

- 请求内容：

  ```json
  {
  	"userId": "06be07ff82754a718a96ef46d1b82b2c", // 用户ID
  	"gameId": "7531b58338104dcd9e7eddcce96c4ae0", // 游戏ID。这是必须的。暂不支持gameId=all的情况。
  	"startDate" : "2022-04-01", //查询范围的起始日期（为yyyy-MM-dd字符串格式）查询结果会包含这个日期
  	"endDate" : "2022-04-05", //查询范围的截止日期（为yyyy-MM-dd字符串格式
  }
  ```
  
	- 关于日期参数的特别说明：

 日期参数采用包含开始不包含截止的方式，例如 startData="2022-04-01" endDate="2022-04-05"时，表示拉取的是以下这4天的数据
 ```json
["2022-04-01","2022-04-02","2022-04-03","2022-04-04"]
 ```
基础参数要求：startDate 必须 < endDate；为避免查询阻塞，限制了一次查询的跨度不可以超过100天


- 正确时响应内容：

  ```json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"details": [
          		{
          			"date": "2022-04-01", // "yyyy-MM-dd"格式日期
          			"onlineTime": 123512, // 该玩家该日期在线时长（单位秒）
          		},
          		{
          			"date": "2022-04-02", // "yyyy-MM-dd"格式日期
          			"onlineTime": 1232, // 该玩家该日期在线时长（单位秒）
          		}
          		{
          			"date": "2022-04-03", // "yyyy-MM-dd"格式日期
          			"onlineTime": 0, // 该玩家该日期在线时长（单位秒）
          		},
          		{
          			"date": "2022-04-04", // "yyyy-MM-dd"格式日期
          			"onlineTime": 72, // 该玩家该日期在线时长（单位秒）
          		},
          	],
          	"totalOnlineTime": 124816, //bigint长整型，该玩家在该游戏该区间日期内总体在线时长
          	"gameId": "7531b58338104dcd9e7eddcce96c4ae0",
          }
      }
  }
  ```
  
  - gameId=all的情况对应玩家当天游戏时长，当前版本暂无可支持的原始数据。如果要查询的是玩家在每日的所有游戏在线时长的直接累加，请使用接口3.2.3。

#### 3.2.3 玩家每日在所有游戏在线时长直接累加数据查询:

- 功能说明：查询玩家每日在各个游戏在线时长的累加。【注：同一个玩家账号若有同时用于多款游戏，该累加值有可能会超过24小时。请接入方注意数据类型的设置】

- 请求方式：POST

- 请求地址：/metaoneData/queryUserGameOnlineTimeDirectSum

- 请求内容：

  ```json
  {
  	"userId": "06be07ff82754a718a96ef46d1b82b2c", // 用户ID
  	"startDate" : "2022-04-01", //查询范围的起始日期（为yyyy-MM-dd字符串格式）查询结果会包含这个日期
  	"endDate" : "2022-04-05", //查询范围的截止日期（为yyyy-MM-dd字符串格式
  }
  ```
  
	- 关于日期参数的特别说明：

 日期参数采用包含开始不包含截止的方式，例如 startData="2022-04-01" endDate="2022-04-05"时，表示拉取的是以下这4天的数据
 ```json
["2022-04-01","2022-04-02","2022-04-03","2022-04-04"]
 ```
基础参数要求：startDate 必须 < endDate；为避免查询阻塞，限制了一次查询的跨度不可以超过100天


- 正确时响应内容：

  ````json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"details": [
          		{
          			"date": "2022-04-01", // "yyyy-MM-dd"格式日期
          			"sumOfOnlineTime": 123512, // 该玩家该日期所有游戏在线时长的加和（单位秒），该值可能超过24小时，注意使用的数据类型
          		},
          		{
          			"date": "2022-04-02", // "yyyy-MM-dd"格式日期
          			"sumOfOnlineTime": 1232, // 该玩家该日期所有游戏在线时长的加和（单位秒），该值可能超过24小时，注意使用的数据类型
          		}
          		{
          			"date": "2022-04-03", // "yyyy-MM-dd"格式日期
          			"sumOfOnlineTime": 0, // 该玩家该日期所有游戏在线时长的加和（单位秒），该值可能超过24小时，注意使用的数据类型
          		},
          		{
          			"date": "2022-04-04", // "yyyy-MM-dd"格式日期
          			"sumOfOnlineTime": 72, // 该玩家该日期所有游戏在线时长的加和（单位秒），该值可能超过24小时，注意使用的数据类型
          		},
          	],
          	"totalSumOfOnlineTime": 124816, //bigint长整型，该玩家在该区间日期内所有游戏的在线时长的累加值
          }
      }
  }
  ````

#### 3.2.4 玩家每日活跃整体情况数据查询(待定)：

### 3.3 公会数据模块(待定)：

含基于公会的数据数据查询和公会玩家关系同步相关流程的接口

### 3.4 道具信息数据模块：

#### 3.4.1 通过道具Id(tokenId）查询道具信息：

- 功能说明：通过道具Id，查询道具信息。

- 请求方式：GET

- 请求地址：/metaoneData/queryGameItemById

- 请求头：

HeaderName | 是否必须 | HeaderValue说明
------- | ------- | -------
Authorization | 是 | 当前保存的token<br/>【token需要每日通过调用获取token接口获取并更新/保存起来】
lang | 否 | 语言，缺省值"en"英文，当前仅支持中文与英文 

- 请求参数：


参数名 | 是否必须 |说明
------- | ------- | -------
itemId | 是 | 道具ID/tokenId



- 正确时响应内容：

  ````json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"ownerId": "道具拥有者的ID/钱包ID",
          	"details": {
					"itemImage": "https://xxxx/xxx.png", //道具配图url地址
					"gameName": "游戏名称", 
					"attributes" :[  //所有扩展字段会以如下格式放在attributes数组内
						{
							"attrName" : "字段的标题或名称", 
							"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
						},	
						{
							"attrName" : "字段的标题或名称", 
							"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
						},
						...
					]
				},
          }
      }
  }
  ````
  
#### 3.4.2 查询指定玩家（钱包ID）名下的道具：
  
- 功能说明：通过道具Id，查询道具信息。由于道具信息数据量可能偏大。建议选择合适的分页大小进行分页加载。返回结果中，itemId、itemImage和gameName都是必选字段

- 请求方式：GET

- 请求地址：/metaoneData/listUserGameItems

- 请求头：

HeaderName | 是否必须 | HeaderValue说明
------- | ------- | -------
Authorization | 是 | 当前保存的token<br/>【token需要每日通过调用获取token接口获取并更新/保存起来】
lang | 否 | 语言，缺省值"en"英文，当前仅支持中文与英文 

- 请求参数：


参数名 | 是否必须 |说明
------- | ------- | -------
userId | 是 | 玩家Id/钱包ID
pageNum | 是 | 分页页码，第一页从0开始
pageSize | 是 | 分页大小，必须为正整数


- 正确时响应内容：

  ````json
  {
      "stateCode": 1,
      "resultInfo": {
          "desc": "Success",
          "data": {
          	"totalCount": 23131, //该玩家名下的道具总数
          	"itemCount": 20, //当前请求查询到的结果数量
          	"items": [
          		{
		          	"itemId": "道具ID/tokenID",
		          	"details": {
						"itemImage": "https://xxxx/xxx.png", //道具配图url地址
						"gameName": "游戏名称", 
						"attributes" :[  //所有扩展字段会以如下格式放在attributes数组内
							{
								"attrName" : "字段的标题或名称", 
								"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
							},	
							{
								"attrName" : "字段的标题或名称", 
								"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
							},
						]
					},
          		},
          		{
		          	"itemId": "道具ID/tokenID",
		          	"details": {
						"itemImage": "https://xxxx/xxx.png", //道具配图url地址
						"gameName": "游戏名称", 
						"attributes" :[  //所有扩展字段会以如下格式放在attributes数组内
							{
								"attrName" : "字段的标题或名称", 
								"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
							},	
							{
								"attrName" : "字段的标题或名称", 
								"attrValue" : "字符串格式的字段取值", //所有类型的值都统一转为字符串
							},
						]
					},
          		},
          		...
          	]
          }
      }
  }
  ````









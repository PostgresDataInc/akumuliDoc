# 端点API

{% api-method method="get" host="http://<host>:8181" path="/api/stats" %}
{% api-method-summary %}
统计
{% endapi-method-summary %}

{% api-method-description %}

此端点获取正在运行的Akumuli实例的存储统计信息，也可以用作测试服务有效性的检测端点。
{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}
Storage statistics successfully retrieved.
{% endapi-method-response-example-description %}

```javascript
{    "volume_0":    {        "free_space": "0",        "file_name": "\/root\/.akumuli\/db_0.vol"    },    "volume_1":    {        "free_space": "0",        "file_name": "\/root\/.akumuli\/db_1.vol"    },    "volume_2":    {        "free_space": "0",        "file_name": "\/root\/.akumuli\/db_2.vol"    },    "volume_3":    {        "free_space": "2027974656",        "file_name": "\/root\/.akumuli\/db_3.vol"    }}
```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="post" host="http://<host>:8181" path="/api/query" %}
{% api-method-summary %}
读取查询
{% endapi-method-summary %}

{% api-method-description %}
此端点用于从数据库中检索时序数据，客户端必须提供合法的查询，响应使用分块传输编码返回结果，而结果以RESP协议编码。
{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-body-parameters %}
{% api-method-parameter name="查询体" type="object" required=true %}
JSON编码查询
{% endapi-method-parameter %}
{% endapi-method-body-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```
+RESP encoded data
+just like this
```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=400 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```
-RESP encoded error message
```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="post" host="http://<host>:8181" path="/api/search" %}
{% api-method-summary %}
搜索
{% endapi-method-summary %}

{% api-method-description %}
此端点用于检索元数据，例如 系列名、标签值。
{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-path-parameters %}
{% api-method-parameter name="查询体" type="string" required=true %}
JSON编码查询
{% endapi-method-parameter %}
{% endapi-method-path-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```
+RESP encoded output
```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=400 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```
-RESP encoded error message
```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="post" host="http://<host>:8181" path="/api/suggest" %}
{% api-method-summary %}
Suggest
{% endapi-method-summary %}

{% api-method-description %}
This endpoint can be used to retrieve metric names, tag names, and tag values. It powers autocomplete function of the **akumuli-datasource** for Grafana. 
{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}
{% api-method-path-parameters %}
{% api-method-parameter name="Query Body" type="string" required=false %}
JSON encoded query
{% endapi-method-parameter %}
{% endapi-method-path-parameters %}
{% endapi-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```
+RESP encoded list or results
```
{% endapi-method-response-example %}

{% api-method-response-example httpCode=400 %}
{% api-method-response-example-description %}

{% endapi-method-response-example-description %}

```
-RESP encoded error message
```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}

{% api-method method="get" host="http://<host>:8181" path="/api/function-names" %}
{% api-method-summary %}
列出函数
{% endapi-method-summary %}

{% api-method-description %}
此端点检索可以用在查询中的函数列表。
{% endapi-method-description %}

{% api-method-spec %}
{% api-method-request %}

{% api-method-response %}
{% api-method-response-example httpCode=200 %}
{% api-method-response-example-description %}
List of functions
{% endapi-method-response-example-description %}

```
absaccumulatecmacusumdiffdivideewmaewma-errorfrequent-itemsheavy-hittersmultiplyratescalesmasma-errorsumtop
```
{% endapi-method-response-example %}
{% endapi-method-response %}
{% endapi-method-spec %}
{% endapi-method %}




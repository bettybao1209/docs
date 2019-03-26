# importprivkey 方法

显示钱包中未提取的 gas 数量。

> [!Note]
> 执行此命令前需要在 Neo-CLI 节点中打开钱包。

## 参数说明

key ：WIF 格式的私钥。

## 调用示例

请求正文：

```json
{
  "jsonrpc": "2.0",
  "method": "importprivkey",
  "params": ["L5c6jz6Rh8arFJW3A5vg7Suaggo28ApXVF2EPzkAXbm94ThqaA6r"],
  "id": 1
}
```

响应正文：

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "address": "Ad8S24trcuchyLfEbJWqRP7BUScUT4t2pw",
        "haskey": true,
        "label": null,
        "watchonly": false
    }
}

```

响应说明：

返回地址信息。
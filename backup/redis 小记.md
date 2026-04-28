## 计算KEYS占用内存大小

```lua
-- 初始化游标为 "0"，用于 SCAN 命令的初始值
local cursor = "0"
-- 创建一个空表 result，用于存储结果
local result = {}
-- 使用 repeat 循环，直到游标返回 "0" 表示扫描结束
repeat
    -- 使用 SCAN 命令逐步获取键，SCAN 命令返回一个表，第一个元素是新的游标，第二个元素是键的列表
    local res = redis.call('SCAN', cursor)
    -- 更新游标为新的游标值，以便在下一次迭代中继续扫描
    cursor = res[1]
    -- 获取键的列表
    local keys = res[2]
    -- 遍历每个键
    for _, key in ipairs(keys) do
        -- 使用 MEMORY USAGE 命令获取键的内存使用情况
        local memory = redis.call('MEMORY', 'USAGE', key)
        -- 将键和其内存使用情况格式化为字符串，并插入到结果表中
        table.insert(result, key .. ": " .. memory .. " bytes")
    end
    -- 添加延迟以降低 Redis 的 CPU 占用率，单位为秒
    redis.call('DEBUG', 'SLEEP', 0.1)    
-- 继续循环，直到游标为 "0" 表示扫描结束
until cursor == "0"
-- 返回结果表
return result
```

**使用redis-cli**

```shell
redis-cli -h xxx -p 6379 -a $(cat ./pwd) --eval mem.lua > mem_result.txt
cat mem_result.txt |awk -F ':' '{print $1 $2 $NF}'|uniq -c | sort -nr > redis.txt
```


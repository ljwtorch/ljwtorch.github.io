## 计算KEYS占用内存大小

```lua
local keys = redis.call('KEYS', '*')
local result = {}
for _, key in ipairs(keys) do
    local memory = redis.call('MEMORY', 'USAGE', key)
    table.insert(result, key .. ": " .. memory .. " bytes")
end
return result
```

**使用redis-cli**

```shell
redis-cli -h xxx -p 6379 -a $(cat ./pwd) --eval mem.lua > mem_result.txt
cat mem_result.txt |awk -F ':' '{print $1 $2 $NF}'|uniq -c | sort -nr > redis.txt
```


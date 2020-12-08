例如，游戏中玩家击杀BOSS后会按如下概率掉落奖励：10%神兵 20%饰品 30%戒指 40%精魄。我们要程序实现掉落符合上述概率。  

通常有三种算法：  
#### 一般算法

构造一个容量为100的数组array，其中10个元素填充为神兵，20个元素填充为饰品，30个元素填充为戒指...构造完之后，在1-100之间取随机数rand，那么array[rand]对应的值即为玩家的掉落奖励。这种方法的优点是简单，且构造完成之后生成随机数的时间复杂度是O(1)，缺点是精度不够高且在类型很多的时候，占用的空间很大，其构造的时间复杂度和空间复杂度都是O(MN)，N指类型数，M由类型对应的最低概率确定。

#### 离散算法

通过概率分布构造几个点，[10, 30, 60, 100]（后面的值是前面一次累加的概率之和）。再生成1-100的随机数，看它落在哪个区间，比如50落在(30, 60]，那玩家就抽到了戒指。在查找时，可以使用线性查找，或效率更高的二分查找，时间复杂度是O(logN)。
```lua
-- 构造一个cdf
function DiscreteMethod(pdf)
    local len = #pdf
    local cdf = clone(pdf)
    for i = 2, len do
        cdf[i] = cdf[i] + cdf[i-1]
    end
    cdf[len] = 1    -- 因为浮点型精度问题，最后一个值强制为1
    return cdf
end

function next_rand(cdf)
    local len = #cdf
    local left = 1
    local right = len
    local randNum = math.random()
    while (left < right) do
        local mid = left + math.floor((right - left) / 2)
        if randNum > cdf[mid] then
            left = mid
        else
            right = mid
        end
        if left + 1 >= right then
            left = right
        end
    end
    return left
end
```

#### Alias Method

对于上面所说的PDF[0.1, 0.2, 0.3, 0.4]，我们将每个概率都当作一列，概率值为其高度，我们最终要构造出一个每列高度都为1的矩形。  
要完成这个操作，我们需要先将每个元素乘以4（概率类型的数量），此时一定会有大于1和小于1的值，我们之后要做的就是用大于1的数去补足小于1的数，使得最后每种概率最后都为1，注意，这里一定要遵循一个限制：每列至多是两种概率的组合。  
![alias method](/assets/image/2020-12-08-0.png)  
![alias method](/assets/image/2020-12-08-1.png)  

最终我们会得到两个数组，一个是记录原始的列对应的概率值数组prob[0.4, 0.8, 0.6, 1]，一个是用于记录用于补充当前列的列号alias[3, 4, 4, nil]。  

在构造得到这两个数组之后，我们后面要得到玩家当前的掉落类型，只需要随机获取一列```col```，然后再让当前列的概率值```prob[col]```与一个随机小数```rand```比较，如果```rand < prob[col]```，那么结果就是```col```，否则结果就是```alias[col]```，即用于补足当前列的列号。

```lua
-- pdf 里各值之和为1
function AliasMethod(pdf)
    local len = #pdf
    local prob_arr = {} -- 记录当前列的概率值
    local alias = {}    -- 记录补充列的列号
    local small = {}
    local large = {}
    for i = 1, len do
        pdf[i] = pdf[i] * len
        if pdf[i] < 1 then
            table.insert(small, i)
        else
            table.insert(large, i)
        end
    end

    while (#small ~= 0 and #large ~= 0) do
        -- 用 lIdx 列去补充 sIdx 列
        local sIdx = table.remove(small, 1)
        local lIdx = table.remove(large, 1)
        prob_arr[sIdx] = pdf[sIdx]
        alias[sIdx] = lIdx

        pdf[lIdx] = pdf[lIdx] - (1 - pdf[sIdx])
        if pdf[lIdx] < 1 then
            table.insert(small, lIdx)
        else
            table.insert(large, lIdx)
        end
    end

    while (#small~=0) do
        local sIdx = table.remove(small, 1)
        prob_arr[sIdx] = 1
    end

    while (#large~=0) do
        local lIdx = table.remove(large, 1)
        prob_arr[lIdx] = 1
    end
    return prob_arr, alias
end

function next_rand(prob_arr, alias)
    local col = math.random(1, #prob_arr)
    return math.random() < prob_arr[col] and col or alias[col]
end
```
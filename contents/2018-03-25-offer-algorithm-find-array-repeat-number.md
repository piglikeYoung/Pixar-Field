
## 找出数组中重复的数字

在一个长度为n的数组中，所有数字的取值范围都在[0,n-1]，数组中某些数字是重复的，但不知道有几个数字重复了。

| 解法 | 是否改变原数组 | 时间复杂度 | 空间复杂度 |
|:-------:|:----|:----:|:----:|
| 借助哈希表 | 否 | o(n) | o(n) |
| 根据数字特点排序 | 是 | o(n) | o(1) |

### 借助哈希表

借助哈希表，不会修改原始数据，时间复杂度o(n)，空间复杂度o(n)

```swift
func duplicate1(data: [Int]) -> Int {
    guard data.count > 2 else {
        return -1;
    }
    
    for i in 0..<data.count {
        if data[i] < 0 || data[i] > (data.count - 1) {
            return -1
        }
    }
    
    var hashTable: [String: String] = [String: String]()
    // 遍历数组
    for index in 0..<data.count {
        let value = String(data[index]);
        if let _ = hashTable[value] {
            return data[index]
        } else {
            hashTable[value] = "1"
        }
    }
    return -1;
}

let num = duplicate1(data: [6,3,6,0,2,5,3])
print(num) // 6
```

### 根据数字特点排序

根据数字特点排序，会修改原始数据，时间复杂度o(n)，空间复杂度o(1)


```swift
func duplicate2(data: [Int]) -> Int {
    var numbers = data
    
    guard numbers.count > 2 else {
        return -1;
    }
    
    for i in 0..<numbers.count {
        if numbers[i] < 0 || numbers[i] > (numbers.count - 1) {
            return -1
        }
    }
    
    for index in 0..<numbers.count {
        while numbers[index] != index {
            print(index)
            if numbers[index] == numbers[numbers[index]] {
                return numbers[index]
            }
            
            let temp = numbers[index]
            numbers[index] = numbers[temp]
            numbers[temp] = temp
            
            print(numbers)
        }
    }
    return -1;
}

let num2 = duplicate2(data: [6,3,6,0,2,5,3])
print(num2) // 3

```

## 不修改数组找出重复的数字

在一个长度为n+1的数组里所有数字都在1~n的范围内，所以数组至少有个一个数字是重复的。请找出数组中任意一个重复的数字，但不能修改输入的数组。

| 解法 | 是否改变原数组 | 时间复杂度 | 空间复杂度 |
|:-------:|:----|:----:|:----:|
| 二分查找 | 否 | o(nlogn) | o(1) |

### 二分查找思路

不修改原始数据，时间复杂度o(nlogn)，空间复杂度o(1)

```swift
func duplicate3(data: [Int]) -> Int {
    guard data.count > 0 else {
        return -1;
    }
    
    var start = 1
    var end = data.count - 1;
    while end >= start {
        print("start:\(start)")
        print("end:\(end)")
        let middle = ((end - start) >> 1) + start
        print("middle:\(middle)")
        let count = countRange(numbers: data, start: start, end: middle)
        if end == start {
            if count > 1 {
                return start
            } else {
                break
            }
        }
        
        if count > (middle - start + 1) {
            end = middle
        } else {
            start = middle + 1
        }
    }
    return -1;
}

func countRange(numbers: [Int], start: Int, end: Int) -> Int {
    
    if numbers.count == 0 {
        return 0
    }
    
    var count = 0
    for index in 0..<numbers.count {
        if numbers[index] >= start && numbers[index] <= end {
            count += 1
        }
    }
    print("count:\(count)")
    return count
    
}

let num3 = duplicate3(data: [4,4,6,1,2,5,3])
print(num3) // 4
```


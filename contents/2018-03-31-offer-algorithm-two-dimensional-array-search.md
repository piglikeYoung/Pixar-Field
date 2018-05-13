
## 二维数组中的查找

一个二维数组中，每一行从左到右递增，每一列从上到下递增。输入一个整数，判断数组中是否含有该整数


```swift
func P44_TwoDimensionalArraySearch(data: [[Int]], target: Int) -> Bool {
    var found = false
    guard data.count > 0 else {
        return found
    }
    
    // 总行数
    let rows = data.count
    // 总列数
    let columns = data[0].count
    
    var row = 0
    var column = columns - 1
    while row < rows && column >= 0 {
        if data[row][column] == target {
            found = true
            break
        } else if (data[row][column] > target) {
            column -= 1;
        } else {
            row += 1;
        }
    }
    return found
}

let array = [
    [1, 2, 4, 9],
    [4, 6, 11, 14],
    [5, 8, 12, 16],
    [9, 10, 15, 18],
    ]

let found = P44_TwoDimensionalArraySearch(data: array, target: 4)
print(found)
```


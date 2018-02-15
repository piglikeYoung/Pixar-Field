## 前言

最近开发的时候遇到到聊天文本框的字符长度限制的需求，比如粘贴过来的文本超过最大字符限制，将文本截取。但是使用 **substringToIndex** 截取的时候遇到恰好是emoji表情的时候就会出现 emoji 长度不是 1 的情况，截取后出现**半边A**的编码错误的情况。
![emoji-substring-error](http://p44bkxib3.bkt.clouddn.com/emoji-substring-error.jpg)

## 解决方法

### 方法一（有bug）
刚开始我觉得既然 emoji 多了一个字符，那我少截取一个字符吧，写了一个方法：

```objc
- (NSString *)subStringWith:(NSString *)string ToIndex:(NSInteger)index{

    NSString *result = string;

    const char *res = [result substringToIndex:index].UTF8String;
    if (res == NULL) {
        result = [result substringToIndex:index - 1];
    } else {
        result = [result substringToIndex:index];
    }

    return result;
}
```

将截取的字符转码成 **UTF8** 编码的字符集，然后判断转码是否成功，假如成功了说明截取成功，没有将 emoji 截取错误；反之截取失败，然后少截取一个字符，保持 emoji 完整。

自测后发现，上面的方法放在以前还行，但是现在出现了很多新的 emoji ，比如国旗emoji，占了4个字符长度，利用上面的方法还是会切除失败。

### 方法二（完美解决）

只好上网查找并看了下NSString的API，找到两个API：

```objc
- (NSRange)rangeOfComposedCharacterSequenceAtIndex:(NSUInteger)index;
- (NSRange)rangeOfComposedCharacterSequencesForRange:(NSRange)range API_AVAILABLE(macos(10.5), ios(2.0), watchos(2.0), tvos(9.0));
```

尝试下后发现它可以根据字符组成的序列判断截取的地方是否一个完整的字符，假设是个 emoji 它会将 emoji 完整截取。


```objc
/**
 *  监听输入文字变化
 */
- (void)inputTextLimited {
    NSString *resultString = _inputTextView.text;
    
    if ([resultString length] > _maxTextLength){
        NSRange rangeIndex = [resultString rangeOfComposedCharacterSequenceAtIndex:_maxTextLength];
        _inputTextView.text = [_inputTextView.text substringToIndex:rangeIndex.location];
    }
}
```







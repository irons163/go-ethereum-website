---
title: "Welcome"
date: 2018-10-03T11:09:53+08:00
draft: true
---



commit:32ae696
comment:1. Moved string util 2.Make Serializing block 3.Changed Tx serialization to return bytes instead of a string


封裝（pack）與解封裝（unpack）這筆資料（另一個常用的稱呼為 ＂marshal＂與 ＂unmarshal＂）
這邊架構還有些混亂，serialization.go負責的RlpEncode與block.go中的MarshalRlp及transaction.go中的MarshalRlp，這三者概念容易混淆。
RlpEncode中負責將傳入的列表按造Rlp編碼定義來編碼，因為編碼不會破壞列表中元素的先後順序，所以可以說是序列化(serialization)。
而MarshalRlp像是定義一個序列化的結構，在MarshalRlp中定義一個struct，列如 {A,B,C}，然後透過RlpEncode變成binary，binary可用於儲存或傳輸，然後再經由反向的RlpDecode與Unmarshal還原成struct {A,B,C}。

這邊還沒有RlpDecode與Unmarshal，但是已經有預留伏筆，

result:

    init Ethereum VM
    stack size = 256

    # processing Tx (8fee28c5311d91212d92cbf14548e9e96ab39a)
    # fee = 0.000000, ops = 12, sender = 1234567890, value = 20

    # processing Tx (3ab78afb9e495acc6eabd8982730dbb679db2f)
    # fee = 0.000000, ops = 2, sender = 1234567890, value = 20
    0   67 [10 6 0 0 0 0]
    1   67 [10 6 0 0 0 0]
    # finished processing Tx

    2   66 [10 10 0 0 0 0]
    3   67 [255 7 0 0 0 0]
    4   81 [20 255 0 0 0 0]
    ... JMP 7 ...
    7   66 [30 31 0 0 0 0]
    8   67 [255 22 0 0 0 0]
    9   81 [31 255 0 0 0 0]
    ... JMP 22 ...
    # finished processing Tx

    rlp encoded Tx "\x01\x00\x010\x00\n1234567890\x00\x010\x00\x0220\x00\x010\x01\x00\x06395843\x00\x06657986\x00\t335612448\x00\x06524099\x00\b16716881\x00\x010\x00\b13114947\x00\a2039362\x00\a1507139\x00\b16719697\x00\a1048387\x00\x0565360"
    block enc "\x01\x00\x010\x00\x00\x00\x00\x00 mi\u007f\xb0\b\x91A\xd0P\x1f\xb7\xf9\x1cd\xb1ӑ{\xc2ѕ&X\xa4\u007f\xad\xea\x95\v\xed\xc8="
    block hash "39e507d0bc23032af0fc1ad2d979c27d31968fc9ed69c4e3000fed172305e24d"

    Process finished with exit code 0

commit:eef6b5d
comment:Updated rlp encoding. (requires verification)
file:
rlp_test.go
test result:

    === RUN   TestEncode
    --- PASS: TestEncode (0.00s)
    PASS
    coverage: 24.5% of statements

    Process finished with exit code 0

result:

    init Ethereum VM
    stack size = 256

    # processing Tx (8fee28c5311d91212d92cbf14548e9e96ab39a)
    # fee = 0.000000, ops = 12, sender = 1234567890, value = 20

    # processing Tx (3ab78afb9e495acc6eabd8982730dbb679db2f)
    # fee = 0.000000, ops = 2, sender = 1234567890, value = 20
    0   67 [10 6 0 0 0 0]
    1   66 [10 10 0 0 0 0]
    1   67 [10 6 0 0 0 0]
    # finished processing Tx

    3   32 [10 1 20 0 0 0]
    4   81 [20 255 0 0 0 0]
    ... JMP 0 ...
    0   67 [10 6 0 0 0 0]
    1   66 [10 10 0 0 0 0]
    2   32 [10 1 20 0 0 0]
    3   67 [255 7 0 0 0 0]
    4   81 [20 255 0 0 0 0]
    ... JMP 7 ...
    7   66 [30 31 0 0 0 0]
    8   67 [255 22 0 0 0 0]
    9   81 [31 255 0 0 0 0]
    ... JMP 22 ...
    # finished processing Tx

commit:195d72d
comment:Implemented decoding rlp
file:

    func Decode(data []byte, pos int) (interface{}, int) {
    char := int(data[pos])
    slice := make([]interface{}, 0)
    switch {
    case char < 24:
    return append(slice, data[pos]), pos + 1
    case char < 56:
    b := int(data[pos]) - 23
    return FromBin(data[pos+1 : pos+1+b]), pos + 1 + b
    case char < 64:
    b := int(data[pos]) - 55
    b2 := int(FromBin(data[pos+1 : pos+1+b]))
    return FromBin(data[pos+1+b : pos+1+b+b2]), pos+1+b+b2
    case char < 120:
    b := int(data[pos]) - 64
    return data[pos+1:pos+1+b], pos+1+b
    case char < 128:
    b := int(data[pos]) - 119
    b2 := int(FromBin(data[pos+1 : pos+1+b]))
    return data[pos+1+b : pos+1+b+b2], pos+1+b+b2
    case char < 184:
    b := int(data[pos]) - 128
    pos++
    for i := 0; i < b; i++ {
    var obj interface{}

    obj, pos = Decode(data, pos)
    slice = append(slice, obj)
    }
    return slice, pos
    case char < 192:
    b := int(data[pos]) - 183
    //b2 := int(FromBin(data[pos+1 : pos+1+b])) (ref imprementation has an unused variable)
    pos = pos+1+b
    for i := 0; i < b; i++ {
    var obj interface{}

    obj, pos = Decode(data, pos)
    slice = append(slice, obj)
    }
    return slice, pos

    default:
    panic(fmt.Sprintf("byte not supported: %q", char))
    }

    return slice, 0
    }
    
    
分析：

RLP 編碼定義如下：

型別  |  首位元組範圍  |  編碼內容
------------ | -------------
[0x00, 0x7f]的單個位元組  |  [0x00, 0x7f]  |  位元組內容本身
0-55 位元組長的字串  |  [0x80, 0xb7]  |  0x80加上字串長度，後跟字串二進位制內容
超過55個位元組的字串  |  [0xb8, 0xbf]  |  0xb7加上字串長度的長度，後跟字串二進位制內容
0-55個位元組長的列表（所有項的組合長度） |  [0xc0, 0xf7]  |  0xc0加上所有的項的RLP編碼串聯起來的長度得到的單個位元組，後跟所有的項的RLP編碼的串聯組成。
列表的內容超過55位元組  |  [0xf8, 0xff]  |  0xC0加上所有的項的RLP編碼串聯起來的長度的長度得到的單個位元組，後跟所有的項的RLP編碼串聯起來的長度的長度，再後跟所有的項的RLP編碼的串聯組成


例子
字串 "pig" = [ 0x83, 'p', 'i', 'g' ]
列表 [ "pig", "dog" ] = [ 0xc8, 0x83, 'p', 'i', 'g', 0x83, 'd', 'o', 'g' ]
空字串 ('null') = [ 0x80 ]
空列表 = [ 0xc0 ]
數字12 ('x0c') = [ 0x0c ]
數字 1024 ('x04x00') = [ 0x82, 0x04, 0x00 ]
巢狀列表 [ [], [[]], [ [], [[]] ] ] = [ 0xc7, 0xc0, 0xc1, 0xc0, 0xc3, 0xc0, 0xc1, 0xc0 ]
字串 "Lorem ipsum dolor sit amet, consectetur adipisicing elit" = [ 0xb8, 0x38, 'L', 'o', 'r', 'e', 'm', ' ', ... , 'e', 'l', 'i', 't' ]

RLP分析
以上我們可以看出RLP編碼的設計思想，就是通過首位元組快速判斷一串編碼的型別，充分利用了一個位元組的儲存空間，將0x7f以後的值賦予了新的含義，RLP最大的優點是在充分利用位元組的情況下，同時支援列表結構，也就是說可以很輕易的利用RLP儲存一個樹狀結構。

程序處理RLP編碼時也非常容易，根據首位元組就可以判斷出這段編碼的型別，同時呼叫不同的方法進行解碼，和JSON編碼類似，支援巢狀的結構，通過遞迴呼叫可以將整個RLP快速還原成一顆樹，或者轉譯成一個JSON結構，便於其他程序使用。



slice 一開始為空
case char < 24:
    return append(slice, data[pos]), pos + 1 //加入slice
case char < 56:
b := int(data[pos]) - 23 // -23
return FromBin(data[pos+1 : pos+1+b]), pos + 1 + b // 透過FromBin將data[pos+1 : pos+1+b]從binary轉成int64。
case char < 64:
b := int(data[pos]) - 55 // -55
b2 := int(FromBin(data[pos+1 : pos+1+b])) // 透過FromBin將data[pos+1 : pos+1+b]從binary轉成int64。
return FromBin(data[pos+1+b : pos+1+b+b2]), pos+1+b+b2 // 透過FromBin將data[pos+1 : pos+1+b]從binary轉成int64。
case char < 120: 
b := int(data[pos]) - 64 // -64
return data[pos+1:pos+1+b], pos+1+b
case char < 128:
b := int(data[pos]) - 119
b2 := int(FromBin(data[pos+1 : pos+1+b]))
return data[pos+1+b : pos+1+b+b2], pos+1+b+b2
case char < 184:
b := int(data[pos]) - 128
pos++
for i := 0; i < b; i++ {
var obj interface{}

obj, pos = Decode(data, pos)
slice = append(slice, obj)
}
return slice, pos
case char < 192:
b := int(data[pos]) - 183
//b2 := int(FromBin(data[pos+1 : pos+1+b])) (ref imprementation has an unused variable)
pos = pos+1+b
for i := 0; i < b; i++ {
var obj interface{}

obj, pos = Decode(data, pos)
slice = append(slice, obj)
}
return slice, pos

Encode:

    WriteSliceHeader := func(length int) {
    if length < 56 {
    // buff.WriteString(string(length + 128))
    buff.WriteByte(byte(length + 128)) // Change from use WriteString to use WriteByte 
    } else {
    b2 := ToBin(uint64(length), 0)
    // buff.WriteString(string(len(b2) + 183) + b2)
    buff.WriteByte(byte(len(b2) + 183)) // Change from use WriteString to use WriteByte 
    buff.WriteString(b2)
    }
    

rlp_test.go

    func TestEncode(t *testing.T) {
    strRes := "Cdog"
    bytes := Encode("dog")
    str := string(bytes)
    if str != strRes {
    t.Error(fmt.Sprintf("Expected %q, got %q", strRes, str))
    }
    dec,_ := Decode(bytes, 0)
    fmt.Printf("raw: %v encoded: %q == %v\n", dec, str, "dog")


    sliceRes := "\x83CdogCgodCcat"
    strs := []string{"dog", "god", "cat"}
    bytes = Encode(strs)
    slice := string(bytes)
    if slice != sliceRes {
    t.Error(fmt.Sprintf("Expected %q, got %q", sliceRes, slice))
    }

    dec,_ = Decode(bytes, 0)
    fmt.Printf("raw: %v encoded: %q == %v\n", dec, slice, strs)
    } 
    
result:

    init Ethereum VM
    stack size = 256

    # processing Tx (8fee28c5311d91212d92cbf14548e9e96ab39a)
    # fee = 0.000000, ops = 12, sender = 1234567890, value = 20

    # processing Tx (3ab78afb9e495acc6eabd8982730dbb679db2f)
    # fee = 0.000000, ops = 2, sender = 1234567890, value = 20
    0   67 [10 6 0 0 0 0]
    0   67 [10 6 0 0 0 0]
    # finished processing Tx

    2   66 [10 10 0 0 0 0]
    3   67 [255 7 0 0 0 0]
    4   81 [20 255 0 0 0 0]
    ... JMP 7 ...
    7   66 [30 31 0 0 0 0]
    8   67 [255 22 0 0 0 0]
    9   81 [31 255 0 0 0 0]
    ... JMP 22 ...
    # finished processing Tx

    rlp encoded Tx "\x01\x00\x010\x00\n1234567890\x00\x010\x00\x0220\x00\x010\x01\x00\x06395843\x00\x06657986\x00\t335612448\x00\x06524099\x00\b16716881\x00\x010\x00\b13114947\x00\a2039362\x00\a1507139\x00\b16719697\x00\a1048387\x00\x0565360"
    block enc "\x01\x00\x010\x00\x00\x00\x00\x00 mi\u007f\xb0\b\x91A\xd0P\x1f\xb7\xf9\x1cd\xb1ӑ{\xc2ѕ&X\xa4\u007f\xad\xea\x95\v\xed\xc8="
    block hash "39e507d0bc23032af0fc1ad2d979c27d31968fc9ed69c4e3000fed172305e24d"

    Process finished with exit code 0
    
test result:

    === RUN   TestEncode
    raw: [100 111 103] encoded: "Cdog" == dog
    raw: [[100 111 103] [103 111 100] [99 97 116]] encoded: "\x83CdogCgodCcat" == [dog god cat]
    --- PASS: TestEncode (0.00s)
    === RUN   TestRlpEncode
    --- PASS: TestRlpEncode (0.00s)
    PASS

    Process finished with exit code 0
    

commit:8b993ac
comment:Reset stack pointer on run
file:
vm.go

更新statck pointer運作機制，舊的寫法可能產生多線程問題。

    type Vm struct {
    // Memory stack
    stack map[string]string
    // Index ptr
    // iptr int // Remove
    memory map[string]map[string]string
    }

    func (vm *Vm) RunTransaction(tx *Transaction, cb TxCallback) {
    ...
    stPtr := 0 // Each transaction for run is with reset ptr beginning.

    // for vm.iptr < len(tx.data) { ... // 多次RunTransaction共用vm.iptr可能產生多線程問題
    for stPtr < len(tx.data) { ...


commit:8b993ac
comment:Reset stack pointer on run
file:
vm.go
// XXX In the future there's no need to cast to string because they'll end up being big numbers (strings)
這段註解在這次的更改中終於解決了，不需要特地轉成字串來序列化，因為捨棄了舊的RlpEncode方法使用新的Encode方法，新方法都是用[]byte來處理。

- "0", // TODO last Tx
+ tx.lastTx,
這個暫時的參數"0"現在被移除了，正式加入了lastTx作為序列化結構之一。
在NewTransaction時要記得初始化，tx.lastTx = "0"。

加入了func (tx *Transaction) UnmarshalRlp(data []byte)，做為transaction的反序列化方法。


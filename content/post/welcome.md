---
title: "Welcome"
date: 2018-10-03T11:09:53+08:00
draft: true
---









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


commit:38b37f2
comment:Removed slice appending for integers
file:
rlp.go




commit:db5092c
comment:Add TestMultiEncode
file:
rlp_test.go

=== RUN   TestEncode
raw: [100 111 103] encoded: "Cdog" == dog
raw: [[100 111 103] [103 111 100] [99 97 116]] encoded: "\x83CdogCgodCcat" == [dog god cat]
--- PASS: TestEncode (0.00s)
=== RUN   TestMultiEncode
"\x83\x83A1A2A3\x84FstringGstring2d\x86A0J1234567890A\x00B20A0\x82F395843F657986y\x00p\x86A0J1234567890A\x00B20A0\x8cF395843F657986I335612448F524099H16716881A0H13114947G2039362G1507139H16719697G1048387E65360Dtest"
[[[49] [50] [51]] [[115 116 114 105 110 103] [115 116 114 105 110 103 50] [134 65 48 74 49 50 51 52 53 54 55 56 57 48 65 0 66 50 48 65 48 130 70 51 57 53 56 52 51 70 54 53 55 57 56 54] [134 65 48 74 49 50 51 52 53 54 55 56 57 48 65 0 66 50 48 65 48 140 70 51 57 53 56 52 51 70 54 53 55 57 56 54 73 51 51 53 54 49 50 52 52 56 70 53 50 52 48 57 57 72 49 54 55 49 54 56 56 49 65 48 72 49 51 49 49 52 57 52 55 71 50 48 51 57 51 54 50 71 49 53 48 55 49 51 57 72 49 54 55 49 57 54 57 55 71 49 48 52 56 51 56 55 69 54 53 51 54 48]] [116 101 115 116]]
--- PASS: TestMultiEncode (0.00s)
PASS
coverage: 61.0% of statements

Process finished with exit code 0



commit: 2bc701f
comment: use Benchmark to test perfomance of EncodeDecode.
file:
rlp_test.go

基本的 benchmark 測試也是透過 go test 指令，不同的是要加上 -bench=.，這樣才會跑 benchmark 部分，否則預設只有跑測試程式，大家可以看到 -4 代表目前的 CPU 核心數，也就是 GOMAXPROCS 的值，另外 -run 可以用在跑特定的測試函示，但是假設沒有指定 -run 時，你會看到預設跑測試 + benchmark，所以這邊補上 -run=none 的用意是不要跑任何測試，只有跑 benchmark，最後看看輸出結果，其中 1000000 代表一秒鐘可以跑 100 萬次，每一次需要 1253 ns，如果你想跑兩秒，請加上此參數在命令列 -benchtime=2s，但是個人覺得沒什麼意義。

```go
func BenchmarkEncodeDecode(b *testing.B) {
    for i := 0; i < b.N; i++ {
        bytes := Encode([]string{"dog", "god", "cat"})
        Decode(bytes, 0)
    }
}
```

    goos: darwin
    goarch: amd64
    BenchmarkEncodeDecode-4        1000000          1253 ns/op
    PASS

    Process finished with exit code 0


commit: ac8a06f
comment: 1.(un)marshal blocks and transactions 2.Test code updated
file:
rlp_test.go

```go
func (block *Block) UnmarshalRlp(data []byte) {
    t, _ := Decode(data,0)
    if slice, ok := t.([]interface{}); ok {
        if header, ok := slice[0].([]interface{}); ok {
            if number, ok := header[0].(uint8); ok {
                block.number = uint32(number)
            }

            if prevHash, ok := header[1].([]byte); ok {
                block.prevHash = string(prevHash)
            }

            // sha of uncles is header[2]   

            if coinbase, ok := header[3].([]byte); ok {
                block.coinbase = string(coinbase)
            }

            // state is header[header[4]

            // sha is header[5]

            // It's either 8bit or 64
            if difficulty, ok := header[6].(uint8); ok {
                block.difficulty = uint32(difficulty)
            }
            if difficulty, ok := header[6].(uint64); ok {
                block.difficulty = uint32(difficulty)
            }

            if time, ok := header[7].([]byte); ok {
                fmt.Sprintf("Time is: ", string(time))
            }

            if nonce, ok := header[8].(uint8); ok {
                block.nonce = uint32(nonce)
            }
        }

        if txSlice, ok := slice[1].([]interface{}); ok {
            block.transactions = make([]*Transaction, len(txSlice))

            for i, tx := range txSlice {
                if t, ok := tx.([]byte); ok {
                    tx := &Transaction{}
                    tx.UnmarshalRlp(t)

                    block.transactions[i] = tx
                }
            }
        }
    }
}
```
transaction.go:
- return []byte(Encode(preEnc))
+ return Encode(preEnc)

稍微優化一下，轉[]byte是不必要的，因為本Encode的回傳值就是[]byte。

```go
t := blck.MarshalRlp()
copyBlock := &Block{}
copyBlock.UnmarshalRlp(t)

fmt.Println(blck)
fmt.Println(copyBlock)
```

序列化與反序列化block，並打印出來。

    &{1 1234 [] me 10 {13755934970731041952 1196413 0x11a71a0} 0 [0xc0000a4090 0xc0000a4000]}
    &{1 1234 [] me 10 {0 0 <nil>} 0 [0xc0000a4120 0xc0000a41b0]}

    Process finished with exit code 0

這邊要注意，現在傳入func Encode(object interface{}) []byte 中的參數限定為uint32, uint64, string, []string ([]byte處理被隱藏起來了)，傳錯會當掉。
```go
/* I made up the block. It should probably contain different data or types. It sole purpose now is testing */
// 測試模擬用
header := []interface{}{
    block.number,
    block.prevHash,
    //Sha of uncles
    "",
    block.coinbase,
    //root state
    "",
    string(Sha256Bin([]byte(RlpEncode(encTx)))),
    block.difficulty,
    block.time.String(),
    block.nonce,
    // extra?
}

encoded := Encode([]interface{}{header, encTx})
```

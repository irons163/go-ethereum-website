---
title: "eef6b5d"
date: 2018-10-08T14:33:44+08:00
draft: true
---
##### commit: eef6b5d
##### comment: Updated rlp encoding. (requires verification)
##### file: rlp_test.go

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

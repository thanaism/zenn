---
title: 'EntryPointとPaymaster'
---

## bundlerの立て替えをどう回収するか

CAが起点になれない問題を解決するために、alt-mempoolと代理送信をbundlerが担ってくれるようになりました。
しかし、ここで次の問題が出ます。

bundlerが素のtxを送る以上、そのガス代を最初に払うのはbundler自身です。
単にユーザーのために立て替えているだけでは、一方的な持ち出しになる。ではどうやってbundlerというエコシステムを成り立たせているのでしょうか？

端的に言えば、ガスを立て替えた分、あとから回収する機構が用意されています。
payloadの実行に伴って、使用したガス相当をユーザー側のウォレット、または第三者からbundlerへ戻す。
これを保証するのが**EntryPoint**と呼ばれるチェーンごとにシングルトンのコントラクトです。

複雑になってきたので、ここまでの内容をいったん整理しましょう。

ユーザーはネイティブtxを組み立てる代わりにUserOperationという署名済みpayloadを用意し、その検証はコントラクトの任意ロジックで行うというのがAAの核でした。
そして、そのUserOperationの代理送信をするのがbundlerでしたが、このままではガスの払い戻し機構が存在しません。

---

ガス払い戻しのために、bundlerはERC-4337を実装したコントラクトウォレットに直接UserOperationを送るのではなく、**ガスの払い戻しラッパーであるEntryPointを経由させて**、目的のERC-4337コントラクトに送ります。

なぜなら、ガスの払い戻しを個々のコントラクトウォレット側の責務にするには、ウォレットの任意コード内に共通のガス精算ルールを強制する必要があり、それは困難かつ非効率だからです。

したがって、bundlerが生成するtxは常に、払い戻しを強制化できる共通のEntryPointのメソッドをcallするという形になります。

具体的にはEntryPointの`handleOps()`メソッドがそれに該当します。

かなり単純化すると、`handleOps()`がやっているのは次のような処理です。

```solidity
function handleOps(UserOperation[] ops, address beneficiary) {
    // 1. validation loop
    for (op in ops) {
        validationGasBefore = gasleft();
        wallet = op.sender; // コントラクトウォレットのアドレス

        wallet.validateUserOp(op); // コントラクトウォレットの検証フェーズ

        if (op.paymaster != address(0)) {
            paymaster.validatePaymasterUserOp(op); // 第三者による代理支払の場合は支払可否を検証
        }

        require(EntryPoint内のdepositで最大ガス代を払える);

        op.preOpGas = validationGasBefore - gasleft() + op.preVerificationGas;
    }

    // 2. execution loop
    for (op in ops) {
        executionGasBefore = gasleft();

        wallet.execute(op.callData); // コントラクトウォレットの実行フェーズ

        actualGas = op.preOpGas + (executionGasBefore - gasleft());
        actualGasCost = actualGas * op.gasPrice;
        walletまたはpaymasterのdepositからactualGasCostを差し引く;
    }

    bundlerの指定したbeneficiaryへ集めた手数料を支払う;
}
```

厳密な実装はもっと複雑ですが、重要なのは、EntryPointが「検証する」「実行する」「ガス代を回収してbundlerへ渡す」という順序を一箇所で握っていることです。
また、個別のUserOperation処理で発生したgas代は、EntryPoint内でUserOperationを処理するごとに`gasleft()`グローバル関数の結果を取得した差分と固定オーバーヘッドから算定されていることがわかります（直感に反するかもしれませんがtx内でコントラクトからガス残量は参照できます）。

したがって、コントラクトウォレット側は検証用の`validateUserOp`メソッドと実行用の`execute`メソッドだけを用意しておけばよく、支払・検証・実行の分離はEntryPointがそのフローを強制してくれます。

### Paymasterコントラクト

擬似コードの中にサラッと含めてしまいましたが、この払い戻しという形式の機構であれば、支払い元を柔軟に変更できます。つまり、第三者にガスを支払ってもらうという構成も可能です。
これがいわゆるPaymasterコントラクトという存在で、ユーザーは事前にPaymasterから代理支払いの許可を署名なりで取得し、その取得した支払許可証を自身のUserOperationに含めます。

するとEntryPointが払い戻しを処理する際に、Paymasterコントラクトに「あなたの支払許可証が届いてるので支払ってください」と仲介する感じですね。
その検証処理を担うのがPaymasterの`validatePaymasterUserOp()`となります。

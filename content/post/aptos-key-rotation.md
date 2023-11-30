---
title: "Aptos Key Rotation"
date: 2023-07-08
draft: false
subtitle:    ""
description: ""
author:      "Ashley Hsu"
image:       ""
tags:        ["Aptos"]
categories:  ["Blockchain"]
---

# Overview

在本文中，我們將深入探討 Aptos 密鑰輪換的概念及其重要性。在開始之前，讓我們先了解什麼是 Aptos 帳戶、公鑰和私鑰的作用。當私鑰洩漏時，你的帳戶就可能遭到攻擊，但若想保留原有帳戶並確保資產安全，就需要使用密鑰輪換技術。Aptos 帳戶支持密鑰輪換，讓你更改私鑰而不必創建新帳戶，繼而保留原有帳戶的資產和身份。

接下來，我們將介紹如何使用 Aptos cli 及 Python SDK 完成密鑰輪換，讓你能夠隨時保持帳戶資產及身份的安全。 


# 什麼是帳戶（account）？

![](https://hackmd.io/_uploads/B1qK7JEt3.png)

每個 Aptos 帳戶代表著區塊鏈上一個可以發送交易的實體，並由特定的 32 字節帳戶地址識別。這個帳戶是 Move 模組和 Move 資源的容器，可以包含區塊鏈資產（例如 coin 和 NFT）。這些資產在區塊鏈帳戶中表示為資源，同時帳戶也可以執行各種操作，例如將資源發送給其他人。就像在 Web2 的世界中，你的電子郵件地址代表著你的電子郵件帳戶，而在 Aptos 區塊鏈中，帳戶地址代表著帳戶本身，你可以透過這個地址進行收發資產等操作。

## 公鑰（public key）、私鑰（private key）

私鑰在 Aptos 區塊鏈與其他區塊鏈一樣，可以用來簽署交易以使其被認可和驗證，就像在現實生活中需要簽名或輸入密碼才能夠轉帳。簽署過後的交易才能夠對你的帳戶資產進行更改，但是只有擁有私鑰的人才能夠簽署這筆交易，因此私鑰是非常重要且敏感的一組數據。

在其他區塊鏈上，你的公開身份即為私鑰對應的公鑰，地址可以從公鑰被推算出來，公鑰也被用來驗證簽章。然而，Aptos 支援密鑰輪換，導致公鑰可能會發生變化，因此使用地址來代表帳戶。傳統上，每個帳戶都有一個特定的地址作為其識別符號，而且這個地址是不會改變的。在 Aptos 中，即使您更改了身份驗證密鑰（其中包含公鑰），帳戶地址仍然是唯一的，並且不會發生變化。

## 身份驗證密鑰（authentication key）

Aptos 區塊鏈支援單簽和多簽帳戶。多簽帳戶允許多個用戶共同執行數位簽章來管理帳戶資產。例如，想像一個擁有兩把鎖和兩把鑰匙的保險箱，一把鑰匙由 Alice 持有，另一把則由 Bob 掌管。打開此保險箱的唯一辦法就是兩個人同時提供鑰匙開鎖，只有其中一把鑰匙時則無法打開。多簽帳戶與此類似，它是代表多方的單一帳戶，所有發生的交易都需要所有參與方的簽名，也可以被設定一個閾值，舉個例子，2-of-3 代表三位中只要有兩個簽署即可完成交易。

![](https://hackmd.io/_uploads/H11DHlEKh.png)

多簽帳戶使用多個（私鑰，公鑰）密鑰對，沒有單個公鑰會代表這個帳戶。因此，我們需要一個代表此帳戶的密鑰，以便封裝多簽帳戶中所有用戶的公鑰，這個代表此帳戶的密鑰也就是所謂的身份驗證密鑰。

身份驗證密鑰是表示多簽帳戶中所有用戶的一種方式。簡而言之，身份驗證密鑰是通過連接所有參與用戶的公鑰的串聯散列創建的。
```
auth_key = sha3-256(pubkey_1 | . . . | pubkey_n | K | 0x01)
```
其中`K`是驗證交易所需的簽名閾值，`0x01`為 1-byte 多簽方案標識符。

對於單簽帳戶，也存在身份驗證密鑰，單簽帳戶的身份驗證密鑰只封裝了一個公鑰來代表帳戶，這樣做是為了在所有類型的帳戶上保持一致性。
```
auth_key = sha3-256(pubkey_A | 0x00)
```
其中 `0x00` 為 1-byte 單簽方案標識符。

我們可以說，身份驗證密鑰是私鑰的廣義公共表示。

## 帳戶地址

在帳戶創建過程中，產生一組公鑰及私鑰後，會計算出一個 32 字節的身份驗證密鑰，這個身份驗證密鑰將作為帳戶地址。

單簽帳戶：

```
auth_key = sha3-256(pubkey_A | 0x00)
```
其中 `0x00` 為 1-byte 單簽方案標識符。

多簽帳戶：
```
auth_key = sha3-256(pubkey_1 | . . . | pubkey_n | K | 0x01)
```
其中`K`是驗證交易所需的簽名閾值，`0x01`為 1-byte 多簽方案標識符。


但是，密鑰輪換後，身份驗證密鑰會發生變化，當生成新的公鑰-私鑰對時，公鑰將輪換密鑰。**但帳戶地址不會改變**。因此，僅在最初，32 字節身份驗證密鑰才會與 32 字節帳戶地址相同。帳戶與金鑰解耦的方法使 Aptos 能夠無縫新增新的數位簽名演算法以支援公鑰和私鑰型別。

> 帳戶創建後，儘管私鑰、公鑰和認證密鑰可能發生變化，但帳戶地址將**保持不變**。



# 為什麼我們需要密鑰輪換？

當私鑰洩漏時，你的帳戶就可能遭到攻擊。在 web2 中，多數人會定期修改密碼，減少被盜的風險，還有在 Instagram 或 Facebook 帳戶被盜後，你可以透過其他方式驗證身份，修改密碼並拿回自己的帳號。但是，在大多數區塊鏈中，情況並不那麼簡單。私鑰與公鑰成對出現，而鏈上身份與私鑰綁定，唯一的解決方法是創建一個新帳戶，使用新的公鑰和私鑰對並將所有資產轉移到該帳戶中。

然而，這種方法會造成很多問題，例如失去原有的帳戶資產、OG 身份的象徵、以及參與過的活動的紀念等。因此，密鑰輪換技術應運而生。Aptos 帳戶支援密鑰輪換，讓你更改私鑰而不必創建新帳戶，從而保留原有帳戶的資產和身份。這意味著你可以保持原有地址，其他人仍然可以使用你原本的地址向你發送資產。唯一會改變的是身份驗證密鑰，因為它是由公鑰計算出來的。

密鑰輪換技術實現了**鏈上身份**與**安全性**分離，當私鑰洩漏時，你可以替換掉私鑰，從而保護資產免於受到損失或竊取。此外，密鑰輪換也使得我們可以定期更改密鑰，減少私鑰被盜取的風險，同時也為使用多簽帳戶提供了更多彈性。

# Authentication key rotation

密鑰輪換，在 Aptos 中準確來說稱為身份驗證密鑰輪換（Authentication key rotation）。 

[account.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/account.move) 模塊包含了 Aptos 帳戶相關的所有函數。
用戶可以透過 `account::rotate_authentication_key` 函數來達成密鑰輪換。
```rust=
public entry fun rotate_authentication_key(
    account: &signer,
    from_scheme: u8,
    from_public_key_bytes: vector<u8>,
    to_scheme: u8,
    to_public_key_bytes: vector<u8>,
    cap_rotate_key: vector<u8>,
    cap_update_table: vector<u8>,
) acquires Account, OriginatingAddress {

    ...
}
```

為了授權輪換，我們需要兩個簽名：
1. `cap_rotate_key` 是指帳戶目前擁有者對 `RotationProofChallenge` 的簽名，證明用戶打算並具有能力輪換此帳戶的身份驗證金鑰
2. `cap_update_table` 是指所需輪換至的新金鑰（帳戶擁有者想要輪換的金鑰）對`RotationProofChallenge` 的簽名，證明用戶擁有新的私鑰，並有權更新 `OriginatingAddress` 映射與新地址映射 `<new_address, originating_address>`。

為了驗證這兩個簽名，我們需要它們對應的公鑰和公鑰方案：我們使用 `from_scheme` 和`from_public_key_bytes`驗證`cap_rotate_key`，使用`to_scheme`和`to_public_key_bytes`驗證`cap_update_table`。

單簽為`ED25519_SCHEME`，多簽則為`MULTI_ED25519_SCHEME`。



## 使用 Aptos cli 達成單簽帳戶密鑰輪換

初始狀態，可以看到 account address 與 authentication key 相同。
![](https://hackmd.io/_uploads/Bywsu-NY2.png)

1. 產生密鑰：
```bash
aptos key generate --key-type ed25519 --output-file output.key
{
  "Result": {
    "PrivateKey Path": "output.key",
    "PublicKey Path": "output.key.pub"
  }
}
```

2. 利用 aptos cli 做 key rotation，參數可以選擇直接填入私鑰或 key file:
```
aptos account rotate-key --new-private-key-file output.key
```
```bash
aptos account rotate-key --new-private-key 0xae249782eedbfafcc7da157542d1f97d97385cbda36338c806475a3ce127fd90

Do you want to submit a transaction for a range of [52100 - 78100] Octas at a gas unit price of 100 Octas? [yes/no] >
yes
{
  "transaction_hash": "0x75c412d82e038f87cec14c6c8b08c7fb08290118437e18433700c0e5d0262854",
  "gas_used": 521,
  "gas_unit_price": 100,
  "sender": "148f1f6f88d690a04451fa8a548f21129b537cf511cedaa66a6ad93abc83a607",
  "sequence_number": 0,
  "success": true,
  "timestamp_us": 1688636116823320,
  "version": 571106848,
  "vm_status": "Executed successfully"
}
Do you want to create a profile for the new key? [yes/no] >
yes
Enter the name for the profile
Update
Profile Update is saved.
{
  "Result": {
    "message": "Profile Update is saved.",
    "transaction": {
      "transaction_hash": "0x75c412d82e038f87cec14c6c8b08c7fb08290118437e18433700c0e5d0262854",
      "gas_used": 521,
      "gas_unit_price": 100,
      "sender": "148f1f6f88d690a04451fa8a548f21129b537cf511cedaa66a6ad93abc83a607",
      "sequence_number": 0,
      "success": true,
      "timestamp_us": 1688636116823320,
      "version": 571106848,
      "vm_status": "Executed successfully"
    }
  }
}
```



3. 回到 Aptos explorer 查看，authentication key 已改變，account address 維持相同。
![](https://hackmd.io/_uploads/H1CMsWNY3.png)


aptos cli 也可以透過 public key 來反查出帳戶地址：
```
aptos account lookup-address --public-key-file output.key.pub
{
  "Result": "148f1f6f88d690a04451fa8a548f21129b537cf511cedaa66a6ad93abc83a607"
}
```

## 使用 Python SDK 將單簽帳戶輪換為多簽帳戶

以 aptos-core 中 python sdk 的 [multisig 範例](https://github.com/aptos-labs/aptos-core/blob/main/ecosystem/python/sdk/examples/multisig.py)作為參考，可以根據自己的使用情境進行修改。

總結一下流程：
1. 建立一個單簽帳戶
2. 建立一個多簽帳戶，這邊是建立一個 2-of-3 多簽帳戶
3. 簽署 rotation proof challenge
4. 執行輪換 authentication key

### 環境安裝

#### 安裝 Poetry

推薦使用 [Poetry](https://python-poetry.org/) 進行 Python 套件管理。
1. 安裝 Poetry
```bash
curl -sSL https://install.python-poetry.org | python3 -
```
2. 將 Poetry 加入 PATH
依照自己的系統環境，加到 `.zshrc` 或 `.bashrc`
```bash
export PATH="$HOME/.local/bin:$PATH"
```
3. 確認 Poetry 安裝完成
```
poetry --version
```

#### 安裝 Aptos Python SDK
```
pip3 install aptos-sdk
```
更多詳細資訊，可參考[官網文檔](https://aptos.dev/sdks/python-sdk)。

### 執行 Aptos SDK Example

```bash
git clone https://github.com/aptos-labs/aptos-core.git
```
```
cd aptos-core/ecosystem/python/sdk
```
- 重現專案的 Poetry 虛擬環境
```
poetry env use python3
```
- 安裝套件
```
poetry install
```
- 執行
```
poetry run python -m examples.multisig
```

### 設定
引入要用到的庫
```python=
from aptos_sdk.account import Account, RotationProofChallenge
from aptos_sdk.account_address import AccountAddress
from aptos_sdk.authenticator import Authenticator, MultiEd25519Authenticator
from aptos_sdk.bcs import Serializer
from aptos_sdk.client import FaucetClient, RestClient
from aptos_sdk.ed25519 import MultiPublicKey, MultiSignature
from aptos_sdk.transactions import (
    EntryFunction,
    RawTransaction,
    Script,
    ScriptArgument,
    SignedTransaction,
    TransactionArgument,
    TransactionPayload,
)
from aptos_sdk.type_tag import StructTag, TypeTag
```


設定連接節點、水龍頭資訊，這邊使用 devnet 環境：
```python=
NODE_URL = os.getenv("APTOS_NODE_URL", "https://fullnode.devnet.aptoslabs.com/v1")
FAUCET_URL = os.getenv(
    "APTOS_FAUCET_URL",
    "https://faucet.devnet.aptoslabs.com",
)
```
初始化客戶端
```python=
rest_client = RestClient(NODE_URL)
faucet_client = FaucetClient(FAUCET_URL, rest_client)
```

### 建立單簽帳戶
```python=
deedee = Account.generate()
faucet_client.fund_account(deedee.address(), 50_000_000)
deedee_balance = rest_client.account_balance(deedee.address())
```

```
Deedee's address:    0x7ee730f9bb87e78b0f51b4399113af2ed6bc50d0a8e0bc27f914c287af6a9d49
Deedee's auth key:   0x7ee730f9bb87e78b0f51b4399113af2ed6bc50d0a8e0bc27f914c287af6a9d49
Deedee's public key: 0x641cb41098c6cea2c2643508ee28e116459c666e353d6dc708290c5dd8f27169
Deedee's balance:    50000000
```

### 建立多簽帳戶

這邊我們選擇建立 2-of-3 多簽帳戶，所以先建立三個單簽帳戶，這樣會有三對公鑰私鑰對。

```python=
# generate account
alice = Account.generate()
bob = Account.generate()
chad = Account.generate()

# fund account
faucet_client.fund_account(alice.address(), 10_000_000)
faucet_client.fund_account(bob.address(), 20_000_000)
faucet_client.fund_account(chad.address(), 30_000_000)
```

```
=== Account addresses ===
Alice: 0xe2f0ac3bf3066f83c3f13df40eb1f1e980b15bbf6198dfee0d4bfd741baf92a8
Bob:   0x55234e90dec96a74938a4da2f7815b97494fc3c4b4d8782fa9c0e83399613310
Chad:  0x9b2ff4d016b1dead4c79487275bcb332ce56d0c707bc07cf0f15e223e64122a0

=== Authentication keys ===
Alice: 0xe2f0ac3bf3066f83c3f13df40eb1f1e980b15bbf6198dfee0d4bfd741baf92a8
Bob:   0x55234e90dec96a74938a4da2f7815b97494fc3c4b4d8782fa9c0e83399613310
Chad:  0x9b2ff4d016b1dead4c79487275bcb332ce56d0c707bc07cf0f15e223e64122a0

=== Public keys ===
Alice: 0x53a03cc129319935ddabc8ac2afb0c5e320cd5bda347413fb6fe33bcdd0b7fb3
Bob:   0x51a95405c021b0449d11d71797ecb72e931c045a55f69ae96aa6c87d55ba8098
Chad:  0x74ac180a4c5b4f6e14f13b8a5e6dd19d102c0e3543268816dcb6484f0da5444d
```
接下來使用上一步產生的公鑰去生成 2-of-3 多簽帳戶公鑰和地址。

```python=
# generate public key from Alice, Bob and Chad's public key
threshold = 2
multisig_public_key = MultiPublicKey(
    [alice.public_key(), bob.public_key(), chad.public_key()], threshold
)
# get address
multisig_address = AccountAddress.from_multi_ed25519(multisig_public_key)
# fund account
faucet_client.fund_account(multisig_address, 40_000_000)
```

```
=== 2-of-3 Multisig account ===
Account public key: 2-of-3 Multi-Ed25519 public key
Account address:    0x77e7728ef2189e48b3b898087c460a63f14955bcc6e4915b95aab8f864be8d05
```

### 簽署 rotation proof challenge
在調用 `rotate_authentication_key` 之前，需要準備傳入的參數，分別是 `cap_rotate_key` 及 `cap_update_table`。
`cap_rotate_key` 表 Deedee 批准此身份驗證密鑰輪換。
`cap_update_table` 驗證多簽帳戶是否批准身份驗證密鑰輪換。

```python=
rotation_proof_challenge = RotationProofChallenge(
    sequence_number=0,
    originator=deedee.address(),
    current_auth_key=deedee.address(),
    new_public_key=multisig_public_key.to_bytes(),
)

serializer = Serializer()
rotation_proof_challenge.serialize(serializer)
rotation_proof_challenge_bcs = serializer.output()

cap_rotate_key = deedee.sign(rotation_proof_challenge_bcs).data()

cap_update_table = MultiSignature(
    multisig_public_key,
    [
        (bob.public_key(), bob.sign(rotation_proof_challenge_bcs)),
        (chad.public_key(), chad.sign(rotation_proof_challenge_bcs)),
    ],
).to_bytes()
```

```
=== Signing rotation proof challenge ===
cap_rotate_key:   0x497d25f09c29df422e620dd8eaf44ee9b7bbc2844b5143ade652d9a0d084cd1b59e66e6d0c8ebd3a4e6d5b70ddc0f635bbe321ea6ac2902d2ab07e3248e5ac08
cap_update_table: 0xe40f69589e89beca67362c2cadb145df92d77351bfe77aa9318af88f1453576b2fc8e8723fca961e696014a0077946a762c18d7a64547b3ec1f0539f88889303d89de34be00ada58151a124cc3610226ca1c0ca9814be9040bba3c6d02f169a53b922ce6cb026165746c8e90837affac8692206dc9eb0f82f0117742d331670a60000000
```

### 執行輪換 authentication key

現在可以提交身份驗證密鑰輪換的交易。執行後，輪換後的身份驗證密鑰與多簽帳戶的地址相匹配。
```python=
from_scheme = Authenticator.ED25519
from_public_key_bytes = deedee.public_key().key.encode()
to_scheme = Authenticator.MULTI_ED25519
to_public_key_bytes = multisig_public_key.to_bytes()

entry_function = EntryFunction.natural(
    module="0x1::account",
    function="rotate_authentication_key",
    ty_args=[],
    args=[
        TransactionArgument(from_scheme, Serializer.u8),
        TransactionArgument(from_public_key_bytes, Serializer.to_bytes),
        TransactionArgument(to_scheme, Serializer.u8),
        TransactionArgument(to_public_key_bytes, Serializer.to_bytes),
        TransactionArgument(cap_rotate_key, Serializer.to_bytes),
        TransactionArgument(cap_update_table, Serializer.to_bytes),
    ],
)

signed_transaction = rest_client.create_bcs_signed_transaction(
    deedee, TransactionPayload(entry_function)
)

# auth key before key rotation
auth_key = rest_client.account(deedee.address())["authentication_key"]
print(f"Auth key pre-rotation: {auth_key}")

# send key rotation tx
tx_hash = rest_client.submit_bcs_transaction(signed_transaction)
rest_client.wait_for_transaction(tx_hash)
print(f"Transaction hash:      {tx_hash}")

# auth key after key rotation
auth_key = rest_client.account(deedee.address())["authentication_key"]
print(f"New auth key:          {auth_key}")
print(f"1st multisig address:  {multisig_address}")
```

```
=== Submitting authentication key rotation transaction ===
Auth key pre-rotation: 0x7ee730f9bb87e78b0f51b4399113af2ed6bc50d0a8e0bc27f914c287af6a9d49
Transaction hash:      0x16046f72559268dc4025d28bbc2d8c91ab7a780e237cbf430965477910014df0
New auth key:          0x77e7728ef2189e48b3b898087c460a63f14955bcc6e4915b95aab8f864be8d05
1st multisig address:  0x77e7728ef2189e48b3b898087c460a63f14955bcc6e4915b95aab8f864be8d05
```


# 總結
Aptos 帳戶讓鏈上地址身份與私鑰解耦，提供了單簽及多簽帳戶，最重要的是具有密鑰輪換的功能。
地址在創建帳號後維持不變，即使在密鑰輪換後仍然維持相同。
密鑰輪換改變的是公鑰私鑰對以及身份驗證密鑰。

# ？
如果想把兩個帳號換成同個 auth key 可以嗎？  
不可以。
```bash
{
  "Error": "Simulation failed with status: Move abort in 0x1::table: 0x6407"
}
```

有個例外，當 auth key == address 的時候可以。  
```bash
address: 
0xf2e692d2c041973da02f1dc9bd2d8aae30af6f27e706104d8f62cb07a73fcd54
auth key:
0x1be04e2fc6271fcfe03ecaf8e0dba4c0aed93439a6e3efcf40239a547b6a4268
===
address: 
0x1be04e2fc6271fcfe03ecaf8e0dba4c0aed93439a6e3efcf40239a547b6a4268
auth key:
0x1be04e2fc6271fcfe03ecaf8e0dba4c0aed93439a6e3efcf40239a547b6a4268
```


# Reference
- https://aptos.dev/concepts/accounts/
- https://medium.com/@martian-wallet/accounts-in-aptos-1ecc3f0b1213
- https://forum.aptoslabs.com/t/aptos-review-account-system/150437
- https://aptos.dev/tutorials/your-first-multisig/
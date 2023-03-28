# Lesson 1: 簡單的 FunC 智慧合約
## 介紹

在這個教程中，我們將使用 FunC 語言在 The Open Network 測試網上撰寫您的第一個智慧合約，並使用 [toncli](https://github.com/disintar/toncli) 將其部署到測試網上，並使用 Fift 語言中的訊息進行測試。

> *部署是將合約轉移到網路上（在這種情況下，是將智慧合約轉移到區塊鏈上）

## 需求
完成此教程，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面。

## 智慧合約
我們將創建的智慧合約應具備以下功能：

- 將一個名為 *total* 的 64 位元無符號數字儲存在其數據中；
- 當接收到內部入站訊息時，合約必須從訊息正文中取出一個 32 位元無符號整數，將其加到 *total* 中並將其儲存在合約數據中；
- 智慧合約必須提供一個 *get total* 方法，允許您返回 *total* 的值；
- 如果入站訊息的正文少於 32 位元，則合約必須拋出一個例外。

## 使用 toncli 創建專案

在控制台中運行以下命令：

    toncli start wallet
	cd wallet


Toncli 創建了一個簡單的錢包專案，您可以在其中看到 4 個文件夾：
- build;
- func;
- fift;
- test;

在此階段，我們對 func 和 fift 文件夾感興趣，分別在其中使用 FunС 和 Fift 編寫代碼。

##### 什麼是 FunC 和 Fift

FunC 高級語言用於在 TON 上編程智能合約。FunC 程序被編譯成 Fift 組合語言代碼，該代碼為 TON 虛擬機（TVM）生成相應的字節碼（關於 TVM 更多信息可在[此](https://ton-blockchain.github.io/docs/tvm.pdf)找到）。進一步地，這個字節碼（實際上是一個 cell 樹，就像 TON 區塊鏈中的任何其他數據一樣）可以用於在區塊鏈上創建智能合約，也可以在 TON 虛擬機（TON Virtual Machine）的本地實例上運行。

有關 FunC 的更多資訊可點[此](https://ton-blockchain.github.io/docs/#/smart-contracts/)查看

##### 讓我們為我們的代碼準備一個文件

進入 func 文件夾：

    cd func

並打開 code.func 文件，你將看到簡單的錢包智能合約，在屏幕上刪除所有代碼，我們準備開始編寫我們的第一個智能合約。

## 外部方法

在 TON 網路上的智慧合約具有兩個保留方法可以被存取。

首先是 `recv_external()`。當合約從外部世界（即不從 TON）收到請求時，此函數會被執行。例如，當我們自己形成一則訊息並透過 lite-client（關於安裝 [lite-client](https://ton-blockchain.github.io/docs/#/compile?id=lite-client)）傳送時。

其次是 `recv_internal()`。當在 TON 內部發生時，例如當任何合約參照我們的合約時，此函數會被執行。

 > 輕量版用戶端（英文：lite-client）是一個軟體，它連接到完整節點以與區塊鏈互動。它們幫助用戶在不需要同步整個區塊鏈的情況下訪問並與區塊鏈互動。
 
`recv_internal()` 符合我們的條件。

在 `code.fc` 檔案中，我們撰寫：


     () recv_internal(slice in_msg_body) impure {
     ;; 在這裡撰寫程式碼
     }

  > FunC 請用 `;;` 來進行註解

我們將 in_msg_body 切片傳遞給函數並使用 impure 關鍵字。
`impure` 是一個關鍵字，它表示函數會改變智慧合約資料。

例如，如果函數可以修改合約資料儲存、發送訊息或在某些資料無效時引發異常並且函數旨在驗證該資料，則必須指定 `impure` 修飾符。

重要的是：如果沒有指定 `impure` 且不使用函數呼叫的結果，則 FunC 編譯器可能會刪除該函數呼叫。

但為了了解切片是什麼，讓我們先談談 TON 網路的智慧合約中的資料類型。

##### 在 FunC 中的 cell、slice、builder、integer 類型

在我們的簡單智慧合約中，我們僅使用了四種類型：
- Cell - 由 1023 位數據和最多 4 個對其他單元的引用組成的 TVM 單元
- Slice - 用於從 TVM 單元中解析數據的 TVM 單元的部分表示形式
- Builder - 包含最多 1023 位數據和最多四個引用的部分構建的 TVM 單元；可用於創建新的單元
- Integer - 帶符號的 257 位整數

有關 FunC 中的類型的更多信息：
- [簡單的介紹](https://ton-blockchain.github.io/docs/#/smart-contracts/) 
- [詳細描述在此第 2.1 節中](https://ton-blockchain.github.io/docs/fiftbase.pdf)

簡單來說，Cell 是密封的單元，Slice 是當單元可以被讀取時，Builder 是當您組裝單元時。
## 將結果片段轉換為整數

為了將結果片段轉換為整數，添加以下代碼：`int n = in_msg_body~load_uint(32);`

現在，`recv_internal()` 函數如下所示：

     () recv_internal(slice in_msg_body) impure {
		int n = in_msg_body~load_uint(32);
     }

`load_uint` 函數來自於 [FunC standard library](https://ton-blockchain.github.io/docs/#/func/stdlib) 它從片段中加載無符號的 n 位整數。
## 持久性智慧合約數據

要將變量添加到 `total` 中並將值儲存在智慧合約中，讓我們看看 TON 中實現持久性數據/儲存功能的方式。

> 注意：請不要混淆 TON 儲存，上面的儲存是一個便於理解的類比。

TVM 虛擬機是基於堆棧（stack-based）的，因此使用特定的寄存器來儲存合約中的數據，而不是將數據 “堆疊” 在堆棧上是一種良好的實踐。

為了儲存永久數據，指定寄存器 c4，數據類型為 Cell。

有關寄存器的更多詳細信息可以在此處找到 [c4](https://ton-blockchain.github.io/docs/tvm.pdf) 在第 1.3 段中。

##### 從 c4 中取數據

為了「獲取」 c4 中的數據，我們需要使用 FunC 標準庫中的兩個函數。

具體來說：
`get_data` - 從 c4 寄存器獲取 Cell。
`begin_parse` - 將 Cell 轉換為片段。

將該值傳遞給片段 ds。

`slice ds = get_data().begin_parse();`

同樣，我們還將把此片段轉換為 64 位整數，以便根據我們的任務進行總和。 （使用我們已經熟悉的 `load_uint` 函数）

`int total = ds~load_uint(64);`

現在，我們的函數如下所示：

    () recv_internal(slice in_msg_body) impure {
		int n = in_msg_body~load_uint(32);

		slice ds = get_data().begin_parse();
		int total = ds~load_uint(64);
    }

##### 求和

為了完成任務，我們將使用二進制求和運算符 `+` 和賦值運算符 `=`

    () recv_internal(slice in_msg_body) impure {
		int n = in_msg_body~load_uint(32);

		slice ds = get_data().begin_parse();
		int total = ds~load_uint(64);

		total += n;
    }

##### 保存值

為了保持常數值，我們需要進行四個操作：

- 創建 Builder 以用於未來 Cell
- 將值寫入其中
- 從 Builder 創建 Cell
- 將結果 Cell 寫入寄存器

我們將再次使用 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 的功能來實現這一點

`set_data(begin_cell().store_uint(total, 64).end_cell());`

`begin_cell()` - 創建用於未來 Cell 的Builder
`store_uint()` - 寫入 total 的值
`end_cell()`- 創建 Cell
`set_data()` - 將 Cell 寫入寄存器 c4

總結：

    () recv_internal(slice in_msg_body) impure {
		int n = in_msg_body~load_uint(32);

		slice ds = get_data().begin_parse();
		int total = ds~load_uint(64);

		total += n;

		set_data(begin_cell().store_uint(total, 64).end_cell());
    }
	
## 報錯

在我們的 internal function 中剩下的就是如果接收到的變數不是 32 位，則添加異常調用的操作。

我們將使用[内置](https://ton-blockchain.github.io/docs/#/func/builtins)的報錯。

異常可以由有條件的原始數據 `throw_if` 和 `throw_unless` 和無條件的 `throw` 調用來拋出。

讓我們使用 `throw_if` 並傳遞任何錯誤代碼。為了取位，我們使用 `slice_bits()`。

`throw_if(35,in_msg_body.slice_bits() < 32);`

順便提一下，在 TON TVM 虛擬機中，有標準的異常代碼，我們將在測試中真正需要它們。你可以在[這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_exit_codes)查看。

在函數開頭插入：

    () recv_internal(slice in_msg_body) impure {
		throw_if(35,in_msg_body.slice_bits() < 32);

		int n = in_msg_body~load_uint(32);

		slice ds = get_data().begin_parse();
		int total = ds~load_uint(64);

		total += n;

		set_data(begin_cell().store_uint(total, 64).end_cell());
    }
	
## 撰寫一個 Get 函數

在 FunC 中，任何函數都符合以下模式：

`[<forall declarator>] <return_type><function_name(<comma_separated_function_args>) <specifiers>`

現在讓我們撰寫一個 get_total () 函數，它返回一個整數，並具有 method_id 規範（稍後會講到）：
 
    int get_total() method_id {
  	;; 在此處編寫代碼
	}

##### Method_id

method_id 規範允許您從 lite-client 或 ton-explorer 按名稱調用 GET 函數。

大致來說，該卷中的所有函數都有一個數字識別符，get 方法的編號是它們名稱的 crc16 雜湊。

#### 從 c4 獲取數據

為了使函數返回儲存在合約中的總數，我們需要從註冊表中取出數據，我們已經完成了這一步：

    int get_total() method_id {
  		slice ds = get_data().begin_parse();
 	 	int total = ds~load_uint(64);
		
  		return total;
	}
	
## 我們智慧合約的所有代碼

    () recv_internal(slice in_msg_body) impure {
		throw_if(35,in_msg_body.slice_bits() < 32);

		int n = in_msg_body~load_uint(32);

		slice ds = get_data().begin_parse();
		int total = ds~load_uint(64);

		total += n;

		set_data(begin_cell().store_uint(total, 64).end_cell());
    }
	 
    int get_total() method_id {
  		slice ds = get_data().begin_parse();
 	 	int total = ds~load_uint(64);
		
  		return total;
	}
	
## 部署合約到測試網

為了將合約部署到測試網，我們將使用命令行接口 [toncli](https://github.com/disintar/toncli/)
。

`toncli deploy -n testnet`

##### 如果說沒有足夠的 TON，該怎麼辦？

您需要從測試水龍頭獲取它們，可以使用 [@testgiver_ton_bot](https://t.me/testgiver_ton_bot) 機器人進行操作。

您可以在控制台中直接看到錢包地址，在 deploy 命令後，toncli 將在 INFO 行的右側顯示它：Found existing deploy-wallet。

要檢查 TON 是否到達您在測試網上的錢包，可以使用此瀏覽器：https://testnet.tonscan.org/

> 重要提示：這只是一個測試網

## 測試合約

##### 呼叫 recv_internal()

要呼叫 recv_internal() 函數，我們需要在 TON 網路中發送一條消息。

使用 [toncli send](https://github.com/disintar/toncli/blob/master/docs/advanced/send_fift_internal.md) 工具，讓我們編寫一個小的 Fift 腳本，向我們的合約發送一條 32 位元的消息。

##### 消息腳本

首先在 fift 資料夾中創建一個 `try.fif` 檔案，並編寫以下代碼：
 
    "Asm.fif" include
	
	<b
		11 32 u, // number
	b>
	

`"Asm.fif" include` - 需要將消息編譯為位元組碼

現在考慮一下這個消息：

`<b b>` - 創建 Builder cells，更多細節在 [5.2](https://ton-blockchain.github.io/docs/fiftbase.pdf) 段落中有介紹

`10 32 u` - 放入 32 位元無符號整數 10

` // number` - 單行註解

##### 發佈結果消息

在命令行中輸入：

`toncli send -n testnet -a 0.03 --address "address of your contract" --body ./fift/try.fif`

現在我們來測試 GET 函數：

`toncli get get_total`

你應該得到以下結果：

![toncli get send](./img/tonclisendget.png)

## 恭喜你完成了本教程！

##### 練習

正如你可能已經注意到的，我們並沒有測試異常情況。
修改消息，讓智能合約拋出異常。
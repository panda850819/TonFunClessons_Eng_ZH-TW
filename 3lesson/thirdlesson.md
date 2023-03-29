# Lesson 3 Proxy smart contract
## 介紹

在這個課程中，我們將使用 FunC 語言在測試網路 The Open Network 中撰寫一個代理（將所有訊息發送給其擁有者）智慧合約，使用 [toncli](https://github.com/disintar/toncli) 部署到測試網路中，並在下一課中進行測試。.

## 要求

完成此課程，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 的 command line 的接口。

還需要能夠使用 toncli 創建/部署項目，您可以在[第一課](../1lesson/firstlesson.md)中學習如何做到這一點。

## 智慧合約

我們將創建的智慧合約應該具有以下功能：

- 將進入合約的所有訊息轉發給所有者；
- 轉發時，發件人的地址必須排在訊息之前；
- 附加到訊息的 Toncoin 值必須等於傳入訊息減去處理費用（計算和訊息轉發費用）的值；
- 所有者的地址儲存在智慧合約儲存中；
- 從所有者向合約發送訊息時，不應進行轉發。
- ** 我決定從 [FunC 任務](https://github.com/ton-blockchain/func-contest1)中獲取智慧合約的想法，因為它們非常適合熟悉 TON 的智慧合約開發。
## 外部方法

為了讓我們的代理接收訊息，我們將使用外部方法 `recv_internal()`

    () recv_internal() {

    }

##### 外部方法的參數
一個邏輯上的問題出現了 - 如何理解函數應該有哪些參數才能在 TON 網路上接收訊息？

根據 [TON 虛擬機 - TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的文件，當 TON 鏈中的一個帳戶發生事件時，它會觸發一個交易。

每個交易由最多 5 個階段（stage）組成。在[這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=transactions-and-phases)讀取更多 here。

我們感興趣的是**計算階段**。更具體地說，初始化時堆棧上有什麼。對於正常的訊息觸發交易，堆棧的初始狀態如下：

5 個元素：
- 智慧合約餘額（以 nanoTons 為單位）
- 進入消息餘額（以 nanotones 為單位）
- 有關進入消息的 cell
- 進入消息正文，切片類型
- 函數選擇器（對於 recv_internal 函數，其值為 0）

因此，我們得到以下代碼：

    () recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body) {

    }
	
## 發送者地址

根據任務，我們需要獲取發送者的地址。我們將從包含進入消息 `in_msg_full` 的 cell 中取出地址。讓我們將程式碼移動到單獨的函數中。

	() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	  slice sender_address = parse_sender_address(in_msg_full);
	}
	
##### 編寫一個函數

讓我們編寫解析發送者地址的代碼，從消息 cell 中獲取發送者地址並解析它：

	slice parse_sender_address (cell in_msg_full) inline {
	  var cs = in_msg_full.begin_parse();
	  var flags = cs~load_uint(4);
	  slice sender_address = cs~load_msg_addr();
	  return sender_address;
	}

正如您所看到的，該函數具有 `inline` 限定符，其代碼實際上被替換在每次調用函數的地方。

為了獲取地址，我們需要使用 `begin_parse` 函數將 cell 轉換為切片：

	var cs = in_msg_full.begin_parse();

現在我們需要「減去」地址所對應的切片。使用來自 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 的 `load_uint` 函數從切片中加載一個 n 位無符號整數，「減去」標誌（flags）：

	var flags = cs~load_uint(4);

在這節課中，我們不會詳細討論標誌，但您可以在 [3.1.7](https://ton-blockchain.github.io/docs/tblkch.pdf) 中閱讀更多相關內容。

最後，獲取地址。使用 `load_msg_addr()` 函數 - 從切片中加載唯一的前綴，該前綴是有效的 MsgAddress。

	slice sender_address = cs~load_msg_addr();
	return sender_address;

## 收件人地址

我們將從我們之前講過的 [c4](https://ton-blockchain.github.io/docs/tvm.pdf) 中獲取地址。

我們將使用以下函數：
`get_data` - 從 c4 寄存器中獲取 cell。
`begin_parse` - 將 cell 轉換為切片。
`load_msg_addr()` - 從切片中加載唯一的前綴，該前綴是有效的 MsgAddress。

因此，我們得到以下函數：

	slice load_data () inline {
	  var ds = get_data().begin_parse();
	  return ds~load_msg_addr();
	}

只需調用它：

	slice load_data () inline {
	  var ds = get_data().begin_parse();
	  return ds~load_msg_addr();
	}

	slice parse_sender_address (cell in_msg_full) inline {
	  var cs = in_msg_full.begin_parse();
	  var flags = cs~load_uint(4);
	  slice sender_address = cs~load_msg_addr();
	  return sender_address;
	}

	() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	  slice sender_address = parse_sender_address(in_msg_full);
	  slice owner_address = load_data();
	}
	
## 比較地址相等條件

根據任務，如果合約所有者訪問代理智慧合約，則代理不應轉發消息。因此，需要比較兩個地址是否相等。

##### 比較函數

FunC 支援組合語言（即 Fift）定義函數。這可以通過將函數定義為低級 TVM 原語來實現。例如，比較函數的定義如下：

	int equal_slices (slice a, slice b) asm "SDEQ";

您可以看到使用了 `asm` 關鍵字。您可以在 [TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的第 77 頁找到可用的原語列表。

##### 一元運算符

因此，我們將在 `if` 中使用 `equal_slices` 函數：

	() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	  slice sender_address = parse_sender_address(in_msg_full);
	  slice owner_address = load_data();

	  if  equal_slices(sender_address, owner_address) {

	   }
	}
	
但是該函數將檢查確切的相等性，如何檢查不等性呢？這裡可以使用位求反的一元運算符 `~`。現在我們的代碼如下：

	int equal_slices (slice a, slice b) asm "SDEQ";

	slice load_data () inline {
	  var ds = get_data().begin_parse();
	  return ds~load_msg_addr();
	}

	slice parse_sender_address (cell in_msg_full) inline {
	  var cs = in_msg_full.begin_parse();
	  var flags = cs~load_uint(4);
	  slice sender_address = cs~load_msg_addr();
	  return sender_address;
	}

	() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	  slice sender_address = parse_sender_address(in_msg_full);
	  slice owner_address = load_data();

	  if ~ equal_slices(sender_address, owner_address) {

	   }
	}
	
現在，我們需要填充條件操作符的代碼，以便按照任務要求發送傳入的消息。

## 發送訊息
因此，我們仍然需要根據任務填充條件運算符的主體，即發送傳入消息。

##### 消息結構
完整的訊息結構可以在這裡 - [訊息佈局](https://ton-blockchain.github.io/docs/#/smart-contracts/messages?id=message-layout)中找到。但通常我們不需要控制每個字段，因此可以使用來自[範例](https://ton-blockchain.github.io/docs/#/smart-contracts/messages?id=sending-messages)的簡短形式:

	 var msg = begin_cell()
		.store_uint(0x18, 6)
		.store_slice(addr)
		.store_coins(amount)
		.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
		.store_slice(message_body)
	  .end_cell();

如您所見，使用了來自 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib)的函數來構建消息。即建構器原始碼（部分構建的單元格，正如您可能記得第一課所述）。考慮：

 `begin_cell()` - 將為將來的 cell 創建一個建構器
 `end_cell()` - 將創建一個 cell
 `store_uint` - 在建構器中儲存無符號的整數
 `store_slice` - 在建構器中儲存切片
 `store_coins` -  此處的文檔意思為 store_grams - 用於儲存TonCoins。更多細節在[此處](https://ton-blockchain.github.io/docs/#/func/stdlib?id=store_grams)。
  
另外還要考慮到 `store_ref`，需要用來發送地址。  
 
 `store_ref` - 在建構器中儲存 cell 引用，現在我們已經擁有了所有必要的訊息，讓我們組裝消息。
 
 ##### 最後的收件訊息內容

為了將來自 `recv_internal` 的消息正文發送到訊息中，我們將收集單元格，在訊息本身中使用 `store_ref` 建立連結。

	  if ~ equal_slices(sender_address, owner_address) {
		cell msg_body_cell = begin_cell().store_slice(in_msg_body).end_cell();
	   }

##### 收集訊息

根據問題的條件，我們必須發送地址和訊息的正文。因此，我們將 `.store_slice(message_body)` 更改為 `.store_slice(sender_address)` 和 `.store_ref(msg_body_cell)`。

我們得到：

	  if ~ equal_slices(sender_address, owner_address) {
		cell msg_body_cell = begin_cell().store_slice(in_msg_body).end_cell();

		var msg = begin_cell()
			  .store_uint(0x10, 6)
			  .store_slice(owner_address)
			  .store_grams(0)
			  .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
			  .store_slice(sender_address)
			  .store_ref(msg_body_cell)
			  .end_cell();
	   }

它仍然只是發送我們的訊息。

##### 傳送訊息的模式

使用[標準函式庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message)的 `send_raw_message` 函式傳送訊息。

我們已經收集了 msg 變數，現在只需弄清楚 `mode` 即可。每個模式都在[文檔](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message)中有描述。
為了更清楚地說明，讓我們看一個例子。

假設智慧合約的餘額為 100 個 Toncoin，我們收到了一個 60 個 Toncoin 的內部訊息，並發送了一個 10 個 Toncoin 的訊息，總費用為 3 個 Toncoin。

 `mode = 0` - 餘額（100+60-10 = 150 個 Toncoin），傳送（10-3 = 7 個Toncoin）
 `mode = 1` - 餘額（100+60-10-3 = 147 個 Toncoin），傳送（10 個 Toncoin）
 `mode = 64` - 餘額（100-10 = 90 個 Toncoin），傳送（60+10-3 = 67 個 Toncoin）
 `mode = 65` - 餘額（100-10-3 = 87 個 Toncoin），傳送（60+10 = 70 個 Toncoin）
 `mode = 128` - 餘額（0 個 Toncoin），傳送（100+60-3 = 157 個 Toncoin）
 
 上面描述的 Modes 1 和 65 是 mode' = mode + 1。
 
 根據任務的要求，附加到訊息的 Toncoin 的值必須等於傳入訊息的值減去處理費用。
 `mode = 64` 並且 `.store_grams(0)` 就足夠了。
 
此範例將產生以下結果：

假設智慧合約的餘額為 100 個 Toncoin，我們收到了一個 60 個 Toncoin 的內部訊息，並發送了一個 0 個 Toncoin 的訊息（因為 `.store_grams(0)`），總費用為 3 個 Toncoin。

`mode = 64` - 餘額（100 = 100 個 Toncoin），傳送（60-3 = 57 個 Toncoin）

 因此，我們的條件語句如下：
 
	   if ~ equal_slices(sender_address, owner_address) {
		cell msg_body_cell = begin_cell().store_slice(in_msg_body).end_cell();

		var msg = begin_cell()
			  .store_uint(0x10, 6)
			  .store_slice(owner_address)
			  .store_grams(0)
			  .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
			  .store_slice(sender_address)
			  .store_ref(msg_body_cell)
			  .end_cell();
		 send_raw_message(msg, 64);
	   }

完整的智慧合約程式碼：

	int equal_slices (slice a, slice b) asm "SDEQ";

	slice load_data () inline {
	  var ds = get_data().begin_parse();
	  return ds~load_msg_addr();
	}

	slice parse_sender_address (cell in_msg_full) inline {
	  var cs = in_msg_full.begin_parse();
	  var flags = cs~load_uint(4);
	  slice sender_address = cs~load_msg_addr();
	  return sender_address;
	}

	() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	  slice sender_address = parse_sender_address(in_msg_full);
	  slice owner_address = load_data();

	  if ~ equal_slices(sender_address, owner_address) {
		cell msg_body_cell = begin_cell().store_slice(in_msg_body).end_cell();

		var msg = begin_cell()
			  .store_uint(0x10, 6)
			  .store_slice(owner_address)
			  .store_grams(0)
			  .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
			  .store_slice(sender_address)
			  .store_ref(msg_body_cell)
			  .end_cell();
		 send_raw_message(msg, 64);
	   }
	}
	
## 結論

由於訊息和我們的代理函數都是 `internal` 的，所以無法通過 `toncli` 「讀取」合約 - 它只在 TON 內部處理訊息。
那麼如何正確開發此類合約 - 答案來自於測試驅動開發。
我們將在下一課中撰寫[測試](https://en.wikipedia.org/wiki/Test-driven_development)。
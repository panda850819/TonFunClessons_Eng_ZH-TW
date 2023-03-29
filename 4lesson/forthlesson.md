# Lesson 4 測試訊息

## 介紹

在這個教學中，我們將為在 FUNC 語言中在 The Open Network testnet 創建的智能合約編寫測試，並使用 [toncli](https://github.com/disintar/toncli) 執行這些測試。

## 需求

完成這個教學，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 終端機介面並完成[第三課](../3lesson/thirdlesson.md) 。

## 重要提示

下面所述是舊版本的測試。新的 toncli 測試目前可用於 func/fift 的 dev 版本，使用說明在[此](https://github.com/disintar/toncli/blob/master/docs/advanced/func_tests_new.md)，新測試的教學在[此](../11lesson/11lesson)。新測試的發布並不意味著舊測試的教學毫無意義-它們很好地傳達了邏輯，因此有助於通過這些課程。另外請注意，當使用 `toncli run_tests` 時，可以使用 `--old` 標誌來使用舊的測試。

## 智能合約代理的測試

對於我們的代理智能合約，我們將撰寫以下測試：

- test_same_addr () 測試當從所有者向合約發送訊息時，不應進行轉發。
- test_example_data() 測試[第三課](../3lesson/thirdlesson.md)中的其餘條件。

## 在 toncli 下的 FunC 測試結構

我要提醒您，在 toncli 下的每個 FunC 測試中，您需要編寫兩個函數。第一個函數將確定要傳遞給第二個函數進行測試的資料（在 TON 中，更正確地說是狀態，但我希望資料是更容易理解的類比）。

每個測試函數都必須指定一個方法 ID。測試函數的方法 ID 應該從 0 開始。
##### 資料函數

資料函數不接受任何參數，但必須返回：
- 函數選擇器（function selector） - 被測試合約中所調用的函數的 ID;
- tuple - 我們將傳遞給執行測試的函數的（stack）值;
- c4 cell - 在控制寄存器 c4 中的「永久資料」;
- c7 tuple - 在控制寄存器 c7 中的「臨時資料」;
- gas limit integer - gas 限制（為了理解 gas 的概念，建議您首先閱讀[Ethereum](https://ethereum.org/en/developers/docs/gas/)）;

> 簡單來說，gas 衡量了在網路上執行某些操作所需的算力。您可以在[這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/fees)中詳細閱讀。在[附錄 A](https://ton-blockchain.github.io/docs/tvm.pdf) 中也有詳細解釋。

> Stack—— 根據 LIFO 原則（英語中的後進先出，“last in - first out”）組織的元素列表。有關堆棧，請參閱 [wikipedia](https://ru.wikipedia.org/wiki/%D0%A1%D1%82%D0%B5%D0%BA)。

更多有關寄存器 c4 和 c7 的資訊，請參閱[這裡](https://ton-blockchain.github.io/docs/tvm.pdf)的 1.3.1。
##### 測試函數

測試函數必須傳遞以下參數：

- 退出碼（exit code） - 虛擬機的 Return codes，以便我們能夠理解錯誤或否；
- c4 cell - 在控制寄存器 c4 中的「永久資料」
- tuple - 從資料函數傳遞的（stack）值；
- c5 cell - 檢查傳出的訊息
- gas - 使用的 gas

[TVM return codes](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_exit_codes)
## 讓我們開始撰寫測試

在這個教學中的測試中，我們需要一個比較輔助函數。讓我們使用 `asm` 關鍵字將其定義為低級原始函數：

`int equal_slices (slice a, slice b) asm "SDEQ";`


## 測試合約代理呼叫

讓我們寫第一個測試 `test_example_data()` 並分析它的程式碼

##### 資料函數

讓我們從資料函數開始：

	[int, tuple, cell, tuple, int] test_example_data() method_id(0) {
		int function_selector = 0;

		cell my_address = begin_cell()
					.store_uint(1, 2)
					.store_uint(5, 9) 
					.store_uint(7, 5)
					.end_cell();

		cell their_address = begin_cell()
					.store_uint(1, 2)
					.store_uint(5, 9) 
					.store_uint(8, 5) 
					.end_cell();

		slice message_body = begin_cell().store_uint(12345, 32).end_cell().begin_parse();

		cell message = begin_cell()
				.store_uint(0x6, 4)
				.store_slice(their_address.begin_parse()) 
				.store_slice(their_address.begin_parse()) 
				.store_grams(100)
				.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
				.store_slice(message_body)
				.end_cell();

		tuple stack = unsafe_tuple([12345, 100, message, message_body]);

		return [function_selector, stack, my_address, get_c7(), null()];
	}

## 分析

`int function_selector = 0;`

由於我們呼叫了 `recv_internal()`，所以我們將值賦為 0，為什麼是 0？
Fift（即，我們編譯 FunC 程式的地方）有預定義的標識符號，即：
- `main` 和 `recv_internal` 的 id = 0
- `recv_external` 的 id = -1
- `run_ticktock` 的 id = -2

為了檢查發送，我們需要從中發送訊息的地址，此範例中，我們有自己的地址 `my_address` 和他們的地址 `their_address`。

問題是地址應該看起來像什麼，考慮到需要將其分配給 FunC 類型。讓我們轉向 [TL-B schema](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb) 模式，具體來說是第 100 行中的地址描述。

	cell my_address = begin_cell()
				.store_uint(1, 2)
				.store_uint(5, 9) 
				.store_uint(7, 5)
				.end_cell();

`.store_uint(1, 2)` - 0x01 外部的地址;

`.store_uint(5, 9)` - 長度等於 5;

`.store_uint(7, 5)` - 讓我們的地址為 7;

為了具體理解 `TL-B` 和這篇文章，我建議你學習 https://core.telegram.org/mtproto/TL 。

我們也會多收集一個地址，假設是 8 個。

	cell their_address = begin_cell()
				.store_uint(1, 2)
				.store_uint(5, 9) 
				.store_uint(8, 5) 
				.end_cell();

將訊息組裝起來，還需要組裝訊息主體的切片，將數字 12345 放在其中。

訊息主體的切片：slice message_body = begin_cell().store_uint(12345, 32).end_cell().begin_parse();

現在只需要收集訊息本身：

	cell message = begin_cell()
			.store_uint(0x6, 4)
			.store_slice(their_address.begin_parse()) 
			.store_slice(their_address.begin_parse()) 
			.store_grams(100)
			.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
			.store_slice(message_body)
			.end_cell();

需要注意的是，我們在 cells 中收集了地址，所以要使用 `store_slice()` 將它們儲存在訊息中，需要使用 `begin_parse()` 將 cell 轉換為 slice。

正如您可能已經注意到的那樣，發件人和收件人都是相同的地址，這是為了簡化測試並避免產生大量地址，因為按照條件，當僅從所有者發送訊息給合約時，不應進行轉發。

現在讓我們回憶一下函數應該返回什麼：

- 函數選擇器：被測試合約中調用的函數的 ID；
- 元組：我們將傳遞給執行測試的函數的 (堆) 值；
- c4 cell：控制寄存器 c4 中的「永久資料」；
- c7 tuple：控制寄存器 c7 中的「臨時資料」；
- gas 限制整數

正如您可能已經注意到的那樣，我們只需要收集元組並返回資料。根據我們的合約中 `recv_internal ()` 函數的簽名（函數聲明的語法結構），我們將以下值放在其中：

	tuple stack = unsafe_tuple([12345, 100, message, message_body]);

需要注意的是，我們將返回 `my_address`，這是必要的，以檢查地址匹配的條件。

	return [function_selector, stack, my_address, get_c7(), null()];

正如您所看到的，在 c7 中，我們使用 `get_c7()` 將 c7 的當前狀態放置，並在 gas 限制整數中放置 `null()`。

##### 測試函數

程式碼：

	_ test_example(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(1) {
		throw_if(100, exit_code != 0);

		slice actions = actions.begin_parse();
		throw_if(101, actions~load_uint(32) != 0x0ec3c86d); 


		throw_if(102, ~ slice_empty?(actions~load_ref().begin_parse())); 

		slice msg = actions~load_ref().begin_parse();
		throw_if(103, msg~load_uint(6) != 0x10);

		slice send_to_address = msg~load_msg_addr();
		slice expected_my_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(7, 5).end_cell().begin_parse();

		throw_if(104, ~ equal_slices(expected_my_address, send_to_address));
		throw_if(105, msg~load_grams() != 0);
		throw_if(106, msg~load_uint(1 + 4 + 4 + 64 + 32 + 1 + 1) != 0);

		slice sender_address = msg~load_msg_addr();
		slice expected_sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(8, 5).end_cell().begin_parse();
		throw_if(107, ~ equal_slices(sender_address, expected_sender_address));

		slice fwd_msg = msg~load_ref().begin_parse();

		throw_if(108, fwd_msg~load_uint(32) != 12345);
		fwd_msg.end_parse();

		msg.end_parse();
	}

## 分析

`throw_if(100, exit_code != 0);`

檢查返回碼，如果返回碼為非零，函數將拋出異常。
0 - 成功執行智能合約的標準返回碼。

slice actions = actions.begin_parse();
throw_if(101, actions~load_uint(32) != 0x0ec3c86d);

出站訊息寫入 c5 寄存器，因此我們從那裡卸載一個 32 位值（ `load_uint` 是標準 FunC 庫中的函數，它從 slice 中加載一個無符號 n 位整數。）並在其不等於 0x0ec3c86d 時報錯，即它未發送訊息。數字 0x0ec3c86d 可從 [TL-B 模式行 371 中](https://github.com/ton-blockchain/ton/blob/d01bcee5d429237340c7a72c4b0ad55ada01fcc3/crypto/block/block.tlb)獲取，為了確保 `send_raw_message` 使用 `action_send_msg`，讓我們查看標準庫的 [764 行](https://github.com/ton-blockchain/ton/blob/24dc184a2ea67f9c47042b4104bbb4d82289fac1/crypto/vm/tonops.cpp)。

![github tone](./img/send_action.PNG)

在繼續之前，我們需要理解資料如何儲存在 c5 中，從[文件](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=result-of-tvm-execution)中了解到，c5 分別儲存了最後一個動作的兩個 cell 引用和前一個動作的 cell 引用。

有關從動作中完全獲取資料的更多詳細訊息，將在以下程式碼中描述。現在主要的是，我們將解除 c5 中的第一個鏈接，並立即檢查它是否為空，以便稍後取出帶有訊息的單元格。

	throw_if(102, ~ slice_empty?(actions~load_ref().begin_parse()));

我們使用[FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=slice_empty)中的 `slice_empty?` 進行檢查。

我們需要從 "actions" cell 中取出訊息的 slice，使用 `load_ref()` 取出具有訊息的 cell 的引用，並使用 `begin_parse()` 將其轉換為 slice。

	slice msg = actions~load_ref().begin_parse();

讓我們繼續

	slice send_to_address = msg~load_msg_addr();
	slice expected_my_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(7, 5).end_cell().begin_parse();
	throw_if(104, ~ equal_slices(expected_my_address, send_to_address));

讓我們開始閱讀這個訊息。透過 `load_msg_addr()` 從訊息中載入地址並檢查收件人地址，並將載入的地址放入 `expected_my_address` 中，接著使用之前聲明的 `equal_slices()` 檢查它們是否相同。由於該函數用於檢查相等，我們使用位元反轉的一元運算子 ~ 進行不相等的檢查。位元運算在[維基百科](https://en.wikipedia.org/wiki/Bitwise_operation)中有詳細的說明。

    throw_if(105, msg~load_grams() != 0);
    throw_if(106, msg~load_uint(1 + 4 + 4 + 64 + 32 + 1 + 1) != 0);

使用[標準函式庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=load_grams)中的 `load_grams()` 和 `load_uint()` 檢查訊息中的 TON 數量是否不等於 0，以及其他可以在[訊息模式](https://ton-blockchain.github.io/docs/#/smart-contracts/messages)中檢視的服務字段，將它們從訊息中讀取。

	slice sender_address = msg~load_msg_addr();
	slice expected_sender_address = begin_cell().store_uint(1, 2).store_uint(5,9).store_uint(8, 5).end_cell().begin_parse();
	throw_if(107, ~ equal_slices(sender_address, expected_sender_address));

繼續閱讀訊息，我們檢查發件人的地址，就像我們之前檢查收件人地址一樣。

	slice fwd_msg = msg~load_ref().begin_parse();
	throw_if(108, fwd_msg~load_uint(32) != 12345);

最後，我們檢查訊息中的值。首先，讓我們使用 `load_ref()` 從訊息中載入 cell 引用，並將其轉換為切片 `begin_parse()`，然後使用標準 FunC 函數 `load_uint()`（它可以從切片中載入一個 n 位元的無符號整數）讀取一個 32 位元的值，並將其與我們的值 12345 進行比較。

	fwd_msg.end_parse();

	msg.end_parse();

在閱讀完畢後，我們最後要檢查的是切片是否為空，即從我們取得的整個訊息和訊息正文中。值得注意的是，`end_parse()` 如果切片不為空則會拋出異常，這在測試中非常方便。

## 測試相同地址

根據[第三課](../3lesson/thirdlesson.md)的任務，當所有者向合約發送訊息時，不應該進行轉發，我們來進行測試。

## 資料函數
讓我們從資料函數開始：

	[int, tuple, cell, tuple, int] test_same_addr_data() method_id(2) {
		int function_selector = 0;

		cell my_address = begin_cell()
								.store_uint(1, 2) 
								.store_uint(5, 9)
								.store_uint(7, 5)
								.end_cell();

		slice message_body = begin_cell().store_uint(12345, 32).end_cell().begin_parse();

		cell message = begin_cell()
				.store_uint(0x6, 4)
				.store_slice(my_address.begin_parse()) 
				.store_slice(my_address.begin_parse())
				.store_grams(100)
				.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
				.store_slice(message_body)
				.end_cell();

		tuple stack = unsafe_tuple([12345, 100, message, message_body]);

		return [function_selector, stack, my_address, get_c7(), null()];
	}
	
## 分析

資料函數實際上與先前的資料函數沒有什麼不同，唯一的區別在於只有一個地址，因為我們正在測試從我們的地址向智能合約代理發送訊息時會發生什麼。其次，我們會發送給自己，以節省撰寫測試的時間。

##### 測試函數

程式碼：

	_ test_same_addr(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(3) {
		throw_if(100, exit_code != 0);

		throw_if(102, ~ slice_empty?(actions.begin_parse())); 

	}

我們再度檢查返回碼，如果返回碼不為零，該函數將引發異常。

`throw_if(100, exit_code != 0);`

0 - 成功執行智能合約的標準返回碼。

`throw_if(102, ~ slice_empty?(actions.begin_parse()));`


由於代理合約不應該發送訊息，因此我們只需使用 `slice_empty？` 檢查片段是否為空。有關該函數的更多訊息在[此處](https://ton-blockchain.github.io/docs/#/func/stdlib?id=slice_empty)。
## 練習

正如您所看到的，我們尚未測試使用 `send_raw_message` 發送訊息的模式

##### 提示

這是一個「訊息解析」函數的範例：

	(int, cell) extract_single_message(cell actions) impure inline method_id {
		;; ---------------- Parse actions list
		;; prev:^(OutList n)
		;; #0ec3c86d
		;; mode:(## 8)
		;; out_msg:^(MessageRelaxed Any)
		;; = OutList (n + 1);
		slice cs = actions.begin_parse();
		throw_unless(1010, cs.slice_refs() == 2);
		
		cell prev_actions = cs~load_ref();
		throw_unless(1011, prev_actions.cell_empty?());
		
		int action_type = cs~load_uint(32);
		throw_unless(1013, action_type == 0x0ec3c86d);
		
		int msg_mode = cs~load_uint(8);
		throw_unless(1015, msg_mode == 64); 
		
		cell msg = cs~load_ref();
		throw_unless(1017, cs.slice_empty?());
		
		return (msg_mode, msg);
	}
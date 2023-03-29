# Lesson 2 FunC 智能合約的測試
## 介紹

在本教程中，我們將使用 FunC 語言為第一課中在 The Open Network 測試網上創建的智能合約編寫測試，並使用 [toncli](https://github.com/disintar/toncli) 執行它們。

## 要求

要完成本課程，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面並完成[第一課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/1lesson/firstlesson.md)。

## 重要提示

以下所述是舊版本的測試。新的 toncli 測試，目前適用於 func/fift 的 dev 版本，指令在[這裡](https://github.com/disintar/toncli/blob/master/docs/advanced/func_tests_new.md)，新測試的課程在這裡。發布[新的測試](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/11lesson/11lesson.md)並不意味著舊測試的課程毫無意義-它們很好地傳達了邏輯，因此能夠成功通過該課程。此外，請注意，在使用 `toncli run_tests` 時，可以使用 `--old` 標誌使用舊測試。

## 第一個智能合約的測試
對於我們的第一個智能合約，我們將編寫以下測試：

- test_example - 使用數字 10 調用 recv_internal ()
- test_get_total - 測試 get 方法
- test_exception - 檢查添加不符合位條件的數字

## 在 toncli 下的 FunC 測試結構

對於 toncli 下的每個 FunC 測試，我們將編寫兩個函數。第一個函數將確定數據（從 TON 的角度來說，更正確的說法是狀態，但我希望數據是一個更易懂的比喻），我們將把它傳送到第二個函數進行測試。

每個測試函數必須指定一個 method_id。方法 ID 測試函數應該從 0 開始。

##### 創建測試文件

讓我們在上一節的程式碼中，我們將在 *tests* 文件夾中創建 *example.func* 文件來撰寫我們的測試。

##### Data 函數

Data 函數不接受任何參數，但必須返回：
- 函數選擇器（function selector） - 被測試合約中所調用的函數的 ID;
- tuple - 我們將傳遞給執行測試的函數的（stack）值;
- c4 cell - 在控制寄存器 c4 中的「永久數據」;
- c7 tuple - 在控制寄存器 c7 中的「臨時數據」;
- gas limit integer - gas 限制（為了理解 gas 的概念，建議您首先閱讀[Ethereum](https://ethereum.org/en/developers/docs/gas/)）;

> 簡單來說，gas 衡量了在網路上執行某些操作所需的算力。您可以在[這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/fees)中詳細閱讀。在[附錄 A](https://ton-blockchain.github.io/docs/tvm.pdf) 中也有詳細解釋。

> Stack—— 根據 LIFO 原則（英語中的後進先出，“last in - first out”）組織的元素列表。有關堆棧，請參閱 [wikipedia](https://ru.wikipedia.org/wiki/%D0%A1%D1%82%D0%B5%D0%BA)。

更多有關寄存器 c4 和 c7 的資訊，請參閱[這裡](https://ton-blockchain.github.io/docs/tvm.pdf)的 1.3.1。

##### 測試函數
測試函數必須傳遞以下參數：

- 退出碼（exit code） - 虛擬機的 Return codes，以便我們能夠理解錯誤或否；
- c4 cell - 在控制寄存器 c4 中的「永久數據」
- tuple - 從資料函數傳遞的（stack）值；
- c5 cell - 檢查傳出的訊息
- gas - 使用的 gas

[TVM return codes](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_exit_codes)

## 測試 recv_internal () 調用

讓我們撰寫第一個測試 test_example 並分析它的代碼。

##### Data 函數

讓我們從 data 函數開始：

    [int, tuple, cell, tuple, int] test_example_data() method_id(0) {
		int function_selector = 0;

		cell message = begin_cell()     
				.store_uint(10, 32)          
				.end_cell();

		tuple stack = unsafe_tuple([message.begin_parse()]); 

		cell data = begin_cell()             
			.store_uint(0, 64)              
			.end_cell();

		return [function_selector, stack, data, get_c7(), null()];
	}

## 分析

`int function_selector = 0;`

由於我們調用 `recv_internal()`，因此我們分配值為 0，為什麼是 0？Fift（換句話說，在其中我們編譯我們的 FunC 程式）具有預定義的標識符號，即：

- `main` 和 `recv_internal` 的 id = 0
- `recv_external` 的 id = -1
- `run_ticktock` 的 id = -2

		cell message = begin_cell()     
				.store_uint(10, 32)          
				.end_cell();

在訊息的 Cell 中，我們填入佔據 32 位元的無符號的整數 10。

`tuple stack = unsafe_tuple([message.begin_parse()]);`

`tuple` 是另一個 FunC 數據類型。
Tuple - 是堆棧值類型的任意值的有序集合。


使用 begin_parse() 將  *message *cell 轉換為 *slice* ，然後使用 `unsafe_tuple()` 函數將其寫入 *tuple*。

		cell data = begin_cell()             
			.store_uint(0, 64)              
			.end_cell();

在控制寄存器 c4 中放置 64 位元的 0。
只需要返回數據即可：

`return [function_selector, stack, data, get_c7(), null()];`

可以看到，在 c7 中使用 `get_c7()` 放置 c7 的當前狀態，在 gas 限制整數中使用 `null()`。

##### 測試函數

程式碼如下：

	_ test_example(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(1) {
		throw_if(100, exit_code != 0);

		var ds = data.begin_parse();

		throw_if(101, ds~load_uint(64) != 10); 
		throw_if(102, gas > 1000000); 
	}

## 分析

`throw_if(100, exit_code != 0);`

我們檢查返回程式碼，如果返回程式碼不等於零，則該函數將引發異常。
0 — 智能合約成功執行的返回程式碼。

		var ds = data.begin_parse();

		throw_if(101, ds~load_uint(64) != 10); 

我們檢查我們發送的數字是否等於 10，即我們發送了 10 個 32 位元的數字，執行智能合約後，10 個 64 位元的數字被寫入控制寄存器 c4 中。

換句話說，如果不是 10，則會引發異常。

`throw_if(102, gas > 1000000); `

儘管在我們在第一課解決的問題中沒有限制使用 gas，但在智能合約的測試中，重要的是不僅檢查執行邏輯，而且檢查邏輯是否會導致非常大的 gas 消耗，否則合約將無法在主網上運行。

## 測試 Get 函數調用

讓我們撰寫 test_get_total 測試，並分析其代碼。
##### Data 函數

讓我們從 data 函數開始：

	[int, tuple, cell, tuple, int] test_get_total_data() method_id(2) {
		int function_selector = 128253; 
		
		tuple stack = unsafe_tuple([]); 

		cell data = begin_cell()            
			.store_uint(10, 64)              
			.end_cell();

		return [function_selector, stack, data, get_c7(), null()];
	}
	
## 分析

`int function_selector = 128253; `

要了解 GET 函數的 ID，您需要轉到已編譯的智能合約並查看函數分配的 ID。讓我們進入 build 文件夾並打開 contract.fif，找到其中帶有 get_total 的行。

`128253 DECLMETHOD get_total`

對於 get_total 函數，我們不需要傳遞任何參數，因此我們只需聲明一個空的元組。

`tuple stack = unsafe_tuple([]); `

並在 c4 中寫入 10，以進行驗證。

	cell data = begin_cell()            
		.store_uint(10, 64)              
		.end_cell();
			
##### 測試函數

The code:

	_ test_get_total(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(3) {
		throw_if(103, exit_code != 0); 
		int counter = first(stack); 
		throw_if(104, counter != 10); 
	}
	
## 分析

`throw_if(103, exit_code != 0); `

Let's check the return code.

		int counter = first(stack); 
		throw_if(104, counter != 10); 
		
在我們的測試中，對於我們傳遞的值10，重要的是它在堆棧的“頂部”，因此我們使用標準庫 [stdlib.fc](https://ton-blockchain.github.io/docs/#/func/stdlib?id= first) 的第一個函數進行減法，它返回元組的第一個值。

## 測試異常

讓我們編寫 test_exception 測試並分析其代碼。
##### Data 函數

讓我們從 data 函數開始吧

	[int, tuple, cell, tuple, int] test_exception_data() method_id(4) {
		int function_selector = 0;

		cell message = begin_cell()     
				.store_uint(30, 31)           
				.end_cell();

		tuple stack = unsafe_tuple([message.begin_parse()]);

		cell data = begin_cell()            
			.store_uint(0, 64)               
			.end_cell();

		return [function_selector, stack, data, get_c7(), null()];
	}
	
## 分析

正如我們所見，與我們的第一個函數相比，差異極小，即我們放入元組的值為 30 並且佔據 31 位元。

		cell message = begin_cell()     
				.store_uint(30, 31)           
				.end_cell();
				
但在測試函數中，差異將更加顯著。
##### Test 函數

	_ test_exception(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(5) {
		throw_if(100, exit_code == 0);
	}
	
與其他測試函數不同，這裡如果成功執行智能合約，我們期望引發異常。

## 執行測試

為了讓 toncli「理解」測試的位置，您需要在 `project.yaml` 中添加訊息。

	contract:
	  data: fift/data.fif
	  func:
		- func/code.func
	  tests:
		- tests/example.func

現在，使用以下命令運行測試：

`toncli run_tests`

> 如果您正在使用 func 的 dev 版本，則需要添加 `--old` 標誌，即 toncli `run_tests --old`

應該得到如下結果：

![toncli get send](./img/run_tests.png)
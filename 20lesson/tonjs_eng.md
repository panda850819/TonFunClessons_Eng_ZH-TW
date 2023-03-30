# 使用 JS 向 TON 區塊鏈發出請求：如何獲取 NFT 數據

Web3 應用程式或 Dapps 的架構通常看起來像調用智能合約方法的前端。
因此，您需要能夠在區塊鏈中使用 JS 進行請求。 TON 中有一些 JS 範例，因此我決定製作一個小型的教學。

## 簡介

Web3 應用程序通常建立在區塊鏈中存在的標準之上，在 TON 中，這些標準是 NFT 和 Jetton。
對於 NFT 標準，常見的任務是獲取特定收藏的 NFT 地址。因此，在本教程中：

- 我們獲取有關 NFT 收藏的數據
- 按索引（Index）獲取 NFT 地址
而且，所有這些都在 JS 中進行。

### 安裝資源庫
為了對 TON 進行請求，我們需要 `typescript` 和用於處理 TON 的模組。 

要使用 Typescript，我們需要：
- Node.js 是您運行 TypeScript 編譯器的環境。
- TypeScript 編譯器是一個 Node.js 模組，用於將 TypeScript 編譯為 JavaScript。

> 我們不會深入介紹 Node.js，安裝它的說明在[這裡](https://nodejs.org/en/download/)：

> 為了方便使用套件，讓我們使用 `npm` 管理器創建一個 `package.json` 文件：

1. 在控制台中，進入您的專案文件夾（我們將在其中撰寫程式）
2. 在控制台中輸入

	npm init

3. 在 console 中回答問題，並確保創建了 `package.json` 文件。

現在安裝 `typescript`。在終端機中，輸入以下命令：

	npm install typescript

安裝完成後，您可以輸入以下命令檢查當前版本的 TypeScript 編譯器：

	tsc --v

我們還將安裝 `ts-node` 套件，在 console 和 node.js 的 REPL 中執行 TypeScript。

	npm install ts-node

還需要安裝用於處理 TON 的模組：

	npm install ton ton-core ton-crypto

好了，現在我們可以來撰寫合約了。

## 獲取有關收藏品的VMP4VU6

要獲取關於 NFT 收藏品的訊息，我們需要調用收藏品的智能合約的 GET 方法，為此我們需要：
- 使用與 TON 區塊鏈的 Light 服務器互動的特定 API 服務
- 通過這個客戶端調用所需的 GET 方法
- 將接收的資料轉換進可讀表格

在本課程中，我們將使用 [toncenter API](https://github.com/toncenter/ton-http-api)，對於請求，我們將使用 js 客戶端，[ton.js](https://www.npmjs.com/package/ton)。

讓我們創建一個 `collection.ts`。

	import { TonClient } from 'ton';
	
並連接到 toncenter

	import { TonClient } from 'ton';

	export const toncenter = new TonClient({
		endpoint: 'https://toncenter.com/api/v2/jsonRPC',
	});
	
為了簡化範例，我們不使用 API 金鑰，因此我們將被限制每分鐘一個請求，您可以使用[機器人](https://t.me/tonapibot) 來創建金鑰。

現在讓我們查看 TON 上的[NFT 收藏品標準](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md) ，以便了解要調用哪個 GET 方法。該標準顯示我們需要 `get_collection_data()` 函數，它將返回：

- `next_item_index` 是當前部署的 NFT 產品數量
- `collection_content` - 收藏品的內容，格式符合 [TEP-64](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md) 標準。
- `owner_address` - 收藏品擁有者的地址，如果沒有擁有者則為零地址。

讓我們使用語法糖 `async/await` ，為 TON 中的某個收藏品調用此方法：

	import { TonClient } from 'ton';

	export const toncenter = new TonClient({
		endpoint: 'https://toncenter.com/api/v2/jsonRPC',
	});

	export const nftCollectionAddress = Address.parse('UQApA79Qt8VEOeTfHu9yKRPdJ_dADvspqh5BqV87PgWD998f');

	(async () => {
		let { stack } = await toncenter.callGetMethod(
			nftCollectionAddress, 
			'get_collection_data'
		);

	})().catch(e => console.error(e));
	
要將數據轉換為可讀形式，我們將使用 `ton-core` 庫：

	import { Address } from 'ton-core';

讓我們將 nextItemIndex 轉換為字串，減去包含內容的單元格，並轉換地址：

	import { TonClient } from 'ton';
	import { Address } from 'ton-core';

	export const toncenter = new TonClient({
		endpoint: 'https://toncenter.com/api/v2/jsonRPC',
	});

	export const nftCollectionAddress = Address.parse('UQApA79Qt8VEOeTfHu9yKRPdJ_dADvspqh5BqV87PgWD998f');

	(async () => {
		let { stack } = await toncenter.callGetMethod(
			nftCollectionAddress, 
			'get_collection_data'
		);
		let nextItemIndex = stack.readBigNumber();
		let contentRoot = stack.readCell();
		let owner = stack.readAddress();

		console.log('nextItemIndex', nextItemIndex.toString());
		console.log('contentRoot', contentRoot);
		console.log('owner', owner);
	})().catch(e => console.error(e));
	
Run the script with `ts-node`. You should get the following:

![collection](./img/1.PNG)

## 透過索引獲取 NFT 集合元素的地址

現在我們要解決通過 Index 獲取地址的問題，我們將再次調用集合的智能合約的 GET 方法。
根據標準，`get_nft_address_by_index(int index)` 方法適用於此任務，該方法返回 `slice address`。

該方法接受一個 `int index` 參數，乍一看似乎只需將具有 `int` 類型的值傳遞給智能合約即可。當然，這是真的，但由於 TON 虛擬機使用寄存器，因此需要通過元組傳遞具有 `int` 類型的值。為此，`ton.js` 提供了 `TupleBuilder`。

	import { TupleBuilder } from 'ton';

將值 0 寫入元組（Tuple）：

	import { TonClient } from 'ton';
	import { Address } from 'ton-core';
	import { TupleBuilder } from 'ton';

	export const toncenter = new TonClient({
		endpoint: 'https://toncenter.com/api/v2/jsonRPC',
	});

	export const nftCollectionAddress = Address.parse('EQDvRFMYLdxmvY3Tk-cfWMLqDnXF_EclO2Fp4wwj33WhlNFT');

	(async () => {
		let args = new TupleBuilder();
		args.writeNumber(0);


	})().catch(e => console.error(e));

仍需發送請求並使用 `readAddress()` 轉換地址：

	import { TonClient } from 'ton';
	import { Address } from 'ton-core';
	import { TupleBuilder } from 'ton';

	export const toncenter = new TonClient({
		endpoint: 'https://toncenter.com/api/v2/jsonRPC',
	});

	export const nftCollectionAddress = Address.parse('EQDvRFMYLdxmvY3Tk-cfWMLqDnXF_EclO2Fp4wwj33WhlNFT');

	(async () => {
		let args = new TupleBuilder();
		args.writeNumber(0);

		let { stack } = await toncenter.callGetMethod(
			nftCollectionAddress, 
			'get_nft_address_by_index',
			args.build(),
		);
		let nftAddress = stack.readAddress();

		console.log('nftAddress', nftAddress.toString());
	})().catch(e => console.error(e));

使用 `ts-node` 運行程式。你應該會得到以下結果：

![address](./img/2.PNG)

## 結論

作者也在 [Telegram 頻道](https://t.me/ton_learn)中發佈了類似的分析和教學，歡迎您訂閱。 

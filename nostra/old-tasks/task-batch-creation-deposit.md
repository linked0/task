Written on Tue Nov 25 14:07:53 KST 2025

# Summary
This is the list of additional tasks in response to your latest implementation of the detail page. Your workspace should be in the `/Users/jay/work/nostra-server`, `/Users/jay/work/nostra-contracts`, and `/Users/jay/work/task` folders. You don't need to check the folder `/Users/jay/work/web` that you acutally have to avoid.

# Prerequisites

# Tasks 
## Task-1: Create a market 
We already created a create-market page. But it's not working. So we need to make the "Create Market" button work like inserting a new market into the database or splitting the positions into two for each binary market. And currently the dafault contnet of the page has the example for "Who will win the MVP in the World Series?". So we need to replace that with the guide text for creating a market. 

## Task-2: Deopsoit/Withdrawal Model
We need to create a deposit/withdrawal model for the users. The users can deposit their USDC to the contract and withdraw it back. The contract will be used to store the users' USDC and the users can withdraw it back to their wallets. So we just add this kind of popup like the image below that will popup when the user clicks the deposit button next to the portpolio button that you should also add at the top of the landing page that is also the market listing page.
![alt text](../images/image-11.png)

I think we should change the contracts in nostra-contracts folder, which would very big change I think. 

## Task-3: Batching transaction
Currently, the transactions are sent one by one. But we need to batch them to reduce the gas fee or enhance the user experience. If a user orders limit buy or sell, our system currently needs so many user interactions to complete the transaction if there are multiple limit orders that are matched to the user's order. I want to know the best way to implement this which mean a user should interact with the wallet like metamask only once to complete the transaction. 

# What you must do and what you must not do
You must not change the code first. Make a plan and desigin to create a new desing file in the folder `~/work/task/nostra/designs`. Add proper name or date to the file name to avoid overwriting the existing files.

You should provide a plan for each task and a design for each task. You can check the what files should be changed and what tables scheme should be changed and etc. And also check what would need to change in other UI and user experience.

After you've done the plan, I will review the design and if it's good, I will ask you to start to change the code. If it's not good, I will ask you to change the design.

# After Tasks Done
You can summarize the what you've done and the changes you've made to the codebase in the file named `summary.md` in the same folder as the tasks.md And copy it to the foler `~/work/task/nostra/dev-logs` with the name of summary-xxx.md in which xxx is date and time when the summary was created. If there is already the same file, you can add some number to the file name. In the last part of the file you should add the original request of the tasks described in this file with some vivid separation. I don't think you must change the codebase, but you can just plan the implementation or recommend something after a review and research. Any summary is OK because I can try to have you do something after a review.

Go ahead and thanks in advance!
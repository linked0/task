# Summary
This is the list of additional tasks in response to your latest implementation of the detail page. Your workspace should be in the `/Users/jay/work/nostra-server`, `/Users/jay/work/nostra-contracts`, and `/Users/jay/work/task` folders. You don't need to check the folder `/Users/jay/work/web` that you acutally have to avoid.

# Heads-Up
Check the following checkbox whether the task I'm requesting is just a design or a code change.

[ ] Change the code 
[x] Just Design how to implement or research the background

# Tasks 
## Task-1: Advanced Design about AI Boost Market Creation 
We roughly designed the AI boost market creation in the future-plan.html file. You can add more details to the design. Here's my requirements.

- [x] **Fix Market Resolution via Admin API**
  - **Issue:** "Not resolved" error when claiming winnings.
  - **Root Cause:** Admin API was using `ConditionalTokens` directly, but the market oracle was set to `ResolutionOracle` (verified via debug script `0x30f4...` match).
  - **Fix:** Reverted Admin API to use `ResolutionOracle` wrapper, which correctly proxies to the ConditionalTokens contract.
  - **Status:** Verified Oracle address match. Ready for user testing.
- [x] **Fix Market Creation Liquidity**
  - **Issue:** "Minting failed" during batch creation.
  - **Root Cause:** Server Wallet ran out of USDC to split into positions (inventory).
  - **Fix:** Added auto-minting logic to `BatchProcessor` to ensure sufficient USDC balance before splitting positions.
  - **Status:** Implemented and deployed.on.

1. The edit box that will be submitted into the LLM server is located bottom of the page that is right before the "create" button.

2. The LLM Server would be the ChatGPT at first that the web app would connect to. The web app will send the edit box content to the LLM server with API Key. 

3. The request would be a simple text like "I want to create a market for best character of the manga, Slam Dunk". And then the LLM server would return the market data to the web app. It would be like like some json data like this:
{
    title: "Best Character of Slam Dunk",
    description: "Best Character of Slam Dunk",
    category: "Sports",
    outcomes: [
        {
            name: "Sakuragi Hanamichi",
            description: "Genius but not skillful yet"
        },
        { 
            name: "Rukawa Kaede",
            description: "Handsome and calm"
        }
    ]
}
I think this LLM should be finetuned to make this kind of data. So you should the steps to finetune the LLM model using a platform like Huggingface or something.

4. Show me some related code.

## After changing any code while coworking with me 
You should say whether I should rerun api server or not. That goes the same for web server.

## For English skill: Very Important
You should do this first before doing other work.
You should check all my English in Tasks section and used while discussing with you in the terminal and evaluate my English and show me the correct version in to the eng.html also in the same folder as this task.md file that can be replaced so that I can see the file in the browser without checking the changing file name. The new added sentences should be on the top of the file. I mean all the english I used in the terminal should be checked. Remove any title like "English Correction" in the file, I want just the content.

# After Tasks Done
Everytime you change some code or do some discussion you should change the following files constantly. That should have some important information or code about like some error fixed while discussing some issue or answering my question about some code or design.

If we just discussing what to do for some change not going to implement it, you should summarize your recommendation for implementation and how to implement it into some document within `/Users/jay/work/task/nostra/designs` that could be created with current date not replacing the existing file. I think html is more readable than markdown. I want have a design.html also in the same folder as this task.md file that can be replaced so that I can see the file in the browser without checking the changing file name.

You should copy this md file to `/Users/jay/work/task/nostra/old-tasks` folder with current date. And if there is already the same file, you should add the new content to the existing file. You should remove the "For my English skill" section from the md file before copying it.

You must summarize the what you've done and the changes you've made to the codebase in the file named `summary.html` with date in the folder `~/work/task/nostra/dev-logs`. Don't include "For English skill" related content for the summary. If there is already the same file, you just add the new content to the existing file. You should add the changed code showing before and after and technical details and background. The code for ‘before’ and ‘after’ should be placed vertically. You can add link to the changed code to the file that shows in antigravity, at least vscode. I think the summary file should be made after you changed some code not for designing how to implement it. I want have a summary.html also in the same folder as this task.md file that can be replaced so that I can see the file in the browser without checking the changing file name.

I think it's better to make a future plan for the project like fixing some bug or change some hahavior of the codebase like using subgraph for the market data or using web socket for the market data or ect. So you can make a file named `future-plan.html` in the same folder as this task.md file that can be replaced so that I can see the file in the browser without checking the changing file name. Adding future features should be done by my request like "Add it to the future plan".

If you edit html file which should have code int it, so the html doesn't need to have container becuase the the container unnecessarily make the width of the content narrow. And for the code shown in the page, don't carriage return which makes the code hard to read.

Go ahead and thanks in advance!
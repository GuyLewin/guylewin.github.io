---
title: israel-finance-telegram-bot
date: '2019-01-19T13:54:42+00:00'
author: Guy Lewin
layout: post
redirect_from:
  - /israel-finance-telegram-bot/
tags:
  - bank
  - 'credit card'
  - finance
  - notifications
  - telegram
---

When it comes to monitoring your expenses, different people have different methods. Some read the monthly-sent billing summary from each credit card provider / bank, some login to these services repeatedly to check if there’s something new. I prefer getting the expenses as phone notifications while I can still remember what the expense was for, not a month after.

When I was an Android owner I used to pay a (very cheap) subscription fee for an app called [חיוב בטוח](https://play.google.com/store/apps/details?id=com.applaudsoftware.safecharge). This app was perfect for me - it scraped the credit card providers for new transactions and let me choose whether I approve / deny this transaction, so I can review those I denied when I have time. Using this method I found multiple subscriptions that I thought I cancelled and billings that were for the wrong amount.

But although I love this app, I can’t ignore my 3 problems with it:

- It only supports Android (I moved to iPhone and I don’t have this solution anymore)
- I don’t trust closed-source solutions I enter my credit card credentials into (how do I know it’s not being uploaded?)
- It doesn’t support bank accounts, only credit cards

Mainly because of the iPhone transition, I was left without a working solution. So I decided to develop one.

My solution is a small Telegram bot script that can run on your computer, scrapes the credit card **and bank** accounts (using the open-source [Israeli Bank Scrapers](https://github.com/eshaham/israeli-bank-scrapers) project) and sends you notifications about new transactions using Telegram. It also supports "denying transactions" (marking them as denied so you can look into them later on). Most importantly - it’s open-source! So you don’t have to trust me your credentials aren’t being uploaded anywhere, you’re the one running the script on your computer and you can read the 1 file of source code yourself.

If you’re like me and need this solution - have a look at the project’s GitHub page: <https://github.com/GuyLewin/israel-finance-telegram-bot>
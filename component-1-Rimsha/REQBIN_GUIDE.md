# 🧪 ReqBin Testing Guide

## 📌 Purpose
This guide explains how to test the Email Triage System using ReqBin.

## 🚀 Step 1: Open ReqBin
Go to: https://reqbin.com

## ⚙️ Step 2: Setup Request
Method: POST
URL: https://jjmopalinski.app.n8n.cloud/webhook-test/email-triage

## 🧾 Step 3: Select Body
Click Body
Select JSON

## 🧪 Step 4: Paste Test Data
Example:
{
  "sender": "test@example.com",
  "subject": "Need help",
  "body": "Hi, can you help me with my account?",
  "timestamp": "2026-05-03T11:00:00"
}

## ▶️ Step 5: Send Request
Click Send
This triggers the n8n workflow

## 📊 Step 6: Check Results
Go to Airtable and verify:
- Category is assigned
- Urgency is set
- Route Queue is updated
- Status = Classified

## 🧪 Test Different Cases
Try different types of emails:

- Question → “Can you help me?”
- Complaint → “I am unhappy with the service.”
- Request → “Please reset my password.”
- Informational → “Office hours updated.”
- Spam → “Click here to win a prize.”

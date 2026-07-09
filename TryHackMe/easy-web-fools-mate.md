# Fool's Mate
 
| Field       | Info                                                                 |
-------------|----------------------------------------------------------------------|
| **Platform** | TryHackMe |
| **Link** | https://tryhackme.com/room/foolsmate |
| **Difficulty** | Easy |
| **Category** | Web |
| **Date** | 2026-07-09 |
| **Time spent** | ~45 min |
 
---
 
## Challenge Description
 
A chess endgame trainer. The description says: mate in one — white to move. The board agrees, the engine doesn't. Goal: make the winning move and get the flag.
 
---
 
## Process
 
### 1. Reconnaissance
 
```bash
nmap 10.114.148.199
```
 
Result: two open ports — SSH (22) and HTTP (80). No credentials for SSH, so I started from HTTP.
 
![nmap scan showing ports 22 and 80 open](/images/easy-web-fools-mate-01.png)
 
---
 
### 2. Web enumeration
 
Visited `http://10.114.148.199` in the browser. Found a chess board labeled "Mate-in-one · White to move". The winning move is clear: the white rook on a1 goes to a8, delivering checkmate.
 
![chess board showing the starting position](/images/easy-web-fools-mate-02.png)
 
---
 
### 3. Attempted the move — client-side block
 
I tried moving the rook from a1 to a8 directly on the board. A JavaScript popup appeared:
 
> `/usr/lib32 — I'll shut down your PC if you play that.`
 
This is a JavaScript alert — it runs entirely in the browser, not on the server. That means it's **client-side**, and can be bypassed.
 
![JavaScript popup blocking the move](/images/easy-web-fools-mate-03.png)
 
---
 
### 4. Exploring the app behavior
 
I tried other moves to understand how the app worked. Moving a different piece (Ra1-a7) went through and the engine responded with Kh8. This confirmed that the app does communicate with the server — it's only the winning move that gets blocked client-side.
 
![a legal move going through — Ra7 Kh8](/images/easy-web-fools-mate-04.png)
 
---
 
### 5. Network analysis — finding the API
 
I opened DevTools (F12) → Network tab, then made the blocked move again. I could see a **POST request to `/move`** appear in the list — the request was sent to the server even through the popup.
 
I opened the request and checked:
- **Response**: `ok: false`, `error: "illegal move"` — the server received the request but rejected it because the coordinates sent by the browser were wrong (the popup interrupted before the correct `to` value was set).
- **Request body**: `from: "a1"`, `to: "a2"` — the wrong destination.
This told me two things:
1. The server is reachable directly.
2. I just need to send the correct move.
![Network tab showing POST /move with status 400](/images/easy-web-fools-mate-05.png)
 
![Response showing ok: false and error: illegal move](/images/easy-web-fools-mate-06.png)
 
![Request body showing from: a1, to: a2](/images/easy-web-fools-mate-07.png)
 
---
 
### 6. Exploit — curl bypass
 
Since the only thing blocking me was JavaScript in the browser, I sent the correct move directly to the server using `curl`, skipping the browser entirely:
 
```bash
curl -X POST http://10.114.148.199/api/move \
  -H "Content-Type: application/json" \
  -d '{"from":"a1", "to": "a8"}'
```
 
The server responded with `ok: true`, `status: checkmate`, and the flag.
 
![curl command and server response with the flag](/images/easy-web-fools-mate-08.png)
 
---
 
## Lessons Learned
 
- A JavaScript popup is client-side — it blocks the user, not the server. If the request still reaches the server (even with wrong data), the endpoint is directly reachable.
- Checking the Network tab in DevTools shows exactly what requests are being made and what data is being sent, even when the UI blocks you.
- `curl` lets you talk to a web API directly, bypassing all browser-side logic. Once you know the endpoint and the expected JSON format, you control the request completely.
- The hint `/usr/lib32` in the popup was a distraction — flavor text designed to send you down a rabbit hole.

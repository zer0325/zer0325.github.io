---
layout: post
title: Lab - Username enumeration via different responses
---

### Description

 This lab is vulnerable to username enumeration and password brute-force
 attacks. It has an account with a predictable username and password, which can
 be found in the following wordlists:

    Candidate usernames
    Candidate passwords

 To solve the lab, enumerate a valid username, brute-force this user's
 password, then access their "My account" page. 

### Solution

In this lab, we are going to use ZAP as the proxy. First we login with a test
username and password to see what the response is. Here, the returned page has
an "invalid username" text on it. Using ZAPs fuzzer we are going to perform
username enumeration.

In ZAP, select the request that returns the "invalid username" page. 

![apprentice lab03 zap request result](/assets/images/authentication/apprentice_lab03_request_result.png)

Right-click and select "Open/Resend with Request Editor..." to open the request
editor window.

![apprentice lab03 request editor](/assets/images/authentication/apprentice_lab03_request_editor.png)

Click send to repeat the request. Right-click and select "Fuzz.." to open the
fuzzer. 

![apprentice lab03 fuzzer](/assets/images/authentication/apprentice_lab03_fuzzer.png)

In fuzzer, highlight "testuser" and click on "Add..." to open the payload
selection tab.

![apprentice lab03 payload selector](/assets/images/authentication/apprentice_lab03_payload.png)

In the "Type" combo box, select "File". Click on "Select..." and choose the
"username" wordlist provided lab.

![apprentice lab03 file selector](/assets/images/authentication/apprentice_lab03_username.png)

Click on "Add" on the "Payload Preview" dialog.

![apprentice lab03 payload preview](/assets/images/authentication/apprentice_lab03_payload_preview.png)

Click on "Start Fuzzer" in the fuzzer dialog.

![apprentice lab03 start fuzzer](/assets/images/authentication/apprentice_lab03_fuzzer_final.png)

In the results page below, note the "Size Resp. Body" column. Only one result
has a different value.

![apprentice lab03 results page](/assets/images/authentication/apprentice_lab03_results_page.png)

Select one of the result with similar value and double click on the "Request &
Response" tab to maximize its window. Scroll-down on the "Response" tab and you
will see an "Invalid Username" text. 

![apprentice lab03 invalid username response](/assets/images/authentication/apprentice_lab03_invalid_username_response.png)

Double click again on the "Request & Response" tab to get back to the previous
window size. This time select the last row or the result with the different
value and double clik again on the "Request & Response" tab. Scroll down again
on the "Response" tab and you will see an "Incorrect password" text.

![apprentice lab03 incorrect password response](/assets/images/authentication/apprentice_lab03_incorrect_password.png)

Looking back at the results page above, the result with the "Incorrect
password" text has a payload value of **aq**. This is our username.

Do the fuzzing procedure above but this time select the "password" as the placeholder
for our payload and the "password" wordlist as the file. Make sure also that the
value for the username is "aq".

![apprentice lab03 password payload](/assets/images/authentication/apprentice_lab03_password_payload.png)

![apprentice lab03 password wordlist](/assets/images/authentication/apprentice_lab03_password_wordlist.png)

Looking at the results page below, "Task ID" 86 has a response code of **302
Found** and has a payload value of **access**. This is our valid password.

![apprentice lab03 final results pages](/assets/images/authentication/apprentice_lab03_final_result.png)


Using ZAPs fuzzer we are able to enumerate a valid username and brute-force its
password. 

The screenshot below shows that we able to login and thus solve the lab.

![apprentice lab03 result](/assets/images/authentication/apprentice_lab03.png)
